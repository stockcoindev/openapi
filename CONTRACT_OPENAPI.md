# 合约 OpenAPI 完整技术文档

## 目录

1. [基础信息与认证](#1-基础信息与认证)
2. [签名算法详解](#2-签名算法详解)
3. [公共 REST 接口（无需签名）](#3-公共-rest-接口无需签名)
4. [私有 REST 接口（需要签名）](#4-私有-rest-接口需要签名)
5. [WebSocket 用户私有流](#5-websocket-用户私有流)
6. [错误码详解](#6-错误码详解)

---

## 1. 基础信息与认证

### 1.1 服务端点

| 服务类型 | 测试环境地址 |
| :--- | :--- |
| **REST API** | `****` |
| **WebSocket 私有流** | `****/openapi/v1/ws/{listenKey}` |

### 1.2 认证方式

所有私有接口（Signed Endpoints）需要在 HTTP 请求头中携带 API Key：

```
X-BH-APIKEY: <your_api_key>
```

**重要说明**：
- API Key 必须绑定到 `FUTURES` 账户类型（account_type = 3）
- 每个请求必须包含 `timestamp` 参数（毫秒级时间戳）
- 每个请求必须包含 `signature` 参数（签名，详见第 2 章）

---

## 2. 签名算法详解

### 2.1 签名流程

签名算法使用 **HMAC-SHA256**，具体步骤如下：

#### 步骤 1: 收集参数
- 收集所有 **Query String** 参数（URL 参数）
- 如果是 POST/PUT/DELETE 请求且使用 **Form-Data**，收集所有 Body 参数
- **注意**：`signature` 参数不参与签名计算
- **注意**：如果 `data` 是 JSON Array（如批量下单），Body 不参与签名

#### 步骤 2: 参数排序
- 将所有参数按 **Key 的字母顺序**升序排列
- 使用 `requests.Request.prepare()` 或类似方法确保排序一致

#### 步骤 3: 构建查询字符串
- 将 Query 参数序列化为 `key1=value1&key2=value2` 格式，得到 `query_string`
- 将 Form-Data Body 参数同样序列化为 `key1=value1&key2=value2` 格式，得到 `body_string`
- **关键**：如果 Query 参数存在，Body 参数也存在，则直接拼接：`total_params = query_string + body_string`（**中间不加任何分隔符**）

#### 步骤 4: 计算签名
```python
import hmac
import hashlib

signature = hmac.new(
    secret_key.encode('utf-8'),
    total_params.encode('utf-8'),
    hashlib.sha256
).hexdigest()
```

#### 步骤 5: 添加签名
- 将计算得到的 `signature` 作为 Query 参数添加到请求中

### 2.2 签名示例

**示例 1: GET 请求**
```
请求: GET /openapi/v1/futures/balance?symbol=NVDA-PERP-USDT&timestamp=1767930000000

步骤：
1. Query 参数: {"symbol": "NVDA-PERP-USDT", "timestamp": 1767930000000}
2. 排序后: symbol=NVDA-PERP-USDT&timestamp=1767930000000
3. total_params = "symbol=NVDA-PERP-USDT&timestamp=1767930000000"
4. signature = HMAC_SHA256(secret_key, total_params)
5. 最终 URL: /openapi/v1/futures/balance?symbol=NVDA-PERP-USDT&timestamp=1767930000000&signature=<signature>
```

**示例 2: POST 请求（Form-Data）**
```
请求: POST /openapi/v1/futures/order

Query 参数: {"timestamp": 1767930000000}
Body 参数: {"symbol": "NVDA-PERP-USDT", "side": "BUY_OPEN", "quantity": "10000", "clientOrderId": "test123"}

步骤：
1. Query 排序: timestamp=1767930000000
2. Body 排序: clientOrderId=test123&quantity=10000&side=BUY_OPEN&symbol=NVDA-PERP-USDT
3. total_params = "timestamp=1767930000000" + "clientOrderId=test123&quantity=10000&side=BUY_OPEN&symbol=NVDA-PERP-USDT"
   = "timestamp=1767930000000clientOrderId=test123&quantity=10000&side=BUY_OPEN&symbol=NVDA-PERP-USDT"
4. signature = HMAC_SHA256(secret_key, total_params)
5. 最终请求: POST /openapi/v1/futures/order?timestamp=1767930000000&signature=<signature>
   Body: symbol=NVDA-PERP-USDT&side=BUY_OPEN&quantity=10000&clientOrderId=test123
```

**示例 3: POST 请求（JSON Body，如批量下单）**
```
请求: POST /openapi/v1/futures/order/batch

Query 参数: {"timestamp": 1767930000000, "orderSource": "PYTHON_TEST"}
Body: [{"symbolId": "NVDA-PERP-USDT", ...}]  (JSON Array)

步骤：
1. Query 排序: orderSource=PYTHON_TEST&timestamp=1767930000000
2. Body 是 JSON Array，不参与签名
3. total_params = "orderSource=PYTHON_TEST&timestamp=1767930000000"
4. signature = HMAC_SHA256(secret_key, total_params)
```

---

## 3. 公共 REST 接口（无需签名）

### 3.1 获取合约信息 - getContracts

**接口**: `GET /openapi/v1/getContracts`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| expired | String | 否 | "false" | 是否包含已过期/下架的合约，可选值: "true", "false" |

**响应格式**: Array of Objects

**响应字段详解** (`ContractsResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| symbol | String | 合约交易对 ID，如 `NVDA-PERP-USDT` |
| symbolName | String | 合约显示名称 |
| baseToken | String | 基础资产币种，如 `NVDA` |
| quoteToken | String | 计价资产币种，如 `USDT` |
| lastPrice | Decimal | 最新成交价格 |
| baseVolume | Decimal | 24小时基础资产成交量 |
| quoteVolume | Decimal | 24小时成交额（计价资产） |
| bid | Decimal | 当前买一价 |
| ask | Decimal | 当前卖一价 |
| high | Decimal | 24小时最高价 |
| low | Decimal | 24小时最低价 |
| productType | String | 产品类型标识 |
| openInterest | String | 当前持仓量 |
| indexPrice | Decimal | 指数价格 |
| index | String | 指数标识 |
| indexBaseToken | String | 指数基础币种 |
| startTs | Long | 合约上线/开盘时间戳（毫秒） |
| endTs | Long | 合约交割/到期时间戳（毫秒），永续合约通常为 0 |
| fundingRate | Decimal | 当前资金费率 |
| nextFundingRate | Decimal | 下期预测资金费率 |
| nextFundingRateTs | Integer | 下次结算时间戳（毫秒） |

### 3.2 获取合约信息 - contracts（别名）

**接口**: `GET /openapi/v1/contracts`

**说明**: 与 `/openapi/v1/getContracts` 功能完全相同，仅路径不同。

**请求参数**: 同 3.1

**响应格式**: 同 3.1

### 3.3 查询保险基金

**接口**: `GET /openapi/v1/futures/insurance`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 否 | - | 合约 ID，不传则返回所有合约 |
| fromId | Long | 否 | 0 | 起始记录 ID，用于翻页 |
| limit | Integer | 否 | 20 | 返回记录数，最大 500 |

**响应格式**: Array of Objects

**响应字段详解** (`InsuranceFundResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| id | Long | 记录唯一 ID |
| timestamp | Long | 记录时间戳（毫秒） |
| value | String | 保险基金余额 |
| unit | String | 资产单位，如 `USDT` |

### 3.4 查询实时资金费率

**接口**: `GET /openapi/v1/futures/fundingRate`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 否 | - | 合约 ID，不传则返回所有合约 |
| state | String | 否 | "current" | 状态，目前固定为 `"current"` |

**响应格式**: Array of Objects

**响应字段详解** (`FundingRateResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| symbol | String | 合约 ID |
| rate | String | 当前实时资金费率 |
| nextFundingTime | Long | 下次结算时间戳（毫秒） |

### 3.5 查询历史资金费率

**接口**: `GET /openapi/v1/futures/fundingRate/history`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 否 | - | 合约 ID |
| limit | Integer | 否 | 20 | 返回记录数，最大 1000 |

**响应格式**: Array of Objects

**响应字段详解** (`HistoryFundingRateResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| id | Long | 结算记录 ID |
| symbol | String | 合约 ID |
| settleTime | Long | 结算时间戳（毫秒） |
| settleRate | String | 结算时的资金费率 |

---

## 4. 私有 REST 接口（需要签名）

所有接口均需要在 Header 中添加 `X-BH-APIKEY`，并在 Query 参数中添加 `timestamp` 和 `signature`。

### 4.1 查询账户余额

**接口**: `GET /openapi/v1/futures/balance`

**请求参数**: 无（仅需要 timestamp 和 signature）

**响应格式**: Object

**响应字段详解** (`FuturesBalanceResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| balance | String | 账户总余额（Wallet Balance） |
| availableBalance | String | 可用保证金（已包含全仓未实现盈亏） |
| positionMargin | String | 当前持仓占用的保证金 |
| orderMargin | String | 当前挂单锁定的保证金 |
| asset | String | 资产币种，如 `USDT` |
| crossUnRealizedPnl | String | 全仓持仓的总未实现盈亏 |

### 4.2 查询当前持仓

**接口**: `GET /openapi/v1/futures/positions`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 否 | 合约 ID，不传则返回所有持仓 |
| side | String | 否 | 持仓方向，可选值: `LONG`, `SHORT` |

**响应格式**: Array of Objects

**响应字段详解** (`FuturesPositionResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| symbol | String | 合约交易对 ID |
| side | String | 持仓方向: `LONG` (多仓), `SHORT` (空仓) |
| position | String | 总持仓张数 |
| available | String | 可平仓数量（未被挂单锁定的数量） |
| avgPrice | String | 持仓均价（开仓平均成本价） |
| leverage | String | 当前使用的杠杆倍数 |
| lastPrice | String | 市场最新价格 |
| positionValue | String | 仓位名义价值 |
| flp | String | 强平价 |
| margin | String | 仓位已占用的保证金 |
| marginRate | String | 保证金率 |
| unrealizedPnL | String | 未实现盈亏（浮动盈亏） |
| profitRate | String | 盈亏率 |
| realizedPnL | String | 累计已实现盈亏 |
| accountId | String | 账户 ID（仅做市账户支持） |
| minMargin | String | 维持该仓位所需的最低保证金 |

### 4.3 下单（New Order）

**接口**: `POST /openapi/v1/futures/order`

**请求参数（Form-Data）**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 是 | - | 合约交易对 ID |
| side | String | 是 | - | 订单方向，可选值: `BUY_OPEN` (开多), `SELL_OPEN` (开空), `BUY_CLOSE` (平空), `SELL_CLOSE` (平多) |
| orderType | String | 是 | - | 订单类型，可选值: `LIMIT`, `STOP` |
| quantity | String | 是 | - | 下单数量（张数） |
| price | String | 否 | "0" | 委托价格。市价单时传 `"0"` |
| priceType | String | 否 | "INPUT" | 价格类型，可选值: `INPUT` (限价), `MARKET` (市价) |
| triggerPrice | String | 否 | - | 计划委托触发价格（仅 `orderType=STOP` 时有效） |
| leverage | String | 否 | - | 杠杆倍数 |
| overPrice | String | 否 | - | 超价参数 |
| timeInForce | String | 否 | "GTC" | 生效策略，可选值: `GTC` (Good Till Cancel), `IOC` (Immediate Or Cancel), `FOK` (Fill Or Kill) |
| clientOrderId | String | 是 | - | 客户端自定义订单 ID（必须唯一） |
| orderSource | String | 否 | "" | 订单来源标识 |

**响应格式**: Object

**响应字段详解** (`FuturesOrderResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| time | Long | 订单创建时间戳（毫秒） |
| updateTime | Long | 订单最后更新时间戳（毫秒） |
| orderId | Long | 系统生成的订单 ID |
| accountId | Long | 账户 ID |
| clientOrderId | String | 客户端自定义订单 ID |
| symbol | String | 合约交易对 ID |
| price | String | 委托价格 |
| leverage | String | 杠杆倍数 |
| origQty | String | 原始下单数量 |
| executedQty | String | 已成交数量 |
| executeQty | String | 已成交数量（别名） |
| executedAmount | String | 已成交金额 |
| avgPrice | String | 成交均价 |
| marginLocked | String | 订单锁定的保证金 |
| orderType | String | 订单类型: `LIMIT`, `STOP` |
| side | String | 订单方向: `BUY_OPEN`, `SELL_OPEN`, `BUY_CLOSE`, `SELL_CLOSE` |
| timeInForce | String | 生效策略: `GTC`, `IOC`, `FOK` |
| status | String | 订单状态: `NEW`, `PARTIALLY_FILLED`, `FILLED`, `CANCELED`, `REJECTED` |
| isClose | Boolean | 是否为平仓单 |
| priceType | String | 价格类型: `INPUT`, `MARKET`, `OPPONENT`, `QUEUE`, `OVER` |
| triggerPrice | String | 计划委托触发价格 |
| isLiquidationOrder | Boolean | 是否为系统强平单 |
| contractMultiplier | String | 合约乘数 |

### 4.4 查询订单详情

**接口**: `GET /openapi/v1/futures/order`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约交易对 ID |
| orderId | Long | 否* | 系统订单 ID（与 clientOrderId 二选一） |
| clientOrderId | String | 否* | 客户端订单 ID（与 orderId 二选一） |
| orderType | String | 否 | 订单类型，默认 `LIMIT` |

*注：`orderId` 和 `clientOrderId` 必须提供其中一个。

**响应格式**: Object（同 4.3 的响应字段）

### 4.5 撤销订单

**接口**: `DELETE /openapi/v1/futures/order`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约交易对 ID |
| orderId | Long | 否* | 系统订单 ID（与 clientOrderId 二选一） |
| clientOrderId | String | 否* | 客户端订单 ID（与 orderId 二选一） |
| orderType | String | 否 | 订单类型，默认 `LIMIT` |

*注：`orderId` 和 `clientOrderId` 必须提供其中一个。

**响应格式**: Object（同 4.3 的响应字段）

### 4.6 查询当前挂单

**接口**: `GET /openapi/v1/futures/orders/open`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 否 | "" | 合约 ID，不传则返回所有合约的挂单 |
| orderId | Long | 否 | - | 起始订单 ID，用于翻页 |
| orderType | String | 否 | "LIMIT" | 订单类型: `LIMIT`, `STOP` |
| limit | Integer | 否 | 20 | 返回条数，最大 500 |

**响应格式**: Array of Objects（每个元素同 4.3 的响应字段）

### 4.7 查询历史订单

**接口**: `GET /openapi/v1/futures/orders/history`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 否 | "" | 合约 ID |
| orderId | Long | 否 | 0 | 起始订单 ID |
| orderType | String | 否 | "LIMIT" | 订单类型: `LIMIT`, `STOP` |
| startTime | Long | 否 | 0 | 开始时间戳（毫秒） |
| endTime | Long | 否 | 0 | 结束时间戳（毫秒） |
| limit | Integer | 否 | 20 | 返回条数，最大 500 |

**响应格式**: Array of Objects（每个元素同 4.3 的响应字段）

### 4.8 查询成交记录

**接口**: `GET /openapi/v1/futures/trades`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 否 | "" | 合约 ID |
| fromId | Long | 否 | 0 | 起始成交 ID |
| toId | Long | 否 | 0 | 结束成交 ID |
| startTime | Long | 否 | 0 | 开始时间戳（毫秒） |
| endTime | Long | 否 | 0 | 结束时间戳（毫秒） |
| limit | Integer | 否 | 20 | 返回条数，最大 500 |

**响应格式**: Array of Objects

**响应字段详解** (`FuturesMatchResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| time | Long | 成交时间戳（毫秒） |
| tradeId | Long | 成交 ID |
| orderId | Long | 订单 ID |
| matchOrderId | Long | 对手方订单 ID |
| symbolId | String | 合约交易对 ID |
| price | String | 成交价格 |
| quantity | String | 成交数量（张） |
| feeTokenId | String | 手续费币种 |
| fee | String | 手续费金额 |
| makerRebate | String | Maker 返佣 |
| orderType | String | 订单类型 |
| side | String | 订单方向 |
| pnl | String | 盈亏（平仓单才有） |

### 4.9 查询风险限额

**接口**: `GET /openapi/v1/futures/riskLimit`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID |

**响应格式**: Array of Objects

**响应字段详解** (`FuturesRiskLimitResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| riskLimitId | String | 风险限额 ID（已废弃） |
| riskLimitAmount | String | 风险限额（最大持仓量） |
| maintainMargin | String | 维持保证金率（如 0.03 表示 3%） |
| initialMargin | String | 起始保证金率（如 0.1 表示 10%） |
| side | String | 订单方向: `BUY_OPEN`, `SELL_OPEN` |

### 4.10 查询账户杠杆与模式

**接口**: `GET /openapi/v1/futures/accountLeverage`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID |

**响应格式**: Array of Objects

**响应字段详解** (`AccountLeverageResponse`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| symbolId | String | 合约 ID |
| leverage | String | 当前杠杆倍数 |
| marginType | String | 保证金模式: `ISOLATED` (逐仓), `CROSSED` (全仓) |

### 4.11 查询手续费率

**接口**: `GET /openapi/v1/futures/commissionRate`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID |

**响应格式**: Object

**响应字段详解** (`FuturesCommissionRate`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| symbol | String | 合约 ID |
| openMakerFee | String | 开仓挂单成交费率 |
| openTakerFee | String | 开仓吃单成交费率 |
| closeMakerFee | String | 平仓挂单成交费率 |
| closeTakerFee | String | 平仓吃单成交费率 |

### 4.12 查询最佳订单（特殊权限）

**接口**: `GET /openapi/v1/futures/order/best`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID |

**响应格式**: Object

**响应字段详解** (`BestOrderResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| price | String | 当前价格 |
| bid | Object | 买一订单信息 (`BestOrderInfo`) |
| └ time | Long | 时间戳 |
| └ orderId | Long | 订单 ID |
| └ accountId | Long | 账户 ID |
| └ price | String | 价格 |
| └ origQty | String | 数量 |
| └ type | String | 类型 |
| └ side | String | 方向 |
| └ status | String | 状态 |
| ask | Object | 卖一订单信息（结构同 bid） |

### 4.13 查询深度信息（特殊权限）

**接口**: `GET /openapi/v1/futures/depth`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbols | String | 是 | 合约 ID，多个用逗号分隔 |

**响应格式**: Object

**响应字段详解** (`DepthInfoResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| time | Long | 时间戳 |
| level | Integer | 深度层级 |
| exchangeId | Long | 交易所 ID |
| depthInfoList | Array | 深度信息列表 |
| └ symbolId | String | 合约 ID |
| └ bid | Array | 买单列表 |
| └└ price | String | 价格 |
| └└ originalPrice | String | 原始价格 |
| └└ orderInfo | Array | 订单信息 |
| └└└ quantity | String | 数量 |
| └└└ accountId | Long | 账户 ID |
| └ ask | Array | 卖单列表（结构同 bid） |

### 4.14 市场持仓聚合查询（特殊权限）

**接口**: `GET /openapi/v1/futures/pullPositions`

**请求参数**:
| 参数名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | 是 | - | 合约 ID |
| limit | Long | 否 | 20 | 返回条数，最大 100 |

**响应格式**: Array of Objects（同 4.2 的响应字段）

### 4.15 批量撤销订单（按币对）

**接口**: `DELETE /openapi/v1/futures/cancel/batch/symbol`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID，多个用逗号分隔 |
| side | String | 否 | 订单方向，可选值: `BUY_OPEN`, `SELL_OPEN` 等 |

**响应格式**: Object
```
{
  "code": 200,
  "msg": "success",
  "timestamp": 1767930000000
}
```

### 4.16 调整杠杆倍数

**接口**: `POST /openapi/v1/futures/leverage`

**请求参数（Form-Data）**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID |
| leverage | String | 是 | 目标杠杆倍数，如 "10", "20", "50" |

**响应格式**: Object

**响应字段详解** (`AdjustAccountLeverageResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| code | Integer | 状态码，0 为成功 |
| symbolId | String | 合约 ID |
| leverage | String | 调整后的杠杆倍数 |

### 4.17 调整保证金模式

**接口**: `POST /openapi/v1/futures/marginType`

**请求参数（Form-Data）**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID |
| marginType | String | 是 | 保证金模式: `ISOLATED` (逐仓), `CROSSED` (全仓) |

**响应格式**: Object

**响应字段详解** (`AdjustMarginTypeResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| code | Integer | 状态码，0 为成功 |
| symbolId | String | 合约 ID |
| marginType | String | 调整后的保证金模式 |

### 4.18 修改逐仓保证金

**接口**: `POST /openapi/v1/futures/margin/modify`

**请求参数（Form-Data）**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| symbol | String | 是 | 合约 ID |
| side | String | 是 | 仓位方向: `LONG`, `SHORT` |
| amount | String | 是 | 变动数量，正数为增加，负数为减少 |

**响应格式**: Object

**响应字段详解** (`ModifyMarginResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| code | Integer | 状态码，0 为成功 |
| msg | String | 消息 |
| symbol | String | 合约 ID |
| margin | String | 修改后的保证金 |
| timestamp | Long | 时间戳 |

### 4.19 批量下单

**接口**: `POST /openapi/v1/futures/order/batch`

**请求参数**:
- **Query 参数**:
  | 参数名 | 类型 | 必填 | 描述 |
  | :--- | :--- | :--- | :--- |
  | orderSource | String | 否 | 订单来源标识 |
- **Body 参数（JSON Array）**:
  | 参数名 | 类型 | 必填 | 描述 |
  | :--- | :--- | :--- | :--- |
  | symbolId | String | 是 | 合约 ID |
  | orderSide | String | 是 | 订单方向: `BUY_OPEN`, `SELL_OPEN`, `BUY_CLOSE`, `SELL_CLOSE` |
  | orderType | String | 是 | 订单类型: `LIMIT`, `STOP` |
  | quantity | String | 是 | 数量（张） |
  | price | String | 否 | 价格（市价单传 "0"） |
  | priceType | String | 否 | 价格类型: `INPUT`, `MARKET` |
  | clientOrderId | String | 是 | 客户端订单 ID |

**响应格式**: Object

**响应字段详解** (`BatchNewFuturesOrderResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| code | Integer | 状态码 |
| result | Array | 每个订单的处理结果 (`NewFuturesOrderSingleResult`) |
| └ code | Integer | 单个订单的状态码 |
| └ msg | String | 消息 |
| └ order | Object | 订单信息（同 4.3 的响应字段） |

### 4.20 批量撤销订单（按 ID 列表）

**接口**: `DELETE /openapi/v1/futures/cancel/batch/ids`

**请求参数**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| ids | String | 是 | 订单 ID 列表，逗号分隔，如 "123,456,789" |

**响应格式**: Object

**响应字段详解** (`BatchCancelOrderResult`):
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| code | Integer | 状态码 |
| result | Array | 每个订单的撤单结果 (`OrderCancelResult`) |
| └ orderId | Long | 订单 ID |
| └ code | Integer | 撤单结果码，0 为成功 |

---

## 5. WebSocket 用户私有流

### 5.1 获取 listenKey（REST 接口）

**接口**: `POST /openapi/v1/userdata`

**请求参数（Query）**:
| 参数名 | 类型 | 必填 | 描述 |
| :--- | :--- | :--- | :--- |
| timestamp | Long | 是 | 时间戳（毫秒） |
| signature | String | 是 | 签名 |

**请求头**:
```
X-BH-APIKEY: <your_api_key>
```

**响应格式**: Object
```json
{
  "listenKey": "ZOoMNMNPdDkBsCHiMitBtEmBwdkbsWXuTZoCewQvSINgIyULPREHJLuuzhjTEgwl"
}
```

**响应字段**:
| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| listenKey | String | WebSocket 连接令牌，有效期 60 分钟 |

### 5.2 建立 WebSocket 连接

**连接地址**: `****/openapi/v1/ws/{listenKey}`

**示例**:
```
****/openapi/v1/ws/ZOoMNMNPdDkBsCHiMitBtEmBwdkbsWXuTZoCewQvSINgIyULPREHJLuuzhjTEgwl
```

**连接流程**:
1. 通过 REST 接口获取 `listenKey`（见 5.1）
2. 使用 `listenKey` 替换 URL 中的 `{listenKey}` 占位符
3. 建立 WebSocket 连接
4. 连接成功后，服务器会推送订单、持仓、余额等实时数据

### 5.3 心跳维护（PING/PONG）

**服务端 PING**:
服务器会定期发送心跳包：
```json
{
  "ping": 1767930000000
}
```

**客户端 PONG**:
客户端收到 `ping` 后，**必须立即**回复 `pong`：
```json
{
  "pong": 1767930000000
}
```

**重要说明**:
- 如果客户端在收到 `ping` 后 3 分钟内未回复 `pong`，服务器会主动断开连接
- `listenKey` 的有效期为 60 分钟，建议每 30-50 分钟通过 REST 接口延长有效期

### 5.4 推送消息格式

所有推送消息均为 JSON 格式，可能为单个对象或对象数组。

#### 5.4.1 订单更新推送 (`contractExecutionReport`)

**事件类型**: `contractExecutionReport`

**触发时机**: 订单状态变更（创建、部分成交、完全成交、撤销、拒绝）

**推送格式**: Object 或 Array of Objects

**字段详解** (`SocketOrderInfo`):
| 字段名 (JSON Key) | 类型 | 描述 |
| :--- | :--- | :--- |
| e | String | 事件类型，固定为 `"contractExecutionReport"` |
| E | Long | 事件时间戳（毫秒） |
| s | String | 合约交易对 ID，如 `"NVDA-PERP-USDT"` |
| c | String | 客户端订单 ID (`clientOrderId`) |
| S | String | 订单方向: `"BUY_OPEN"`, `"SELL_OPEN"`, `"BUY_CLOSE"`, `"SELL_CLOSE"` |
| o | String | 订单类型: `"LIMIT"`, `"STOP"` |
| f | String | 生效策略: `"GTC"`, `"IOC"`, `"FOK"` |
| q | String | 原始委托数量（张） |
| p | String | 委托价格（市价单为 "0"） |
| X | String | 订单状态: `"NEW"`, `"PARTIALLY_FILLED"`, `"FILLED"`, `"CANCELED"`, `"REJECTED"` |
| i | Long | 系统订单 ID (`orderId`) |
| l | String | 最后成交数量（本次推送的成交数量） |
| z | String | 累计成交数量（该订单已成交的总数量） |
| L | String | 最后成交价格（本次撮合的价格） |
| n | String | 手续费金额 |
| N | String | 手续费币种，如 `"USDT"` |
| Z | String | 累计成交金额（该订单已成交的总金额） |
| v | String | 杠杆倍数 |
| m | Boolean | 是否 Maker 订单: `true` (挂单成交), `false` (吃单成交) |
| w | Boolean | 订单是否仍在订单簿中 |
| u | Boolean | 是否为正常订单 |
| O | Long | 订单创建时间戳（毫秒） |
| U | Long | 订单更新时间戳（毫秒） |
| A | Long | 对手方账户 ID |
| C | Boolean | 是否为平仓单: `true` (平仓), `false` (开仓) |
| M | Long | 对手方订单 ID |

**示例**:
```json
{
  "e": "contractExecutionReport",
  "E": 1767930000000,
  "s": "NVDA-PERP-USDT",
  "c": "test_mkt_1767930",
  "S": "BUY_OPEN",
  "o": "LIMIT",
  "f": "GTC",
  "q": "10000",
  "p": "0",
  "X": "FILLED",
  "i": 2123940267501303552,
  "l": "10000",
  "z": "10000",
  "L": "183.25",
  "n": "0.05",
  "N": "USDT",
  "Z": "1832.5",
  "v": "10",
  "m": false,
  "w": false,
  "u": true,
  "O": 1767930000000,
  "U": 1767930001000,
  "A": 2114553387206869251,
  "C": false,
  "M": 2123940267501303553
}
```

#### 5.4.2 持仓更新推送 (`outboundContractPositionInfo`)

**事件类型**: `outboundContractPositionInfo`

**触发时机**: 持仓数量或均价发生变动

**推送格式**: Object 或 Array of Objects

**字段详解** (`SocketFuturesPositionInfo`):
| 字段名 (JSON Key) | 类型 | 描述 |
| :--- | :--- | :--- |
| e | String | 事件类型，固定为 `"outboundContractPositionInfo"` |
| E | Long | 事件时间戳（毫秒） |
| A | String | 账户 ID |
| s | String | 合约交易对 ID |
| S | String | 持仓方向: `"LONG"` (多仓), `"SHORT"` (空仓) |
| p | String | 持仓均价（开仓平均成本价） |
| P | String | 持仓总量（当前持有的总张数） |
| a | String | 可平仓数量（未被挂单锁定的数量） |
| f | String | 强平价 (`flp`) |
| m | String | 仓位保证金（当前占用的保证金金额） |
| r | String | 已实现盈亏（累计已结算的真实盈亏） |
| up | String | 未实现盈亏（根据最新价计算的浮动盈亏） |
| pr | String | 盈亏率（未实现盈亏占比） |
| pv | String | 仓位价值（名义价值） |
| v | String | 杠杆倍数 |
| mt | String | 保证金模式: `"1"` (逐仓), `"2"` (全仓) |
| mm | String | 维持保证金（该仓位不被强平所需的最低保证金） |

**示例**:
```json
{
  "e": "outboundContractPositionInfo",
  "E": 1767930000000,
  "A": "2114553387206869251",
  "s": "NVDA-PERP-USDT",
  "S": "LONG",
  "p": "183.25",
  "P": "10000",
  "a": "10000",
  "f": "173.0",
  "m": "1832.5",
  "r": "0",
  "up": "50.0",
  "pr": "0.0273",
  "pv": "1832.5",
  "v": "10",
  "mt": "1",
      "mm": "549.75"
}
```

#### 5.4.3 账户余额推送 (`outboundContractAccountInfo`)

**事件类型**: `outboundContractAccountInfo`

**触发时机**: 账户资产发生变动（划转、手续费扣除、盈亏结算）

**推送格式**: Object 或 Array of Objects

**字段详解** (`SocketAccountInfo`):
| 字段名 (JSON Key) | 类型 | 描述 |
| :--- | :--- | :--- |
| e | String | 事件类型，固定为 `"outboundContractAccountInfo"` |
| E | Long | 事件时间戳（毫秒） |
| u | Long | 账户最后更新时间戳（毫秒） |
| m | String | Maker 手续费率 |
| t | String | Taker 手续费率 |
| b | String | 买方手续费率 |
| s | String | 卖方手续费率 |
| T | Boolean | 是否可以交易 |
| W | Boolean | 是否可以提现 |
| D | Boolean | 是否可以充值 |
| B | Array | 余额变动列表 (`SocketBalanceInfo` 数组) |
| └ a | String | 资产名称，如 `"USDT"` |
| └ f | String | 可用余额（自由可支配金额） |
| └ l | String | 锁定金额（挂单锁定的保证金） |

**示例**:
```json
{
  "e": "outboundContractAccountInfo",
  "E": 1767930000000,
  "u": 1767930000000,
  "m": "0.0002",
  "t": "0.0004",
  "b": "0.0004",
  "s": "0.0004",
  "T": true,
  "W": true,
  "D": true,
  "B": [
    {
      "a": "USDT",
      "f": "9167.91671864135",
      "l": "1832.5"
    }
  ]
}
```

---

## 6. 错误码详解

所有接口在发生错误时，都会返回以下格式的 JSON：

```json
{
  "code": -1005,
  "msg": "No Permission"
}
```

### 6.1 通用错误码（10xx）

| 错误码 | 错误名称 | 描述 | 解决方案 |
| :--- | :--- | :--- | :--- |
| -1000 | UNKNOWN | 未知错误 | 联系技术支持 |
| -1001 | DISCONNECTED | 内部错误，无法处理请求 | 重试请求 |
| -1002 | UNAUTHORIZED | 未授权 | 检查 API Key 是否正确 |
| -1003 | TOO_MANY_REQUESTS | 请求过于频繁 | 降低请求频率，使用 WebSocket |
| -1005 | NO_PERMISSION | 无权限 | **API Key 的账户类型不匹配**。合约接口需要 `account_type = 3 (FUTURES)`。请检查 API Key 配置或使用正确的账户类型 |
| -1006 | UNEXPECTED_RESP | 后端返回异常响应 | 联系技术支持 |
| -1007 | TIMEOUT | 请求超时 | 重试请求 |
| -1021 | INVALID_TIMESTAMP | 时间戳超出接收窗口 | 检查本地时间与服务器时间是否同步 |
| -1022 | INVALID_SIGNATURE | **签名验证失败** | **检查签名算法**：1) 参数是否按字母序排序；2) Query 和 Body 是否正确拼接（中间无分隔符）；3) HMAC-SHA256 计算是否正确；4) Secret Key 是否正确 |

### 6.2 请求参数错误（11xx）

| 错误码 | 错误名称 | 描述 | 解决方案 |
| :--- | :--- | :--- | :--- |
| -1102 | MANDATORY_PARAM_EMPTY_OR_MALFORMED | 必填参数缺失或格式错误 | 检查请求参数 |
| -1105 | PARAM_EMPTY | 参数为空 | 提供有效的参数值 |
| -1121 | BAD_SYMBOL | 无效的交易对 | 使用正确的合约 ID |
| -1122 | INVALID_ORDER_ID | 无效的订单 ID | 使用有效的订单 ID |
| -1123 | INVALID_CLIENT_ORDER_ID | 无效的客户端订单 ID | 使用有效的客户端订单 ID |
| -1124 | INVALID_PRICE | 无效的价格 | 检查价格格式和精度 |
| -1126 | INVALID_QUANTITY | 无效的数量 | 检查数量格式和精度 |
| -1130 | INVALID_PARAMETER | 参数值无效 | 检查参数值是否符合要求 |

### 6.3 订单相关错误（11xx, 12xx, 20xx）

| 错误码 | 错误名称 | 描述 | 解决方案 |
| :--- | :--- | :--- | :--- |
| -1115 | INVALID_TIF | 无效的 timeInForce | 使用 `GTC`, `IOC`, `FOK` 之一 |
| -1116 | INVALID_ORDER_TYPE | 无效的订单类型 | 使用 `LIMIT` 或 `STOP` |
| -1117 | INVALID_SIDE | 无效的订单方向 | 使用 `BUY_OPEN`, `SELL_OPEN`, `BUY_CLOSE`, `SELL_CLOSE` 之一 |
| -1132 | ORDER_PRICE_TOO_HIGH | 订单价格过高 | 调整价格 |
| -1133 | ORDER_PRICE_TOO_SMALL | 订单价格过低 | 调整价格 |
| -1135 | ORDER_QUANTITY_TOO_BIG | 订单数量过大 | 减少数量 |
| -1136 | ORDER_QUANTITY_TOO_SMALL | 订单数量过小 | **增加数量，检查合约最小下单单位** |
| -1141 | DUPLICATED_ORDER | 重复的客户端订单 ID | 使用唯一的 `clientOrderId` |
| -1142 | ORDER_CANCELLED | 订单已撤销 | 订单已被撤销，无法再次撤销 |
| -1149 | CREATE_ORDER_FAILED | 下单失败 | 检查账户余额、持仓限制等 |
| -1150 | CANCEL_ORDER_FAILED | 撤单失败 | 订单可能已成交或不存在 |
| -1200 | ORDER_BUY_QUANTITY_TOO_SMALL | **创建订单买入数量过小** | **增加下单数量，确保满足合约的最小下单单位要求** |
| -2013 | NO_SUCH_ORDER | **订单不存在** | 订单 ID 或客户端订单 ID 错误，或订单已被完全成交/撤销 |

### 6.4 WebSocket 相关错误

| 错误码 | 错误名称 | 描述 | 解决方案 |
| :--- | :--- | :--- | :--- |
| -1125 | INVALID_LISTEN_KEY | listenKey 不存在 | 重新获取 listenKey |