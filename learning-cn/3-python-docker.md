# 3.  让 Python 直接连上 bitlight 提供的 Docker 里的 bitcoind

### **🎯 初衷**

在 bitlight 提供的 demo 框架里，我们原本是这样操作的：

* 进入容器，跑 make core-cli 调 Bitcoin Core；
* 再 make alice-cli 跟 BDK 钱包交互；
* 一旦涉及交易签名，**PSBT 文件要在宿主机和容器之间来回传**；
* RGB 的 consignment、Bob 确认这些步骤，也要不断复制粘贴命令。

这种方式虽然能跑通流程，但一到签名、确认，就变成了「命令行打工人」模式：

* 有的数据在 Core 容器里；
* 有的数据在 Alice 钱包容器里；
* 我们需要手动搬运，效率很低，也容易出错。

所以我们的目标就是：**把这些交互迁移到 Python 里，用库直接调用 RPC/BDK，而不是依赖 CLI。**

能做到这一点，场景会大大简化：

```
rpc = AuthServiceProxy("<http://bitcoin:bitcoin@127.0.0.1:18443>")
print(rpc.getblockchaininfo())
```

这样几行 Python，就能替代掉 make core-cli 那些命令，后面再往里加 RGB 钱包逻辑，才能更自然地衔接。本文尝试把我调试连接的过程记录分享给大家。

```python
      [过去：命令行打工人模式]                         [目标：Python 编程模式]
🖥️ 宿主机 (Mac)                                      🖥️ 宿主机 (Mac)
+-----------------------------+               +-----------------------------+
| - 终端 make core-cli        |               | - VSCode / Python IDE        |
| - 终端 make alice-cli       |               | - python-bitcoinrpc 库       |
| - PSBT / consignment 手搬   |               | - 一行 rpc.getblock...        |
+-------------+---------------+               +-------------+---------------+
              |                                             |
              | docker exec / REPL                          | 🔗 HTTP JSON-RPC
              v                                             v
📦 容器: bitcoin-core                             📦 容器: bitcoin-core
+-----------------------------+               +-----------------------------+
| - bitcoind regtest          |               | - bitcoind regtest          |
| - bitcoin.conf              |               | - bitcoin.conf (配置好 RPC)  |
+-----------------------------+               +-----------------------------+
              |                                             
              | PSBT / UTXO / 签名要搬                           
              v                                             
📦 容器: wallet-alice                                
+-----------------------------+               
| - bdk-cli repl              |               
| - 账户 9/*                   |               
+-----------------------------+               
```

***

### **🪜 实际过程与坑点**

#### **1. 第一次尝试：直接 curl → 超时**

当时我们最直接的想法：**宿主机直接打 RPC**。

```
curl --user bitcoin:bitcoin \\
  -H 'content-type: text/plain;' \\
  --data-binary '{"jsonrpc":"1.0","id":"curl","method":"getblockchaininfo","params":[]}' \\
  <http://127.0.0.1:18443/>
```

结果：**完全没反应，curl 超时**。

👉 才意识到：容器里的 18443 端口并没有映射到宿主机。

***

#### **2. 映射端口：写 docker-compose.override.yml → 容器不停重启**

想到的第一个解法：在 docker-compose.override.yml 里加端口映射，同时顺便改 command，想强行让 bitcoind 加上参数。

当时写的是这样的：

```
services:
  bitcoin-core:
    ports:
      - "18443:18443"
    command: >
      sh -lc 'echo "Waiting for bitcoin.conf";
              while [ ! -f /data/.bitcoin/bitcoin.conf ]; do sleep 1; done;
              exec bitcoind -regtest -rpcbind=0.0.0.0 -rpcallowip=0.0.0.0/0'
```

结果：容器起不来，一直 **Restarting**。

👉 这一步踩的坑：**不要轻易覆盖镜像原本的启动命令**。

