# 2.  程序员眼里的 RGB 协议原理与设计

### **1. 所有 RGB 交易 = 普通比特币交易**

* 在链上，**RGB 看起来就是 Taproot 交易**。
* RGB 的承诺（commitment）嵌入在 Taproot 公钥的 tweak（或 script path）里。
* 对比特币主网来说，它只是一个普通的比特币交易，没有“特殊格式”。
* 因此：
  * **mempool.space / explorer** 上显示的就是 BTC 转账。
  * RGB 客户端才能解读里面的 “RGB20 资产变化”。

***

### **2. RGB 逻辑在链下解释**

* 真正的 “RGB 转账” **不在链上运行**，而是在链下：
  * Alice 根据 Bob 的发票（invoice）生成 **consignment**（本地证明包）。
  * Bob 使用 RGB 客户端 **validate + accept**，更新自己的本地 state.dat。
* 账本组成：

```
Bitcoin 主网 (顺序 & 安全)
+ 本地 stash/state 文件 (合约状态)
= RGB 账本
```

* 主网只保证「顺序不可篡改」，RGB 客户端负责「含义解释」。

***

### **3. 为什么要这样？**

* **不污染比特币协议**：RGB 不需要软分叉/硬分叉。
* **保持轻量**：主网节点完全不需要理解 RGB。
* **隐私友好**：只有参与方才知道某笔交易携带了 RGB 承诺。
* **极小的链上开销**：一笔 RGB 转账 ≈ 一笔普通 Taproot 交易的手续费。

***

### **4. 核心流程**

1. **锚定（Commitment）**
   * Alice 在自己控制的 UTXO 上，嵌入合约状态承诺（tapret commitment）。
2. **发票（Invoice）**
   * Bob 生成 “我要收多少资产 + 我的封印方式(UTXO seal)” 并给 Alice。
3. **转移（Transfer）**
   * Alice 根据发票构建 consignment，链上生成带 commitment 的交易。
4. **验证（Validation）**
   * Bob 用链上数据 + Alice 的 consignment 验证有效性，更新自己的账本。

***

### **5. 与铭文 (Ordinals) 的对比**

* 铭文：**链上存数据**，explorer 直接能看到。
* RGB：**链下存证明**，链上只存不可篡改的锚点。
* 结果：
  * 铭文的 “状态解释” 依赖 indexer，一家和另一家可能不同。
  * RGB 的 “状态解释” 自包含在 consignment + 主网交易，只要验证过，任何客户端结果必然一致。

***

### **6. 优势总结**

* **极致扩展性**：智能合约逻辑不落链，比特币无需修改。
* **安全性继承**：状态锚定在 BTC UTXO，不可伪造。
* **隐私性**：链上不可见，只有参与方有完整上下文。
* **成本低**：只消耗一次普通 BTC 交易手续费。

***

### **7. Taproot 承诺机制：tweak & tapret**

_注：看到数学原理头疼的朋友可以略过。当然也欢迎你去上我在登链社区开的比特币公开课，_[https://learnblockchain.cn/course/76，\*里面有这内容\*](https://learnblockchain.cn/course/76%EF%BC%8C*%E9%87%8C%E9%9D%A2%E6%9C%89%E8%BF%99%E5%86%85%E5%AE%B9*)

RGB 的 **链上锚定（commitment）** 本质就是利用 Taproot 的 tweak 属性，把合约状态哈希藏进公钥或 script path。

#### **Key-path tweak 模式 (tapret-key-only)**

* 选一个内部公钥 P，对应私钥 k。
* 生成 commitment 值 c = H(\text{RGB state})。
*   计算 tweak：

    t = H(P \parallel c)
*   得到 tweaked 公钥：

    P’ = P + t \cdot G
* Taproot 地址 = P2TR(P′)，链上看上去就是**普通地址**。

🔑 **效果**：

* 链上看不到任何 “RGB” 的痕迹。
* 只有 reveal 时公开 P, c，别人才能验证 commitment。
* 花费时用 tweaked 私钥 k + t 正常走 key-path 签名。

#### **Script-path tweak 模式 (tapret-tree)**

* 不是直接改 key，而是在 Taproot script tree 里加入一条虚拟分支，叶子脚本里包含 commitment。
* 花费时 reveal 这个 script leaf，别人能直接看到承诺的内容。

🔑 **效果**：

* **key-path tweak**：隐私最好，链上不可见。
* **script-path tweak**：承诺直接暴露在 witness 里，可被外部工具扫描到。

#### **RGB 采用的模式**

* 在实践中，RGB 客户端通常默认用 **tapret-key-only**：
  * 每次转账 → 生成一个新的 tweaked 公钥地址。
  * 保证承诺不可见，只能通过 consignment + 本地验证解读。

***

### **链上眼里 vs Ordinals Indexer vs RGB 客户端**

```python
           Bitcoin 主网眼里            |     Ordinals Indexer 眼里          |        RGB 客户端眼里
───────────────────────────────────── ┼───────────────────────────────────┼────────────────────────────────────
TXID: abc123...                       | TXID: abc123...                   | TXID: abc123...
------------------------------------- | --------------------------------- | -----------------------------------
		输入: 引用前一笔 UTXO                | 输入: 一样                         | 输入: 一样
		输出:                              |  输出:                             | 输出:
  - Taproot 地址 A (1 BTC)             |   - Taproot 地址 A (1 BTC)        |   - Taproot 地址 A (1 BTC)
                                      |     (含 Ordinals inscription:     |     (含 RGB 承诺: Alice➝Bob 2000 TEST)
                                      |      "Hello World" 图片数据)       |
链上看到:                              | Ordinals 解释:                     | RGB 解释:
  - 一笔普通 BTC 转账                   |   - 这是一个 NFT 铭文               |   - 这是一个资产转账
  - 不包含额外语义                      |   - 属于某个 satoshi                |   - 更新 Bob 的 RGB20 余额
```
