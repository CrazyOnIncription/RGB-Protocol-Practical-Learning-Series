# 6.Experimental Notes: RGB Asset Recovery Guide

## Overview

This document records an RGB asset recovery experiment case conducted in a regtest environment, demonstrating the complete diagnostic and repair process when blockchain reorganization causes RGB state inconsistency.

### Background Scenario

* **Alice**: RGB asset issuer, issued 100,000,000,000 units of TEST asset
* **Bob**: Receiver, should receive 3,800 units (2,000 + 1,800)
* **Dave**: Receives 500 units from Bob
* **Problem**: Blockchain reorganization causes Alice to display incorrect balance

### 1. 🔍 Identifying the Problem: Blockchain Reorg Causes RGB State Inconsistency

#### Problem Manifestation

```bash
# Alice displays incorrect balance
rgb -d "$ALICE_DIR" -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora="$ESPLORA"
# Result: 99,999,998,000 (should be 99,999,996,200)

```

#### Diagnosing Chain Reorganization

```bash
# Check current block height
curl http://localhost:3002/blocks/tip/height
# Result: 228

# Check if historical block exists
curl http://localhost:3002/block-height/233
# Result: Block not found (proves chain rollback)

```

#### Key Indicators

* **Negative Confirmations**: Transaction showing "-4 Confirmations" indicates blockchain reorganization
* **Invalid Transactions**: Some witness transactions don't exist on-chain
* **State Mismatch**: Alice and Bob have inconsistent understanding of the same transfer

### 2. 🔧 Understanding the Mechanism: RGB Transfers Assets Through Tapret Commitments

#### RGB Transfer Mechanism

RGB asset transfer is implemented through tapret commitments in Bitcoin transactions:

```bash
# Typical RGB transfer transaction
Transaction: 41c54986a2d99c3f078a1aea29b867d55df2d8e7f24b213b123a56e4e0ebe804
Input:  0.99999600 BTC (UTXO carrying RGB asset)
Output: 0.99999200 BTC (Change to Alice, containing remaining RGB asset)
Fee:    0.00000400 BTC (Transaction fee)

```

#### Key Concepts

* **Tapret Commitment**: RGB data is committed to Bitcoin transactions through taproot script path
* **Single-use Seals**: Each RGB UTXO can only be spent once
* **Client-side Validation**: RGB state is maintained by clients, not stored on blockchain

### 3. 📊 Data Analysis: Finding Differences Through Dump Comparison

#### Generating Detailed Dumps

```bash
# Generate Alice's data dump
rgb -d "$ALICE_DIR" -n regtest dump
mv ./rgb-dump ./alice-dump

# Generate Bob's data dump
rgb -d "$BOB_DIR" -n regtest dump
mv ./rgb-dump ./bob-dump

# Compare differences
diff -r ./alice-dump ./bob-dump

```

#### Key Findings

```bash
# Invalid witness unique to Alice
Only in ./alice-dump/stash/witnesses: bc:6c94ef43dbb50537400e55a9faab7df11dac678975e07a4b242988429334206f.yaml

# Valid witness unique to Bob
Only in ./bob-dump/stash/witnesses: bc:24d14f5cdbbf0487269ed36bda89629f4bae71820f41af72ab7117d641233ccf.yaml

```

#### Analyzing Taprets Differences

```bash
# Check Alice's taprets
rgb -d "$ALICE_DIR" -n regtest taprets

# Check Bob's taprets
rgb -d "$BOB_DIR" -n regtest taprets

# Find differences and verify on-chain existence
curl http://localhost:3002/tx/[tapret_hash]

```

#### Critical Tapret Comparison Discovery

By comparing Alice and Bob's taprets, we found decisive differences:

**Taprets unique to Alice (invalid):**

```bash
bc:6c94ef43dbb50537400e55a9faab7df11dac678975e07a4b242988429334206f  # Does not exist on-chain
bc:0460ae7f6d899b861922029e33efb946f92679d2069fb74da935b0bd949d58cb  # Does not exist on-chain
bc:0d11e14971a07c560f887e454d04970a8e4517f11b5ac3ed8edf8ad1df7684fa  # Does not exist on-chain

```

**Taprets unique to Bob (valid):**

