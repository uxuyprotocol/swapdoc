# Solana 询价与兑换

Solana 链的兑换分两步：先询价拿行情，再取**待签名指令链**。本服务只产出指令与待签名交易，签名与广播由业务方完成。

| 方法名 | 用途 | 是否构建交易 |
|---|---|---|
| `dex_offlineQuoteSol` | 询价：给定输入代币/数量，返回最优路由与预计产出 | 否 |
| `dex_offlineSwapSol` | 兑换：返回可直接组装的指令链，以及已组装好的待签名交易 | 是 |

```
dex_offlineQuoteSol（可选，用于展示行情）
        │
        ▼
dex_offlineSwapSol（simulate:true 预执行验证）
        │
        ├─ 路径 A（推荐）：反序列化 unsignedTransaction → 签名 → 广播
        └─ 路径 B：按 instructionLists + tx.computeBudget 自组 v1 交易
        │
        ▼
业务方签名广播；用 getTransaction(maxSupportedTransactionVersion: 1) 查询回执
```

## 接口信息

| 项目 | 值 |
|---|---|
| 请求方式 | `POST` |
| 请求地址 | `https://gwapi.ourdex.com/solana/<appId>` |
| 鉴权 | `Authorization: Bearer <JWT>` |
| Content-Type | `application/json` |
| `chainIndex` | Solana 固定 `"501"`（兼容值 `"6000001"`），缺省按 URL 链名 |

## 通用约定

- **传输协议**：JSON-RPC 2.0，请求体 `{"jsonrpc":"2.0","id":1,"method":"<方法名>","params":[{…}]}`。
- **地址格式**：Solana base58 字符串。
- **原生 SOL 的表示**：请求与响应中的原生 SOL 统一用 System Program 地址 `11111111111111111111111111111111`。
  - 传 `111…1` = **原生 SOL** 语义：指令链会自动完成 wrap（SOL→wSOL）与 unwrap（wSOL→SOL），并附带 wSOL 临时账户的创建与关闭。
  - 传 `So11111111111111111111111111111111111111112`（wSOL mint）= **字面 wSOL 代币**语义：直接借记/记入用户 wSOL 账户，不做 wrap/unwrap。两者不可混用。
- **数量单位**：除 `platformFee.amount` 外，所有金额字段（`amount`、`fromTokenAmount`、`toTokenAmount`、`minReceiveAmount`）均为**最小单位整数字符串**，如 USDC（6 位精度）1 枚 = `"1000000"`。
- **报价基准**：完全基于服务端本地扫描的链上状态（离线引擎），存在正常的快照时差；生产环境建议 `simulate:true` 做执行前验证。
- **错误形态**：JSON-RPC 标准错误对象 `{"error":{"code":-32000,"message":"…"}}`，`message` 会点名出错的参数。

---

## 询价：`dex_offlineQuoteSol`

只报价，不构建指令。返回最优路由（平铺在 `result` 顶层）与全部候选路由 `routerList`。

### 请求参数

