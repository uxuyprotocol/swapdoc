# EVM 询价与交易

本页介绍 EVM 链的兑换接口：`dex_offlineQuote`（单对询价）、`dex_offlineQuotes`（批量询价）与配套的授权、发交易流程。

接口返回**可执行的路由**（含目标合约与 calldata），业务方自行授权、签名、广播。路由基于服务端本地池状态计算；本地无路由或数据不满足条件时自动回落链上询价，业务方无需区分两种来源。

```
dex_offlineQuote(withTx=true)
      │  result[]：按 amount 降序
      ▼
取 result[0] ──▶ tokenIn 是 ERC20？
      │                  ├─ 是：approve(calldata.to, ≥ amountIn)
      │                  └─ 否：无需授权
      ▼
发送交易 { to: calldata.to, data: calldata.data, value: calldata.value }
      │
      ▼
等待回执，核对到账
```

## 接口信息

| 项目 | 值 |
|---|---|
| 方法名 | `dex_offlineQuote` |
| 请求方式 | `POST` |
| 请求地址 | `https://gwapi.ourdex.com/<chain>/<appId>` |
| 鉴权 | `Authorization: Bearer <JWT>` |
| Content-Type | `application/json` |
| 适用链 | EVM 链（`bnbchain`、`ethereum`、`base`、`xlayer`、`anubis` 等） |

## 请求参数

`params` 为只含一个对象的数组：`params: [{ ... }]`。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `sender` | string | 是 | 交易发起地址（实际签名地址）。合约按 `msg.sender` 校验余额与授权 |
| `receiver` | string | 否 | 收款地址，缺省 = `sender` |
| `tokenIn` | string | 是 | 卖出代币地址；原生币传零地址 `0x0000…0000` |
| `tokenOut` | string | 是 | 买入代币地址；原生币传零地址 |
| `amountIn` | string | 二选一 | 卖出数量，人类可读十进制字符串（`"1"` = 1 枚，`"0.5"` = 0.5 枚） |
| `amountOut` | string | 二选一 | 买入数量（反向询价）：按目标产出反推所需输入，返回值中 `amount` 为所需 `tokenIn` 数量。仅 `withTx=false` 时支持 |
| `slippage` | int | 否 | 滑点，单位 **1e-6**（`5000` = 0.5%）。显式传入时按该值计算 `minAmount`，接口不会自动抬高 |
| `decimals` | int | 否 | 保留字段，离线接口不生效（金额始终按 token 精度换算） |
| `size` | int | 否 | 返回路由数量上限，取值 1–10，默认 3 |
| `orderId` | string | 否 | 业务方订单号。仅支持十进制数字串，会编入 calldata 的 `orderId` 字段（非数字按 `0` 处理）；本接口响应不回显 |
| `withTx` | bool | 否 | `true`：返回 calldata 并额外做一次 `eth_call` 预执行，回写 `executed` / `error`；`false`（默认）：返回 calldata 但不预执行，`executed` 恒为 `false` |
| `chainIn` / `chainOut` | int | 否 | 本接口不支持跨链：传入与当前链不同的 chainId 会直接报错 |
| `extraSlippage` | int | 否 | 保留字段，离线接口不生效 |
| `noFee` | bool | 否 | 手续费口径开关，默认 `false`，一般无需传入 |
| `usdIn` / `usdOut` / `bridgeFee` / `bridgeRouter` / `bridgeAgent` / `fsReceiver` | — | 否 | 跨链与分佣字段，离线接口不生效 |

> `slippage` 不传或传 `0` 时使用引擎推荐值（恒在 0.5%–5% 之间），实际生效值见响应 `slippage`。业务方一般应显式传入。

## 请求示例

```bash
curl -s -X POST 'https://gwapi.ourdex.com/bnbchain/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0","id":1,"method":"dex_offlineQuote",
    "params":[{
      "sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0",
      "receiver":"0xDd841835e929FE458948aD31Ba43B6239c2912a0",
      "tokenIn":"0x55d398326f99059fF775485246999027B3197955",
      "tokenOut":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
      "amountIn":"1",
      "slippage":5000,
      "size":2,
      "withTx":true
    }]
  }'
```

