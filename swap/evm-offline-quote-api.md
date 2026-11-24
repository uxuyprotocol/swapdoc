# EVM 链离线询价（`dex_offlineQuote`）

`dex_offlineQuote` 用于获取 EVM 链上的代币兑换报价。

- **离线优先**：并行计算多条本地路由，并按可兑换数量排序返回。
- **偏差守卫**：对最终路由集进行参考币价偏差检查；陈旧储备导致的虚高路由会被拦截，并回落到在线报价。
- **交易预执行**：设置 `withTx=true` 时返回 calldata，并通过 `eth_call` 预执行。

相关接口：

- `dex_offlineQuotes`：批量查询多个交易对。
- `dex_debugRoute`：排查单个交易对无路由的原因。

## 接口概览

| 项目 | 值 |
|---|---|
| 请求方式 | `POST` |
| 请求地址 | `https://gwapi.ourdex.com/<chain>/YOUR-APP-ID` |
| JSON-RPC 方法 | `dex_offlineQuote` |
| Content-Type | `application/json` |

## 请求参数

| 参数 | 说明 |
|---|---|
| `sender` | 用户地址。 |
| `receiver` | 收款用户地址。 |
| `tokenIn` | 卖出代币地址。 |
| `tokenOut` | 买入代币地址。 |
| `amountIn` | 卖出代币数量，使用按 token 精度换算的人类可读十进制字符串（如 `"0.001"`）。与 `amountOut` 二选一。 |
| `amountOut` | 买入代币数量。反向询价时，按目标产出反推所需输入，返回的 `amount` 为所需 `tokenIn` 数量；仅在 `withTx=false` 时支持。 |
| `slippage` | 滑点，单位为百万分之一；可选，默认 `0`。离线接口不会自动抬高滑点：该值是硬上限，所需滑点超限的路由会被丢弃并回落在线报价。 |
| `extraSlippage` | 额外滑点，可选；会加到 `slippage` 上。 |
| `decimals` | `amountIn` / `amountOut` 的精度，可选；缺省时使用 token 精度。 |
| `size` | 返回的路由数量上限，可选。 |
| `orderId` | 业务方订单 ID，原样回传。 |
| `withTx` | 是否构建 calldata 并预执行。`false`（默认）时返回 calldata 但不预执行，`executed` 恒为 `false`；`true` 时执行一次 `eth_call`，并回写 `executed` / `error`。 |
| `noFee` | 是否免平台手续费。 |
| `chainIn` / `chainOut` / `usdIn` / `usdOut` / `bridgeFee` / `bridgeRouter` / `bridgeAgent` / `fsReceiver` | 离线询价不支持跨链。`chainIn` / `chainOut` 传入与当前链不同的 chainId 会直接报错；其余跨链或分佣字段不生效。 |

## 请求示例

```bash
curl https://gwapi.ourdex.com/bnbchain/YOUR-APP-ID \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","id":1,"method":"dex_offlineQuote","params":[{"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","receiver":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x55d398326f99059fF775485246999027B3197955","tokenOut":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","amountIn":"1","slippage":50,"size":2}]}'
```

## 接口响应

`result` 为路由数组，按可兑换 `amount` 降序；本地与在线均无路由时返回空数组 `[]`。

### 响应参数