`params` 为只含一个对象的数组。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `fromTokenAddress` | string | 是 | 支付代币 mint；原生 SOL 传 `111…1` |
| `toTokenAddress` | string | 是 | 收取代币 mint；原生 SOL 传 `111…1` |
| `amount` | string | 是 | 兑换数量，**最小单位整数字符串**。按 `exactIn` 语义解释（即输入数量） |
| `slippagePercent` | string | 否 | 滑点百分比字符串，`"0.5"` = 0.5%，范围 [0, 100)。显式传入时按该值计算 `minReceiveAmount`，接口不会自动抬高；缺省由引擎推荐（见返回 `suggestSlippagePercent`）。开启 `autoSlippage` 时本字段被忽略 |
| `swapMode` | string | 否 | `exactIn`（默认）/ `maxIn`。`maxIn` 当前仅回显该模式：报价仍按 `exactIn` 处理，不做产出反推输入 |
| `autoSlippage` | bool | 否 | 开启自动滑点：忽略显式 `slippagePercent`，改用引擎推荐值，并在 `maxAutoSlippagePercent` 内抬高实际生效滑点、重算最小到账 |
| `maxAutoSlippagePercent` | string | 否 | 自动滑点上限百分比；缺省不设额外上限 |
| `priceImpactProtectionPercent` | string | 否 | 价格影响保护阈值，默认 `90`，`100` = 关闭；超阈值的路由被逐条丢弃，全部路由都超阈值时返回错误 |
| `userWalletAddress` | string | 否 | 用户钱包地址；询价阶段仅作路由参考 |
| `dexIds` | string | 否 | 仅允许的 DEX 程序白名单，逗号分隔（与 `size` 独立） |
| `excludeDexIds` | string | 否 | 排除的 DEX 程序黑名单，逗号分隔 |
| `size` | int | 否 | 候选路由数量，取值 1–10，默认 3 |
| `excludePoolAddresses` | string | 否 | 排除的池地址，逗号分隔，最多 20 个 |
| `directRoute` | bool | 否 | 只走单池直达路由 |

**保留但当前不生效的参数**（静默接受，勿依赖）：`useTokenLedger`、`positiveSlippagePercent`、`positiveSlippageFeeAddress`、`feePercent`、`referrerAddress`、`closeAuthorityAddress`、`maxCalldataSize`、`maxAccounts`、`forJitoBundle`、`disableRFQ`、`singleRouteOnly`、`singlePoolPerHop`、`assetAwareRouting`。

### 响应参数

`result` 顶层为最优路由，`routerList` 为全部候选路由（结构相同）。

| 字段 | 类型 | 说明 |
|---|---|---|
| `chainIndex` | string | 固定 `"501"` |
| `swapMode` | string | 本次报价模式（`exactIn` / `maxIn`） |
| `fromToken` | object | 支付代币信息，结构见下 |
| `fromTokenAmount` | string | 支付数量（最小单位；原生 SOL 为 lamports） |
| `toToken` | object | 收取代币信息，结构见下 |
| `toTokenAmount` | string | 预计收到数量（最小单位） |
| `priceImpactPercent` | string | 价格影响百分比，`"0.19"` = 0.19% |
| `router` | string | 路由摘要（经过的代币序列） |
| `dexRouterList` | array | 逐跳明细：`dexProtocol{dexName, percent}`、`fromToken`/`toToken`、`fromTokenIndex`/`toTokenIndex` |
| `minReceiveAmount` | string | 按实际生效滑点折算的最小应收到数量（最小单位） |
| `slippagePercent` | string | 本次实际生效的滑点百分比（自动滑点或守卫抬升后为新值） |
| `suggestSlippagePercent` | string | 引擎推荐的滑点百分比 |
| `routerList` | array | 全部候选路由 |

`fromToken` / `toToken` / `dexRouterList[].fromToken|toToken` 子结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `tokenContractAddress` | string | 代币 mint 地址（请求侧用 `111…1` 时回显 `111…1`；中间跳恒为真实 mint） |
| `tokenSymbol` | string | 代币符号（本地库缺失时可能为空） |
| `tokenUnitPrice` | string | 代币美元单价（本地库缺失时为空） |
| `decimal` | string | 精度位数 |
| `isHoneyPot` | bool | 蜜罐标记（Solana 侧恒 `false`） |
| `taxRate` | string | 转账税率（Solana 侧恒 `"0"`） |

> 询价不返回 `estimateGasFee` / `tradeFee` / `contextSlot`。

### 请求示例

```bash
curl -s -X POST 'https://gwapi.ourdex.com/solana/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0","id":1,"method":"dex_offlineQuoteSol",
    "params":[{
      "fromTokenAddress":"11111111111111111111111111111111",
      "toTokenAddress":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "amount":"100000000",
      "slippagePercent":"0.5"
    }]
  }'
```

