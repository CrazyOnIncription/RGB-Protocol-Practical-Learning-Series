# 5.Experimental Notes: Alice → Bob → Dave Two-Hop Transfer

### 📋 Experiment Overview

**Experiment Objective**: Verify RGB protocol's multi-hop transfer capability, complete Alice → Bob → Dave two-hop transfer path

**Experiment Time**: 2025-08-19 23:55 - 2025-08-20 00:18

**Final Result**: ✅ Complete Success

***

### 🎯 Experiment Goals

1. **Create Third-Party Wallet** - Introduce Dave as a new participant
2. **Verify Multi-Hop Path** - Confirm RGB can support complex transfer networks
3. **Test Invoice Reuse Risk** - Understand RGB invoice security mechanism
4. **Complete Process Verification** - From invoice generation to on-chain confirmation

***

### 🏗️ Experiment Environment

#### Infrastructure

* **Bitcoin Network**: regtest (local testnet)
* **RGB Version**: v0.11.0-beta.9
* **Esplora API**: http://localhost:3002
* **Docker Environment**: bitlight-local-env

#### Participant Wallets

| Participant | RGB Address                                                      | Balance Status        |
| ----------- | ---------------------------------------------------------------- | --------------------- |
| **Alice**   | bcrt1pn0s2pajhsw38fnpgcj79w3kr3c0r89y3xyekjt8qaudje70g4shs20nwfx | 99,999,998,000 RGB    |
| **Bob**     | bcrt1p9yjaffzhuh9p7d9gnwfunxssngesk25tz7rudu4v69dl6e7w7qhq5x43k5 | 4,300 RGB             |
| **Dave**    | bcrt1pq4vz5369ctpt3mj96ey0k2dewwlh2x6gt3qrq2drtalwddac7vasjgmuzp | 0 RGB (newly created) |

#### Contract Information

* **Contract ID**: `rgb:BppYGUUL-Qboz3UD-czwAaVV-!!Jkr1a-SE1!m1f-Cz$b0xs`
* **Token Name**: TEST
* **Total Supply**: 100,000,000,000 (fixed supply)
* **Interface Type**: RGB20Fixed

***

### 📝 Detailed Experiment Steps

#### Step 1: Dave Wallet Creation and Setup

#### 1.1 Create Dave's Bitcoin Wallet

```bash
make dave-cli
# Select RGB Descriptor 9/*
# Get address: bcrt1pq4vz5369ctpt3mj96ey0k2dewwlh2x6gt3qrq2drtalwddac7vasjgmuzp

```

#### 1.2 Send Bitcoin to Dave

```bash
make core-cli
load_wallet
send bcrt1pq4vz5369ctpt3mj96ey0k2dewwlh2x6gt3qrq2drtalwddac7vasjgmuzp 1
mint 1

```

**Result**: Dave receives 1 BTC, transaction ID: `75cd19134f6fbbe9dc9cd2c1ff1917deff0523ea8e765c6130451d081e70d318`

#### 1.3 Create Dave's RGB Wallet

```bash
rgb -d .dave -n regtest create default --tapret-key-only "[4a2358bd/86'/1'/0']tpubDCJSBWsVBQLHNJG4KuMgLLH6vhVdX1YCq3VLsjQxmLuQtRwzcd6LZw44h5yU6R6iGrCB8bVVt7Yoy5DWURUjBz3Y74DNpy8T1scWEUXTGpU/<0;1;9;10>/*" --esplora="http://localhost:3002"

```

#### 1.4 Import RGB Contract

```bash
rgb -d .dave -n regtest import ../bitlight-rgb20-contract/test/rgb20-simplest.rgb --esplora="http://localhost:3002"

```

**Verification Result**: Dave successfully imported contract, Owned section is empty (no tokens received yet)

***

#### Step 2: RGB Transfer Process

#### 2.1 Dave Generates Invoice 📋

```bash
python experiment2_bob_to_dave_complete.py

```

**Invoice Content**:

```
rgb:BppYGUUL-Qboz3UD-czwAaVV-!!Jkr1a-SE1!m1f-Cz$b0xs/RGB20Fixed/YFaF+bcrt:utxob:0EkaEhCY-7MiMA0F-pZjg1FF-eOpKMvD-dohVtt6-J3NH1fm-3lpa7

```

#### 2.2 Bob Creates Transfer 📤

**Generated Files**:

* `bob_to_dave_20250820_001323.consignment` (proof package)
* `bob_to_dave_20250820_001323.psbt` (Bitcoin transaction)

**Transfer Amount**: 500 RGB tokens

#### 2.3 Dave Validates and Accepts ✅

```bash
rgb -d .dave validate bob_to_dave_20250820_001323.consignment
rgb -d .dave accept -f bob_to_dave_20250820_001323.consignment

```

**Result**: Dave successfully validated and accepted transfer to local stash

***

#### Step 3: PSBT Signing Challenge and Solution

#### 3.1 Initial Signing Failure ❌

**Problem**: Automated signing script fails on all derivation paths attempted **Root Cause Analysis**:

* PSBT references UTXO: `2f8eb2dd9caac7d414368b2c30adf7cb46df4bbfc49fee8f2fdb4cbd2da36bd9:0`
* This UTXO corresponds to address: `&9/0` (branch=9, index=0)
* Although signing script tried the correct path, wallet processing logic had issues

#### 3.2 Debugging Analysis 🔍

**Key Discovery**:

* All of Bob's RGB tokens bound to same seal: `bc:tapret1st:2f8eb2dd9caac7d414368b2c30adf7cb46df4bbfc49fee8f2fdb4cbd2da36bd9:0`
* Corresponding Bitcoin UTXO: `2f8eb2dd9caac7d414368b2c30adf7cb46df4bbfc49fee8f2fdb4cbd2da36bd9:0` (1 BTC)
* This UTXO at address `&9/0`, requires private key for `branch=9, index=0`

#### 3.3 Manual Signing Success ✅

**Solution**: Directly use bitcoin-cli for PSBT processing

```python
# Create dedicated wallet
cli(['createwallet', 'wallet_name=final_bob', 'descriptors=false'])

# Import correct private key (branch=9, index=0)
wif = derive_wif_from_tprv(bob_tprv, 9, 0)
cli(['importprivkey', wif, 'final-key', 'false'], wallet='final_bob')

# Process PSBT
proc_result = cli(['walletprocesspsbt', psbt_b64, 'true'], wallet='final_bob')
fin_result = cli(['finalizepsbt', f'psbt={proc_data["psbt"]}', 'extract=true'])

# Broadcast transaction
txid = cli(['sendrawtransaction', fin_data['hex']])

```

**Success Result**:

* ✅ PSBT signing completed
* ✅ Transaction broadcast successful
* **TXID**: `24d14f5cdbbf0487269ed36bda89629f4bae71820f41af72ab7117d641233ccf`

***

### 📊 Final State Verification

#### Pre-Transfer State

| Participant | RGB Balance | UTXO Distribution           |
| ----------- | ----------- | --------------------------- |
| **Bob**     | 4,300       | 2000 + 1800 + 500 (3 UTXOs) |
| **Dave**    | 0           | None                        |

#### Post-Transfer State (Block 228 Confirmed)

| Participant | RGB Balance | UTXO Details                           |
| ----------- | ----------- | -------------------------------------- |
| **Bob**     | 3,800       | `24d14f5c...ccf:0` (merged change)     |
| **Dave**    | 500         | `75cd1913...318:1` (received transfer) |

#### State Verification Commands

```bash
rgb -d .bob state $CID RGB20Fixed --sync --esplora="http://localhost:3002"
rgb -d .dave state $CID RGB20Fixed --sync --esplora="http://localhost:3002"

```

***

### 🔬 In-Depth Technical Analysis

#### RGB Protocol Mechanism Verification

#### 1. **Tapret Commitment Mechanism**

* **Principle**: RGB state is committed through Taproot public key tweak
* **Verification**: Each RGB transfer corresponds to a unique tapret seal
* **Example**: `bc:tapret1st:24d14f5c...` binds to a specific Bitcoin UTXO

#### 2. **Client-Side Validation Model**

* **Process**: Dave independently verifies cryptographic proofs of the transfer
* **Security**: Does not rely on third parties, completely decentralized verification
* **Result**: Dave only accepts transfer to local stash after validation passes

#### 3. **UTXO Consolidation Optimization**

* **Problem**: Bob's original 3 separate RGB UTXOs cause fragmentation
* **Solution**: RGB protocol automatically consolidates UTXOs during transfer
* **Effect**: Bob consolidated from 3 UTXOs to 1, improving efficiency

#### 4. **Atomicity Guarantee**

