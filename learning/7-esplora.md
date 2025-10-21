# 7.Esplora Blockchain Explorer Deployment Practice on macOS

When setting up an Esplora service for RGB protocol testing environment, there was a need to provide Bitcoin testnet REST API query service on macOS. This article documents the complete deployment process from Docker solution to native compilation, as well as key decisions in resource optimization.

### Network Limitations of Docker Solution

Initial attempts to use Docker for quick deployment encountered macOS-specific network isolation issues. `host.docker.internal` in the Docker container resolved to `192.168.65.254`, while Bitcoin Core was configured to listen on `127.0.0.1:18332` and the local IP, resulting in the container being unable to access the host's Bitcoin service.

Error logs showed:

```
WARN - failed to connect daemon at 192.168.65.254:18332: Connection refused
```

Various Docker network configurations (host mode, custom networks) were attempted without success, ultimately deciding to adopt a native compilation approach.

### Key Differences in Version Selection

The system already had standard electrs installed, but it was discovered that it only provides Electrum RPC protocol and does not support HTTP REST API. The esplora-electrs variant maintained by Blockstream was needed:

bash

```bash
git clone https://github.com/Blockstream/electrs.git esplora-electrs
cd esplora-electrs
cargo build --release
```

Main differences between versions:

* Standard electrs: `-cookie-file` parameter, only supports Electrum protocol
* esplora-electrs: `-cookie` parameter, supports `-http-addr` option

### Resource Consumption Challenges of Full Node Mode

Initial attempts at full indexing mode encountered severe resource management issues. Monitoring data showed abnormal database bloat, growing from the expected \~150GB to a peak of 337GB, far exceeding the normal range. Key issues included:

**Abnormal Resource Consumption:**

* Database size abnormally grew to 337GB (normal should be \~150GB)
* System memory usage consistently at 98%, frequent swap operations
* CPU load maintained at 3.0-4.0, system resources near saturation
* Disk I/O reached abnormal levels of 18TB reads and 4.6TB writes

**Unstable Synchronization Process:**

* Block sync often stalled at specific heights (e.g., 4,655,778)
* Historical indexing progress frequently rolled back (from 4,296,773 back to 3,518,999 blocks)
* Process repeatedly terminated by OOM killer, causing repeated work

**Database Management Issues:**

* RocksDB compaction process unable to complete normally
* Severe database file fragmentation
* Index data not persisted in time, progress lost after restart

This abnormal resource consumption pattern indicated that full node mode was difficult to run stably under current hardware configuration.

### Performance Optimization with Light Mode

To address resource constraint issues, light mode configuration was enabled:

bash

```bash
./target/release/electrs \\
  --network testnet \\
  --daemon-dir "/Volumes/MAC_Programs" \\
  --daemon-rpc-addr "127.0.0.1:18332" \\
  --db-dir "/Volumes/DEV/esplora-electrs-db" \\
  --http-addr "127.0.0.1:3001" \\
  --monitoring-addr "127.0.0.1:14225" \\
  --lightmode \\
  -vvv
```

Optimization results:

* Storage requirements: Reduced from 500GB to 300GB
* Database size: Compressed from 160GB to 90GB
* Memory footprint: Significantly reduced, avoiding OOM termination
* Functional impact: Retained UTXO queries and address history functionality

### Technical Analysis of Indexing Process

Through log analysis, three main processing stages of electrs were identified:

**Stage 1 - Transaction Store**

```
INFO 4,655,777 blocks were added
```

Store all transaction data to txstore database.

**Stage 2 - History Index**

```
INFO 4,296,773 blocks were indexed
```

Build address-to-transaction mapping relationships.

**Stage 3 - Real-time Sync**

```
INFO hash=... height=4656666 @ 2025-08-31T13:47:11Z (1 left to index)
```

Switch to real-time processing mode, processing new blocks one by one.

After each stage completes, RocksDB compaction is executed:

```
DEBUG starting full compaction on RocksDB { path: "/Volumes/DEV/esplora-electrs-db/testnet/newindex/history" }
```

### Deployment Architecture Design

**Storage Allocation Strategy:**

* Bitcoin Core data: /Volumes/MAC\_Programs (\~200GB)
* Esplora index data: /Volumes/DEV (\~90GB)
* Distributed across different disks to optimize I/O performance

**Service Port Configuration:**

* Bitcoin RPC: 127.0.0.1:18332
* Esplora HTTP API: 127.0.0.1:3001
* Electrum RPC: 127.0.0.1:60001
* Prometheus monitoring: 127.0.0.1:14225

### Service Status Verification

Key indicators of completed indexing:

* Processing mode switched from batch historical data to real-time single-block processing
* Data source switched from "using BlkFiles" to "using Bitcoind"
* Started actively processing mempool transactions

API service verification:

bash

```bash
curl -s http://127.0.0.1:3001/api/blocks/tip/height
curl -s http://127.0.0.1:3001/api/blocks/tip/hash
curl -s http://127.0.0.1:3001/api/address/{address}
```

### Technical Summary

Key lessons from deploying Esplora service on macOS:

1. **Platform Compatibility**: Native compilation is more reliable than containerized solutions, avoiding network layer complexity
2. **Version Selection**: Accurately identify functional differences between software variants, choose versions that meet requirements
3. **Resource Optimization**: Light mode significantly reduces resource requirements while maintaining core functionality
4. **Process Monitoring**: Understand processing stages through log analysis, facilitating problem diagnosis and progress assessment

The complete indexing process took approximately 12 hours, ultimately providing stable Bitcoin testnet REST API service, establishing a reliable blockchain data query infrastructure for RGB protocol development testing.