### 响应示例（节选）

```json
{
  "jsonrpc":"2.0","id":1,
  "result":{
    "chainIndex":"501",
    "swapMode":"exactIn",
    "fromToken":{"tokenContractAddress":"11111111111111111111111111111111","tokenSymbol":"SOL","decimal":"9"},
    "fromTokenAmount":"100000000",
    "toToken":{"tokenContractAddress":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v","tokenSymbol":"USDC","decimal":"6"},
    "toTokenAmount":"1523402",
    "priceImpactPercent":"0.021",
    "minReceiveAmount":"1515783",
    "slippagePercent":"0.5",
    "suggestSlippagePercent":"0.1",
    "dexRouterList":[{"dexProtocol":{"dexName":"raydium  clmm","percent":"100"}}],
    "routerList":["…同结构候选路由…"]
  }
}
```

---

## 兑换：`dex_offlineSwapSol`

返回可直接签名广播的指令链。继承询价全部参数，另加下列字段（`userWalletAddress` 在本端点**必填**）。

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userWalletAddress` | string | 是 | 用户钱包地址（base58）。指令链中的用户账户（ATA、swap 权限账户）均按此地址构建 |
| `computeUnitPrice` | string | 否 | 优先费单价，单位 micro-lamports / CU（如 `"100000"`）。服务端换算为 v1 交易内联的优先费总额，回显在 `tx.computeBudget.priorityFeeLamports`；`0` / 缺省 = 只付基础费 |
| `computeUnitLimit` | string | 否 | CU 上限。缺省时服务端自动解析：`simulate:true` 且拿到了模拟消耗时按「消耗 +10%」注入，否则用默认预算（每指令 20 万 CU，封顶 140 万）；回显在 `tx.computeBudget.computeUnitLimit` |
| `simulate` | bool | 否 | 建议 `true`：服务端对组装后的完整指令链做 `simulateTransaction` 预执行，结果在 `simulateResult`。**预执行失败不要广播** |
| `feePayer` | string | 否 | 代付地址：作为交易 payer 并承担 ATA 创建租金（原生 SOL 输入的一次性 wSOL 临时账户由钱包出资）。缺省 = `userWalletAddress`。返回 `tx.feePayer`，组包时必须以它为 payer 并收集其签名 |
| `swapReceiverAddress` | string | 否 | 代收地址（A 兑给 B）：末跳输出归属接收方 B，接收方无需签名；签名者与输入侧仍在钱包 A。原生 SOL 输出同样支持 |
| `memo` | string | 否 | 交易备注：生成 SPL Memo 指令（UTF-8） |
| `platformFee` | object | 否 | 平台手续费，结构见下；在指令链尾部追加一笔转账 |

`platformFee` 子对象：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `to` | string | 是（费用生效时） | 平台收款地址 |
| `amount` | string | 是 | 手续费数量，**可读金额**（非最小单位）：0.1 USDT 传 `"0.1"`；`≤ 0` 时整个费用块跳过。精度超过代币精度会报错 |
| `contract` | string | 否 | 手续费代币 mint。缺省 / `"SOL"` / `111…1` / wSOL mint = 原生 SOL；传其他 mint 则做 SPL / Token-2022 转账 |
| `from` | string | 否 | 手续费扣款地址，缺省 = `userWalletAddress`；设为第三地址时该地址也必须签名 |

手续费代币支持范围：经典 SPL 代币（收款方 ATA 缺失时自动创建）、Token-2022（含 transfer-fee / transfer-hook 扩展）。

### 响应参数

| 字段 | 类型 | 说明 |
|---|---|---|
| `instructionLists` | array | 待组装的指令链，按执行顺序排列，结构见下。**不含 ComputeBudget 指令**（v1 交易拒绝指令形式的预算，见 `tx.computeBudget`） |
| `createTokenAccountList` | array | 本次交易将创建的账户地址（幂等创建的 ATA、一次性 wSOL 临时账户）。ATA 租金由 `feePayer` 承担，原生 SOL 输入的一次性 wSOL 临时账户租金由钱包出资 |
| `routerResult` | object | 最优路由报价，结构与询价响应相同，另含 `estimateGasFee`、`tradeFee` |
| `routerResult.estimateGasFee` | string | 预估网络费（lamports）= 5000 基础费 + CU 上限 × `computeUnitPrice` / 10⁶ |
| `routerResult.tradeFee` | string | 预估网络费的美元价值（仅在平台能取到 SOL 价格时返回） |
| `tx` | object | 交易摘要：`from`、`to`（`swapReceiverAddress` 回显，未设时为空）、`minReceiveAmount`、`slippagePercent`、`feePayer`、`computeBudget` |
| `wsolRentFee` | number | 路由涉及原生 SOL 时的 wSOL 临时账户租金（`2039280` lamports），否则为 `0`。除 `swapReceiverAddress` + 原生 SOL 输出外，租金在交易结束随账户关闭退还 |
| `unsignedTransaction` | string | **已组装、待签名的完整交易**（base64，v1 wire 格式）：嵌入最新 finalized blockhash、按必需签名者预留零签名槽、内联 compute 预算，payer = `tx.feePayer` |
| `transactionVersion` | string | 恒为 `"v1"` |
| `executionEngine` | string | 恒为 `"uxuy-router"`：整条路由在 uxuy-router 程序内原子执行，余额差 + 最小到账兜底 |
| `routerList` | array | 全部候选路由 |
| `simulateResult` | object | `simulate:true` 时的预执行结果，结构见下 |

`instructionLists[]` 指令结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `programId` | string | 目标程序地址（base58） |
| `data` | string | 指令数据，**base64** 编码 |
| `accounts` | array | 账户元信息，顺序即传参顺序：`{pubkey, isSigner, isWritable}` |

`tx.computeBudget` 子结构（v1 内联预算，自行组装交易时必须写入消息配置）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `computeUnitLimit` | number | CU 上限。v1 消息不带上限 = 0 CU（交易必败），服务端恒显式解析 |
| `priorityFeeLamports` | number | 优先费总额（lamports），由 `computeUnitPrice` 按生效 CU 上限换算 |
| `loadedAccountsDataSizeLimit` | number | 加载账户数据上限（字节），服务端恒设为 64 MiB |

`simulateResult` 结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `ok` | bool | `true` = 按当前构造可执行成功 |
| `err` | string | 失败时的错误摘要（含程序日志定位） |
| `logs` | array | 完整模拟日志（失败排查看尾部 `Program log: Error` 行） |
| `consumedUnits` | number | 模拟消耗的 CU |
| `appliedUnitLimit` | number | 服务端自动注入的 CU 上限（仅请求未显式传 `computeUnitLimit` 时返回） |

### 指令链结构

服务端产出的顺序即执行顺序，典型形态：

```
（compute 预算不是指令——在 v1 消息内联配置里，见 tx.computeBudget）
[memo]                             ← memo 非空时
[CreateIdempotent × N]             ← 缺失的 ATA（租金由 feePayer 承担）
[原生 SOL 输入的 wrap 指令组]        ← 一次性 wSOL 临时账户（CreateAccountWithSeed + InitializeAccount3，租金由钱包出资）+ Transfer + SyncNative
[中间 mint 的一次性临时账户]         ← 仅路径包含中间 mint 时（由路由 authority 持有）
[1 条 uxuy-router swap 指令]        ← 整条路由只有一条合约指令，内含全部跳
[CloseAccount]                     ← 仅路由触及原生 SOL（解包 + 退租金）
[CreateIdempotent × 2]             ← 仅 SPL 平台费：收款方/扣款方 ATA 缺失时
[memo]                             ← 仅 Token-2022 手续费 + memo 非空（紧邻转账）
[平台费转账]                        ← System.Transfer 或 Token / Token-2022 转账
```

### 组装、签名与广播

交易为 **v1 格式（SIMD-0385）**：序列化 ≤ 4096 字节、去重账户 ≤ 64、指令 ≤ 64、签名 ≤ 12；compute 预算内联在消息配置中，**不接受 ComputeBudget 指令**。两条路径二选一，结果一致。

**路径 A（推荐）：直接用返回的交易**

```js
// 需要支持 SIMD-0385 v1 反序列化的 web3.js 版本
const tx = VersionedTransaction.deserialize(Buffer.from(res.unsignedTransaction, 'base64'));
tx.sign([walletKeypair]);                          // 用户钱包
if (feePayerKeypair) tx.sign([feePayerKeypair]);    // 传了 feePayer 时
await connection.sendRawTransaction(tx.serialize(), { maxSupportedTransactionVersion: 1 });
```

`unsignedTransaction` 内的 blockhash 约 1 分钟后过期，**拿到后请及时签名广播**。

**路径 B：按 `instructionLists` 自行组装 v1 交易**

1. 指令按 `instructionLists` 原序；
2. 自取 `latestBlockhash` 作为消息 lifetime；
3. 把 `tx.computeBudget` 写进消息内联配置（不要生成 ComputeBudget 指令）；
4. payer = `tx.feePayer`（未传 `feePayer` 时即用户钱包）。

两条路径的共同约定：

1. 需要收集的签名：**feePayer**（网络费 + ATA 租金）＋ **用户钱包**（兑换发起方 / 平台费扣款方）；`platformFee.from` 为第三地址时再加该地址。指令中 `isSigner=true` 的账户集合即签名依据。
2. 平台费扣款账户需已有足额代币（费用币 = 兑换输出代币时，输出在同链前序指令到账，天然满足）。
3. 广播与查询 RPC 统一带 `maxSupportedTransactionVersion: 1`；`getTransaction` 不带会报 `Transaction version (1) is not supported`。

### 优先费

**费用构成**：基础费（5000 lamports/签名）＋ 优先费（可选，拥堵时提升打包优先级）。v1 按声明总额收优先费，费用可预期；不传 `computeUnitPrice` = 只付基础费。

**关键约束**：优先费在服务端构建时内联锁定。拿到 `unsignedTransaction` 后**不能**再补一条 `SetComputeUnitPrice`（v1 拒绝 ComputeBudget 指令）。要调整费率只能重新调接口（带新的 `computeUnitPrice`），或走路径 B 自组交易。

**推荐两段式定价**：

```
① 探测（不广播）：dex_offlineSwapSol { simulate:true }（不传 computeUnitPrice）
     → simulateResult.consumedUnits、tx.computeBudget.computeUnitLimit
