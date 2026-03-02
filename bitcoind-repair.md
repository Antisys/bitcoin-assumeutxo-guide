# Bitcoind LevelDB Repair & Recovery (2026-03-01/02)

## Incident Summary
ThunderHub `npm build` on Pi (107MB free RAM) OOM-killed bitcoind 210+ times, corrupting both `blocks/index` and `chainstate` LevelDB databases. Fixed without reindex or resync.

## Pi Bitcoin Setup
- **Docker**: `bitcoinknots/bitcoin:28.1.knots20250305`, `network_mode: host`, volume `/mnt/bitcoin-usb/bitcoin:/home/bitcoin/.bitcoin`
- **Docker compose**: `/home/pi/docker-services/docker-compose.yml`
- **Data**: `/mnt/bitcoin-usb/bitcoin/` (blocks, chainstate, etc.)
- **Config**: `/mnt/bitcoin-usb/bitcoin/bitcoin.conf`
- **SSH**: `sshpass -p dr600sr ssh pi@192.168.1.111`
- **Bitcoin Knots specific**: blk*.dat files are XOR-encrypted with obfuscation key `c089d26bd9252055` (8 bytes, rotated by file position `key[pos % 8]`)
- **Config change**: Added `checklevel=0` to skip block disconnect verification on startup (the 7 recovered blocks lack undo/rev data)

## Root Cause
OOM crash left chainstate 7 blocks AHEAD of block index:
- Chainstate best block: height 938,891 (`00000000000000000001de5d530ce1927516db1d37ab3a46aa2280d20fd77747`)
- Block index highest: height 938,884 (`00000000000000000001b0c79d75e5e9e84cc772719f0e8aec4855ed3738224e`)
- Blocks 938,885-938,891 were in memory (never flushed to blk*.dat files)
- Bitcoin Core fails because chainstate references a block not in the index

## Fix Steps

### 1. LevelDB Repair (both databases)
```python
import plyvel
# Remove LOCK, WAL (*.log), and small corrupt files (<10k) first
plyvel.repair_db("/mnt/bitcoin-usb/bitcoin/blocks/index")
plyvel.repair_db("/mnt/bitcoin-usb/bitcoin/chainstate")  # ~6-7 min for 11GB on USB
```

**Critical repair cycle** (must repeat until clean):
1. Delete WAL files (`*.log` in DB dir)
2. Delete small/corrupt `.ldb` files (< 10KB)
3. Delete `LOCK` file
4. Run `plyvel.repair_db()`
5. Check for new small files → if any, go to step 1
6. After final repair, do NOT delete WAL (it contains the clean state)

**Why WAL deletion matters**: LevelDB recovers WAL on open, recreating corrupt data from crash. Must delete WAL BEFORE repair to break the cycle.

### 2. Download Missing Blocks
Fetched 7 blocks from `https://blockstream.info/api/block/{hash}/raw` and `https://blockstream.info/api/block-height/{height}`.

### 3. Write Blocks to blk File
- Created `blk05420.dat` with standard format: `magic(4) + size(4) + block_data` per block
- **MUST XOR-encrypt** with Bitcoin Knots obfuscation key (`c089d26bd9252055`), rotating by absolute file position

### 4. Add Block Index Entries
Block index entry format (`b` + 32-byte hash internal order → value):
```
1. ser_version (varint) — use same as existing blocks (259900)
2. nHeight (varint)
3. nStatus (varint) — 0x8d = BLOCK_OPT_WITNESS|BLOCK_HAVE_DATA|BLOCK_VALID_SCRIPTS (no UNDO)
4. nTx (varint)
5. nFile (varint) — if HAVE_DATA or HAVE_UNDO
6. nDataPos (varint) — if HAVE_DATA (points to block data AFTER magic+size header)
7. [nUndoPos (varint) — if HAVE_UNDO, skipped here]
8. block_header (80 bytes raw: version+prev_hash+merkle+time+bits+nonce)
```

### 5. Fix File Info and Last Block Pointer
- File info key: `b'f' + struct.pack('<i', file_num)` (4-byte LE int, NOT varint!)
- File info value: `varint(nBlocks) + varint(nSize) + varint(nUndoSize) + varint(nHeightFirst) + varint(nHeightLast) + varint(nTimeFirst) + varint(nTimeLast)`
- Last block file key: `b'l'` → value: `struct.pack('<i', file_num)` (4-byte LE int, NOT varint!)
- **Must compact after writing**: `db.compact_range()` to flush WAL to SST files

### 6. Bitcoin Core VarInt Encoding
```python
def write_varint(n):
    tmp = []
    while True:
        tmp.append((n & 0x7F) | (0x80 if tmp else 0x00))
        if n <= 0x7F:
            break
        n = (n >> 7) - 1
    return bytes(reversed(tmp))
```
Note: This is NOT standard protobuf varint. It subtracts 1 after each shift.

## Chainstate Obfuscation
- Obfuscation key stored at LevelDB key `\x0e\x00obfuscate_key` — first byte is length, rest is key
- All chainstate values are XOR'd with the key (rotating)
- Best block stored at key `B` — value is 32-byte hash XOR'd with obfuscation key
- To deobfuscate: `bytes(b ^ obf_key[i % len(obf_key)] for i, b in enumerate(raw_value))`
- Result is in internal byte order; reverse for display format

## Key Gotchas
- `plyvel` installed as user `pi`, not available via `sudo python3` — use `sudo PYTHONPATH=/home/pi/.local/lib/python3.12/site-packages python3`
- Files owned by `messagebus:messagebus` inside Docker (uid 101) — need `sudo chmod` for access
- `checkblocks=0` in bitcoin.conf means "check ALL blocks" (not none!) — use `checklevel=0` instead
- Block index keys use INTERNAL byte order (little-endian) for block hashes
- Bitcoin Knots `blocksdir` obfuscation applies to entire blk/rev files (including magic+size headers)
- After plyvel writes, must `db.compact_range()` to flush to SST — otherwise data only in WAL which may be lost

## Verification
```bash
# Check bitcoind is syncing
docker logs bitcoind --tail 5
# Should show "UpdateTip: new best=... height=NNNNNN"
```
