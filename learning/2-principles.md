# 2. RGB Protocol Principles and Design from a Programmer's Perspective

### 1. All RGB Transactions = Regular Bitcoin Transactions

* On-chain, RGB looks like Taproot transactions.
* RGB's commitment is embedded in the Taproot public key's tweak (or script path).
* To the Bitcoin mainnet, it's just a regular Bitcoin transaction, with no "special format".
* Therefore:
  * mempool.space / explorer shows it as a BTC transfer.
  * Only RGB clients can interpret the "RGB20 asset changes" within it.

### 2. RGB Logic is Interpreted Off-Chain

* The real "RGB transfer" doesn't run on-chain, but off-chain:
  * Alice generates a consignment (local proof package) based on Bob's invoice.
  * Bob uses RGB client to validate + accept, updating his local state.dat.
* Ledger composition:

```
Bitcoin Mainnet (ordering & security)
+ Local stash/state files (contract state)
= RGB Ledger

```

* Mainnet only guarantees "immutable ordering", RGB client is responsible for "semantic interpretation".

### 3. Why Design it This Way?

* No pollution of Bitcoin protocol: RGB doesn't require soft fork/hard fork.
* Stays lightweight: Mainnet nodes don't need to understand RGB at all.
* Privacy-friendly: Only participants know that a certain transaction carries RGB commitment.
* Minimal on-chain overhead: One RGB transfer ≈ one regular Taproot transaction fee.

### 4. Core Workflow

1. **Anchoring (Commitment)**
   * Alice embeds contract state commitment (tapret commitment) on a UTXO she controls.
2. **Invoice**
   * Bob generates "how much asset I want to receive + my seal method (UTXO seal)" and gives it to Alice.
3. **Transfer**
   * Alice constructs consignment based on invoice, generates on-chain transaction with commitment.
4. **Validation**
   * Bob uses on-chain data + Alice's consignment to verify validity, updates his own ledger.

### 5. Comparison with Inscriptions (Ordinals)

* **Inscriptions**: Store data on-chain, explorer can see it directly.
* **RGB**: Store proofs off-chain, only store immutable anchor points on-chain.
* Result:
  * Inscriptions' "state interpretation" depends on indexer, different indexers may differ.
  * RGB's "state interpretation" is self-contained in consignment + mainnet transactions, as long as verified, any client must have consistent results.

### 6. Advantages Summary

* **Ultimate Scalability**: Smart contract logic doesn't go on-chain, Bitcoin requires no modification.
* **Security Inheritance**: State is anchored to BTC UTXOs, cannot be forged.
* **Privacy**: Not visible on-chain, only participants have complete context.
* **Low Cost**: Only consumes one regular BTC transaction fee.

### 7. Taproot Commitment Mechanism: tweak & tapret

Note: Friends who get headaches from mathematical principles can skip this. Of course, you're also welcome to attend my Bitcoin public course on LearnBlockchain community, https://learnblockchain.cn/course/76, _which covers this content_

RGB's on-chain anchoring (commitment) essentially utilizes Taproot's tweak property to hide contract state hash in the public key or script path.

#### Key-path tweak Mode (tapret-key-only)

* Choose an internal public key P, corresponding private key k.
* Generate commitment value c = H(\text{RGB state}).
* Calculate tweak:

```
t = H(P || c)
```

* Get tweaked public key:

```
P' = P + t · G
```

* Taproot address = P2TR(P′), looks like a regular address on-chain.

🔑 **Effect**:

* No trace of "RGB" visible on-chain.
* Only when revealing P, c can others verify the commitment.
* When spending, use tweaked private key k + t to normally go through key-path signing.

#### Script-path tweak Mode (tapret-tree)

* Instead of directly modifying the key, add a virtual branch in the Taproot script tree, with the leaf script containing the commitment.
* When spending, reveal this script leaf, others can directly see the committed content.

🔑 **Effect**:

* key-path tweak: Best privacy, not visible on-chain.
* script-path tweak: Commitment directly exposed in witness, can be scanned by external tools.

#### RGB's Adopted Mode

* In practice, RGB clients typically default to tapret-key-only:
  * Each transfer → generates a new tweaked public key address.
  * Ensures commitment is invisible, can only be interpreted through consignment + local validation.

### On-Chain View vs Ordinals Indexer vs RGB Client

```markup
           Bitcoin Mainnet's View      |     Ordinals Indexer's View        |        RGB Client's View
───────────────────────────────────── ┼───────────────────────────────────┼────────────────────────────────────
TXID: abc123...                       | TXID: abc123...                   | TXID: abc123...
------------------------------------- | --------------------------------- | -----------------------------------
    Input: References previous UTXO   | Input: Same                       | Input: Same
    Output:                           | Output:                           | Output:
  - Taproot address A (1 BTC)         |   - Taproot address A (1 BTC)    |   - Taproot address A (1 BTC)
                                      |     (contains Ordinals inscription:|     (contains RGB commitment: 
                                      |      "Hello World" image data)    |      Alice➝Bob 2000 TEST)
                                      |                                   |
On-chain sees:                        | Ordinals interprets:              | RGB interprets:
  - A regular BTC transfer            |   - This is an NFT inscription    |   - This is an asset transfer
  - No additional semantics           |   - Belongs to a certain satoshi  |   - Updates Bob's RGB20 balance

```
