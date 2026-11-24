# Solana 离线询价 / 兑换指令接口对接文档

> 适用版本：`feature/solInstr` 分支（含 feePayer / memo / platformFee 支持，commit `5732773` 及以后）；`uxuy-router` 分支起默认全量经 uxuy-router 合约执行（无需参数）
>
| 接口 | 方法名 | 用途 |
|---|---|---|
| 询价 | `dex_offlineQuoteSol` | 纯报价：给定输入代币/数量，返回最优路由与预计产出（不构建指令） |
| 兑换指令 | `dex_offlineSwapSol` | 返回可直接组装签名的**指令链**（instructionLists），业务侧自行组包、签名、广播 |

整体流程：

```
dex_offlineQuoteSol（可选，展示行情）
        │
        ▼
dex_offlineSwapSol（组指令；建议 simulate:true 预执行验证）
        │
        ▼
业务侧：反序列化 unsignedTransaction（v1 交易）→ 部分签名 → 广播
        （或按 instructionLists + tx.computeBudget 自组 v1 交易）
```

---

## 1. 通用约定

- **传输协议**：JSON-RPC 2.0，`POST`，`Content-Type: application/json`。
- **URL 格式**：`https://<服务地址>/solana/<apiKey>`。`solana` 为链名（大小写不敏感），`apiKey` 走平台 JWT 鉴权。
- **地址格式**：Solana base58 字符串。
- **原生 SOL 的表示（重要）**：请求与返回中的原生 SOL 一律用 System Program 地址
  `11111111111111111111111111111111` 表示。
  - `fromTokenAddress` / `toTokenAddress` 传 `111…1` = 原生 SOL 语义：服务端会在指令链自动完成
    **wrap（SOL→wSOL）与 unwrap（wSOL→SOL）**，并附带 wSOL 临时账户的创建与关闭。
  - 传 `So11111111111111111111111111111111111111112`（wSOL mint）= **字面 wSOL 代币**语义：
    直接借记/记入用户的 wSOL ATA 余额，**不做** wrap/unwrap。两者不可混用。
- **数量单位**：除 `platformFee.amount` 外，所有金额字段（`amount`、`fromTokenAmount`、
  `toTokenAmount`、`minReceiveAmount`）均为**最小单位整数字符串**（OKX 语义），
  如 USDC(6 位精度) 的 1 枚 = `"1000000"`。
- **金额基准**：报价完全基于服务端本地扫描的链上状态（离线引擎，零链上询价），存在正常的快照时差；
  生产建议开启 `simulate:true` 做执行前验证。