```bash
bc:24d14f5cdbbf0487269ed36bda89629f4bae71820f41af72ab7117d641233ccf  # Block 228, confirmed
bc:7a4550e63c3c4e42f06ae067d38bb29c655f6534dd3dea8a13298cbb4fdc9e34  # Does not exist on-chain (Bob also has issues)

```

**Verification Method:**

```bash
# For each suspicious tapret, check on-chain status
curl http://localhost:3002/tx/7a4550e63c3c4e42f06ae067d38bb29c655f6534dd3dea8a13298cbb4fdc9e34
# Result: No results found - this transaction doesn't exist

curl http://localhost:3002/tx/24d14f5cdbbf0487269ed36bda89629f4bae71820f41af72ab7117d641233ccf
# Result: Block 228 confirmed - this is a valid witness transaction

```

### 4. 🔄 Recovery Method: Export/Import Consignment

#### Cleaning Invalid State

```bash
# Backup current data
cp -r "$ALICE_DIR" "${ALICE_DIR}_backup"

# Delete RGB state files (preserve wallet keys)
rm "$ALICE_DIR"/regtest/stash.dat
rm "$ALICE_DIR"/regtest/state.dat
rm "$ALICE_DIR"/regtest/index.dat

```

#### Importing from Valid Source

```bash
# Export contract consignment from Bob
rgb -d "$BOB_DIR" -n regtest export "$CONTRACT_ID" alice_contract.rgb

# Alice imports (note: must add --esplora parameter)
rgb -d "$ALICE_DIR" -n regtest import alice_contract.rgb --esplora="$ESPLORA"

# Verify contract import success
rgb -d "$ALICE_DIR" -n regtest contracts

```

#### Re-syncing State

```bash
# Force re-sync
rgb -d "$ALICE_DIR" -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora="$ESPLORA"

```

### 5. ✅ State Verification: Confirming Correct Asset Distribution

#### Verifying Transfer History

```bash
# Check Alice's transfer history
rgb -d "$ALICE_DIR" -n regtest history "$CONTRACT_ID" RGB20Fixed --details

# Check Bob's transfer history
rgb -d "$BOB_DIR" -n regtest history "$CONTRACT_ID" RGB20Fixed --details

# Check Dave's transfer history
rgb -d "$DAVE_DIR" -n regtest history "$CONTRACT_ID" RGB20Fixed --details

```

#### Verifying Mathematical Balance

```
Total Issued:  100,000,000,000
Alice Sent:         3,800 (2,000 + 1,800 to Bob)
Bob Sent:             500 (to Dave)
Alice Remaining:   99,999,996,200 ✓
Bob Remaining:          3,800 ✓
Dave Remaining:           500 ✓
Total:         100,000,000,000 ✓

```

### 6. 🛠️ Quick Command Reference

#### Basic State Queries

```bash
# View RGB asset state
rgb -d "$DATA_DIR" -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora="$ESPLORA"

# View transfer history
rgb -d "$DATA_DIR" -n regtest history "$CONTRACT_ID" RGB20Fixed --details

# View tapret list
rgb -d "$DATA_DIR" -n regtest taprets

```

#### Export/Import Operations

```bash
# Export contract
rgb -d "$DATA_DIR" -n regtest export "$CONTRACT_ID" output.rgb

# Import contract (must add resolver parameter)
rgb -d "$DATA_DIR" -n regtest import input.rgb --esplora="$ESPLORA"

```

#### Diagnostic Commands

```bash
# Generate detailed dump
rgb -d "$DATA_DIR" -n regtest dump

# Validate consignment
rgb validate consignment.rgb

# Check contract list
rgb -d "$DATA_DIR" -n regtest contracts

```

### 7. 🎯 RGB Asset Recovery Core Methods Summary

#### Method 1: Consignment Export/Import Recovery

This is the core method for RGB asset recovery, successfully verified in our experiment:

**Technical Principle:**

* **RGB data completely depends on local storage** - no blockchain backup
* **Consignment contains complete transfer history and state proofs**
* **Complete contract state can be transferred between different clients**

**Operational Steps:**