| 字段 | 说明 |
|---|---|
| `id` | 询价路径 ID。 |
| `paths` | 询价路径，按 `tokenIn` → 中间 token（可为空）→ `tokenOut` 排序。每项结构见下表。 |
| `pairs` | 路径中的池信息，顺序与询价路径对应。每项结构见下表。 |
| `dexList` | 路径中各跳的 DEX 信息，结构见下表。 |
| `amount` | 最终可兑换的 `tokenOut` 数量，未除精度；反向询价时为所需 `tokenIn` 数量。 |
| `minAmount` | 最终最小兑换数量，未除精度，由 `amount` 与生效滑点推导。 |
| `slippage` | 生效滑点，单位为百万分之一。 |
| `suggestSlippage` | 建议滑点，单位为百万分之一。当路由实际偏差超过生效滑点但仍可执行时，该值会大于 `slippage`，业务方可在执行前参考并放宽滑点。 |
| `price` | 币价，按最终可兑换 `amount` / `amountIn` 计算；反向询价时取倒数口径。 |
| `value` | 该笔交易对应的 USD 价值；仅在可定价时返回，可用于业务方限制。 |
| `orderId` | 原样回传的业务方订单 ID。 |
| `calldata` | 封装的 calldata，字段见下表。 |

#### `paths[]`

| 字段 | 说明 |
|---|---|
| `chainId` | 链 ID。 |
| `token` | Token 合约地址。 |
| `name` | Token 名称。 |
| `symbol` | Token 符号。 |
| `decimals` | Token 精度。 |

#### `pairs[]`

| 字段 | 说明 |
|---|---|
| `chainId` | 链 ID。 |
| `pair` | 根据 `tokenIn` 和 `tokenOut` 拼接的 pool 标记地址，业务方可不处理。 |
| `tokenIn` / `tokenOut` | 该跳输入 / 输出 Token 合约地址。 |
| `pools` | 该跳的 pool 信息，字段见下表。 |

#### `pairs[].pools[]`

| 字段 | 说明 |
|---|---|
| `pool` | Pool 合约地址；Uniswap v4 中为 poolId。 |
| `factory` | Pool 所在的 factory 合约地址；Uniswap v4 中为 PoolManager。 |
| `router` | Pool 对应的 router 合约地址。 |
| `protocol` | Factory 协议类型。 |
| `amount` | 该跳可兑换的输出数量，未除精度。 |
| `in` | 该跳输入数量，未除精度。 |
| `price_impact` | 价格影响，单位为百万分之一。 |
| `slippage` | 预估滑点，单位为百万分之一。 |
| `reserve` | Pool 快照数据（储备、价格、流动性等诊断字段），业务方可不处理。 |

#### `dexList[]`

| 字段 | 说明 |
|---|---|
| `chainId` | 链 ID。 |
| `tokenIn` / `tokenOut` | 该跳输入 / 输出 Token 合约地址。 |
| `items` | DEX 信息列表，字段见下表。 |

#### `dexList[].items[]`

| 字段 | 说明 |
|---|---|
| `name` | DEX 名称。 |
| `logo` | DEX Logo 地址。 |
| `amountIn` | 输入数量，未除精度。 |
| `amountOut` | 可兑换输出数量，未除精度。 |

#### `calldata`

| 字段 | 说明 |
|---|---|
| `to` | 执行合约地址。 |
| `value` | 当 `tokenIn` 为原生币时的输入数量，未除精度。 |
| `data` | Calldata 数据。 |
| `executed` | 预执行结果。`withTx=false` 时恒为 `false`；`withTx=true` 时，`false` 表示预执行失败，`true` 表示成功。 |
| `error` | 预执行失败时的错误信息。 |

