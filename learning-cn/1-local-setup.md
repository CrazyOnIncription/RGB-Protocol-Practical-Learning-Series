# 1.本地 RGB20 发行与转账踩坑记录（bitlight-local-env-public）

### **1. 背景与目标**

基于 [bitlightlabs/bitlight-local-env-public](https://github.com/bitlightlabs/bitlight-local-env-public) 搭建 **regtest** 本地链，完整走一遍 **RGB20 资产发行 → 转账 → 校验**。

**关键点**：教程对应的 rgb CLI 是 **v0.11.0-beta.9**（旧版）。新版 CLI 命令不兼容，必须用项目容器内自带的版本或自己编译同版本。

***

### **2. 环境准备**

#### **启动本地链**

```
git clone <https://github.com/bitlightlabs/bitlight-local-env-public>
cd bitlight-local-env-public
make start
```

#### **进入钱包容器（会打印 XPRV / XPUB / 地址）**

```
make alice-cli   # 进入 alice 钱包 REPL
# 记下 Alice 的 Fixed XPUB（后续做 descriptor）
# 记下 Alice 的 Bitcoin 地址（后续打钱）

make bob-cli     # 进入 bob 钱包 REPL
# 同样记下 Bob 的 Fixed XPUB 与地址
```

#### **准备比特币测试币**

```
make core-cli
mint 1  # 简单方式，给挖矿钱包挖一个块，后续可按教程 send 到 Alice/Bob 地址
```

***

### **3. 发行与转账（可复现步骤）**

> 下文所有命令都在
>
> **宿主机**

#### **3.1 用 XPUB 创建 RGB 钱包**

把在容器里看到的 Fixed XPUB（含 <0;1;9;10>/\*）填到环境变量：

```
ALICE_DESC="[5183a8d8/86'/1'/0']tpubDDtdVYn7LWnWNUXADgoLGu48aLH4dZ17hYfRfV9rjB7QQK3BrphnrSV6pGAeyfyiAM7DmXPJgRzGoBdwWvRoFdJoMVpWfmM9FCk8ojVhbMS/<0;1;9;10>/*"
BOB_DESC="[3abb3cbb/86'/1'/0']tpubDDeBXqUqyVcbe75SbHAPNXMojntFu5C8KTUEhnyQc74Bty6h8XxqmCavPBMd1fqQQAAYDdp6MAAcN4e2eJQFH3v4txc89s8wvHg98QSUrdL/<0;1;9;10>/*"
```

创建钱包（老版 CLI 没有 init 子命令）：

```
rgb -d .alice -n regtest create default --tapret-key-only "$ALICE_DESC"
rgb -d .bob   -n regtest create default --tapret-key-only "$BOB_DESC"
```

查看 UTXO（确认两边都有资金）：

```
rgb -d .alice -n regtest utxos
rgb -d .bob   -n regtest utxos
```

#### **3.2 导入合约（发行已在合约文件中完成）**

先在你的合约项目（例如 bitlight-rgb20-contract）里生成 rgb20-simplest.rgb，然后在宿主机导入：

```
CONTRACT_FILE="/绝对路径/bitlight-rgb20-contract/test/rgb20-simplest.rgb"

rgb -d .alice -n regtest import "$CONTRACT_FILE" --esplora=http://localhost:3002
rgb -d .bob   -n regtest import "$CONTRACT_FILE" --esplora=http://localhost:3002
```

查看合约列表并记下 **合约 ID**（有 !、$ 等特殊字符，请用 **单引号**）：

```
rgb -d .alice -n regtest contracts
# 例：BppYGUUL-Qboz3UD-czwAaVV-!!Jkr1a-SE1!m1f-Cz$b0xs
RGB20_CONTRACT='BppYGUUL-Qboz3UD-czwAaVV-!!Jkr1a-SE1!m1f-Cz$b0xs'
```

#### **3.3 Bob 生成收款发票**

老版 invoice **不带金额参数**，需要手动把金额字段插入：

```
INV_NOAMT=$(rgb -d .bob -n regtest invoice "$RGB20_CONTRACT")
# 把 “/RGB20Fixed/” 替换为 “/RGB20Fixed/TadF+”
INVOICE=$(printf '%s' "$INV_NOAMT" | sed 's#RGB20Fixed/#RGB20Fixed/TadF+#')
echo "$INVOICE"
```

#### **3.4 Alice 制作转账（生成 consignment + PSBT）**

```
rgb -d .alice -n regtest transfer "$INVOICE" transfer.consignment alice.psbt
```

#### **3.5 Bob 校验并接受转账**

```
rgb -d .bob -n regtest validate transfer.consignment
rgb -d .bob -n regtest accept -f transfer.consignment
```

#### **3.6 Alice 签名 PSBT（用 bdk-cli）**

把 alice.psbt 转成 base64，粘进 **bdk-cli REPL** 的 wallet sign：

```
# 在宿主机
PB64=$(base64 < alice.psbt | tr -d '\\r\\n')
echo "$PB64"  # 复制整段

# 在钱包容器 (make alice-cli) 里执行：
wallet sign --psbt "<上面整段 base64>"
```

取 REPL 返回的 JSON 里 "psbt" 的 **signed base64**，保存并解码：

```
cat > alice.psbt.signed.b64 <<'EOF'
<把 JSON 里 "psbt" 的完整 base64 粘这里>
EOF
base64 -D -i alice.psbt.signed.b64 > alice.psbt.signed
```

#### **3.7 Finalize、生成原始交易并广播**

```
rgb -d .alice -n regtest finalize alice.psbt.signed alice.final.tx

# 用 Esplora API 广播
HEX=$(xxd -p -c 999 alice.final.tx)
TXID=$(curl -s -X POST <http://localhost:3002/tx> -d "$HEX")
echo "TXID=$TXID"
```

#### **3.8 挖块确认 & 同步状态**

```
make core-cli
mint 1

# 同步并查看 RGB 状态（两边各查一次）
rgb -d .alice -n regtest state "$RGB20_CONTRACT" RGB20Fixed --sync --esplora=http://localhost:3002
rgb -d .bob   -n regtest state "$RGB20_CONTRACT" RGB20Fixed --sync --esplora=http://localhost:3002

# 历史
rgb -d .alice -n regtest history "$RGB20_CONTRACT"
rgb -d .bob   -n regtest history "$RGB20_CONTRACT"
```

***

### **4. 踩坑与要点**

1. **版本对齐**：教程针对 rgb-wallet v0.11.0-beta.9。新版本命令集不同，务必用容器内或自己编译出的 **同版本** CLI。
2. **数据目录**：旧版把数据放在 .alice/regtest、.bob/regtest。不要和新版的 bitcoin.testnet/bitcoin.regtest 混用。
3. **特殊字符**：合约 ID 与发票包含 !、$ 等，**zsh 必须用单引号** '...'，否则会被历史扩展吃掉。
4. **链上解析**：涉及链交互的命令（import、state --sync 等）加 --esplora=http://localhost:3002，否则会报 _no resolver specified_。
5. **发票金额**：invoice 不带金额，需把 RGB20Fixed/ 手工替换为 RGB20Fixed/TadF+\<amount>。
6. **签名工具**：bdk-cli 新版参数是 wallet --database-type sqlite --ext-descriptor --int-descriptor，REPL 里用 wallet sign --psbt "\<base64>" 最稳。
7. **广播方式**：宿主的 bitcoin-cli 未连接容器数据目录，直接用 **Esplora /tx 接口** 广播最省事。
8. **同步刷新**：完成广播后用 state ... --sync --esplora=... 强制刷新见证与余额；必要时 mint 1 挖块确认。

***

### **5. 参考仓库**

* 本地链与钱包容器：[https://github.com/bitlightlabs/bitlight-local-env-public](https://github.com/bitlightlabs/bitlight-local-env-public)
* RGB v0.11 源码（可本机编译 rgb CLI 同版本）：[https://github.com/RGB-WG/rgb](https://github.com/RGB-WG/rgb)
* RGB20 合约样例：[https://github.com/bitlightlabs/bitlight-rgb20-contract](https://github.com/bitlightlabs/bitlight-rgb20-contract)



**附流程图**：

```jsx
┌──────────┐
│  Alice   │
│(发行方)   │
└─────┬────┘
      │
      │ 1. 在 REPL 中发行 RGB20 合约
      │    - 指定 Ticker / Name / Precision / Amount
      │    - 在发行时选择一个本地 UTXO 作为锚定点
      ▼
┌──────────┐
│ Bitcoin  │
│ 区块链    │
└─────┬────┘
      │
      │ 2. 存在一个承载 RGB 合约的“锚定交易”
      │    (锚定 UTXO: Alice 钱包中的某个输出)
      ▼
┌──────────┐
│   Bob    │
│(接收方)   │
└─────┬────┘
      │
      │ 3. Bob 生成发票 (invoice)
      │    - 包含 Bob 的收款 seal + 资产 ID
      ▼
┌──────────┐
│  Alice   │
│(发行方)   │
└─────┬────┘
      │
      │ 4. Alice 根据发票生成转账 consignment
      │ 5. Alice 用自己的私钥签名交易 (PSBT)
      │ 6. 广播到 Bitcoin 链上
      ▼
┌──────────┐
│ Bitcoin  │
│ 区块链    │
└─────┬────┘
      │
      │ 7. 交易确认后，双方同步状态
      ▼
┌──────────┐
│  Alice   │
│   Bob    │
└──────────┘
```

```markup
# 本地 RGB20 发行 & 转账 — 简化命令表

1️⃣ 启动环境
--------------------------------
git clone <https://github.com/bitlightlabs/bitlight-local-env-public>
cd bitlight-local-env-public
make start

2️⃣ 进入 Alice 钱包容器 (发行方)
--------------------------------
make alice-cli
# 选择对应 descriptor 进入 REPL

3️⃣ 发行 RGB20 合约
--------------------------------
rgb -d .alice -n regtest issue \\
  --ticker TEST --name "Test asset" \\
  --precision 2 --amount 100000000000 \\
  --iface RGB20Fixed

# 记录输出的 CONTRACT_ID

4️⃣ 进入 Bob 钱包容器 (接收方)
--------------------------------
make bob-cli
# 生成发票 (invoice)
rgb -d .bob -n regtest invoice "$CONTRACT_ID" RGB20Fixed 2000 > bob.invoice

5️⃣ Alice 根据发票创建转账
--------------------------------
# 宿主机执行
rgb -d .alice -n regtest transfer bob.invoice > alice.psbt

6️⃣ 签名交易 (Alice 容器内)
--------------------------------
PB64=$(base64 < alice.psbt | tr -d '\\r\\n')
wallet sign --psbt "$PB64" > signed.json
# 提取 signed.json 中的 psbt 字段，保存为 alice.psbt.signed

7️⃣ 广播交易 (宿主机)
--------------------------------
xxd -p -c 999 alice.final.tx > hex.txt
HEX=$(cat hex.txt)
docker compose -p bitlight-local-env exec -T core-cli \\
  bitcoin-cli -regtest sendrawtransaction "$HEX"

8️⃣ 挖矿确认
--------------------------------
make core-cli
mint 1

9️⃣ 同步并查看状态
--------------------------------
rgb -d .alice -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora=http://localhost:3002
rgb -d .bob   -n regtest state "$CONTRACT_ID" RGB20Fixed --sync --esplora=http://localhost:3002
```

```markup
完整的：RGB 资产转账链上结构 & CLI 操作对照

**STEP 1: Alice 发行资产（Anchor Commitment）**
-------------------------------------------
链上结构：
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

Bitlight CLI：
# 进入 Alice REPL
make alice-cli
# 选择 RGB Descriptor
# 发行 RGB20 资产
rgb -d .alice -n regtest issue --ticker TEST --name "Test asset" --amount 100000000000 RGB20Fixed

**STEP 2: Bob 发票（Invoice / Seal）**
-----------------------------------
链上结构：
[ Bob Internal PubKey ] --or--> [ Bob blind seal hash ]
       |
       +-- (Optional) blinding factor r
       |
       --> Invoice sent off-chain to Alice:
           "Please lock N units of Asset_X to this seal"

Bitlight CLI：
# 进入 Bob REPL
make bob-cli
# 生成发票（seal 封印信息在这里）
rgb -d .bob -n regtest invoice "$RGB20_CONTRACT" 2000
# 得到一串 rgb:... 发票字符串，发给 Alice

**STEP 3: Alice 转账（Commit to Bob's Seal）**
-------------------------------------------
链上结构：
[ UTXO_A: Alice's asset anchor ]
       |
       --> 构建交易：
           txin: spend UTXO_A
           txout0: Bob's tweaked pubkey (包含新的 RGB commitment)
           txout1: Alice 找零
       |
       --> 新 commitment_hash 描述：
           "Alice: -N units, Bob: +N units"

Bitlight CLI：
# Alice 用 Bob 的发票生成转账 consignment
rgb -d .alice -n regtest transfer "$RGB20_CONTRACT" <发票字符串>
# 得到 PSBT，Alice 在宿主环境用 bdk-cli 签名
# 将签名好的 PSBT 转成最终交易 hex
# 广播交易（docker 内 core-cli 或 REST API）

**STEP 4: Bob 验证（Reveal & Accept）**
-------------------------------------
链上结构：
Bob 检查 txout0：
   pubkey' ?= Bob_internal_pubkey + H( Bob_internal_pubkey || commitment_hash )*G
匹配则：
   - 校验 consignment 证明
   - 更新本地 RGB 状态数据库：
       UTXO_B now anchors Bob's +N units of Asset_X

Bitlight CLI：
# Bob 同步状态
rgb -d .bob -n regtest state "$RGB20_CONTRACT" RGB20Fixed --sync --esplora=http://localhost:3002
# Bob 查看历史
rgb -d .bob -n regtest history "$RGB20_CONTRACT"

```