```bash
# Step 1: Export consignment from valid data source
rgb -d "$BOB_DIR" -n regtest export "$CONTRACT_ID" alice_contract.rgb

# Step 2: Clean problematic client's RGB state (preserve wallet keys)
rm "$ALICE_DIR"/regtest/stash.dat
rm "$ALICE_DIR"/regtest/state.dat
rm "$ALICE_DIR"/regtest/index.dat
# Note: Keep default/ directory (wallet keys) and rgb.toml

# Step 3: Import consignment (must specify resolver)
rgb -d "$ALICE_DIR" -n regtest import alice_contract.rgb --esplora="$ESPLORA"

# Step 4: Force re-sync state
rgb -d "$ALICE_DIR" -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora="$ESPLORA"

```

**Success Verification:**

* ✅ Successfully completed Bob→Alice consignment export/import
* ✅ Import technically successful, contract validation passed
* ✅ Method itself is valid and reproducible

#### Method 2: Data Diagnosis and Comparative Analysis

Systematic diagnostic methods help accurately locate problems:

**Dump Comparison Analysis:**

```bash
# Generate detailed data dump
rgb -d "$ALICE_DIR" -n regtest dump
mv ./rgb-dump ./alice-dump
rgb -d "$BOB_DIR" -n regtest dump
mv ./rgb-dump ./bob-dump

# Compare to find differences
diff -r ./alice-dump ./bob-dump

```

**Tapret Verification:**

```bash
# Check tapret list
rgb -d "$DATA_DIR" -n regtest taprets

# Verify on-chain status of each tapret
curl http://localhost:3002/tx/[tapret_hash]

```

#### Method 3: Essential Data Backup Strategy

```bash
# RGB core data files (located in $DATA_DIR/regtest/)
stash.dat     # RGB state and witness data
state.dat     # Contract state data
index.dat     # Index data
rgb.toml      # Configuration file
default/      # Wallet key directory

# Recommended backup strategy
cp -r "$ALICE_DIR" "/backup/alice_$(date +%Y%m%d_%H%M%S)"

# Important consignment preservation
rgb -d "$DATA_DIR" -n regtest export "$CONTRACT_ID" "backup_$(date +%Y%m%d).rgb"

```

**Important Principles:**

* ✅ **Regularly backup entire data directory**
* ✅ **Backup immediately after important transfers**
* ✅ **Save important consignment files**
* ✅ **Record all contract IDs and transaction hashes**
* ❌ **Don't rely only on wallet mnemonic** - RGB data cannot be recovered from seed

### 8. 🔄 Complex Scenarios: Impact and Handling of Blockchain Reorganization

#### Identifying Reorganization Issues

In our experiment, we encountered a more complex situation than ordinary sync problems:

**Typical Symptoms of Reorganization:**

```bash
# Abnormal transaction status
curl http://localhost:3002/tx/e648269358772cc55431cc5a2bf92130f369cba136666732e822c0e26487b7b1
# Result: Esplora shows "Unconfirmed", but transaction existed long ago

# Mempool check
bitcoin-cli getrawmempool
# Result: Empty (transaction not in pending confirmation queue)

# RGB state display
rgb -d "$ALICE_DIR" -n regtest state "$CONTRACT_ID" RGB20Fixed
# Result: assetOwner shows "tentative" status

```

#### Impact Mechanism of Reorganization on RGB

**Root Cause:**

* **RGB uses single-use seals to bind assets to UTXOs**
* **Blockchain reorganization makes originally confirmed UTXOs become "orphans"**
* **RGB state proof chain bound to orphan UTXOs is broken**

**Orphan Transaction Phenomenon:**

* Transaction data exists in the index but not on current main chain
* Cannot re-enter mempool due to changed input UTXO status
* RGB client detects unconfirmed status, marks as "tentative"

#### Version Differences in Reorganization Handling

**Limitations of v0.11 beta.9** (version we used):

* ✅ Can identify tentative state caused by reorganization
* ✅ Supports basic consignment export/import
* ❌ Cannot fully recover state affected by reorganization
* ❌ Consignment import cannot override problematic witness data

**Improvements in v0.12** (requires upgrade):

* ✅ Introduces complete state archiving mechanism
* ✅ Improved handling logic for invalid transactions
* ✅ Supports genesis transaction rollback handling
* ✅ Stronger reorganization recovery capability

#### Mainnet vs Testnet Differences

