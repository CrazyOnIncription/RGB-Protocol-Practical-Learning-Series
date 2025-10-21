# 3.Connecting Python Directly to bitcoind in bitlight's Docker

### **🎯 Original Intent**

In the bitlight demo framework, we originally operated like this:

* Enter the container, run make core-cli to call Bitcoin Core;
* Then make alice-cli to interact with BDK wallet;
* Once transaction signing is involved, **PSBT files have to be passed back and forth between host and container**;
* RGB's consignment, Bob's confirmation and other steps also require constant copy-pasting of commands.

Although this approach works, once it comes to signing and confirming, it becomes "command-line worker mode":

* Some data is in the Core container;
* Some data is in the Alice wallet container;
* We need to manually transfer data, which is inefficient and error-prone.

So our goal is: **Migrate these interactions into Python, call RPC/BDK directly using libraries, instead of relying on CLI.**

If we can achieve this, the scenario will be greatly simplified:

```python
rpc = AuthServiceProxy("http://bitcoin:bitcoin@127.0.0.1:18443")
print(rpc.getblockchaininfo())
```

With just a few lines of Python like this, we can replace the make core-cli commands. Then adding RGB wallet logic later can be more naturally integrated. This article attempts to document and share my debugging and connection process.

```python
      [Past: Command-line Worker Mode]                    [Goal: Python Programming Mode]
🖥️ Host (Mac)                                      🖥️ Host (Mac)
+-----------------------------+               +-----------------------------+
| - Terminal make core-cli    |               | - VSCode / Python IDE        |
| - Terminal make alice-cli   |               | - python-bitcoinrpc library  |
| - Manual PSBT / consignment |               | - One line rpc.getblock...   |
+-------------+---------------+               +-------------+---------------+
              |                                             |
              | docker exec / REPL                          | 🔗 HTTP JSON-RPC
              v                                             v
📦 Container: bitcoin-core                        📦 Container: bitcoin-core
+-----------------------------+               +-----------------------------+
| - bitcoind regtest          |               | - bitcoind regtest          |
| - bitcoin.conf              |               | - bitcoin.conf (RPC config)  |
+-----------------------------+               +-----------------------------+
              |                                             
              | PSBT / UTXO / signing to transfer                           
              v                                             
📦 Container: wallet-alice                                
+-----------------------------+               
| - bdk-cli repl              |               
| - Account 9/*               |               
+-----------------------------+               
```

***

### **🪜 Actual Process and Pitfalls**

#### **1. First Attempt: Direct curl → Timeout**

Our most direct idea at the time: **Host directly calls RPC**.

```bash
curl --user bitcoin:bitcoin \
  -H 'content-type: text/plain;' \
  --data-binary '{"jsonrpc":"1.0","id":"curl","method":"getblockchaininfo","params":[]}' \
  http://127.0.0.1:18443/
```

Result: **No response at all, curl timeout**.

👉 Then we realized: The 18443 port inside the container was not mapped to the host.

***

#### **2. Port Mapping: Write docker-compose.override.yml → Container Keeps Restarting**

The first solution we thought of: Add port mapping in docker-compose.override.yml, while also modifying the command to force bitcoind to add parameters.

What we wrote at the time was like this:

```yaml
services:
  bitcoin-core:
    ports:
      - "18443:18443"
    command: >
      sh -lc 'echo "Waiting for bitcoin.conf";
              while [ ! -f /data/.bitcoin/bitcoin.conf ]; do sleep 1; done;
              exec bitcoind -regtest -rpcbind=0.0.0.0 -rpcallowip=0.0.0.0/0'
```

Result: Container wouldn't start, kept **Restarting**.

👉 The pitfall in this step: **Don't casually override the image's original startup command**.

The Bitlight image has its own initialization logic (like generating cookies, wallets, etc.), which I completely cut off with this command, and it crashed directly.

Conclusion: **Keep the original startup logic intact, only do port mapping**.

***

#### **3. Finding the Configuration File bitcoin.conf**

Since overriding the command is unreliable, let's go back and find: bitcoind actually reads the configuration file by default.

So we entered the container to look:

```bash
docker compose -p bitlight-local-env exec bitcoin-core ls -l /data/.bitcoin
```

Sure enough, we found bitcoin.conf.

Then confirm the content:

```bash
docker compose -p bitlight-local-env exec bitcoin-core cat /data/.bitcoin/bitcoin.conf
```

Conclusion: **To modify RPC configuration, modify bitcoin.conf**.

***

#### **4. Modifying bitcoin.conf**

At first, we only added rpcuser and rpcpassword, but curl kept returning 403 Forbidden.

After repeated attempts, we found that two more lines were needed:

* rpcbind=0.0.0.0 → Otherwise it only listens on the container's loopback.
* rpcallowip=127.0.0.1 and rpcallowip=0.0.0.0/0 → Otherwise host requests are directly rejected.

Final version:

```ini
regtest=1
server=1
rpcuser=bitcoin
rpcpassword=bitcoin
rpcbind=0.0.0.0
rpcallowip=127.0.0.1
rpcallowip=0.0.0.0/0
```

After modification, run docker compose restart bitcoin-core.

***

#### **5. curl Debugging: Witnessing the Journey from Timeout → 403 → Success**

This step is particularly critical.

* **Without port mapping** → curl directly hangs.
* **Without rpcbind/rpcallowip configuration** → 403 Forbidden.
* **Without rpcuser/rpcpassword configuration** → 401 Unauthorized.
* **When everything is correct** → Successfully returns JSON:

```json
{"result":{"chain":"regtest","blocks":209,"headers":209,"bestblockhash":"..."}}
```

👉 Conclusion: Get curl working first to avoid a bunch of inexplicable exceptions in Python.

***

#### **6. Python Connection: Goal Achieved**

Finally, install python-bitcoinrpc on the host:

```bash
pip install python-bitcoinrpc
```

Code only requires three lines:

```python
from bitcoinrpc.authproxy import AuthServiceProxy

rpc = AuthServiceProxy("http://bitcoin:bitcoin@127.0.0.1:18443")
print(rpc.getblockchaininfo())
```

Output:

```python
{'chain': 'regtest', 'blocks': 209, ...}
```

At this point: **Python → Docker bitcoind → Successfully connected ✅**.

***

### **📌 Pitfall Avoidance Summary**

1. **Don't override container startup commands**, keep the init process, extend RPC configuration with bitcoin.conf.
2. **rpcbind/rpcallowip must be explicitly written**, otherwise all host requests are rejected.
3. **curl step-by-step verification is a necessary path**: timeout, 403, 401 correspond to different configuration issues.
4. **Python is just the last step**, you have to get through every pitfall before it can run.

### **🌈 Significance and Outlook**

Why go through all this trouble?

*   **Reuse bitlight's provided infrastructure**

    We don't need to rebuild regtest and wallet environment ourselves, directly utilize their demo framework. **bitlight's** demo infra is well done and can quickly help everyone set up the infrastructure.
*   **Connect Python with bitcoind**

    Previously, we had to make core-cli and make alice-cli in the container, and manually transfer PSBT / consignment. Now with Python calling RPC, we can accomplish the same thing.
*   **Lay the foundation for RGB programming**

    With this direct connection path, the next step is to move RGB wallet logic and consignment processes into Python scripts, instead of copy-pasting between different containers.

In other words: This step puts us truly at the **starting point of RGB "programmability"**.

In the future, we can do the following in Python:

* Call Core to get UTXOs;
* Call BDK wallet for signing;
* Pass consignment to Bob for verification;
* Automate the entire RGB process.
