# dex\_offlineQuote

EVM 链离线询价接口。
- 询价优先：多路径并行本地计算，按可兑换数量排序返回
- 守卫：最终路由集会对照学习到的参考币价做偏差体检，陈旧储备导致的虚高路由被拦下并回落在线报价
- `withTx=true` 时附带 calldata 并做一次 eth_call 预执行

配套批量接口：`dex_offlineQuotes`（一次请求询多对）。排查接口：`dex_debugRoute`（解释一对币为何无路由）。

## **请求接口**

`POST https://gwapi.ourdex.com/<chain>/YOUR-APP-ID`

### **METHOD**

`Method: dex_offlineQuote`

### **HEADERS**

`Content-Type: application/json`

### **请求参数**

`sender` - 用户 Address

`receiver` - 收款用户 Address

`tokenIn` - 卖出 Token Address

`tokenOut` - 买入 Token Address

`amountIn` - 卖出 Token Amount，十进制人类单位字符串（如 `"0.001"`），按 token 精度换算；`amountOut` 与 `amountIn` 二选一

`amountOut` - 买入 Token Amount（反向询价：给定目标产出反推所需输入，返回的 `amount` 为所需 `tokenIn` 数量）。反向询价仅在 `withTx=false` 时支持

`slippage` - 滑点，百万分之一，可不传（默认 0）。离线接口无自动滑点抬升：滑点是硬上限，路由偏差所需滑点超限时该路由被弃并回落在线

`extraSlippage` - 额外滑点，非必需，会加到 `slippage` 上

`decimals` - 指定 `amountIn`/`amountOut` 的精度，非必需；缺省按 token 精度

`size` - 返回的路由数量上限，非必需

`orderId` - 业务方订单ID，原样回传

`withTx` - 是否构建 calldata 并预执行。`false`（默认）：返回 calldata 但不预执行，`executed` 恒为 `false`；`true`：做一次 eth_call 预执行，结果回写 `executed`/`error`

`noFee` - 是否免平台手续费

`chainIn`/`chainOut`/`usdIn`/`usdOut`/`bridgeFee`/`bridgeRouter`/`bridgeAgent`/`fsReceiver` - 离线询价不支持跨链：`chainIn`/`chainOut` 传与当前链不同的 chainId 会直接报错；其余跨链/分佣字段不生效

### **请求示例**

```bash
curl https://gwapi.ourdex.com/bnbchain/YOUR-APP-ID \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","id":1,"method":"dex_offlineQuote","params":[{"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","receiver":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x55d398326f99059fF775485246999027B3197955","tokenOut":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","amountIn":"1","slippage":50,"size":2}]}'
```

## **接口响应**

`result` 为路由数组，按可兑换 `amount` 降序；本地与在线均无路由时返回空数组 `[]`。

### **响应参数**

`id` - 询价路径ID

`paths` - 询价的路径，按 `tokenIn` - 中间路径 Token（可没有） - `tokenOut` 排序

- `chainId` - chainId
- `token` - Token Address
- `name` - Token 名称
- `symbol` - Token Symbol
- `decimals` - Token 精度

`pairs` - 询价的池子，跟询价路径顺序匹配

- `chainId` - chainId
- `pair` - 根据 `tokenIn` 和 `tokenOut` 拼接的标记 pool 的地址，业务可不处理
- `tokenIn` - 该跳输入 Token 合约地址
- `tokenOut` - 该跳输出 Token 合约地址
- `pools` - pool 信息
    - `pool` - pool 合约地址（uniswapv4 为 poolId）
    - `factory` - pool 所在 factory 合约地址（uniswapv4 为 PoolManager）
    - `router` - pool 对应的 router 合约地址
    - `protocol` - factory 协议类型
    - `amount` - 该跳可兑换的输出数量，未除精度
    - `in` - 该跳输入数量，未除精度
    - `price_impact` - 价格影响，百万分之一
    - `slippage` - 预估滑点，百万分之一
    - `reserve` - 该池快照数据（储备/价格/流动性等诊断字段），业务可不处理

`dexList` - dex 信息

- `chainId` - chainId
- `tokenIn`/`tokenOut` - 该跳 Token 合约地址
- `items` - dex 信息
    - `name` - dex 名称
    - `logo` - dex logo 地址
    - `amountIn` - 输入数量，未除精度
    - `amountOut` - 可兑换输出数量，未除精度

`amount` - 最终兑换的 `tokenOut` 数量，未除精度（反向询价时为所需 `tokenIn` 数量）

`minAmount` - 最终兑换的最小数量，未除精度（由 `amount` 与生效滑点推导）

`slippage` - 生效滑点，百万分之一

`suggestSlippage` - 建议滑点，百万分之一。路由实际偏差超过生效滑点但仍在可执行范围时，该值大于 `slippage`，业务方执行前可参考放宽

`price` - 币价，最终可兑换 `amount`/`amountIn`（反向询价时取倒数口径）

`value` - 该笔交易对应的 USD 价值（仅能定价时返回），用于业务方做限制

`orderId` - 回传业务方订单ID

`calldata` - 封装的 calldata 数据

- `to` - 执行合约地址
- `value` - 当 `tokenIn` 为原生币时，输入的 amount（未除精度）
- `data` - calldata 数据
- `executed` - 预执行结果。`withTx=false` 时恒为 `false`；`withTx=true` 时 `false` 为预执行失败、`true` 为预执行成功
- `error` - 预执行失败时的报错信息

### **响应 Body 示例**

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

# dex\_offlineQuotes

批量离线询价。与 `dex_offlineQuote` 相同的询价与守卫语义，一次请求询多对 token；每对独立回落在线报价，单对失败不影响其他对。

## **请求示例**

```bash
curl https://gwapi.ourdex.com/bnbchain/YOUR-APP-ID \
    -X POST \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","id":1,"method":"dex_offlineQuotes","params":[[{"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","receiver":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x55d398326f99059fF775485246999027B3197955","tokenOut":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","amountIn":"1","slippage":50},{"sender":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","receiver":"0xDd841835e929FE458948aD31Ba43B6239c2912a0","tokenIn":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82","tokenOut":"0x2170Ed0880ac9A755fd29B2688956BD959F933F8","amountIn":"10","slippage":50}]]}'
```

`params` 为 `dex_offlineQuote` 参数对象的数组。

### **响应参数**

`result` 为数组，顺序与请求一致，每项：

`tokenIn`/`tokenOut` - 该对的 Token Address

`routers` - 路由数组，结构同 `dex_offlineQuote` 的 `result`；无路由时省略

`error` - 该对失败时的报错信息，成功时省略

### **响应 Body 示例**

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

## **行为说明**

- **离线优先**：报价数据全部来自本地（池储备、tick、bin 快照），不消耗链上 RPC；本地无路由或快照不足时回落在线 Quoter 询价。
- **守卫回落**：最终路由集对照价格学习通道的参考币价做偏差体检；陈旧储备造成的虚高路由被弃、回落在线报价，业务方无感知（结果仍可用，只是来源为链上状态）。
- **uniswapv4 hooks 池**：hooks 权限位不影响互换数学（且非动态费率）的 v4 池参与离线报价；swap-affecting hooks（beforeSwap、swap returns-delta）与动态费率池自动走在线报价路径，无需业务方区分。