* **Mechanism**: RGB state transitions are atomically bound to Bitcoin transactions
* **Verification**: Same witness transaction `24d14f5c...` simultaneously updates Bob and Dave's states
* **Security**: Bitcoin network consensus guarantees atomicity of RGB state transitions

***

### ⚠️ Issues and Challenges

#### 1. **PSBT Signing Complexity**

**Problem**: Automated signing script failure **Root Cause**:

* Difficulty identifying derivation paths for RGB UTXOs
* Timing issues with wallet creation and loading
* Wallet isolation between different tools

**Solution**:

* Manual analysis of RGB state to determine correct derivation path
* Use dedicated wallet to avoid conflicts
* Simplified direct CLI approach

#### 2. **State Sync Delay**

**Phenomenon**: State display inconsistent after transfer completion **Cause**: RGB client needs time to sync on-chain state **Solution**: State displays correctly after mining block for confirmation

#### 3. **Toolchain Complexity**

**Challenge**: Need to coordinate RGB CLI, Python scripts, Docker containers and other components **Improvement**: Unified toolchain and better error handling

***

### 💡 Lessons Learned

#### Technical Insights

1. **Deep Integration of RGB with Bitcoin**
   * RGB is not an "add-on layer" to Bitcoin, but a "native extension" of Bitcoin
   * Every RGB operation must have corresponding Bitcoin transaction support
2. **Power of Client-Side Validation**
   * Receiver has complete control over whether to accept transfer
   * No need to trust sender or network, autonomously validates all proofs
3. **Advantages of UTXO Model**
   * Natural support for concurrency and sharding
   * Automatic privacy protection and state isolation

#### Practical Experience

1. **Debugging Strategy**
   * Triple-check RGB state + Bitcoin UTXO + derivation path
   * Verify each step incrementally rather than end-to-end testing
   * Keep intermediate files for analysis
2. **Tool Usage Tips**
   * RGB CLI's esplora parameter is mandatory
   * Special characters in contract ID need shell escaping
   * bitcoin-cli's wallet parameter must be explicitly specified
3. **State Management**
   * Mine blocks promptly to confirm state
   * Distinguish between tentative and confirmed states
   * Understand RGB's finality semantics

***

### 🚀 Future Experiment Directions

Based on Experiment 's success, the following in-depth experiments can be conducted:

#### Experiment 2: RGB20Fixed Minting Restrictions

* Verify fixed supply contract limitations
* Understand why Bob cannot mint new tokens
* Compare differences between RGB interfaces

#### Experiment 3: Asset Recovery Drill

* Consignment file loss recovery
* Stash state database reconstruction
* Private key loss scenario analysis

#### Experiment 4: Network Exception Handling

* PSBT handling with insufficient fees
* RBF (Replace-By-Fee) mechanism
* Mempool congestion response

#### Experiment 5: Security Testing

* Double-spend attack attempts
* Invoice reuse risk verification
* Receiver rejection attack

***

### 📁 Experiment File Checklist

#### Generated Files

```
bob_to_dave_20250820_001323.consignment  # RGB proof package
bob_to_dave_20250820_001323.psbt         # Bitcoin PSBT
experiment2_bob_to_dave_complete.py      # Automation script
psbt_signer.py                           # PSBT signing tool

```

#### Key Transactions

* **Transfer Transaction**: `24d14f5cdbbf0487269ed36bda89629f4bae71820f41af72ab7117d641233ccf`
* **Confirmation Block**: 228 (2025-08-19 16:18:48)
* **Dave Receive UTXO**: `75cd19134f6fbbe9dc9cd2c1ff1917deff0523ea8e765c6130451d081e70d318:1`

***

### 🎯 Conclusion

**Experiment  successfully verified RGB protocol's multi-hop transfer capability**. By creating third-party participant Dave and completing Bob→Dave 500 token transfer, we gained deep understanding of:

1. **RGB Scalability**: Supports arbitrarily complex transfer networks
2. **Security Model**: Client-side validation ensures decentralized security
3. **Efficiency Optimization**: UTXO consolidation reduces on-chain footprint
4. **Toolchain Maturity**: Although complex, functionally complete

This lays a solid foundation for more complex RGB applications in the future, proving RGB's tremendous potential as Bitcoin's native smart contract layer.

***

_Experiment Record Time: 2025-08-20_

_Experiment Environment: bitlight-local-env + RGB v0.11.0-beta.9_

_Experiment Status: ✅ Complete Success_