- **错误形态**：JSON-RPC 标准错误对象 `{"error":{"code":-32000,"message":"…"}}`，
  message 会点名出错的参数（见 [第 7 节](#7-常见错误)）。

请求信封示例：

```json
{"jsonrpc":"2.0","id":1,"method":"dex_offlineSwapSol","params":[{ …参数对象… }]}
```

---

## 2. `dex_offlineQuoteSol` — 询价

### 2.1 请求参数

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `chainIndex` | string | 否 | 链编号，Solana 固定 `"501"`（兼容值 `"6000001"`）；缺省按 URL 链名。传其他值报错 |
| `fromTokenAddress` | string | **是** | 支付代币 mint（base58）；原生 SOL 传 `111…1` |
| `toTokenAddress` | string | **是** | 收取代币 mint（base58）；原生 SOL 传 `111…1` |
| `amount` | string | **是** | 兑换数量，**最小单位**整数字符串（`"1000000"` = 1 USDC）。`swapMode=exactIn` 时为输入数量；`maxIn` 时为期望产出数量 |
| `slippagePercent` | string | 否 | 滑点百分比字符串，`"0.5"` = 0.5%，范围 [0, 100)；缺省由引擎智能推荐（见返回 `suggestSlippagePercent`）。显式传入时是硬上限，偏差守卫不会抬高它，只会丢弃超出该容忍度的路由 |
| `userWalletAddress` | string | 否 | 用户钱包地址；询价阶段仅做地址路由参考，非必需 |
| `swapMode` | string | 否 | `exactIn`（默认，按输入定量）/ `maxIn`（按产出定量，反推输入） |
| `autoSlippage` | bool | 否 | 开启自动滑点（引擎按行情推荐并应用）。偏差守卫发现报价相对参考价偏高、原滑点下的 `minReceiveAmount` 不可达时，会在 `maxAutoSlippagePercent` 内抬高**实际生效**的滑点并重算最小到账 |
| `maxAutoSlippagePercent` | string | 否 | 自动滑点的上限百分比；缺省不设上限（守卫最多抬到覆盖参考价偏差所需的容忍度，仍受 30% 慷慨偏差弃路由线约束） |
| `priceImpactProtectionPercent` | string | 否 | 价格影响保护阈值百分比，默认 `90`，`100` = 关闭。超阈值直接报错拒绝 |
| `dexIds` | string | 否 | 允许的 DEX 程序白名单，逗号分隔（强制路由到指定 DEX 时用；**必须配合 `size` ≥ 10**，否则默认只取 3 条候选路由会把指定 DEX 截掉） |
| `excludeDexIds` | string | 否 | 排除的 DEX 程序黑名单，逗号分隔 |
| `size` | int | 否 | 候选路由数量（默认 3；配合 `dexIds` 排查时建议 ≥10） |
| `excludePoolAddresses` | string | 否 | 排除的池地址，逗号分隔，最多 20 个（swap 端点预执行失败后换路重试的核心机制） |
| `directRoute` | bool | 否 | 只走单池直达路由 |

**兼容但当前不生效的参数**（静默接受，勿依赖）：`useTokenLedger`、`positiveSlippagePercent`、
`positiveSlippageFeeAddress`、`feePercent`、`referrerAddress`、
`closeAuthorityAddress`、`maxCalldataSize`、`maxAccounts`、`forJitoBundle`、`disableRFQ`、
`singleRouteOnly`、`singlePoolPerHop`、`assetAwareRouting`。
（`swapReceiverAddress` 已实现生效，见 3.2 参数表。）

### 2.2 返回字段（result）

最优路由**平铺**在 result 顶层（OKX /quote 形状），`routerList` 内为全部候选路由（相同结构）。

| 字段 | 类型 | 说明 |
|---|---|---|
| `chainIndex` | string | 固定 `"501"` |
| `swapMode` | string | 本次报价的模式（`exactIn` / `maxIn`） |
| `fromToken` | object | 支付代币信息，结构见下表 |
| `fromTokenAmount` | string | 支付数量（最小单位；原生 SOL 为 lamports） |
| `toToken` | object | 收取代币信息，结构见下表 |
| `toTokenAmount` | string | 预计收到数量（最小单位） |
| `priceImpactPercent` | string | 价格影响百分比，`"0.19"` = 0.19% |
| `router` | string | 路由摘要（经过的代币序列） |
| `dexRouterList` | array | 逐跳明细，每项含 `dexProtocol{dexName, percent}`（DEX 名与占比）、`fromToken`/`toToken`（代币信息）、`fromTokenIndex`/`toTokenIndex`（该跳在路径中的序号） |
| `minReceiveAmount` | string | 按实际生效滑点折算的最小应收到数量（最小单位；`exactIn` 模式） |
| `slippagePercent` | string | 本次实际生效的滑点百分比（未显式传入时为引擎智能值；偏差守卫抬升后为新值） |
| `suggestSlippagePercent` | string | 引擎推荐的滑点百分比（扩展字段；显式滑点或守卫抬升后可能高于 `slippagePercent`） |
| `routerList` | array | 全部候选路由，每项结构与上表完全相同（扩展字段） |

`fromToken` / `toToken`（及 `dexRouterList[].fromToken/toToken`）子结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `tokenContractAddress` | string | 代币 mint 地址（若请求侧用 `111…1`，端点处回显 `111…1`；中间跳恒为链上真实 mint） |
| `tokenSymbol` | string | 代币符号（本地库缺失时可能为空） |
| `tokenUnitPrice` | string | 代币美元单价（本地库缺失时为空） |
| `decimal` | string | 精度位数 |
| `isHoneyPot` | bool | 蜜罐标记（Solana 侧恒 `false`） |
| `taxRate` | string | 转账税率（Solana 侧恒 `"0"`） |

> 询价端点**不构建指令**，`estimateGasFee` / `tradeFee` / `contextSlot` 不返回。

---

## 3. `dex_offlineSwapSol` — 兑换指令

### 3.1 请求参数

继承 [2.1 询价参数](#21-请求参数) 全部字段（`userWalletAddress` 在本端点**必填**：指令是钱包维度的），另加：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userWalletAddress` | string | **是** | 用户钱包地址（base58）。指令链中的 ATA、swap authority 均按此地址构建 |
| `computeUnitPrice` | string | 否 | 优先费单价，**micro-lamports / CU**（如 `"100000"`）；换算为 v1 内联配置里的**总额 lamports** 优先费（按生效 CU 上限），回显在 `tx.computeBudget.priorityFeeLamports`；`0`/缺省 = 只付基础费 |
| `computeUnitLimit` | string | 否 | 计算单元上限（CU）。v1 消息无上限 = 0 CU 必败，故恒显式解析：显式值 > 模拟注入的紧凑值（消耗+10%，`simulate:true` 时）> 默认预算（每指令 20 万 CU 封顶 140 万）；回显在 `tx.computeBudget.computeUnitLimit` |
| `simulate` | bool | 否 | 建议 `true`：服务端对组装后的完整指令链做 simulateTransaction 预执行，结果附在 `simulateResult`。**这是执行正确性的门禁**，预执行失败的业务侧不应广播 |
| `feePayer` | string | 否 | **代付地址**：接管全部 ATA 创建（租金）的 funder；服务端尺寸检查与模拟也以它为交易 payer。缺省 = `userWalletAddress`。返回 `tx.feePayer` 回显——业务组包时必须以它为 `TransactionPayer` 并收集其签名 |
| `swapReceiverAddress` | string | 否 | **代收地址**（A 兑给 B）：末跳输出侧代币账户（含其幂等创建，funder = feePayer）归属接收方 B；签名者/输入侧/中间跳/平台费仍在钱包 A。接收方无需签名。回显在 `tx.to`。**原生 SOL 输出（`111…1`）同样支持**：路由改用**从钱包 A 派生的一次性 wSOL 临时账户**（with-seed：地址 = sha256(A‖seed‖TokenProgram)，CreateAccountWithSeed 以 **A 自己的签名**授权建户——零额外签名者，`instructionLists` 可照常自组；owner=A、末跳 wSOL 落入其中、链尾 CloseAccount 解包直付 B，金额动态=实际全部产出+账户租金）。**A 自己的 wSOL 持仓完全无关**（链路不碰 A 的 wSOL 账户，无需清空）。seed 由服务端每请求随机生成（同 A 并发在途交易互不冲突）。两点口径：临时账户租金随解包归 B 不退还；SOL 计价 platformFee 从钱包自有 SOL 扣（解包余额已归 B） |
| `memo` | string | 否 | 交易备注：生成 SPL Memo 指令（`MemoSq4gq…` 程序，UTF-8，自动去首尾空白），置于 compute-budget 指令之后。若手续费代币是 Token-2022，会在费用转账前**紧邻再插一份** memo（memo-transfer 扩展的接收账户只认紧邻前置的备注） |
| `platformFee` | object | 否 | **平台手续费**，结构见下表；整个链尾追加一笔费用转账（在 wSOL 解包之后，SOL 计价的费用此时才有余额；`swapReceiverAddress` + 原生 SOL 输出时解包余额归接收方，SOL 计价费用改从钱包自有 SOL 扣，钱包需保有足额 lamports） |

`platformFee` 子对象：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `to` | string | **是**（当费用生效时） | 平台收款地址（base58 钱包地址） |
| `amount` | string | **是** | 手续费数量，**可读金额**（非最小单位！）：0.1 USDT 传 `"0.1"`。`≤ 0` 时整个费用块静默跳过（此情况下 `to` 缺失也不报错）；金额精度超过代币精度（如 `"0.0000001"` USDT @6 位）报错 |
| `contract` | string | 否 | 手续费代币 mint。缺省 / `"SOL"` / `111…1` / wSOL mint = **原生 SOL**（链尾追加一笔 System 转账，单位 lamports）。传其他 mint 则做 SPL/Token-2022 代币转账 |
| `from` | string | 否 | 手续费扣款地址，缺省 = `userWalletAddress`。**设为第三地址时该地址也必须签名**（其代币账户 owner 是 signer） |

手续费代币支持范围：

- **经典 SPL Token**（USDT/USDC 等）：普通 transfer，收款方 ATA 缺失时自动幂等创建（funder = feePayer）。
- **Token-2022**：完整支持——有 transfer-fee 扩展的 mint 自动使用 `TransferCheckedWithFee` 并按当前
  epoch 计算链上将收取的费用（向上取整、封顶 maximumFee）；有 transfer-hook 的 mint 自动解析并附加
  hook 所需的额外账户。
- 交换路径本身仍仅支持经典 SPL 代币（Token-2022 池返回 no route，属预期）。

### 3.2 返回字段（result）

| 字段 | 类型 | 说明 |
|---|---|---|
| `instructionLists` | array | **待组装的指令链**，按执行顺序排列，结构见下。v1 交易形态：**不含 ComputeBudget 指令**（v1 拒绝指令形式，预算走消息内联配置，见 `tx.computeBudget`），自行组装时依序拼进 v1 交易即可 |
| `createTokenAccountList` | array | 本交易将创建的账户地址列表（幂等创建的 ATA：路由侧与平台费收款侧；以及 `swapReceiverAddress` + 原生 SOL 输出时的一次性 wSOL 临时账户——非 ATA、CreateAccountWithSeed 派生自钱包）；租金均由 `feePayer` 承担 |
| `routerResult` | object | 最优路由的报价信息，结构与 [2.2 询价返回](#22-返回字段result) 相同，另加两个字段↓ |
| `routerResult.estimateGasFee` | string | 预估网络费（lamports 字符串）= 5000 基础费 + 生效 CU 上限 × computeUnitPrice / 10⁶ |
| `routerResult.tradeFee` | string | 预估网络费的美元价值（按库存 SOL 价格，保留 8 位） |
| `tx` | object | 交易摘要：`from`（用户钱包）、`to`（= `swapReceiverAddress` 回显，未设时为空）、`minReceiveAmount`、`slippagePercent`、`feePayer`（**组包 payer 依据，扩展字段**）、`computeBudget`（**v1 内联预算，扩展字段**，见下） |
| `wsolRentFee` | uint64 | 路由含原生 SOL（wrap/unwrap）时的 wSOL 临时账户租金，`2039280` lamports；不涉原生 SOL 时为 `0`。租金刚好在交易结束时随账户关闭**退还给用户**，此项仅供展示（`swapReceiverAddress` + 原生 SOL 输出时除外：租金随解包归接收方，不退还） |
| `unsignedTransaction` | string | **已组装、待签名广播的完整交易**（base64，**v1 wire 格式**，SIMD-0385）：与 `instructionLists` 同一指令链，嵌入最新 finalized recentBlockhash、按必需签名者数量预留的零签名槽、内联 compute 预算，payer = `tx.feePayer`。反序列化为 v1 交易后对零签名槽（自己的签名账户）部分签名即可广播。**签名要及时**——内置 blockhash 约 1 分钟后过期。回单查询：`getTransaction` 需传 `maxSupportedTransactionVersion: 1`（默认 0 会报 "Transaction version (1) is not supported"） |
| `transactionVersion` | string | 恒为 `"v1"`：`unsignedTransaction` 及按 `instructionLists` 自组交易的 wire 格式版本（反序列化时按此选类型） |
| `routerList` | array | 全部候选路由（扩展字段） |
| `simulateResult` | object | `simulate:true` 时返回的预执行结果，结构见下 |

`instructionLists[]` 指令结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `programId` | string | 目标程序地址（base58） |
| `data` | string | 指令数据，**base64** 编码 |
| `accounts` | array | 账户元信息列表，顺序即传参顺序：`{pubkey: 地址, isSigner: 是否需签名, isWritable: 是否可写}` |

`tx.computeBudget` 子结构（v1 内联 compute 预算——自行组装 v1 交易时必须放进消息配置，指令形式会被 v1 拒绝）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `computeUnitLimit` | uint32 | CU 上限。v1 消息不带上限 = **0 CU（交易必败）**，所以服务端恒显式解析：请求显式值 > 模拟注入的紧凑值（消耗+10%）> 默认预算（每指令 20 万 CU 封顶 140 万） |
| `priorityFeeLamports` | uint64 | 优先费**总额**（lamports）。由请求的 `computeUnitPrice`（microLamports/CU）按生效 CU 上限换算；`0`/缺省 = 只付基础费 |
| `loadedAccountsDataSizeLimit` | uint32 | 加载账户数据上限（字节）。v1 不设 = 0 字节（集群直接拒 `MaxLoadedAccountsDataSizeExceeded`），服务端恒设 64MiB（与 legacy 默认一致） |

`simulateResult` 结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `ok` | bool | `true` = 链按当前构造可执行成功 |
| `err` | string | 失败时的错误摘要（含程序日志定位）；模拟 RPC 本身故障也会写在这里 |
| `logs` | array | 完整模拟日志（失败排查看尾部 `Program log: Error` 行） |
| `consumedUnits` | uint64 | 模拟消耗的计算单元 |
| `appliedUnitLimit` | uint64 | 服务端自动注入的 CU 上限（仅当请求未显式传 `computeUnitLimit` 且生效时返回；以 v1 内联配置形式生效，回显在 `tx.computeBudget`） |

### 3.3 指令链结构（业务侧组装参考）

服务端产出的顺序即执行顺序，典型形态：

```
（compute 预算不是指令——在 v1 消息内联配置里，见 tx.computeBudget）
[memo]                                          ← memo 非空时
[CreateIdempotent × N]                          ← 缺失的 ATA（funder = feePayer）
[CreateAccountWithSeed + InitializeAccount3]     ← 仅 swapReceiverAddress + 原生 SOL 输出：一次性 wSOL 临时账户（with-seed 派生自钱包、base=钱包签名授权建户零额外签名者、租金由 feePayer 垫、owner=钱包）
[System.Transfer + SyncNative]                  ← 仅 fromToken 为原生 SOL（把本金 wrap 进 wSOL ATA）
[swap 指令 × 跳数]
[CloseAccount]                                  ← 仅路由触及原生 SOL（解包 + 退租金；swapReceiverAddress + 原生 SOL 输出时，解包本金与租金直付接收方）
[CreateIdempotent × 2]                          ← 仅 SPL 平台费：收款方/扣款方 ATA 缺失时
[memo]                                          ← 仅 Token-2022 手续费 + memo 非空（紧邻转账）
[平台费转账]                                     ← System.Transfer（原生 SOL）或 Token/Token-2022 转账
```

### 3.4 优先费：定价、内联与修改约束

**费用构成**：基础费（5000 lamports/签名，恒有）＋ 优先费（可选，拥堵时提升打包优先级）。
v1 按声明的总额收优先费（不按实际消耗），费用完全可预期；不传 `computeUnitPrice` = 只付基础费。

**关键约束：优先费在服务端构建时内联锁定。** v1 交易拒绝 ComputeBudget 指令——拿到
`unsignedTransaction` 后**不能**像 legacy 那样再补一条 `SetComputeUnitPrice`；`tx.computeBudget`
回显的就是交易里实际内嵌的值。要调整费率只能：重新调接口（带新 `computeUnitPrice`），或走
路径 B 用 `instructionLists` 自组（换成自己的内联配置）。

**推荐两段式定价流程**：

```
① 探测（不广播）：dex_offlineSwapSol { simulate:true }（不传 computeUnitPrice）
     → simulateResult.consumedUnits（实际消耗）
     → tx.computeBudget.computeUnitLimit（生效上限）
② 定价：computeUnitPrice = 目标优先费(lamports) × 10⁶ ÷ computeUnitLimit
     例：想付 0.0001 SOL（100 000 lamports）、上限 115 593 CU → ≈ 865 000
③ 正式请求：带 computeUnitPrice 重调 → 校验 tx.computeBudget.priorityFeeLamports
     与 simulateResult.ok → 签名广播（见组包契约）
```

**优先费参考值的获取方式**：

| 方式 | 说明 |
|---|---|
| 固定分档 | 空闲 `50000` / 正常 `100000` / 拥堵 `500000+`（µL/CU；12 万 CU 链约合 0.000006 / 0.000012 / 0.00006+ SOL） |
| 标准 RPC | `getRecentPrioritizationFees([热账户])` 取高分位 → 按上式换算 µL/CU（无需第三方） |
| Helius | `getPriorityFeeEstimate`（priorityLevel 或 recommended；带短缓存，参考业务侧 sol.js 实现） |

**组包与签名契约**（v1-only；两条路径二选一，结果完全一致）：

交易为 **v1 格式（SIMD-0385）**：消息首字节 `0x81`，集群约束为序列化 ≤ **4096 字节**、
去重账户 ≤ **64**、指令 ≤ **64**、签名 ≤ **12**（服务端已按同一约束组链换路，正常无需关心）；
compute 预算内联在消息配置（**不接受 ComputeBudget 指令**）。

- **路径 A（推荐）——直接用返回的交易**：`unsignedTransaction`（base64）按 v1 反序列化，
  对全部签名账户部分签名后广播：

  ```js
  // 需要 web3.js 版本支持 SIMD-0385 v1 反序列化（1.x 未证实时先验证或用支持库）
  const tx = VersionedTransaction.deserialize(Buffer.from(res.unsignedTransaction, 'base64'));
  tx.sign([walletKeypair]);                    // 用户钱包（swap authority / 平台费 owner）
  if (feePayerKeypair) tx.sign([feePayerKeypair]); // 代付地址（网络费 + ATA 租金）
  // platformFee.from 为第三地址时还需其签名
  await connection.sendRawTransaction(tx.serialize(), {maxSupportedTransactionVersion: 1});
  ```

- **路径 B——按 `instructionLists` 自行组装 v1 交易**：
  1. 指令按 `instructionLists` 原序；
  2. 自取 `latestBlockhash` 作为消息 lifetime；
  3. `tx.computeBudget` 放进消息内联配置（不要生成 ComputeBudget 指令）；
  4. payer = `tx.feePayer`（未传 `feePayer` 时即用户钱包）。

两条路径共同的约定：

1. 需要收集的签名：**feePayer**（网络费 + ATA 租金）＋ **用户钱包**（swap authority / 平台费 owner）；
   `platformFee.from` 为第三地址时再加该地址。指令里 `isSigner=true` 的账户集合即签名依据。
2. 平台费转出的代币余额由用户承担：扣款 ATA 中需已有足额代币（费用币 = 兑换输出代币时，输出
   恰好在同链前序指令到账，天然满足）。
3. 广播/查询 RPC 需带 `maxSupportedTransactionVersion: 1`。

---

## 4. 示例

### 4.1 询价

```bash
curl -s -X POST 'https://<服务地址>/solana/<apiKey>' \
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

返回（节选）：

```json
{
  "jsonrpc":"2.0","id":1,
  "result":{
    "chainIndex":"501",
    "swapMode":"exactIn",
    "fromToken":{"tokenContractAddress":"11111111111111111111111111111111","tokenSymbol":"SOL","decimal":"9"},
    "fromTokenAmount":"100000000",
    "toToken":{"tokenContractAddress":"EPjFWdd5…","tokenSymbol":"USDC","decimal":"6"},
    "toTokenAmount":"1523402",
    "priceImpactPercent":"0.021",
    "minReceiveAmount":"1515783",
    "slippagePercent":"0.5",
    "suggestSlippagePercent":"0.1",
    "dexRouterList":[{"dexProtocol":{"dexName":"raydium  clmm","percent":"100"}}],
    "routerList":[ …同结构候选路由… ]
  }
}
```

### 4.2 兑换指令（feePayer + memo + USDT 平台费 0.1）

```bash
curl -s -X POST 'https://<服务地址>/solana/<apiKey>' \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0","id":1,"method":"dex_offlineSwapSol",
    "params":[{
      "fromTokenAddress":"11111111111111111111111111111111",
      "toTokenAddress":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "amount":"100000000",
      "slippagePercent":"0.5",
      "userWalletAddress":"HRF4WPP5…用户钱包…",
      "simulate":true,
      "feePayer":"9h17Nry5…代付地址…",
      "memo":"order-20260929-0001",
      "platformFee":{
        "to":"Gh9ZwEmd…平台收款地址…",
        "amount":"0.1",
        "contract":"Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB"
      }
    }]
  }'
```

返回（节选）：

```json
{
  "jsonrpc":"2.0","id":1,
  "result":{
    "instructionLists":[
      {"programId":"MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr","data":"b3JkZXIt…","accounts":[]},
      {"programId":"ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL","data":"AQ==",
       "accounts":[{"pubkey":"9h17Nry5…代付地址…","isSigner":true,"isWritable":true}, …]},
      {"programId":"whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc","data":"…","accounts":[ …15 个账户… ]},
      {"programId":"Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB 的 Token 程序","data":"…","accounts":[ … ]}
    ],
    "createTokenAccountList":["…将创建的 ATA…"],
    "routerResult":{ …同询价结构 + estimateGasFee/tradeFee… },
    "tx":{
      "from":"HRF4WPP5…",
      "to":"",
      "minReceiveAmount":"1515783",
      "slippagePercent":"0.5",
      "feePayer":"9h17Nry5…",
      "computeBudget":{"computeUnitLimit":67358,"priorityFeeLamports":6735}
    },
    "wsolRentFee":2039280,
    "unsignedTransaction":"gQAAAAAAAAAA…（base64，v1 待签名完整交易，首字节 0x81）",
    "transactionVersion":"v1",
    "simulateResult":{"ok":true,"consumedUnits":61234,"appliedUnitLimit":67358}
  }
}
```

---

## 5. 支持的 DEX（指令可执行）

**Token-2022 代币**：以下协议的路径含 Token-2022 mint 时同样可执行（per-mint 程序感知构建）——Orca Whirlpool、Raydium CLMM、Pancake CLMM（swap_v2）、Meteora DLMM（仅 exactIn）、PumpSwap（buy_exact_quote_in 已带 pump_amm 2026-09 IDL 的 track_volume 参数，None 语义）、Raydium CPMM、Byreal。**仅 Raydium V4 限经典 SPL**（单共享 token 程序槽的程序设计限制）。带 transfer-fee/hook 扩展的 T2022 mint 在 swap 腿由 AMM 程序内部处理，`simulate:true` 预执行门验证。

报价图按后台 DeFiSwapFactory 配置收录。仅报价不可执行的：Orca legacy。指令不支持的路由会被跳过并换下一条候选。

**pump.fun bonding curve（2026-10 起）**：曲线池（6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P，Protocol=pumpfun）毕业后即出图（CompleteEvent 置位 curveComplete），报价走 v2 常数积 + TradeEvent 实测的方向费率。执行侧合约新增两个 Dex（buy=15 `buy_exact_sol_in`、sell=16 `sell`，需合约升级部署后生效）：
- **原生 SOL 桥接**：曲线以原生 SOL 结算。buy 腿合约先 close 一次性 wSOL 临时账户换出原生 SOL（PDA 场景由 payer 预补租金差、事后扫回），sell 腿合约把收到的原生 SOL 精额转入 wSOL 目标账户并 sync。因此 **curve-buy 的 wSOL 中间账户强制走一次性临时账户**（即使托管已预热）——compose 已特例处理；显式 wSOL 输入进 curve-buy 报 `ErrFallback`（会 close 钱包常驻 wSOL ATA）。
- **buy 尾部账户**：必须带 2 个 buyback 槽（第二槽须为授权的 pump-fees 金库，两个槽都传钉死的 5YxQFdt…）；sell 无尾部账户。track_volume 缺省（None，不建 user_volume_accumulator，无租金成本）。
- **fee_recipient 按币授权**：钉死 pump 默认 7VtfL8…；自设分润配置的币（sharing-config）会拒绝它（6000 NotAuthorized），该路由在预执行门被淘汰并换次优。注册 uxuy 自己的 referrer 可解锁全部币并捕获返佣。
- **v2 代币报价曲线（USDC/USDT quote）已可执行（Dex 17/18，2026-10 第二批）**：`buy_exact_quote_in_v2`/`sell_v2` 走 token 结算（无原生 SOL 桥接）。quote 侧用户槽是 **hop authority 的 canonical ATA**（pump 校验）：hop≥1 的 quote 中间账户强制走 PDA 的常驻 ATA（幂等创建、租金一次性摊销，**托管账户不可用于 v2 quote 腿**）；base 侧仅校验 owner（托管/临时账户合法）。**v2 sell 形状受限**：只能是 hop0 末跳（无 receiver）或中间跳——hop≥1 末跳禁止（PDA 的 ATA ≠ 用户收款账户），带 receiver 报 ErrFallback。显式 wSOL 输入进 hop0 曲线 buy 现在路由 v2（此前 ErrFallback）。
- 曲线币多为 Token-2022：CPI 槽按 per-mint 程序填（T2022 已支持）。

**PumpSwap 费率精度（2026-09-30 起）**：pump_amm（pAMMBay6oce…）对链上实际收费与定价做了逐池对齐——名义费率 lp 20 + protocol 5 + creator 15 = 40bps（注册表默认/新池兜底），另由链上 swap 事件实时反解每池、每方向的**有效费率**（含 AMM 内部定价相对金库余额 ~0.6% 的结构性收缩）写入池 Extra，询价按方向取用。池子首个事件到达前按 40bps 兜底，随后自动校准；无需业务侧配置。

---

## 6. 预执行失败重试（换路机制）

`simulate:true` 预执行失败时，取 `routePools`（指令链中 swap 程序的池账户）填入
`excludePoolAddresses` 重新调用 `dex_offlineSwapSol` 即可排除坏路换下一条（上限 20 个池）。

---

## 6a. 运营配置：合约身份与托管账户（gdexUI）

以下配置均在 gdexUI → 链配置 → Swap 页，保存后**热生效**（引擎在列表/KV 缓存过期后
的下一次询价同步，最长约 60 秒，无需重启）；solana 链保存时即校验 contract/authority
成对匹配，不匹配直接报错拒绝保存：

| gdexUI 字段 | 存储 key | 说明 |
|---|---|---|
| UxuySwap Contract | `contract` | uxuy-router 程序 id（base58）。Solana 链填 base58；EVM 链仍为 hex 地址 |
| UxuySwap Authority PDA | `authority` | 签中间跳/费用的 authority PDA = PDA(`["uxuy_sa"]`, 程序id)。**仅 Solana 链显示**，与程序 id 成对校验（不匹配整对拒绝，原身份保持） |

Solana 链上这两行**共用一个保存按钮**（在 Authority PDA 行）整对提交：换合约就两个字段
一起贴、一起存，后端先校验配对再落库，不存在"半新半旧"的中间态。清空两字段保存 = 回退
编译内置默认身份。
| Swap Intermediary Tokens → custodyAccount | 每行代币 | 该中间币的托管账户（SPL owner 须为 authority PDA） |

两者都未配置（或都已清空）时使用编译内置的当前主网身份（`AN6zjJfT…` / `5rvVfdVC…`）。
**更换合约
（新程序 id）时**：改这两个字段即可切换引擎侧指向，但托管账户的 owner 绑定旧 PDA ——
需先用旧合约清空/关闭旧托管账户回收租金，再按新 authority 预热新托管账户并更新
custodyAccount 配置。

---

## 7. 常见错误

| message | 触发条件 |
|---|---|
| `userWalletAddress required (instructions are wallet-specific)` | swap 端点未传用户钱包 |
| `userWalletAddress invalid` / `feePayer invalid: <v>` | 非 base58 地址 |
| `amount must be a positive integer in smallest units, e.g. 1000000 (= 1 USDC)` | amount 非正整数最小单位 |
| `slippagePercent must be a decimal like 0.5 (=0.5%)` / `…within [0, 100)` | 滑点格式/范围错误 |
| `swapMode must be exactIn or maxIn` | swapMode 非法 |
| `chainIndex must be 501 (solana)` | chainIndex 传错 |
| `platformFee.to required` / `platformFee.to invalid: <v>` | 费用收款地址缺失/非法（费用生效时） |
| `platformFee.amount invalid: <v>` | 费用金额不是数字 |
| `platformFee.amount <金额> exceeds <N> decimals` | 费用金额精度超过代币精度（如 0.0000001 USDT） |
| `platformFee.contract invalid: <v>` / `platform fee mint <m> not found on chain` | 费用代币地址非法/链上不存在 |
| `price impact 12.3456% exceeds the protection threshold 90.0000%` | 价格影响超保护阈值 |
| `no executable route: quotes only touch programs without instruction support […]` | 全部路由无指令支持 |
| `no executable route: N non-SPL-mint, N over v1 tx limits (4096B/64 addr/64 ix/12 sig), N failed to build (last: …)` | 候选路由分别因非 SPL 代币/超 v1 交易约束（4096 字节、64 账户、64 指令）/构建失败被淘汰 |
| `dexIds contains factory not configured in the registry: <id>` / `all routes excluded by dex filter` | DEX 过滤后无路由（注意配合 `size`） |
| `excludePoolAddresses supports at most 20 pools` | 排除池超上限 |
> **合约执行（唯一路径）**：`dex_offlineSwapSol` 的交易**只**经 uxuy-router 合约（`AN6zjJfTD5Cpj2x56sWsAod1Jb5viiQH4Xcg3tm5DgNY`，主网/devnet 同址）组装——整条路由为一条合约指令，余额差 + `min_return` 整单原子兜底；响应新增 `executionEngine: "uxuy-router"` 字段自描述。中间跳代币走预热的全局托管账户（无每笔建户/租金垫付；未预热的 mint 自动退回每笔临时账户形态）。`swapReceiverAddress` 含原生 SOL 输出：**接收人获得实际产出（不含租金），临时账户租金退还付款人 A**，钱包自身 wSOL 持仓不受影响。边界：若接收地址余额低于租金豁免线，合约助手会将其补足到豁免线（补足部分由付款人承担，原始 V1 行为）——余额豁免的接收地址（常态）恰好获得产出。
>
> **无静默回退**（产品决策 2026-10-13）：合约暂不支持程序（pump_amm 尾部账户、byreal 动态费池）的路由在候选阶段剔除，报价退回次优可执行路由；全部候选不可执行时显式报错 `no executable route: quotes only touch programs without instruction support [...]`。费用币为 Token-2022 等合约路径拒绝的形态同样显式报错（错误信息来自 `router.ErrFallback`），不再静默换用逐腿路径。