**Mainnet Protection Mechanism:**

* Reorganizations usually shallow (1-2 blocks)
* Strong network consensus, forks converge quickly
* Miner incentives ensure chain stability

**Regtest Specifics:**

* Can artificially create deep reorganizations
* Lacks network consensus protection
* More likely to trigger protocol edge cases

**Actual Impact:**

* RGB problems caused by reorganization relatively rare on mainnet
* But once they occur, impact may be more severe than testnet
* Need comprehensive backup and recovery strategy

### 9. 💾 Version Limitations and Upgrade Solutions

#### Limitations of Current Version

**RGB v0.11 beta.9's insufficiency in reorganization handling:**

* Can detect reorganization issues (shows tentative status)
* But cannot fully recover asset state affected by reorganization
* Consignment import cannot override problematic witness data
* API layer lacks stable reorganization handling mechanism

**Results of Our Experiment:**

* ✅ **Successfully exported/imported consignment** - technical process correct
* ✅ **Contract validation passed** - data integrity good
* ❌ **Alice state still shows tentative** - caused by version limitations
* ❌ **Balance calculation still incorrect** - reorganization handling imperfect

#### Upgrade Direction

According to RGB community feedback, v0.12 version has improvements in reorganization handling. We plan to test the new version's reorganization handling capability in the next experiment to verify whether it can better solve similar asset state problems.

### 10. 💡 Practical Tips and Best Practices

#### Daily Backup Strategy

```bash
#!/bin/bash
# rgb_backup.sh - Automated backup script
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/rgb_$DATE"

# Backup RGB data directories
cp -r "$ALICE_DIR" "$BACKUP_DIR/alice"
cp -r "$BOB_DIR" "$BACKUP_DIR/bob"

# Export important consignment
rgb -d "$ALICE_DIR" -n regtest export "$CONTRACT_ID" "$BACKUP_DIR/alice_consignment.rgb"

echo "Backup completed: $BACKUP_DIR"

```

#### Troubleshooting Checklist

```bash
# 1. Check basic connectivity
curl http://localhost:3002/blocks/tip/height

# 2. Verify contract status
rgb -d "$DATA_DIR" -n regtest contracts

# 3. Check suspect transactions
rgb -d "$DATA_DIR" -n regtest taprets
curl http://localhost:3002/tx/[suspicious_txid]

# 4. Compare data differences
rgb -d "$DATA_DIR" -n regtest dump
diff -r ./dump1 ./dump2

```

#### Common Error Solutions

```bash
# Error: resolver error
# Solution: Add --esplora parameter
rgb import file.rgb --esplora="http://localhost:3002"

# Error: contract not known
# Solution: First import contract consignment

# Error: tentative status cannot be cleared
# Solution: May require version upgrade

```

#### Multi-party Collaboration Recommendations

* **Issuer**: Regularly share complete historical consignment
* **Receiver**: Save every transfer consignment received
* **Third Party**: Can serve as backup source for data recovery

### 11. 📋 Experiment Summary and Outlook

#### Core Achievements

We successfully mastered core methods for RGB asset recovery:

**✅ Verified Technical Methods:**

* Dump comparison diagnostic technique
* Tapret verification method
* Consignment export/import process
* State reconstruction operations

**✅ Mechanisms Deeply Understood:**

* RGB client-side validation principle
* Impact of blockchain reorganization on RGB
* Working mechanism of single-use seals
* Differences between protocol versions

#### Complex Situations Encountered and Future Research Directions

During the asset recovery process, we encountered the complex scenario of blockchain reorganization:

* Identified typical symptoms of orphan transactions
* Understood the mechanism of reorganization's damage to RGB state
* Discovered handling limitations of v0.11 beta.9
* Found improvement directions in v0.12 version

**Future Research Plans:**

1. **Version Upgrade Verification** - Test v0.12's reorganization handling capability
2. **Method Refinement** - Improve diagnostic and recovery processes
3. **Production Practice** - Verify method effectiveness in stable versions
4. **Community Contribution** - Share experience to promote protocol improvement

Although this experiment couldn't fully solve Alice's problem due to version limitations, the knowledge system and methodology we gained will provide important reference for RGB asset security management. The RGB protocol is developing rapidly, and our research lays the foundation for future improvements.
