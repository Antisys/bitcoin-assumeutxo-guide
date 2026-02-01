# Bitcoin AssumeUTXO Quick Sync Guide

A step-by-step guide to quickly bootstrap a Bitcoin full node using AssumeUTXO snapshots. This method allows your node to be fully operational in under an hour instead of days.

## What is AssumeUTXO?

AssumeUTXO is a feature in Bitcoin Core 28+ that allows you to load a pre-generated UTXO (Unspent Transaction Output) snapshot, enabling your node to:

1. **Immediately sync from a recent block** (e.g., block 840,000) to the chain tip
2. **Validate historical blocks in the background** while the node is already usable
3. **Achieve the same security as traditional sync** once background validation completes

### How It Works

When you load an assumeutxo snapshot:
- A new chainstate is created from the snapshot
- Your node starts syncing forward from the snapshot height (840,000) to the current tip
- Meanwhile, background validation downloads and verifies all historical blocks (0 to 840,000)
- Once background validation completes, your node has full trustless security

### Security Considerations

- The snapshot hash is **hardcoded in Bitcoin Core** - you cannot load arbitrary snapshots
- If someone gave you a fake snapshot, it would be rejected because the hash wouldn't match
- Even if somehow loaded, the next mined block would fail validation (hash mismatch)
- Background validation eventually verifies the entire chain anyway

## Requirements

- **Bitcoin Core 28+** or **Bitcoin Knots 28+** (earlier versions don't support mainnet assumeutxo)
- **~15GB disk space** for the snapshot + blockchain data
- **Docker** (optional, but used in this guide)

## Step-by-Step Guide

### Step 1: Set Up Bitcoin Node with Connections Disabled

First, configure your node to not connect to peers while we load the snapshot:

```bash
# bitcoin.conf
mainnet=1
server=1
rpcuser=bitcoin
rpcpassword=your_secure_password
dbcache=450
maxconnections=0  # Disable connections during snapshot load
```

Start the node:
```bash
# Using Docker with Bitcoin Knots 28
docker run -d \
  --name bitcoind \
  -v /srv/bitcoin:/data/.bitcoin \
  -p 8333:8333 \
  -p 8332:8332 \
  bitcoinknots/bitcoin:28
```

### Step 2: Download the UTXO Snapshot

The snapshot for block 840,000 is ~9.2GB. Download via torrent for reliability:

```bash
# Install aria2 for torrent support
sudo apt-get install -y aria2

# Download via torrent (most reliable method)
cd /srv/bitcoin
aria2c --seed-time=0 \
  "magnet:?xt=urn:btih:596c26cc709e213fdfec997183ff67067241440c&dn=utxo-840000.dat&tr=udp://tracker.bitcoin.sprovoost.nl:6969"
```

Alternative direct download (may be unreliable for large files):
```bash
wget https://lopp.net/static/utxo-840000.dat
```

### Step 3: Verify the Snapshot Checksum

**This is critical** - verify the SHA256 hash matches the expected value:

```bash
sha256sum /srv/bitcoin/utxo-840000.dat
```

Expected output:
```
dc4bb43d58d6a25e91eae93eb052d72e3318bd98ec62a5d0c11817cefbba177b  utxo-840000.dat
```

If the checksum doesn't match, delete the file and re-download.

### Step 4: Load the Snapshot

Load the snapshot using the `loadtxoutset` RPC command:

```bash
# Note: Use -rpcclienttimeout=0 to prevent timeout during long operation
docker exec bitcoind bitcoin-cli \
  -rpcuser=bitcoin \
  -rpcpassword=your_secure_password \
  -rpcclienttimeout=0 \
  loadtxoutset /data/.bitcoin/utxo-840000.dat
```

This will take 15-45 minutes depending on your hardware. You'll see progress in the logs:

```bash
docker logs -f bitcoind
```

Example output:
```
[snapshot] 37000000 coins loaded (20.91%, 149 MB)
[snapshot] 38000000 coins loaded (21.48%, 298 MB)
FlushSnapshotToDisk: flushing coins cache (460 MB) started
FlushSnapshotToDisk: completed (35619.93ms)
...
[snapshot] 177000000 coins loaded (100.00%, 460 MB)
```

### Step 5: Enable Connections and Sync

Once the snapshot is loaded, re-enable peer connections:

```bash
# Edit bitcoin.conf
sed -i 's/maxconnections=0/maxconnections=40/' /srv/bitcoin/bitcoin.conf

# Restart the node
docker restart bitcoind
```

### Step 6: Verify Sync Status

Check that your node is syncing from block 840,000:

```bash
docker exec bitcoind bitcoin-cli \
  -rpcuser=bitcoin \
  -rpcpassword=your_secure_password \
  getblockchaininfo
```

Expected output:
```json
{
  "chain": "main",
  "blocks": 840002,
  "headers": 934557,
  "verificationprogress": 0.73,
  "initialblockdownload": true
}
```

Your node will now:
1. Sync forward from 840,000 to the current tip (~94,000 blocks)
2. Validate historical blocks in the background

## Timeline Comparison

| Method | Time to Usable Node | Full Validation |
|--------|---------------------|-----------------|
| Traditional Sync | 2-7 days | Same |
| AssumeUTXO | 30-60 minutes | Background (days) |

## Available Snapshots

| Block Height | Bitcoin Core Version | Magnet Link |
|--------------|---------------------|-------------|
| 840,000 | 28.x | `magnet:?xt=urn:btih:596c26cc709e213fdfec997183ff67067241440c` |
| 880,000 | 29.x (upcoming) | `magnet:?xt=urn:btih:559bd78170502971e15e97d7572e4c824f033492` |

## Troubleshooting

### "assumeutxo block hash in snapshot metadata not recognized"
You're using Bitcoin Core 27 or earlier. Upgrade to Bitcoin Core 28+ or Bitcoin Knots 28+.

### "Couldn't open file for reading"
Check the file path. In Docker, use the container's internal path (e.g., `/data/.bitcoin/utxo-840000.dat`), not the host path.

### Snapshot loading is slow
This is normal with limited RAM. The `dbcache` setting controls how much RAM is used for caching. With 450MB dbcache, the node flushes to disk every ~3 million coins. More RAM = faster loading.

### Download keeps failing
Use the torrent/magnet link instead of direct HTTP download. aria2c handles interruptions gracefully and can resume downloads.

## Generating Your Own Snapshot

If you have a fully synced node, you can generate a snapshot for others:

```bash
# Bitcoin Core 28.x syntax
bitcoin-cli dumptxoutset /path/to/utxo-840000.dat 840000

# Bitcoin Core 29.x syntax (different parameter format)
bitcoin-cli -named dumptxoutset utxo.dat rollback=840000
```

**Warning**: This operation rolls back your node's chainstate and can take hours on slower hardware.

## References

- [Bitcoin Optech - AssumeUTXO](https://bitcoinops.org/en/topics/assumeutxo/)
- [Bitcoin Core assumeutxo design doc](https://github.com/bitcoin/bitcoin/blob/master/doc/design/assumeutxo.md)
- [Lopp's Blog - Bitcoin Node Sync with UTXO Snapshots](https://blog.lopp.net/bitcoin-node-sync-with-utxo-snapshots/)

## License

This guide is released under CC0 (Public Domain). Use it however you like.