### 响应示例

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": [
        {
            "id": "1904101388857217024",
            "paths": [
                {
                    "chainId": 56,
                    "token": "0x55d398326f99059fF775485246999027B3197955",
                    "name": "Tether USD",
                    "symbol": "USDT",
                    "decimals": 18
                },
                {
                    "chainId": 56,
                    "token": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
                    "name": "bnbchain",
                    "symbol": "BNB",
                    "decimals": 18
                },
                {
                    "chainId": 56,
                    "token": "0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
                    "name": "PancakeSwap Token",
                    "symbol": "Cake",
                    "decimals": 18
                }
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
                                "tokens": {
                                    "0": "0x55d398326f99059fF775485246999027B3197955",
                                    "1": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c"
                                },
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
                        {
                            "name": "Pancake V3",
                            "logo": "https://asset-gdex.ourdex.com/dapps/exchange.pancakeswap.finance.png",
                            "amountIn": 1000000000000000000,
                            "amountOut": 1657883247489423
                        }
                    ]
                },
                {
                    "chainId": 56,
                    "tokenIn": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
                    "tokenOut": "0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
                    "items": [
                        {
                            "name": "PancakeSwap V2",
                            "logo": "https://asset-gdex.ourdex.com/dapps/exchange.pancakeswap.finance.png",
                            "amountIn": 1657883247489423,
                            "amountOut": 152610607955654845
                        }
                    ]
                }
            ],
            "amount": 152610607955654845,
            "minAmount": 152534256787178091,
            "slippage": 50,
            "suggestSlippage": 50,
            "price": 0.15261060795565485,
            "value": 1.000000,
            "orderId": "",
            "calldata": {
                "to": "0x020818b41B5D536570E5224626fbf95dA4D0B453",
                "value": 0,
                "data": "5103aafa0000...",
                "executed": false,
                "error": ""
            }
        }
    ]
}
```

## 批量离线询价（`dex_offlineQuotes`）

批量离线询价。与 `dex_offlineQuote` 相同的询价与守卫语义，一次请求询多对 token；每对独立回落在线报价，单对失败不影响其他对。

### 请求示例

```bash
curl https://gwapi.ourdex.com/bnbchain/YOUR-APP-ID \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","id":1,"method":"dex_offlineQuotes","params":[[{"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","receiver":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x55d398326f99059fF775485246999027B3197955","tokenOut":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","amountIn":"1","slippage":50},{"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","receiver":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","tokenOut":"0x2170Ed0880ac9A755fd29B2688956BD959F933F8","amountIn":"10","slippage":50}]]}'
```

`params` 为 `dex_offlineQuote` 参数对象的数组。

### 响应参数

`result` 为数组，顺序与请求一致。每项字段如下：

| 字段 | 说明 |
|---|---|
| `tokenIn` / `tokenOut` | 该交易对的 Token 地址。 |
| `routers` | 路由数组，结构与 `dex_offlineQuote` 的 `result` 相同；无路由时省略。 |
| `error` | 该交易对失败时的错误信息；成功时省略。 |

### 响应示例

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": [
        {
            "tokenIn": "0x55d398326f99059fF775485246999027B3197955",
            "tokenOut": "0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
            "routers": [
                {
                    "id": "1904101388857217024",
                    "paths": [],
                    "pairs": [],
                    "dexList": [],
                    "amount": 152610607955654845,
                    "minAmount": 152534256787178091,
                    "suggestSlippage": 50,
                    "slippage": 50,
                    "price": 0.15261060795565485
                }
            ]
        },
        {
            "tokenIn": "0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
            "tokenOut": "0x2170Ed0880ac9A755fd29B2688956BD959F933F8",
            "routers": [
                {
                    "id": "1904101388857217025",
                    "paths": [],
                    "pairs": [],
                    "dexList": [],
                    "amount": 2673809953416,
                    "minAmount": 2672471400284,
                    "suggestSlippage": 50,
                    "slippage": 50,
                    "price": 0.0002673809953416
                }
            ]
        }
    ]
}
```

### 行为说明

- **离线优先**：报价数据全部来自本地（池储备、tick、bin 快照），不消耗链上 RPC；本地无路由或快照不足时回落在线 Quoter 询价。
- **守卫回落**：最终路由集对照价格学习通道的参考币价做偏差体检；陈旧储备造成的虚高路由被弃、回落在线报价，业务方无感知（结果仍可用，只是来源为链上状态）。
- **uniswapv4 hooks 池**：hooks 权限位不影响互换数学（且非动态费率）的 v4 池参与离线报价；swap-affecting hooks（beforeSwap、swap returns-delta）与动态费率池自动走在线报价路径，无需业务方区分。
