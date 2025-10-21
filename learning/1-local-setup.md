# 1. Local RGB20 Issuance and Transfer Troubleshooting Record (bitlight-local-env-public)

### **1. Background & Objectives**

Based on [bitlightlabs/bitlight-local-env-public](https://github.com/bitlightlabs/bitlight-local-env-public), set up a **regtest** local chain and complete the entire process of **RGB20 asset issuance → transfer → validation**.

**Key Point**: The tutorial corresponds to rgb CLI **v0.11.0-beta.9** (old version). New CLI commands are incompatible, must use the version bundled in the project container or compile the same version yourself.

***

### **2. Environment Setup**

#### **Start Local Chain**

```
git clone https://github.com/bitlightlabs/bitlight-local-env-public
cd bitlight-local-env-public
make start
```

#### **Enter Wallet Container (will print XPRV / XPUB / addresses)**

```
make alice-cli   # Enter alice wallet REPL
# Note down Alice's Fixed XPUB (for later descriptor creation)
# Note down Alice's Bitcoin address (for funding later)

make bob-cli     # Enter bob wallet REPL
# Similarly note down Bob's Fixed XPUB and address
```

#### **Prepare Bitcoin Test Coins**

```
make core-cli
mint 1  # Simple method, mine one block to the mining wallet, can later send to Alice/Bob addresses per tutorial
```

***

### **3. Issuance & Transfer (Reproducible Steps)**

> All commands below are run on the
>
> **host machine**

#### **3.1 Create RGB Wallet Using XPUB**

Fill the Fixed XPUB (containing <0;1;9;10>/\*) seen in the container into environment variables:

```
ALICE_DESC="[5183a8d8/86'/1'/0']tpubDDtdVYn7LWnWNUXADgoLGu48aLH4dZ17hYfRfV9rjB7QQK3BrphnrSV6pGAeyfyiAM7DmXPJgRzGoBdwWvRoFdJoMVpWfmM9FCk8ojVhbMS/<0;1;9;10>/*"
BOB_DESC="[3abb3cbb/86'/1'/0']tpubDDeBXqUqyVcbe75SbHAPNXMojntFu5C8KTUEhnyQc74Bty6h8XxqmCavPBMd1fqQQAAYDdp6MAAcN4e2eJQFH3v4txc89s8wvHg98QSUrdL/<0;1;9;10>/*"
```

Create wallets (old version CLI has no init subcommand):

```
rgb -d .alice -n regtest create default --tapret-key-only "$ALICE_DESC"
rgb -d .bob   -n regtest create default --tapret-key-only "$BOB_DESC"
```

Check UTXOs (confirm both sides have funds):

```
rgb -d .alice -n regtest utxos
rgb -d .bob   -n regtest utxos
```

#### **3.2 Import Contract (issuance already completed in contract file)**

First generate rgb20-simplest.rgb in your contract project (e.g., bitlight-rgb20-contract), then import on host machine:

```
CONTRACT_FILE="/absolute/path/bitlight-rgb20-contract/test/rgb20-simplest.rgb"

rgb -d .alice -n regtest import "$CONTRACT_FILE" --esplora=http://localhost:3002
rgb -d .bob   -n regtest import "$CONTRACT_FILE" --esplora=http://localhost:3002
```

Check contract list and note down the **contract ID** (contains special characters like !, $, use **single quotes**):

```
rgb -d .alice -n regtest contracts
# Example: BppYGUUL-Qboz3UD-czwAaVV-!!Jkr1a-SE1!m1f-Cz$b0xs
RGB20_CONTRACT='BppYGUUL-Qboz3UD-czwAaVV-!!Jkr1a-SE1!m1f-Cz$b0xs'
```

#### **3.3 Bob Generates Payment Invoice**

Old version invoice **has no amount parameter**, need to manually insert amount field:

```
INV_NOAMT=$(rgb -d .bob -n regtest invoice "$RGB20_CONTRACT")
# Replace "/RGB20Fixed/" with "/RGB20Fixed/TadF+"
INVOICE=$(printf '%s' "$INV_NOAMT" | sed 's#RGB20Fixed/#RGB20Fixed/TadF+#')
echo "$INVOICE"
```

#### **3.4 Alice Creates Transfer (generates consignment + PSBT)**

```
rgb -d .alice -n regtest transfer "$INVOICE" transfer.consignment alice.psbt
```

#### **3.5 Bob Validates and Accepts Transfer**

```
rgb -d .bob -n regtest validate transfer.consignment
rgb -d .bob -n regtest accept -f transfer.consignment
```

#### **3.6 Alice Signs PSBT (using bdk-cli)**

Convert alice.psbt to base64, paste into **bdk-cli REPL**'s wallet sign:

```
# On host machine
PB64=$(base64 < alice.psbt | tr -d '\r\n')
echo "$PB64"  # Copy entire string

# In wallet container (make alice-cli) execute:
wallet sign --psbt "<entire base64 string from above>"
```

Take the **signed base64** from "psbt" field in the returned JSON from REPL, save and decode:

```
cat > alice.psbt.signed.b64 <<'EOF'
<paste complete base64 from "psbt" field in JSON here>
EOF
base64 -D -i alice.psbt.signed.b64 > alice.psbt.signed
```

#### **3.7 Finalize, Generate Raw Transaction and Broadcast**

```
rgb -d .alice -n regtest finalize alice.psbt.signed alice.final.tx

# Broadcast using Esplora API
HEX=$(xxd -p -c 999 alice.final.tx)
TXID=$(curl -s -X POST http://localhost:3002/tx -d "$HEX")
echo "TXID=$TXID"
```

#### **3.8 Mine Block for Confirmation & Sync State**

```
make core-cli
mint 1

# Sync and check RGB state (check both sides once each)
rgb -d .alice -n regtest state "$RGB20_CONTRACT" RGB20Fixed --sync --esplora=http://localhost:3002
rgb -d .bob   -n regtest state "$RGB20_CONTRACT" RGB20Fixed --sync --esplora=http://localhost:3002

# History
rgb -d .alice -n regtest history "$RGB20_CONTRACT"
rgb -d .bob   -n regtest history "$RGB20_CONTRACT"
```

***

### **4. Pitfalls & Key Points**

1. **Version Alignment**: Tutorial targets rgb-wallet v0.11.0-beta.9. New version command sets are different, must use **same version** CLI from container or self-compiled.
2. **Data Directory**: Old version places data in .alice/regtest, .bob/regtest. Don't mix with new version's bitcoin.testnet/bitcoin.regtest.
3. **Special Characters**: Contract ID and invoice contain !, $, etc., **zsh must use single quotes** '...', otherwise will be consumed by history expansion.
4. **On-chain Parsing**: Commands involving chain interaction (import, state --sync, etc.) add --esplora=[http://localhost:3002](http://localhost:3002), otherwise will report _no resolver specified_.
5. **Invoice Amount**: invoice has no amount, need to manually replace RGB20Fixed/ with RGB20Fixed/TadF+\<amount>.
6. **Signing Tool**: bdk-cli new version parameters are wallet --database-type sqlite --ext-descriptor --int-descriptor, in REPL use wallet sign --psbt "\<base64>" most stable.
7. **Broadcast Method**: Host's bitcoin-cli not connected to container data directory, directly use **Esplora /tx endpoint** for broadcast most convenient.
8. **Sync Refresh**: After completing broadcast use state ... --sync --esplora=... to force refresh witness and balance; mint 1 to mine block for confirmation if necessary.

***

### **5. Reference Repositories**

* Local chain and wallet container: [https://github.com/bitlightlabs/bitlight-local-env-public](https://github.com/bitlightlabs/bitlight-local-env-public)
* RGB v0.11 source code (can compile rgb CLI same version locally): [https://github.com/RGB-WG/rgb](https://github.com/RGB-WG/rgb)
* RGB20 contract example: [https://github.com/bitlightlabs/bitlight-rgb20-contract](https://github.com/bitlightlabs/bitlight-rgb20-contract)



**Flow Diagram**:

```jsx
┌──────────┐
│  Alice   │
│(Issuer)   │
└─────┬────┘
      │
      │ 1. Issue RGB20 contract in REPL
      │    - Specify Ticker / Name / Precision / Amount
      │    - Select a local UTXO as anchor point during issuance
      ▼
┌──────────┐
│ Bitcoin  │
│Blockchain │
└─────┬────┘
      │
      │ 2. There exists an "anchor transaction" carrying RGB contract
      │    (Anchor UTXO: an output in Alice's wallet)
      ▼
┌──────────┐
│   Bob    │
│(Receiver) │
└─────┬────┘
      │
      │ 3. Bob generates invoice
      │    - Contains Bob's receiving seal + asset ID
      ▼
┌──────────┐
│  Alice   │
│(Issuer)   │
└─────┬────┘
      │
      │ 4. Alice generates transfer consignment based on invoice
      │ 5. Alice signs transaction with her private key (PSBT)
      │ 6. Broadcast to Bitcoin chain
      ▼
┌──────────┐
│ Bitcoin  │
│Blockchain │
└─────┬────┘
      │
      │ 7. After transaction confirmation, both sides sync state
      ▼
┌──────────┐
│  Alice   │
│   Bob    │
└──────────┘
```



```jsx
# Local RGB20 Issuance & Transfer — Simplified Command Table

1️⃣ Start Environment
--------------------------------
git clone https://github.com/bitlightlabs/bitlight-local-env-public
cd bitlight-local-env-public
make start

2️⃣ Enter Alice Wallet Container (Issuer)
--------------------------------
make alice-cli
# Select corresponding descriptor to enter REPL

3️⃣ Issue RGB20 Contract
--------------------------------
rgb -d .alice -n regtest issue \
  --ticker TEST --name "Test asset" \
  --precision 2 --amount 100000000000 \
  --iface RGB20Fixed

# Record the output CONTRACT_ID

4️⃣ Enter Bob Wallet Container (Receiver)
--------------------------------
make bob-cli
# Generate invoice
rgb -d .bob -n regtest invoice "$CONTRACT_ID" RGB20Fixed 2000 > bob.invoice

5️⃣ Alice Creates Transfer Based on Invoice
--------------------------------
# Execute on host machine
rgb -d .alice -n regtest transfer bob.invoice > alice.psbt

6️⃣ Sign Transaction (Inside Alice Container)
--------------------------------
PB64=$(base64 < alice.psbt | tr -d '\r\n')
wallet sign --psbt "$PB64" > signed.json
# Extract psbt field from signed.json, save as alice.psbt.signed

7️⃣ Broadcast Transaction (Host Machine)
--------------------------------
xxd -p -c 999 alice.final.tx > hex.txt
HEX=$(cat hex.txt)
docker compose -p bitlight-local-env exec -T core-cli \
  bitcoin-cli -regtest sendrawtransaction "$HEX"

8️⃣ Mine for Confirmation
--------------------------------
make core-cli
mint 1

9️⃣ Sync and Check State
--------------------------------
rgb -d .alice -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora=http://localhost:3002
rgb -d .bob   -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora=http://localhost:3002
```



```go
Complete: RGB Asset Transfer On-chain Structure & CLI Operation Reference

**STEP 1: Alice Issues Asset (Anchor Commitment)**
-------------------------------------------
On-chain Structure:
[ Alice Internal PubKey ]
       |
       +-- commitment_hash = H( RGB_contract_state )
       |
       +-- tweak = H( internal_pubkey || commitment_hash )
       |
       +-- tweaked_pubkey = internal_pubkey + tweak*G
       |
       --> On-chain Taproot Output:
           scriptPubKey = OP_1 <tweaked_pubkey>
           (UTXO_A as anchor for RGB asset)

Bitlight CLI:
# Enter Alice REPL
make alice-cli
# Select RGB Descriptor
# Issue RGB20 asset
rgb -d .alice -n regtest issue --ticker TEST --name "Test asset" --amount 100000000000 RGB20Fixed

**STEP 2: Bob Invoice (Invoice / Seal)**
-----------------------------------
On-chain Structure:
[ Bob Internal PubKey ] --or--> [ Bob blind seal hash ]
       |
       +-- (Optional) blinding factor r
       |
       --> Invoice sent off-chain to Alice:
           "Please lock N units of Asset_X to this seal"

Bitlight CLI:
# Enter Bob REPL
make bob-cli
# Generate invoice (seal information here)
rgb -d .bob -n regtest invoice "$RGB20_CONTRACT" 2000
# Get an rgb:... invoice string, send to Alice

**STEP 3: Alice Transfer (Commit to Bob's Seal)**
-------------------------------------------
On-chain Structure:
[ UTXO_A: Alice's asset anchor ]
       |
       --> Construct transaction:
           txin: spend UTXO_A
           txout0: Bob's tweaked pubkey (contains new RGB commitment)
           txout1: Alice change
       |
       --> New commitment_hash describes:
           "Alice: -N units, Bob: +N units"

Bitlight CLI:
# Alice generates transfer consignment using Bob's invoice
rgb -d .alice -n regtest transfer "$RGB20_CONTRACT" <invoice string>
# Get PSBT, Alice signs in host environment using bdk-cli
# Convert signed PSBT to final transaction hex
# Broadcast transaction (docker internal core-cli or REST API)

**STEP 4: Bob Validate (Reveal & Accept)**
-------------------------------------
On-chain Structure:
Bob checks txout0:
   pubkey' ?= Bob_internal_pubkey + H( Bob_internal_pubkey || commitment_hash )*G
If matches:
   - Validate consignment proof
   - Update local RGB state database:
       UTXO_B now anchors Bob's +N units of Asset_X

Bitlight CLI:
# Bob syncs state
rgb -d .bob -n regtest state "$RGB20_CONTRACT" RGB20Fixed --sync --esplora=http://localhost:3002
# Bob checks history
rgb -d .bob -n regtest history "$RGB20_CONTRACT"
```