Bitlight 的镜像里有自己的初始化逻辑（比如生成 cookie、钱包等），被我这一刀全砍掉了，直接挂了。

结论：**保持原有启动逻辑不动，只做端口映射即可**。

***

#### **3. 找到配置文件 bitcoin.conf**

既然 command 覆盖不靠谱，那就回头找：bitcoind 其实默认会读配置文件。

于是我们进容器看：

```
docker compose -p bitlight-local-env exec bitcoin-core ls -l /data/.bitcoin
```

果然发现了 bitcoin.conf。

再确认内容：

```
docker compose -p bitlight-local-env exec bitcoin-core cat /data/.bitcoin/bitcoin.conf
```

结论：**要改 RPC 配置，改 bitcoin.conf 就对了**。

***

#### **4. 修改 bitcoin.conf**

刚开始我们只加了 rpcuser 和 rpcpassword，结果 curl 一直报 403 Forbidden。

反复试，才发现要多加两行：

* rpcbind=0.0.0.0 → 否则只监听容器内回环。
* rpcallowip=127.0.0.1 和 rpcallowip=0.0.0.0/0 → 否则宿主机请求直接拒绝。

最终版本：

```
regtest=1
server=1
rpcuser=bitcoin
rpcpassword=bitcoin
rpcbind=0.0.0.0
rpcallowip=127.0.0.1
rpcallowip=0.0.0.0/0
```

改完再 docker compose restart bitcoin-core。

***

#### **5. curl 调试：一步步见证从超时 → 403 → 成功**

这一步特别关键。

* **没映射端口时** → curl 直接卡死。
* **没配 rpcbind/rpcallowip 时** → 403 Forbidden。
* **没配 rpcuser/rpcpassword 时** → 401 Unauthorized。
* **全都正确时** → 成功返回 JSON：

```
{"result":{"chain":"regtest","blocks":209,"headers":209,"bestblockhash":"..."}}
```

👉 结论：用 curl 先调通，才能避免 Python 里一堆莫名其妙的异常。

***

#### **6. Python 连通：目标达成**

最后，在宿主机装 python-bitcoinrpc：

```
pip install python-bitcoinrpc
```

代码只要三行：

```
from bitcoinrpc.authproxy import AuthServiceProxy

rpc = AuthServiceProxy("<http://bitcoin:bitcoin@127.0.0.1:18443>")
print(rpc.getblockchaininfo())
```

输出：

```
{'chain': 'regtest', 'blocks': 209, ...}
```

至此：**Python → Docker 内 bitcoind → 成功打通 ✅**。

***

### **📌 避坑总结**

1. **不要覆盖容器启动命令**，保持 init 流程，用 bitcoin.conf 扩展 RPC 配置。
2. **rpcbind/rpcallowip 必须显式写**，否则宿主机请求全被拒。
3. **curl 分步验证是必经之路**：超时、403、401，分别对应不同配置问题。
4. **Python 只是最后一步**，前面每个坑都得过掉才能跑起来。

### **🌈 意义与展望**

为什么要折腾这一大圈？

*   **复用 bitlight 提供的基础设施**

    我们不用自己重新搭建 regtest、钱包环境，直接利用他们的 demo 框架。**bitlight**的 demo infra，做得不错，能快速帮大家做好设施。
*   **打通 Python 与 bitcoind**

    以前必须在容器里 make core-cli、make alice-cli，遇到 PSBT / consignment 还得手动搬运，现在只要 Python 调 RPC，就能完成同样的事情。
*   **为 RGB 编程打下基础**

    有了这条直连链路，下一步就能把 RGB 的钱包逻辑、consignment 流程搬进 Python 脚本里，而不是在不同容器之间复制粘贴。

换句话说：这一步让我们真正站在 **RGB「可编程」的起点**。

未来，我们就可以在 Python 里：

* 调 Core 拿 UTXO；
* 调 BDK 钱包签名；
* 传 consignment 给 Bob 验证；
* 自动化整个 RGB 流程。