## 响应参数

`result` 为路由数组：正向询价按 `amount` 降序；反向询价（`amountOut`）的 `amount` 是所需输入数量，按升序排列。本地与在线均无路由时为空数组。

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 路由 ID |
| `paths` | array | 兑换路径（`tokenIn` → 中间 token（可为空）→ `tokenOut`），结构见 [`paths[]`](#paths) |
| `pairs` | array | 路径中各跳的池信息，顺序与路径一致，结构见 [`pairs[]`](#pairs) |
| `dexList` | array | 各跳命中的 DEX 信息，结构见 [`dexList[]`](#dexlist) |
| `amount` | int | 最终可兑换的 `tokenOut` 数量，**未除精度**；反向询价时为所需 `tokenIn` 数量 |
| `minAmount` | int | 最小兑换数量，未除精度，由 `amount` 与生效滑点推出，链上低于该值整笔 revert |
| `slippage` | int | 本次生效滑点，单位 1e-6 |
| `suggestSlippage` | int | 引擎建议滑点，单位 1e-6。当路由实际偏差超过生效滑点但仍可执行时会大于 `slippage`，可参考后放宽 |
| `price` | number | 币价：按 token 精度换算成人类单位后的 `amount` / `amountIn` 比值；反向询价取倒数口径 |
| `calldata` | object | 交易数据，结构见 [`calldata`](#calldata) |

### paths[]

| 字段 | 类型 | 说明 |
|---|---|---|
| `chainId` | int | 链 ID |
| `token` | string | Token 合约地址 |
| `name` | string | Token 名称 |
| `symbol` | string | Token 符号 |
| `decimals` | int | Token 精度 |

### pairs[]

| 字段 | 类型 | 说明 |
|---|---|---|
| `chainId` | int | 链 ID |
| `pair` | string | 路由内部使用的池标记（两个 token 地址排序拼接后的 SHA1），业务方可不处理 |
| `tokenIn` / `tokenOut` | string | 该跳输入 / 输出 Token 地址 |
| `pools` | array | 该跳的池信息，结构见下 |

#### pairs[].pools[]

| 字段 | 类型 | 说明 |
|---|---|---|
| `pool` | string | Pool 合约地址；Uniswap v4 中为 poolId |
| `factory` | string | Pool 所在 factory 地址；Uniswap v4 中为 PoolManager |
| `router` | string | Pool 对应的 router 地址 |
| `protocol` | string | 协议类型，如 `uniswapv2`、`uniswapv3` |
| `amount` | int | 该跳可兑换的输出数量，未除精度 |
| `in` | int | 该跳输入数量，未除精度 |
| `price_impact` | int | 价格影响，单位 1e-6 |
| `slippage` | int | 预估滑点，单位 1e-6 |
| `reserve` | object | Pool 快照数据（储备、价格、流动性等诊断字段），业务方可不处理 |

### dexList[]

| 字段 | 类型 | 说明 |
|---|---|---|
| `chainId` | int | 链 ID |
| `tokenIn` / `tokenOut` | string | 该跳输入 / 输出 Token 地址 |
| `items` | array | DEX 明细，结构见下 |

#### dexList[].items[]

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | DEX 名称 |
| `logo` | string | DEX Logo 地址 |
| `amountIn` | int | 输入数量，未除精度 |
| `amountOut` | int | 可兑换输出数量，未除精度 |

### calldata

| 字段 | 类型 | 说明 |
|---|---|---|
| `to` | string | 交易目标合约。**同时也是 ERC20 授权的 spender** |
| `value` | int | 原生币买入时需随交易发送的 `msg.value`，未除精度；非原生币为 `0` |
| `data` | string | 交易的 `data`，**不带 `0x` 前缀**的 hex 字符串（部分库需自行补 `0x`） |
| `executed` | bool | 预执行结果。`withTx=false` 时恒 `false`；`withTx=true` 时 `true` 表示成功、`false` 表示失败 |
| `error` | string | 预执行失败或构建 calldata 失败时的错误信息，成功时为空。构建失败时该路由只有 `error`，没有 `to` / `data` |

> 数量类字段（`amount`、`minAmount`、`calldata.value`、池的 `amount`/`in` 等）为**未除精度的整数**（JSON number），可能超过 2<sup>53</sup>。请用 `BigInt` 等方式解析，避免精度丢失。

## 响应示例

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    {
      "id": "1904101388857217024",
      "paths": [
        {"chainId": 56, "token": "0x55d398326f99059fF775485246999027B3197955", "name": "Tether USD", "symbol": "USDT", "decimals": 18},
        {"chainId": 56, "token": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c", "name": "bnbchain", "symbol": "BNB", "decimals": 18},
        {"chainId": 56, "token": "0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82", "name": "PancakeSwap Token", "symbol": "Cake", "decimals": 18}
      ],
      "pairs": [
        {
          "chainId": 56,
          "pair": "0x209dd2d8a3264512ffeff53fe3dc12065250d920",
          "tokenIn": "0x55d398326f99059fF775485246999027B3197955",
          "tokenOut": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
          "pools": [
            {
              "pool": "0x47a90a2d92a8367a91efa1906bfc8c1e05bf10c4",
              "factory": "0xdB1d10011AD0Ff90774D0C6Bb92e5C5c8b4461F7",
              "router": "0xB971eF87ede563556b2ED4b1C0b0019111Dd85d2",
              "protocol": "uniswapv3",
              "amount": 1657883247489423,
              "in": 1000000000000000000,
              "price_impact": 5,
              "slippage": 106,
              "reserve": {
                "chainId": 56,
                "addr": "0x47a90a2d92a8367a91efa1906bfc8c1e05bf10c4",
                "factory": "0xdB1d10011AD0Ff90774D0C6Bb92e5C5c8b4461F7",
                "tokens": {"0": "0x55d398326f99059fF775485246999027B3197955", "1": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c"},
                "reserve0": "12000000.5",
                "reserve1": "20500.2",
                "time": 1792983200
              }
            }
          ]
        },
        {
          "chainId": 56,
          "pair": "0x9599a23ea0d5513e7e08f7e867093287ea3edee9",
          "tokenIn": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
          "tokenOut": "0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
          "pools": [
            {
              "pool": "0x3f23b4f1a35794306ba4f3176934012dc73312d1",
              "factory": "0xcA143Ce32Fe78f1f7019d7d551a6402fC5350c73",
              "router": "0x10ED43C718714eb63d5aA57B78B54704E256024E",
              "protocol": "uniswapv2",
              "amount": 152610607955654845,
              "in": 1657883247489423,
              "price_impact": 1241,
              "slippage": 621
            }
          ]
        }
      ],
      "dexList": [
        {
          "chainId": 56,
          "tokenIn": "0x55d398326f99059fF775485246999027B3197955",
          "tokenOut": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
          "items": [
            {"name": "Pancake V3", "logo": "https://asset-gdex.ourdex.com/dapps/exchange.pancakeswap.finance.png", "amountIn": 1000000000000000000, "amountOut": 1657883247489423}
          ]
        },
        {
          "chainId": 56,
          "tokenIn": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
          "tokenOut": "0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
          "items": [
            {"name": "PancakeSwap V2", "logo": "https://asset-gdex.ourdex.com/dapps/exchange.pancakeswap.finance.png", "amountIn": 1657883247489423, "amountOut": 152610607955654845}
          ]
        }
      ],
      "amount": 152610607955654845,
      "minAmount": 151847554915876571,
      "slippage": 5000,
      "suggestSlippage": 5000,
      "price": 0.15261060795565485,
      "calldata": {
        "to": "0x020818b41B5D536570E5224626fbf95dA4D0B453",
        "value": 0,
        "data": "5103aafa0000...",
        "executed": true,
        "error": ""
      }
    }
  ]
}
```

> 示例中的合约地址、池数据仅作字段说明，实际值以接口返回为准。

## 交易执行

接口只负责报价与构造 calldata，**授权、签名、广播由业务方完成**。

### 1. 选择路由

`result` 已按可兑换数量排序，通常取 `result[0]`。若业务侧有风控要求（单笔上限、价格影响上限），可在 `result` 中按 `pairs[].pools[].price_impact`、`slippage` 过滤后取第一条。

每条路由都带自己的 `calldata`，选定哪条就用哪条的 `calldata`，不要混用。

### 2. 授权输入代币

- **仅当 `tokenIn` 是 ERC20 时需要授权**；原生币买入（`tokenIn` 为零地址）无需授权，按 `calldata.value` 发送即可。
- **授权对象是 `calldata.to`**：链上真正执行 `transferFrom` 的是这个合约（UXUY Protocol）。**不要**授权给 `pairs[].pools[].router`、`pairs[].pools[].pool` 或任何 DEX 合约——授权错地址会在发送交易时 revert `UP: token allowance is not enough`。
- 授权数量（最小单位）需 **≥ 询价 `amountIn`**。常见做法：一次性授权一个较大额度，减少后续交易的授权笔数；或每次按需授权、交易后复核剩余额度。
- 业务方只需标准 ERC20 `approve` / `allowance`；**不支持**用 EIP-2612 `permit` 签名代替授权（协议对部分 DEX 的授权流转由合约内部处理，业务方无需感知）。
- **不要假设 `calldata.to` 在各链一致**：授权前始终以本次接口返回的 `to` 为准，切换链后必须按新地址重新授权。请勿在代码里硬编码合约地址；需要各链合约地址清单可向 BIPAL 团队索取。

```js
// ethers v6
const erc20 = new ethers.Contract(tokenIn, [
  'function allowance(address owner, address spender) view returns (uint256)',
  'function approve(address spender, uint256 amount) returns (bool)',
], signer);

const spender = calldata.to;                       // UXUY Protocol 合约
const amountInWei = 1000000000000000000n;          // 询价 amountIn 换算成最小单位

if (await erc20.allowance(wallet.address, spender) < amountInWei) {
  const tx = await erc20.approve(spender, ethers.MaxUint256);
  await tx.wait();
}
```

### 3. 发送交易

```js
const tx = await signer.sendTransaction({
  to: calldata.to,
  data: '0x' + calldata.data,     // 接口返回的 data 不带 0x 前缀
  value: BigInt(calldata.value),
});
const receipt = await tx.wait();
if (receipt.status !== 1) throw new Error('swap reverted');
```

| 要点 | 说明 |
|---|---|
| `from` | 必须是询价参数里的 `sender`：合约按 `msg.sender` 校验余额与授权，并向其拉取 `tokenIn` |
| `value` | 必须用 `calldata.value` 原值：原生币买入时非 0，其余情况为 0。漏给会导致 `UP: not enough ETH in transaction`，多给的 ETH 不会退回 |
| `gas` | 按节点估算值上浮 10%–20% 可提高成功率；多跳路由、Uniswap v4 池的 gas 波动较大 |
| `deadline` | calldata 内含截止时间，默认 **5 分钟**（Ethereum 主网 30 分钟）。超时上链会 revert `CommonBase: transaction too old`，需重新询价 |
| 输出接收方 | `receiver`（缺省为 `sender`）收到 `tokenOut`；路由执行期间未经业务方钱包中转 |

### 4. 确认结果

- 回执 `status == 1` 即链上成功；实际到账可用 `tokenOut` 的 `Transfer` 事件或收款地址余额差核对。
- 链上以 `minAmount` 兜底：成交低于该值整笔 revert，不存在"少收但仍成交"的情况。
- 同一用户连续下单时不要复用旧 calldata：前一笔会改变池状态，旧 `minAmount` 可能已不可达，请重新询价。

### 5. 常见失败原因

| 现象 / revert | 原因 | 处理 |
|---|---|---|
| `UP: token allowance is not enough` | 未授权、授权额不足，或授权给了错误地址 | 按 `calldata.to` 重新授权 |
| `UP: sender token balance is not enough` | `sender` 的 `tokenIn` 余额不足 | 核对地址与余额 |
| `UxuySwap: swapped amount less than minAmountOut`（部分场景为 `UP: amount less than minimum`，个别 DEX 透传自身错误） | 成交低于 `minAmount`：滑点过紧、池状态已变 | 重新询价，或放宽 `slippage` |
| `CommonBase: transaction too old` | calldata 超过有效期 | 重新询价获取新 calldata |
| `UP: not enough ETH in transaction` | `msg.value` 未按 `calldata.value` 发送 | 修正 `value` |
| `Pausable: paused` | 目标合约暂停 | 稍后重试或联系团队 |
| 广播前 `eth_call` 就失败 | 路由在最新区块状态下不可执行（余额、授权、池状态） | 换 `result` 中的下一条路由，或重新询价 |

### 6. 预执行（`withTx`）的语义

`withTx=true` 时服务端会用 `eth_call` 对 calldata 预执行一次，结果写在 `calldata.executed` / `calldata.error`：

- 预执行**不改变链上状态**，也不代替业务方发交易。
- 为了让未授权的用户也能拿到预执行结果，服务端在模拟时临时覆盖了输入代币的 `allowance`。因此 **`executed=true` 不代表业务方已完成授权**，真实发交易前仍必须先 approve。
- 预执行失败通常意味着该路由在当前状态下不可执行（余额不足、滑点过紧、池状态变化）。建议换下一条路由或重新询价，不要带着 `executed=false` 的 calldata 上链。

## 批量询价：`dex_offlineQuotes`

一次请求询多对 token，与单对接口语义一致：每对独立计算、独立回落，单对失败不影响其他对。

- `params` 为**参数对象数组的数组**：`params: [[{...}, {...}]]`
- 单次最多 20 对

### 请求示例

```bash
curl -s -X POST 'https://gwapi.ourdex.com/bnbchain/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0","id":1,"method":"dex_offlineQuotes",
    "params":[[
      {"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x55d398326f99059fF775485246999027B3197955","tokenOut":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","amountIn":"1","slippage":5000},
      {"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","tokenOut":"0x2170Ed0880ac9A755fd29B2688956BD959F933F8","amountIn":"10","slippage":5000}
    ]]
  }'
```

### 响应字段

`result` 为数组，顺序与请求一致。

| 字段 | 类型 | 说明 |
|---|---|---|
| `tokenIn` / `tokenOut` | string | 该交易对的 Token 地址 |
| `routers` | array | 路由数组，结构与 `dex_offlineQuote` 的 `result` 相同；无路由时省略 |
| `error` | string | 该交易对失败时的错误信息；成功时省略 |

## 排查：`dex_debugRoute`

询价返回空数组或结果异常时，可调用 `dex_debugRoute` 查看该交易对的路由可达性与池数据：

```bash
curl -s -X POST 'https://gwapi.ourdex.com/bnbchain/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"dex_debugRoute","params":[{"tokenIn":"0x...","tokenOut":"0x..."}]}'
```

返回含直达/两跳可达性、各池快照与剔除原因，主要用于反馈问题时的定位。接入阶段通常不需要调用。

## 说明与限制

- **仅同链询价**：本接口不处理跨链兑换。
- **报价与执行分离**：报价基于服务端本地池状态快照，链上状态在报价后仍会变化，因此以 `minAmount` + 滑点做保护；对时效敏感的场景请缩短"询价 → 广播"的间隔。
- **Uniswap v4 hooks 池**：swap 语义不受 hooks 影响且非动态费率的 v4 池参与离线报价；带 swap-affecting hooks 或动态费率的池自动走在线询价，业务方无需区分。
- **手续费**：平台手续费按业务约定配置，由链上合约在结算时收取；费率与结算币种请与 BIPAL 团队确认。

## 相关接口

| 接口 | 用途 |
|---|---|
| `dex_offlineQuote` | 单对询价 + calldata（本页） |
| `dex_offlineQuotes` | 批量询价（本页） |
| `dex_debugRoute` | 排查无路由原因（本页） |
| `dex_getCoin` | 查询 token 名称、符号、精度等基础信息 |