② 定价：computeUnitPrice = 目标优先费(lamports) × 10⁶ ÷ computeUnitLimit
     例：想付 100000 lamports、上限 115593 CU → ≈ 865000
③ 正式请求：带 computeUnitPrice 重调，校验 tx.computeBudget.priorityFeeLamports
     与 simulateResult.ok 后再签名广播
```

参考值：固定分档（空闲 `50000` / 正常 `100000` / 拥堵 `500000+` µL/CU）；或用标准 RPC `getRecentPrioritizationFees([热账户])` 取高分位换算。

### 完整示例

```bash
curl -s -X POST 'https://gwapi.ourdex.com/solana/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0","id":1,"method":"dex_offlineSwapSol",
    "params":[{
      "fromTokenAddress":"11111111111111111111111111111111",
      "toTokenAddress":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "amount":"100000000",
      "slippagePercent":"0.5",
      "userWalletAddress":"<用户钱包>",
      "simulate":true,
      "feePayer":"<代付地址>",
      "memo":"order-20260929-0001",
      "platformFee":{
        "to":"<平台收款地址>",
        "amount":"0.1",
        "contract":"Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB"
      }
    }]
  }'
```

响应（节选）：

```json
{
  "jsonrpc":"2.0","id":1,
  "result":{
    "instructionLists":[
      {"programId":"MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr","data":"b3JkZXIt…","accounts":[]},
      {"programId":"ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL","data":"AQ==",
       "accounts":[{"pubkey":"<代付地址>","isSigner":true,"isWritable":true}]},
      {"programId":"whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc","data":"…","accounts":[]}
    ],
    "createTokenAccountList":["<将创建的 ATA>"],
    "routerResult":{"fromTokenAmount":"100000000","toTokenAmount":"1523402","minReceiveAmount":"1515783","slippagePercent":"0.5","estimateGasFee":"5000","tradeFee":"0.00085"},
    "tx":{
      "from":"<用户钱包>","to":"",
      "minReceiveAmount":"1515783","slippagePercent":"0.5",
      "feePayer":"<代付地址>",
      "computeBudget":{"computeUnitLimit":67358,"loadedAccountsDataSizeLimit":67108864}
    },
    "wsolRentFee":2039280,
    "unsignedTransaction":"gQAAAAAAAAAA…",
    "transactionVersion":"v1",
    "executionEngine":"uxuy-router",
    "simulateResult":{"ok":true,"consumedUnits":61234}
  }
}
```

---

## 支持的 DEX

报价图按链上池数据自动收录，指令可执行范围覆盖 Solana 主流 AMM：

| 类别 | 说明 |
|---|---|
| 经典 SPL 池 | Raydium V4 / CPMM / CLMM、Orca Whirlpool、Meteora DLMM、PumpSwap、Pancake CLMM 等 |
| Token-2022 池 | 除 Raydium V4（仅经典 SPL）外，上述协议路径含 Token-2022 mint 时可执行，由 `simulate:true` 预执行兜底验证 |
| 曲线 / launchpad 池 | pump.fun 等 launchpad 曲线池纳入报价，毕业后自动切换到 AMM 池，业务方无需处理 |
| 仅报价、不可执行 | Orca legacy 等；此类路由会被自动跳过并换下一条候选 |

不需要为不同 DEX 做区分：接口只返回指令可执行的路由；全部候选都不可执行时返回明确的错误信息。

## 预执行失败与换路

`simulate:true` 预执行失败（`ok=false`）时：

1. 首选重新询价（池状态已变，旧报价的 `minReceiveAmount` 可能已不可达）；
2. 若反复在同一路由失败，用 `excludePoolAddresses` 排除问题池后重试：池地址可从 `instructionLists` 中对应 swap 指令的账户列表获取（AMM 的池账户通常为第一个可写账户），单次最多 20 个。

## 常见错误

| message | 触发条件 |
|---|---|
| `userWalletAddress required (instructions are wallet-specific)` | 兑换端点未传用户钱包 |
| `userWalletAddress invalid` / `feePayer invalid: <v>` | 非 base58 地址 |
| `amount must be a positive integer in smallest units, e.g. 1000000 (= 1 USDC)` | `amount` 非最小单位正整数 |
| `slippagePercent must be a decimal like 0.5 (=0.5%)` / `…within [0, 100)` | 滑点格式或范围错误 |
| `swapMode must be exactIn or maxIn` | `swapMode` 非法 |
| `chainIndex must be 501 (solana)` | `chainIndex` 传错 |
| `platformFee.to required` / `platformFee.to invalid: <v>` | 费用收款地址缺失或非法 |
| `platformFee.amount <金额> exceeds <N> decimals` | 费用金额精度超过代币精度 |
| `price impact 12.3456% exceeds the protection threshold 90.0000%` | 价格影响超保护阈值 |
| `no executable route: …` | 全部候选路由无指令支持或超 v1 交易约束 |
| `excludePoolAddresses supports at most 20 pools` | 排除池超过 20 个 |
