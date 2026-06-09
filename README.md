# StockCoin OpenAPI

Perpetual futures trading API for tokenized stocks. Provides REST endpoints for trading operations and WebSocket interfaces for real-time market data and private account streams.

## Table of Contents

- [Quick Start](#quick-start)
- [Installation](#installation)
- [Authentication](#authentication)
- [REST API](#rest-api)
  - [Public Endpoints](#public-endpoints)
  - [Private Endpoints](#private-endpoints)
- [WebSocket API](#websocket-api)
  - [Market Data Stream](#market-data-stream)
  - [Private Stream](#private-stream)
- [Code Examples](#code-examples)
- [Error Codes](#error-codes)
- [FAQ](#faq)
- [Changelog](#changelog)

---

## Quick Start

```python
import hmac, hashlib, time, urllib.parse, requests

API_KEY = "your_api_key"
SECRET_KEY = "your_secret_key"
BASE_URL = "https://api.stockcoin.ai"

# 1. Build signature (params MUST be sorted alphabetically by key)
params = {"timestamp": int(time.time() * 1000)}
query_string = urllib.parse.urlencode(sorted(params.items()))
signature = hmac.new(SECRET_KEY.encode(), query_string.encode(), hashlib.sha256).hexdigest()
params["signature"] = signature

# 2. Query account balance
resp = requests.get(
    f"{BASE_URL}/openapi/v1/futures/balance",
    params=params,
    headers={"X-BH-APIKEY": API_KEY}
)
print(resp.json())
```

---

## Installation

### Prerequisites

- Python 3.7+
- `requests` library
- `websocket-client` library (for WebSocket)

```bash
pip install requests websocket-client
```

### Configuration

Use environment variables or a config file excluded from version control:

```bash
export STOCKCOIN_API_KEY="your_api_key"
export STOCKCOIN_SECRET_KEY="your_secret_key"
```

> **Important**: Never hardcode API keys in source code.

---

## Authentication

### Endpoints

| Service | URL |
| :--- | :--- |
| REST API | `https://api.stockcoin.ai` |
| Market Data WS | `wss://wsapi.stockcoin.ai/ws/quote/v1` |
| Private WS | `wss://wsapi.stockcoin.ai/openapi/ws/{listenKey}` |

### API Key

All signed endpoints require the API key in HTTP header:

```
X-BH-APIKEY: <your_api_key>
```

- API key must be bound to a **FUTURES** account type (`account_type = 3`).
- Every signed request must include `timestamp` (ms) and `signature` as query parameters.

### Signature Algorithm (HMAC-SHA256)

**Step 1**: Collect all query string parameters and form-data body parameters. Exclude `signature` itself. If the body is a JSON array (e.g., batch orders), the body does **NOT** participate in signing.

**Step 2**: Sort all parameters by **key in alphabetical (ascending) order**.

**Step 3**: Serialize as URL-encoded strings — query params as `query_string`, body params as `body_string`.

**Step 4**: Concatenate directly: `total_params = query_string + body_string` (no `&` separator between them).

**Step 5**: Compute HMAC-SHA256:

```python
import hmac, hashlib

signature = hmac.new(
    secret_key.encode('utf-8'),
    total_params.encode('utf-8'),
    hashlib.sha256
).hexdigest()
```

**Step 6**: Append `signature` as a query parameter.

> **Critical**: Parameters MUST be sorted alphabetically by key before serialization. Failure to sort will cause `-1022 INVALID_SIGNATURE` errors.

#### Signature Examples

**GET Request**:

```
GET /openapi/v1/futures/balance?symbol=NVDA-PERP-USDT&timestamp=1767930000000

Step 1: Query params: {"symbol": "NVDA-PERP-USDT", "timestamp": 1767930000000}
Step 2: Sort by key: symbol, timestamp (already alphabetical)
Step 3: query_string = "symbol=NVDA-PERP-USDT&timestamp=1767930000000"
Step 4: total_params = "symbol=NVDA-PERP-USDT&timestamp=1767930000000"
Step 5: signature = HMAC_SHA256(secret_key, total_params)
Step 6: Final URL: ...?symbol=NVDA-PERP-USDT&timestamp=1767930000000&signature=<sig>
```

**POST Request (Form-Data body)**:

```
POST /openapi/v1/futures/order?timestamp=1767930000000
Body (form-data): symbol=NVDA-PERP-USDT&side=BUY_OPEN&quantity=10000&clientOrderId=test123

Step 1: Query params: {timestamp}; Body params: {symbol, side, quantity, clientOrderId}
Step 2: Sort each group by key
Step 3: query_string = "timestamp=1767930000000"
         body_string = "clientOrderId=test123&quantity=10000&side=BUY_OPEN&symbol=NVDA-PERP-USDT"
Step 4: total_params = "timestamp=1767930000000" + "clientOrderId=test123&quantity=10000&side=BUY_OPEN&symbol=NVDA-PERP-USDT"
        (no separator between query_string and body_string)
Step 5: signature = HMAC_SHA256(secret_key, total_params)
```

**POST Request (JSON Array body — e.g., batch orders)**:

```
POST /openapi/v1/futures/order/batch?timestamp=1767930000000
Body: [{"symbolId": "NVDA-PERP-USDT", ...}]   ← JSON Array, NOT signed

Step 1: Only query params participate: {timestamp}
Step 2: Sort: timestamp
Step 3: total_params = "timestamp=1767930000000"
Step 4: signature = HMAC_SHA256(secret_key, total_params)
```

---

## REST API

### Public Endpoints

No authentication required.

#### Get Contracts

`GET /openapi/v1/getContracts`

Alias: `GET /openapi/v1/contracts` (identical behavior)

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| expired | String | No | `"false"` | Include expired contracts (`"true"` / `"false"`) |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| symbol | String | Contract pair ID, e.g., `NVDA-PERP-USDT` |
| symbolName | String | Display name |
| baseToken | String | Base asset, e.g., `NVDA` |
| quoteToken | String | Quote asset, e.g., `USDT` |
| lastPrice | Decimal | Last traded price |
| baseVolume | Decimal | 24h base volume |
| quoteVolume | Decimal | 24h quote volume |
| bid | Decimal | Best bid price |
| ask | Decimal | Best ask price |
| high | Decimal | 24h high |
| low | Decimal | 24h low |
| productType | String | Product type |
| openInterest | String | Current open interest |
| indexPrice | Decimal | Index price |
| index | String | Index identifier |
| indexBaseToken | String | Index base token |
| startTs | Long | Contract listing timestamp (ms) |
| endTs | Long | Delivery/expiry timestamp (ms), `0` for perpetual |
| fundingRate | Decimal | Current funding rate |
| nextFundingRate | Decimal | Predicted next funding rate |
| nextFundingRateTs | Integer | Next settlement timestamp (ms) |

#### Get Insurance Fund

`GET /openapi/v1/futures/insurance`

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | No | - | Contract ID |
| fromId | Long | No | 0 | Starting record ID for pagination |
| limit | Integer | No | 20 | Max records (up to 500) |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| id | Long | Record ID |
| timestamp | Long | Timestamp (ms) |
| value | String | Insurance fund balance |
| unit | String | Asset unit, e.g., `USDT` |

#### Get Current Funding Rate

`GET /openapi/v1/futures/fundingRate`

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | No | - | Contract ID (omit for all) |
| state | String | No | `"current"` | Fixed as `"current"` |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| symbol | String | Contract ID |
| rate | String | Current funding rate |
| nextFundingTime | Long | Next settlement timestamp (ms) |

#### Get Funding Rate History

`GET /openapi/v1/futures/fundingRate/history`

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | No | - | Contract ID |
| limit | Integer | No | 20 | Max records (up to 1000) |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| id | Long | Settlement record ID |
| symbol | String | Contract ID |
| settleTime | Long | Settlement timestamp (ms) |
| settleRate | String | Funding rate at settlement |

---

### Private Endpoints

All endpoints require `X-BH-APIKEY` header, `timestamp` and `signature` query parameters.

#### Get Account Balance

`GET /openapi/v1/futures/balance`

No additional parameters.

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| balance | String | Total wallet balance |
| availableBalance | String | Available margin (includes cross unrealized PnL) |
| positionMargin | String | Margin used by positions |
| orderMargin | String | Margin locked by open orders |
| asset | String | Asset, e.g., `USDT` |
| crossUnRealizedPnl | String | Total cross unrealized PnL |

#### Get Positions

`GET /openapi/v1/futures/positions`

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | No | Contract ID (omit for all) |
| side | String | No | `LONG` or `SHORT` |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| symbol | String | Contract pair ID |
| side | String | `LONG` / `SHORT` |
| position | String | Total position (contracts) |
| available | String | Available to close (not locked by orders) |
| avgPrice | String | Average entry price |
| leverage | String | Leverage multiplier |
| lastPrice | String | Current market price |
| positionValue | String | Notional value |
| flp | String | Liquidation price |
| margin | String | Position margin |
| marginRate | String | Margin rate |
| unrealizedPnL | String | Unrealized PnL |
| profitRate | String | PnL ratio |
| realizedPnL | String | Cumulative realized PnL |
| minMargin | String | Minimum margin required to maintain position |

#### Place Order

`POST /openapi/v1/futures/order`

**Body (Form-Data)**:

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | Yes | - | Contract pair ID |
| side | String | Yes | - | `BUY_OPEN`, `SELL_OPEN`, `BUY_CLOSE`, `SELL_CLOSE` |
| orderType | String | Yes | - | `LIMIT` or `STOP` |
| quantity | String | Yes | - | Order quantity (contracts) |
| price | String | No | `"0"` | Price (`"0"` for market orders) |
| priceType | String | No | `"INPUT"` | `INPUT` (limit) or `MARKET` (market) |
| triggerPrice | String | No | - | Trigger price (only for `STOP` orders) |
| leverage | String | No | - | Leverage multiplier |
| timeInForce | String | No | `"GTC"` | `GTC`, `IOC`, or `FOK` |
| clientOrderId | String | Yes | - | Unique client order ID |

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| time | Long | Order creation time (ms) |
| updateTime | Long | Last update time (ms) |
| orderId | Long | System order ID |
| clientOrderId | String | Client order ID |
| symbol | String | Contract pair ID |
| price | String | Order price |
| leverage | String | Leverage |
| origQty | String | Original quantity |
| executedQty | String | Filled quantity |
| avgPrice | String | Average fill price |
| marginLocked | String | Locked margin |
| orderType | String | `LIMIT` / `STOP` |
| side | String | `BUY_OPEN` / `SELL_OPEN` / `BUY_CLOSE` / `SELL_CLOSE` |
| timeInForce | String | `GTC` / `IOC` / `FOK` |
| status | String | `NEW`, `PARTIALLY_FILLED`, `FILLED`, `CANCELED`, `REJECTED` |
| priceType | String | `INPUT`, `MARKET` |
| isClose | Boolean | Is close-position order |
| triggerPrice | String | Trigger price (for `STOP` orders) |

#### Query Order

`GET /openapi/v1/futures/order`

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract pair ID |
| orderId | Long | No* | System order ID |
| clientOrderId | String | No* | Client order ID |
| orderType | String | No | Default `LIMIT` |

> *Provide at least one of `orderId` or `clientOrderId`.

**Response**: Same as Place Order response.

#### Cancel Order

`DELETE /openapi/v1/futures/order`

Same parameters as Query Order. **Response**: Same as Place Order response.

#### Get Open Orders

`GET /openapi/v1/futures/orders/open`

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | No | - | Contract ID |
| orderId | Long | No | - | Starting order ID for pagination |
| orderType | String | No | `"LIMIT"` | `LIMIT` or `STOP` |
| limit | Integer | No | 20 | Max records (up to 500) |

**Response**: Array of order objects (same fields as Place Order response).

#### Get Order History

`GET /openapi/v1/futures/orders/history`

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | No | - | Contract ID |
| orderId | Long | No | 0 | Starting order ID |
| orderType | String | No | `"LIMIT"` | `LIMIT` or `STOP` |
| startTime | Long | No | 0 | Start timestamp (ms) |
| endTime | Long | No | 0 | End timestamp (ms) |
| limit | Integer | No | 20 | Max records (up to 500) |

**Response**: Array of order objects.

#### Get Trade History

`GET /openapi/v1/futures/trades`

| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| symbol | String | No | - | Contract ID |
| fromId | Long | No | 0 | Starting trade ID (forward pagination) |
| toId | Long | No | 0 | Ending trade ID (backward pagination) |
| startTime | Long | No | 0 | Start timestamp (ms) |
| endTime | Long | No | 0 | End timestamp (ms) |
| limit | Integer | No | 20 | Max records (up to 500) |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| time | Long | Trade timestamp (ms) |
| tradeId | Long | Trade ID |
| orderId | Long | Order ID |
| matchOrderId | Long | Counterparty order ID |
| symbolId | String | Contract pair ID |
| price | String | Trade price |
| quantity | String | Trade quantity (contracts) |
| feeTokenId | String | Fee token |
| fee | String | Fee amount |
| makerRebate | String | Maker rebate |
| orderType | String | Order type |
| side | String | Order side |
| pnl | String | PnL (for close-position trades) |

#### Get Risk Limit

`GET /openapi/v1/futures/riskLimit`

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract ID |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| riskLimitAmount | String | Max position size |
| maintainMargin | String | Maintenance margin rate (e.g., `0.03` = 3%) |
| initialMargin | String | Initial margin rate (e.g., `0.1` = 10%) |
| side | String | `BUY_OPEN` / `SELL_OPEN` |

#### Get Account Leverage

`GET /openapi/v1/futures/accountLeverage`

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract ID |

**Response** (Array):

| Field | Type | Description |
| :--- | :--- | :--- |
| symbolId | String | Contract ID |
| leverage | String | Current leverage |
| marginType | String | `ISOLATED` or `CROSSED` |

#### Get Commission Rate

`GET /openapi/v1/futures/commissionRate`

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract ID |

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| symbol | String | Contract ID |
| openMakerFee | String | Open position maker fee rate |
| openTakerFee | String | Open position taker fee rate |
| closeMakerFee | String | Close position maker fee rate |
| closeTakerFee | String | Close position taker fee rate |

#### Adjust Leverage

`POST /openapi/v1/futures/leverage`

**Body (Form-Data)**:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract ID |
| leverage | String | Yes | Target leverage, e.g., `"10"` |

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| code | Integer | `0` = success |
| symbolId | String | Contract ID |
| leverage | String | New leverage |

#### Adjust Margin Type

`POST /openapi/v1/futures/marginType`

**Body (Form-Data)**:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract ID |
| marginType | String | Yes | `ISOLATED` or `CROSSED` |

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| code | Integer | `0` = success |
| symbolId | String | Contract ID |
| marginType | String | New margin type |

#### Modify Isolated Margin

`POST /openapi/v1/futures/margin/modify`

**Body (Form-Data)**:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract ID |
| side | String | Yes | `LONG` or `SHORT` |
| amount | String | Yes | Positive to add, negative to reduce |

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| code | Integer | `0` = success |
| msg | String | Message |
| symbol | String | Contract ID |
| margin | String | Updated margin |
| timestamp | Long | Timestamp (ms) |

#### Batch Place Orders

`POST /openapi/v1/futures/order/batch`

**Query Parameters**: `timestamp`, `signature`

**Body (JSON Array)** — body does **NOT** participate in signing:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbolId | String | Yes | Contract ID |
| orderSide | String | Yes | `BUY_OPEN`, `SELL_OPEN`, `BUY_CLOSE`, `SELL_CLOSE` |
| orderType | String | Yes | `LIMIT` or `STOP` |
| quantity | String | Yes | Quantity (contracts) |
| price | String | No | Price (`"0"` for market) |
| priceType | String | No | `INPUT` or `MARKET` |
| clientOrderId | String | Yes | Client order ID |

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| code | Integer | Status code |
| result | Array | Per-order results |
| result[].code | Integer | Individual order status code |
| result[].msg | String | Message |
| result[].order | Object | Order info (same as Place Order response) |

#### Batch Cancel by Symbol

`DELETE /openapi/v1/futures/cancel/batch/symbol`

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| symbol | String | Yes | Contract IDs (comma-separated) |
| side | String | No | Filter by side: `BUY_OPEN`, `SELL_OPEN`, etc. |

**Response**:

```json
{"code": 200, "msg": "success", "timestamp": 1767930000000}
```

#### Batch Cancel by IDs

`DELETE /openapi/v1/futures/cancel/batch/ids`

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| ids | String | Yes | Comma-separated order IDs |

**Response** (Object):

| Field | Type | Description |
| :--- | :--- | :--- |
| code | Integer | Status code |
| result | Array | Per-order cancel results |
| result[].orderId | Long | Order ID |
| result[].code | Integer | `0` = success |

#### Get ListenKey

`POST /openapi/v1/userdata`

Signed request. Returns a `listenKey` valid for 60 minutes.

**Response**:

```json
{"listenKey": "aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789aBcDeFgHiJkLmNoPqRsTuVwXyZ01"}
```

---

## WebSocket API

### Market Data Stream

#### Connection

```
wss://wsapi.stockcoin.ai/ws/quote/v1
```

**Symbol format**: `{exchangeId}.{symbol}`, e.g., `301.NVDA-PERP-USDT`

#### Subscribe / Unsubscribe

```json
{
  "id": "<unique_id>",
  "topic": "<topic_name>",
  "event": "sub",
  "symbol": "301.NVDA-PERP-USDT",
  "params": {}
}
```

| event | Description |
| :--- | :--- |
| `sub` | Subscribe |
| `cancel` | Unsubscribe a specific topic |
| `cancel_all` | Unsubscribe all topics |

#### Data Compression

Set `params.binary` to `"true"` to receive compressed binary data:

```json
{"params": {"binary": "true"}}
```

#### Topics

| Topic | Description | Key Params |
| :--- | :--- | :--- |
| `realtimes` | 24h Ticker | `realtimeInterval: "24h"` (or `"1d"`, `"1d+8"`) |
| `depth` | Full Order Book | `limit: 100` |
| `diffDepth` | Incremental Depth | First message is full snapshot, then incremental |
| `kline_{interval}` | Candlestick | `klineType`, `limit` (max 2000) |
| `trade` | Recent Trades | `limit: 60` |
| `mergedDepth` | Merged Depth | `dumpScale: "2"` (decimal places) |
| `diffMergedDepth` | Incremental Merged Depth | `dumpScale: "2"` |
| `index` | Index Price | - |
| `slowBroker` | Broker Full Ticker | `org: "6001"` |
| `topN` | Top Movers by Change | `limit: "10"`, `org: "6001"` |

**KLine Intervals**: `1m`, `5m`, `15m`, `30m`, `1h`, `2h`, `4h`, `6h`, `12h`, `1d`, `1d+8`, `1w`, `1w+8`, `1M`, `1M+8`

#### Subscribe Examples

**24h Ticker**:

```json
{
  "id": "realtimes",
  "topic": "realtimes",
  "event": "sub",
  "symbol": "301.NVDA-PERP-USDT",
  "params": {"realtimeInterval": "24h"}
}
```

**Order Book Depth**:

```json
{
  "id": "depth301.NVDA-PERP-USDT",
  "topic": "depth",
  "event": "sub",
  "symbol": "301.NVDA-PERP-USDT",
  "limit": 100,
  "params": {"binary": "true"}
}
```

**KLine (15m)**:

```json
{
  "id": "kline_301NVDA-PERP-USDT15m",
  "topic": "kline_15m",
  "event": "sub",
  "symbol": "301.NVDA-PERP-USDT",
  "params": {"klineType": "15m", "limit": 1500, "binary": "true"}
}
```

**Recent Trades**:

```json
{
  "id": "trade301.NVDA-PERP-USDT",
  "topic": "trade",
  "event": "sub",
  "symbol": "301.NVDA-PERP-USDT",
  "limit": 60,
  "params": {"binary": "true"}
}
```

**Merged Depth**:

```json
{
  "id": "301.NVDA-PERP-USDT2",
  "topic": "mergedDepth",
  "event": "sub",
  "symbol": "301.NVDA-PERP-USDT",
  "limit": 22,
  "params": {"dumpScale": "2", "binary": "true"}
}
```

**Index Price**:

```json
{
  "id": "index",
  "topic": "index",
  "event": "sub",
  "symbol": "NVDA-PERP-USDT"
}
```

**Broker Full Ticker (slowBroker)**:

```json
{
  "id": "broker",
  "topic": "slowBroker",
  "event": "sub",
  "params": {"org": "6001", "realtimeInterval": "24h", "binary": "true"}
}
```

**Top Movers (topN)**:

```json
{
  "id": "index",
  "topic": "topN",
  "event": "sub",
  "params": {"limit": "10", "org": "6001", "binary": "false"}
}
```

#### Response Envelope

All messages follow this envelope format:

```json
{
  "symbol": "301.NVDA-PERP-USDT",
  "topic": "realtimes",
  "f": true,
  "data": [{...}, {...}],
  "params": {}
}
```

| Field | Description |
| :--- | :--- |
| `f` | `true` = first full snapshot; `false` = incremental update |
| `data` | Array of data objects, index 0 = oldest, last = newest |

#### Topic Data Fields

**realtimes**:

```json
{
  "t": "1736933460000",
  "s": "NVDA-PERP-USDT",
  "c": "183.25",
  "h": "185.50",
  "l": "182.80",
  "o": "183.00",
  "v": "12550.5",
  "e": "301"
}
```

| Field | Description |
| :--- | :--- |
| t | Timestamp (ms) |
| s | Symbol |
| o | Open price |
| h | High price |
| l | Low price |
| c | Close price |
| v | Volume |
| e | Exchange ID |

**kline_{interval}**:

Same fields as `realtimes` (without `e`).

**depth**:

```json
{
  "t": 1736933400000,
  "s": "NVDA-PERP-USDT",
  "a": [["183.25", "155.92"], ["183.30", "263.53"]],
  "b": [["183.20", "155.92"], ["183.15", "263.53"]]
}
```

| Field | Description |
| :--- | :--- |
| t | Timestamp (ms) |
| s | Symbol |
| a | Asks `[price, qty]` sorted low → high |
| b | Bids `[price, qty]` sorted high → low |

**trade**:

```json
{
  "t": 1736933520000,
  "p": "183.25",
  "q": "15.2",
  "m": true,
  "v": "123"
}
```

| Field | Description |
| :--- | :--- |
| t | Timestamp (ms) |
| p | Price |
| q | Quantity |
| m | `true` = buy, `false` = sell |
| v | Version |

**index**:

```json
{
  "symbol": "NVDA-PERP-USDT",
  "index": "183.25",
  "edp": "183.30",
  "formula": "(183.20[SOURCE_A]+183.25[SOURCE_B]+183.28[SOURCE_C]+183.27[SOURCE_D])/4"
}
```

| Field | Description |
| :--- | :--- |
| symbol | Symbol |
| index | Index price |
| edp | Estimated delivery price |
| formula | Index calculation formula (source prices) |

**slowBroker**:

```json
{
  "t": 1736933460000,
  "s": "NVDA-PERP-USDT",
  "sn": "NVDA-PERP-USDT",
  "c": "183.25",
  "h": "185.50",
  "l": "182.80",
  "o": "183.00",
  "v": "125500",
  "qv": "22983750",
  "m": "0.0014",
  "e": 301
}
```

| Field | Description |
| :--- | :--- |
| t | Timestamp (ms) |
| s | Symbol |
| sn | Symbol name |
| c / h / l / o | Close / High / Low / Open |
| v | Volume |
| qv | Quote volume (price × volume) |
| m | Price change ratio (24h) |
| e | Exchange ID |

**mergedDepth**:

```json
[{
  "e": 301,
  "s": "NVDA-PERP-USDT",
  "t": 1736933460000,
  "v": "237679504_2",
  "b": [["183.20", "26.34"], ["183.15", "20.50"]],
  "a": [["183.25", "23.11"], ["183.30", "14.20"]],
  "o": 0
}]
```

| Field | Description |
| :--- | :--- |
| e | Exchange ID |
| s | Symbol |
| t | Timestamp (ms) |
| v | Version |
| b | Bids `[price, qty]` high → low |
| a | Asks `[price, qty]` low → high |
| o | Send order (sequence) |

#### Incremental Depth (diffDepth / diffMergedDepth)

When subscribed to `diffDepth` or `diffMergedDepth`, the **first** message (`f: true`) is a full snapshot. All subsequent messages are incremental updates.

**Processing incremental updates:**

Each entry `[price, quantity]` in the update should be handled as:

1. **quantity > 0**: Price level changed — update or insert at the correct position.
2. **quantity = 0**: Price level removed — delete from local order book.

**Example:**

```
Initial state: bids = []

Receive: bids = [["183.20", "100"]]
Result:  bids = [["183.20", "100"]]       ← INSERT

Receive: bids = [["183.20", "250"]]
Result:  bids = [["183.20", "250"]]       ← UPDATE (same price, new qty)

Receive: bids = [["183.20", "0"]]
Result:  bids = []                         ← DELETE (qty = 0)
```

#### Params Reference

| Param | Type | Required | Supported Topics | Description |
| :--- | :--- | :--- | :--- | :--- |
| binary | String | No | All | `"true"` to enable data compression |
| org | String | No | slowBroker, topN | Broker org ID |
| limit | String | No | kline, topN | Snapshot count (kline max 2000; topN default 5) |
| klineType | String | Yes (for kline) | kline | KLine interval |
| dumpScale | String | Yes (for merged) | mergedDepth, diffMergedDepth | Merge precision (decimal places) |
| realtimeInterval | String | No | realtimes, slowBroker | `"24h"` (default), `"1d"`, `"1d+8"` |

#### Heartbeat

Client should send ping every 1–2 minutes (server idle timeout: 5 min):

```json
// Client sends:
{"ping": 1736933460000}

// Server responds:
{"pong": 1736933460000}
```

WebSocket ping frames are also supported.

---

### Private Stream

#### Connection Flow

```
1. POST /openapi/v1/userdata         → get listenKey (valid 60 min)
2. Connect wss://wsapi.stockcoin.ai/openapi/ws/{listenKey}
3. Respond to server pings with pong  → must reply within 3 min
4. Renew listenKey every 30–50 min    → re-call POST /openapi/v1/userdata
```

#### Heartbeat

Server sends ping periodically. Client **must** reply within 3 minutes or the connection will be closed:

```json
// Server sends:
{"ping": 1767930000000}

// Client replies:
{"pong": 1767930000000}
```

#### Event: `contractExecutionReport` — Order Update

Triggered when order status changes (created, partially filled, filled, canceled, rejected).

| Field | Type | Description |
| :--- | :--- | :--- |
| e | String | `"contractExecutionReport"` |
| E | Long | Event timestamp (ms) |
| s | String | Symbol |
| c | String | Client order ID |
| S | String | Side: `BUY_OPEN`, `SELL_OPEN`, `BUY_CLOSE`, `SELL_CLOSE` |
| o | String | Order type: `LIMIT`, `STOP` |
| f | String | Time in force: `GTC`, `IOC`, `FOK` |
| q | String | Original quantity |
| p | String | Order price (`"0"` for market) |
| X | String | Status: `NEW`, `PARTIALLY_FILLED`, `FILLED`, `CANCELED`, `REJECTED` |
| i | Long | System order ID |
| l | String | Last filled quantity (this update) |
| z | String | Cumulative filled quantity |
| L | String | Last filled price |
| Z | String | Cumulative filled amount |
| n | String | Fee amount |
| N | String | Fee asset |
| v | String | Leverage |
| m | Boolean | Is maker (`true` = maker, `false` = taker) |
| w | Boolean | Is still in order book |
| O | Long | Order creation timestamp (ms) |
| U | Long | Order update timestamp (ms) |
| C | Boolean | Is close-position order |

**Example**:

```json
{
  "e": "contractExecutionReport",
  "E": 1767930000000,
  "s": "NVDA-PERP-USDT",
  "c": "test_mkt_001",
  "S": "BUY_OPEN",
  "o": "LIMIT",
  "f": "GTC",
  "q": "10000",
  "p": "0",
  "X": "FILLED",
  "i": 1000000000000000001,
  "l": "10000",
  "z": "10000",
  "L": "183.25",
  "Z": "1832.5",
  "n": "0.05",
  "N": "USDT",
  "v": "10",
  "m": false,
  "w": false,
  "O": 1767930000000,
  "U": 1767930001000,
  "C": false
}
```

#### Event: `outboundContractPositionInfo` — Position Update

Triggered when position quantity or average price changes.

| Field | Type | Description |
| :--- | :--- | :--- |
| e | String | `"outboundContractPositionInfo"` |
| E | Long | Event timestamp (ms) |
| s | String | Symbol |
| S | String | Side: `LONG` / `SHORT` |
| p | String | Average entry price |
| P | String | Total position (contracts) |
| a | String | Available to close |
| f | String | Liquidation price |
| m | String | Position margin |
| r | String | Realized PnL (cumulative) |
| up | String | Unrealized PnL |
| pr | String | PnL ratio |
| pv | String | Position value (notional) |
| v | String | Leverage |
| mt | String | Margin type: `"1"` = isolated, `"2"` = cross |
| mm | String | Maintenance margin |

**Example**:

```json
{
  "e": "outboundContractPositionInfo",
  "E": 1767930000000,
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

#### Event: `outboundContractAccountInfo` — Balance Update

Triggered when account assets change (transfer, fee deduction, PnL settlement).

| Field | Type | Description |
| :--- | :--- | :--- |
| e | String | `"outboundContractAccountInfo"` |
| E | Long | Event timestamp (ms) |
| u | Long | Last account update timestamp (ms) |
| T | Boolean | Can trade |
| W | Boolean | Can withdraw |
| D | Boolean | Can deposit |
| B | Array | Balance list |
| B[].a | String | Asset name, e.g., `"USDT"` |
| B[].f | String | Available balance |
| B[].l | String | Locked balance |

**Example**:

```json
{
  "e": "outboundContractAccountInfo",
  "E": 1767930000000,
  "u": 1767930000000,
  "T": true,
  "W": true,
  "D": true,
  "B": [
    {"a": "USDT", "f": "9167.91671864", "l": "1832.5"}
  ]
}
```

---

## Code Examples

Two complete, runnable scripts are provided below. Copy either script, fill in your API credentials, and run directly.

### Example 1: REST API — Complete Contract Trading Demo

> **File**: `test_all_contract_apis.py` — Demonstrates signature generation, all public & private REST endpoints.

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# ============================================================
# StockCoin Contract REST API — Complete Demo
# ============================================================
# Setup:
#   python3 -m venv venv && source venv/bin/activate
#   pip install requests
#   python3 test_all_contract_apis.py
# ============================================================
import json
import time
import hmac
import hashlib
import urllib.parse
import requests
from datetime import datetime


class ContractAPITester:
    """Complete REST API client with signature support."""

    def __init__(self, api_key, secret_key,
                 base_url="https://api.stockcoin.ai",
                 test_symbol="NVDA-PERP-USDT"):
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.secret_key = secret_key
        self.test_symbol = test_symbol

    # ── Signature ────────────────────────────────────────────

    def _generate_signature(self, query_string):
        """HMAC-SHA256 signature."""
        return hmac.new(
            self.secret_key.encode("utf-8"),
            query_string.encode("utf-8"),
            hashlib.sha256,
        ).hexdigest()

    def _build_signing_string(self, params, data=None):
        """
        Build the string to sign.

        Key rules:
        1. Collect query params and form-data body params separately.
        2. Each group is URL-encoded (requests.Request.prepare() ensures
           alphabetical sorting, matching the server's behaviour).
        3. Concatenate directly: query_string + body_string (NO separator).
        4. If the body is a JSON Array (e.g., batch orders), it does NOT
           participate in signing.
        """
        # Query part — use requests.Request.prepare() to get sorted encoding
        query_part = ""
        if params:
            prep = requests.Request("GET", "http://x", params=params).prepare()
            query_part = urllib.parse.urlparse(prep.url).query
            # Remove signature= if accidentally included
            parts = [p for p in query_part.split("&") if not p.startswith("signature=")]
            query_part = "&".join(parts)

        # Body part — only for form-data dicts, NOT for JSON arrays
        body_part = ""
        if data and isinstance(data, dict):
            body_part = urllib.parse.urlencode(sorted(data.items()))

        return query_part + body_part

    # ── HTTP Request ─────────────────────────────────────────

    def _request(self, method, endpoint, params=None, data=None, signed=False):
        """
        Send an HTTP request.

        Args:
            method:   GET / POST / DELETE
            endpoint: API path, e.g., "/openapi/v1/futures/balance"
            params:   Query string parameters (dict)
            data:     Body — dict for form-data, list for JSON array
            signed:   Whether to sign the request

        Returns:
            Response JSON (dict or list)
        """
        url = f"{self.base_url}{endpoint}"
        headers = {}
        params = dict(params) if params else {}

        if signed:
            headers["X-BH-APIKEY"] = self.api_key
            params["timestamp"] = int(time.time() * 1000)

            # Form-data dict participates in signing; JSON list does not
            sig_data = data if isinstance(data, dict) else None
            sig_string = self._build_signing_string(params, sig_data)
            params["signature"] = self._generate_signature(sig_string)

        if method == "GET":
            resp = requests.get(url, params=params, headers=headers)
        elif method == "POST":
            if isinstance(data, list):
                resp = requests.post(url, params=params, json=data, headers=headers)
            elif isinstance(data, dict):
                resp = requests.post(url, params=params, data=data, headers=headers)
            else:
                resp = requests.post(url, params=params, headers=headers)
        elif method == "DELETE":
            resp = requests.delete(url, params=params, headers=headers)
        else:
            raise ValueError(f"Unsupported HTTP method: {method}")

        return resp.json()

    # ── Public Endpoints (no signature) ──────────────────────

    def get_contracts(self):
        """Get all available contracts."""
        return self._request("GET", "/openapi/v1/getContracts", {"expired": "false"})

    def get_insurance(self, symbol=None, limit=20):
        """Get insurance fund records."""
        params = {"limit": limit}
        if symbol:
            params["symbol"] = symbol
        return self._request("GET", "/openapi/v1/futures/insurance", params)

    def get_funding_rate(self, symbol=None):
        """Get current funding rate."""
        params = {"state": "current"}
        if symbol:
            params["symbol"] = symbol
        return self._request("GET", "/openapi/v1/futures/fundingRate", params)

    def get_funding_rate_history(self, symbol=None, limit=20):
        """Get historical funding rate records."""
        params = {"limit": limit}
        if symbol:
            params["symbol"] = symbol
        return self._request("GET", "/openapi/v1/futures/fundingRate/history", params)

    # ── Private Endpoints (signed) ───────────────────────────

    def get_balance(self):
        """Get futures account balance."""
        return self._request("GET", "/openapi/v1/futures/balance", signed=True)

    def get_positions(self, symbol=None, side=None):
        """Get current positions."""
        params = {}
        if symbol:
            params["symbol"] = symbol
        if side:
            params["side"] = side
        return self._request("GET", "/openapi/v1/futures/positions", params, signed=True)

    def place_order(self, symbol, side, order_type, quantity,
                    price="0", price_type="INPUT", leverage="10",
                    time_in_force="GTC", client_order_id=None):
        """
        Place a new order.

        Args:
            symbol:          Contract ID, e.g., "NVDA-PERP-USDT"
            side:            BUY_OPEN / SELL_OPEN / BUY_CLOSE / SELL_CLOSE
            order_type:      LIMIT / STOP
            quantity:        Number of contracts
            price:           Order price ("0" for market orders)
            price_type:      INPUT (limit) / MARKET (market)
            leverage:        Leverage multiplier, e.g., "10"
            time_in_force:   GTC / IOC / FOK
            client_order_id: Unique client order ID
        """
        data = {
            "symbol": symbol,
            "side": side,
            "orderType": order_type,
            "quantity": str(quantity),
            "price": str(price),
            "priceType": price_type,
            "leverage": str(leverage),
            "timeInForce": time_in_force,
            "clientOrderId": client_order_id or f"order_{int(time.time() * 1000)}",
        }
        return self._request("POST", "/openapi/v1/futures/order", data=data, signed=True)

    def get_order(self, symbol, order_id=None, client_order_id=None, order_type="LIMIT"):
        """Query a specific order by orderId or clientOrderId."""
        params = {"symbol": symbol, "orderType": order_type}
        if order_id:
            params["orderId"] = order_id
        if client_order_id:
            params["clientOrderId"] = client_order_id
        return self._request("GET", "/openapi/v1/futures/order", params, signed=True)

    def cancel_order(self, symbol, order_id=None, client_order_id=None, order_type="LIMIT"):
        """Cancel a specific order."""
        params = {"symbol": symbol, "orderType": order_type}
        if order_id:
            params["orderId"] = order_id
        if client_order_id:
            params["clientOrderId"] = client_order_id
        return self._request("DELETE", "/openapi/v1/futures/order", params, signed=True)

    def get_open_orders(self, symbol=None, order_type="LIMIT", limit=20):
        """Get all open (unfilled) orders."""
        params = {"orderType": order_type, "limit": limit}
        if symbol:
            params["symbol"] = symbol
        return self._request("GET", "/openapi/v1/futures/orders/open", params, signed=True)

    def get_history_orders(self, symbol=None, order_type="LIMIT", limit=20,
                           start_time=0, end_time=0):
        """Get historical orders."""
        params = {"orderType": order_type, "limit": limit}
        if symbol:
            params["symbol"] = symbol
        if start_time:
            params["startTime"] = start_time
        if end_time:
            params["endTime"] = end_time
        return self._request("GET", "/openapi/v1/futures/orders/history", params, signed=True)

    def get_trades(self, symbol, from_id=0, to_id=0, start_time=0, end_time=0, limit=20):
        """Get trade (fill) history."""
        params = {"symbol": symbol, "limit": limit}
        if from_id:
            params["fromId"] = from_id
        if to_id:
            params["toId"] = to_id
        if start_time:
            params["startTime"] = start_time
        if end_time:
            params["endTime"] = end_time
        return self._request("GET", "/openapi/v1/futures/trades", params, signed=True)

    def get_risk_limit(self, symbol):
        """Get risk limit tiers for a symbol."""
        return self._request("GET", "/openapi/v1/futures/riskLimit",
                             {"symbol": symbol}, signed=True)

    def get_account_leverage(self, symbol):
        """Get current leverage and margin type."""
        return self._request("GET", "/openapi/v1/futures/accountLeverage",
                             {"symbol": symbol}, signed=True)

    def get_commission_rate(self, symbol):
        """Get trading fee rates."""
        return self._request("GET", "/openapi/v1/futures/commissionRate",
                             {"symbol": symbol}, signed=True)

    def adjust_leverage(self, symbol, leverage):
        """Change leverage for a symbol."""
        return self._request("POST", "/openapi/v1/futures/leverage",
                             data={"symbol": symbol, "leverage": str(leverage)}, signed=True)

    def adjust_margin_type(self, symbol, margin_type):
        """Switch between ISOLATED and CROSSED margin."""
        return self._request("POST", "/openapi/v1/futures/marginType",
                             data={"symbol": symbol, "marginType": margin_type}, signed=True)

    def modify_margin(self, symbol, side, amount):
        """Add or reduce isolated margin. Positive = add, negative = reduce."""
        return self._request("POST", "/openapi/v1/futures/margin/modify",
                             data={"symbol": symbol, "side": side, "amount": str(amount)},
                             signed=True)

    def batch_place_orders(self, orders):
        """
        Place multiple orders at once.

        Args:
            orders: List of order dicts. JSON array body — NOT signed.
                    Each dict: {symbolId, orderSide, orderType, quantity, price,
                                priceType, clientOrderId}
        """
        return self._request("POST", "/openapi/v1/futures/order/batch",
                             data=orders, signed=True)

    def batch_cancel_by_symbol(self, symbol, side=None):
        """Cancel all open orders for given symbol(s)."""
        params = {"symbol": symbol}
        if side:
            params["side"] = side
        return self._request("DELETE", "/openapi/v1/futures/cancel/batch/symbol",
                             params, signed=True)

    def batch_cancel_by_ids(self, ids):
        """Cancel orders by comma-separated order IDs."""
        return self._request("DELETE", "/openapi/v1/futures/cancel/batch/ids",
                             {"ids": ids}, signed=True)

    def get_listen_key(self):
        """Get a listenKey for private WebSocket stream (valid 60 min)."""
        return self._request("POST", "/openapi/v1/userdata", signed=True)

    # ── Test Runner ──────────────────────────────────────────

    def run_all_tests(self):
        """Run through all API endpoints and print results."""
        symbol = self.test_symbol
        print(f"\n{'='*80}")
        print(f"  StockCoin Contract API Test")
        print(f"  Time:   {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"  Server: {self.base_url}")
        print(f"  Symbol: {symbol}")
        print(f"{'='*80}\n")

        tests = [
            # ── Public ──
            ("Get Contracts",            lambda: self.get_contracts()),
            ("Get Insurance Fund",       lambda: self.get_insurance(symbol, 5)),
            ("Get Funding Rate",         lambda: self.get_funding_rate(symbol)),
            ("Get Funding Rate History", lambda: self.get_funding_rate_history(symbol, 5)),

            # ── Account ──
            ("Get Balance",              lambda: self.get_balance()),
            ("Get Positions",            lambda: self.get_positions(symbol)),
            ("Get Account Leverage",     lambda: self.get_account_leverage(symbol)),
            ("Get Commission Rate",      lambda: self.get_commission_rate(symbol)),
            ("Get Risk Limit",           lambda: self.get_risk_limit(symbol)),

            # ── Order Lifecycle ──
            ("Place Limit Order",        lambda: self.place_order(
                symbol=symbol, side="BUY_OPEN", order_type="LIMIT",
                quantity="1000", price="100.00", leverage="10",
                client_order_id=f"demo_{int(time.time()*1000)}")),
            ("Get Open Orders",          lambda: self.get_open_orders(symbol)),
            ("Get History Orders",       lambda: self.get_history_orders(symbol, limit=5)),
            ("Get Trades",               lambda: self.get_trades(symbol, limit=5)),

            # ── Leverage & Margin ──
            ("Adjust Leverage",          lambda: self.adjust_leverage(symbol, "10")),

            # ── Batch Operations ──
            ("Batch Place Orders",       lambda: self.batch_place_orders([
                {
                    "symbolId": symbol,
                    "orderSide": "BUY_OPEN",
                    "orderType": "LIMIT",
                    "quantity": "1000",
                    "price": "100.00",
                    "priceType": "INPUT",
                    "clientOrderId": f"batch_{int(time.time()*1000)}_1",
                }
            ])),
            ("Batch Cancel by Symbol",   lambda: self.batch_cancel_by_symbol(symbol)),

            # ── WebSocket Listen Key ──
            ("Get Listen Key",           lambda: self.get_listen_key()),
        ]

        passed, failed = 0, 0
        for name, fn in tests:
            try:
                result = fn()
                status = "PASS"
                # Check for API error codes
                if isinstance(result, dict) and result.get("code") and result["code"] < 0:
                    status = "FAIL"
                    failed += 1
                else:
                    passed += 1
                print(f"  [{status}] {name}")
                print(f"         {json.dumps(result, ensure_ascii=False)[:200]}")
            except Exception as e:
                failed += 1
                print(f"  [FAIL] {name}")
                print(f"         Error: {e}")
            time.sleep(0.2)

        print(f"\n{'='*80}")
        print(f"  Results: {passed} passed, {failed} failed, {passed+failed} total")
        print(f"{'='*80}\n")


if __name__ == "__main__":
    # ── Configuration ────────────────────────────────────────
    # Option 1: Set environment variables
    #   export STOCKCOIN_API_KEY="your_api_key"
    #   export STOCKCOIN_SECRET_KEY="your_secret_key"
    #
    # Option 2: Replace the strings below directly (NOT recommended for production)
    import os

    tester = ContractAPITester(
        api_key=os.environ.get("STOCKCOIN_API_KEY", "your_api_key"),
        secret_key=os.environ.get("STOCKCOIN_SECRET_KEY", "your_secret_key"),
        base_url="https://api.stockcoin.ai",
        test_symbol="NVDA-PERP-USDT",
    )
    tester.run_all_tests()
```

---

### Example 2: WebSocket — Private Stream (Order / Position / Balance)

> **File**: `test_contract_ws.py` — Connects to the private WebSocket, handles heartbeat, and prints real-time account events.

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# ============================================================
# StockCoin Contract WebSocket — Private Stream Demo
# ============================================================
# Setup:
#   python3 -m venv venv && source venv/bin/activate
#   pip install requests websocket-client
#   python3 test_contract_ws.py
# ============================================================
import json
import time
import hmac
import hashlib
import urllib.parse
import requests
import websocket
from datetime import datetime


class ContractWSTester:
    """Private WebSocket client for real-time order, position and balance updates."""

    def __init__(self, api_key, secret_key,
                 base_url="https://api.stockcoin.ai",
                 ws_base="wss://wsapi.stockcoin.ai"):
        self.api_key = api_key
        self.secret_key = secret_key
        self.base_url = base_url.rstrip("/")
        self.ws_base = ws_base.rstrip("/")
        self.listen_key = None

    # ── Signature (reused from REST client) ──────────────────

    def _generate_signature(self, query_string):
        return hmac.new(
            self.secret_key.encode("utf-8"),
            query_string.encode("utf-8"),
            hashlib.sha256,
        ).hexdigest()

    # ── Get Listen Key via REST ──────────────────────────────

    def get_listen_key(self):
        """
        Call POST /openapi/v1/userdata to obtain a listenKey.
        The listenKey is valid for 60 minutes. Renew it every 30-50 min.
        """
        url = f"{self.base_url}/openapi/v1/userdata"
        params = {"timestamp": int(time.time() * 1000)}
        query_string = urllib.parse.urlencode(sorted(params.items()))
        params["signature"] = self._generate_signature(query_string)
        headers = {"X-BH-APIKEY": self.api_key}

        resp = requests.post(url, params=params, headers=headers)
        if resp.status_code == 200:
            self.listen_key = resp.json().get("listenKey")
            print(f"[OK] Got listenKey: {self.listen_key[:20]}...")
            return self.listen_key
        else:
            print(f"[ERROR] Failed to get listenKey: {resp.status_code} {resp.text}")
            return None

    # ── Message Processing ───────────────────────────────────

    def process_message(self, data):
        """Route incoming messages to the appropriate handler."""
        if isinstance(data, list):
            for item in data:
                self.process_message(item)
            return

        if not isinstance(data, dict):
            print(f"[RAW] {data}")
            return

        event = data.get("e")
        EVENT_LABELS = {
            "contractExecutionReport":       "ORDER UPDATE",
            "outboundContractPositionInfo":   "POSITION UPDATE",
            "outboundContractAccountInfo":    "BALANCE UPDATE",
        }

        label = EVENT_LABELS.get(event)
        if label:
            print(f"\n{'='*25} {label} {'='*25}")
            print(f"Time: {datetime.now().strftime('%H:%M:%S')}")

            # Print key fields based on event type
            if event == "contractExecutionReport":
                print(f"  Symbol:     {data.get('s')}")
                print(f"  Side:       {data.get('S')}")
                print(f"  Status:     {data.get('X')}")
                print(f"  Price:      {data.get('p')}")
                print(f"  Filled:     {data.get('z')} / {data.get('q')}")
                print(f"  Last Price: {data.get('L')}")
                print(f"  Fee:        {data.get('n')} {data.get('N')}")
                print(f"  OrderId:    {data.get('i')}")
                print(f"  ClientId:   {data.get('c')}")

            elif event == "outboundContractPositionInfo":
                print(f"  Symbol:     {data.get('s')}")
                print(f"  Side:       {data.get('S')}")
                print(f"  Size:       {data.get('P')}")
                print(f"  AvgPrice:   {data.get('p')}")
                print(f"  Margin:     {data.get('m')}")
                print(f"  uPnL:       {data.get('up')}")
                print(f"  Leverage:   {data.get('v')}")
                print(f"  Liq.Price:  {data.get('f')}")

            elif event == "outboundContractAccountInfo":
                for b in data.get("B", []):
                    print(f"  {b['a']}: free={b['f']}  locked={b['l']}")

            print(f"{'='*65}\n")
        else:
            print(f"[UNKNOWN EVENT: {event}] {json.dumps(data, indent=2, ensure_ascii=False)}")

    # ── WebSocket Callbacks ──────────────────────────────────

    def on_message(self, ws, message):
        try:
            data = json.loads(message)

            # Respond to server heartbeat (CRITICAL — must reply within 3 min)
            if isinstance(data, dict) and "ping" in data:
                ws.send(json.dumps({"pong": data["ping"]}))
                return

            self.process_message(data)

        except Exception as e:
            print(f"[ERROR] {e}, raw: {message[:200]}")

    def on_open(self, ws):
        print("[OK] WebSocket connected! Listening for private events...")
        print("     (Place/cancel orders via REST API to see updates here)\n")

    def on_error(self, ws, error):
        print(f"[WS ERROR] {error}")

    def on_close(self, ws, status_code, msg):
        print(f"[WS CLOSED] code={status_code} msg={msg}")

    # ── Connect ──────────────────────────────────────────────

    def connect(self):
        """Get listenKey and connect to private WebSocket stream."""
        if not self.get_listen_key():
            return

        url = f"{self.ws_base}/openapi/ws/{self.listen_key}"
        print(f"[INFO] Connecting to: {url}\n")

        ws = websocket.WebSocketApp(
            url,
            on_open=self.on_open,
            on_message=self.on_message,
            on_error=self.on_error,
            on_close=self.on_close,
        )
        ws.run_forever()


if __name__ == "__main__":
    # ── Configuration ────────────────────────────────────────
    # Option 1: Set environment variables
    #   export STOCKCOIN_API_KEY="your_api_key"
    #   export STOCKCOIN_SECRET_KEY="your_secret_key"
    #
    # Option 2: Replace the strings below directly (NOT recommended for production)
    import os

    tester = ContractWSTester(
        api_key=os.environ.get("STOCKCOIN_API_KEY", "your_api_key"),
        secret_key=os.environ.get("STOCKCOIN_SECRET_KEY", "your_secret_key"),
        base_url="https://api.stockcoin.ai",
        ws_base="wss://wsapi.stockcoin.ai",
    )
    tester.connect()
```

---

## Error Codes

Error response format:

```json
{"code": -1005, "msg": "No Permission"}
```

### General

| Code | Name | Description |
| :--- | :--- | :--- |
| -1000 | UNKNOWN | Unknown error |
| -1001 | DISCONNECTED | Internal error |
| -1002 | UNAUTHORIZED | Invalid API key |
| -1003 | TOO_MANY_REQUESTS | Rate limit exceeded |
| -1005 | NO_PERMISSION | API key account type mismatch (requires FUTURES type) |
| -1006 | UNEXPECTED_RESP | Unexpected backend response |
| -1007 | TIMEOUT | Request timeout |
| -1021 | INVALID_TIMESTAMP | Timestamp out of recv window |
| -1022 | INVALID_SIGNATURE | Signature verification failed |

### Parameter

| Code | Name | Description |
| :--- | :--- | :--- |
| -1102 | MANDATORY_PARAM_EMPTY | Missing or malformed required parameter |
| -1105 | PARAM_EMPTY | Empty parameter |
| -1121 | BAD_SYMBOL | Invalid symbol |
| -1130 | INVALID_PARAMETER | Invalid parameter value |

### Order

| Code | Name | Description |
| :--- | :--- | :--- |
| -1115 | INVALID_TIF | Invalid timeInForce |
| -1116 | INVALID_ORDER_TYPE | Invalid order type |
| -1117 | INVALID_SIDE | Invalid order side |
| -1132 | ORDER_PRICE_TOO_HIGH | Price exceeds upper limit |
| -1133 | ORDER_PRICE_TOO_SMALL | Price below lower limit |
| -1135 | ORDER_QUANTITY_TOO_BIG | Quantity exceeds limit |
| -1136 | ORDER_QUANTITY_TOO_SMALL | Quantity below minimum |
| -1141 | DUPLICATED_ORDER | Duplicate client order ID |
| -1142 | ORDER_CANCELLED | Order already cancelled |
| -1149 | CREATE_ORDER_FAILED | Order creation failed |
| -1150 | CANCEL_ORDER_FAILED | Cancellation failed |
| -1200 | ORDER_BUY_QUANTITY_TOO_SMALL | Buy quantity below contract minimum |
| -2013 | NO_SUCH_ORDER | Order not found |

### WebSocket

| Code | Name | Description |
| :--- | :--- | :--- |
| -1125 | INVALID_LISTEN_KEY | listenKey not found or expired |
| -10000 | INVALID_REQUEST | Invalid request |
| -10001 | JSON_FORMAT_ERROR | Invalid JSON |
| -10002 | INVALID_EVENT | Invalid event |
| -10003 | REQUIRED_EVENT | Event required |
| -10004 | INVALID_TOPIC | Invalid topic |
| -10005 | REQUIRED_TOPIC | Topic required |
| -10007 | PARAM_EMPTY | Params required |
| -10008 | PERIOD_EMPTY | Period required |
| -10009 | PERIOD_ERROR | Invalid period |
| -100010 | SYMBOLS_ERROR | Invalid symbols |
| -100007 | ORG_ID_REQUIRED | OrgId required |
| -100008 | ORG_ID_INVALID | OrgId must be a number |

---

## FAQ

**Q: How do I get an API key?**
A: Register on StockCoin.ai, navigate to API Management, and create a new API key with FUTURES account type.

**Q: What's the rate limit?**
A: If you receive error code `-1003`, you are being rate-limited. Reduce request frequency and implement exponential backoff.

**Q: My signature keeps failing (-1022). What should I check?**
A: Common causes:
1. Parameters must be **sorted alphabetically by key** before serialization.
2. `timestamp` must be current time in **milliseconds** (not seconds).
3. The `signature` parameter itself must **not** be included in the signing string.
4. For POST with form-data body, body params must be included in signing. For JSON array body (batch orders), body is excluded.
5. Query string and body string are concatenated directly — no `&` separator between them.

**Q: How do I keep the private WebSocket alive?**
A: Reply to every `{"ping": ...}` with `{"pong": ...}` within 3 minutes. Renew the `listenKey` via `POST /openapi/v1/userdata` every 30–50 minutes before it expires (60 min TTL).

**Q: What does `side` mean in order placement?**
A: `BUY_OPEN` = open long, `SELL_OPEN` = open short, `BUY_CLOSE` = close short, `SELL_CLOSE` = close long.

**Q: How do I handle incremental depth (diffDepth)?**
A: The first message is a full snapshot. For subsequent messages, check each `[price, qty]`: if `qty > 0`, update/insert; if `qty = 0`, delete that price level.

---

## Changelog

- **2026-01-15**: Added `slowBroker` and `topN` WebSocket topics.
- **2025-12-20**: Added `formula` field to `index` topic response.
- **2025-11-10**: Added `mergedDepth` and `diffMergedDepth` WebSocket topics.
- **2025-10-25**: Added `index` (index price) WebSocket topic.
- **2025-09-15**: Added `diffDepth` (incremental depth) WebSocket topic.

---

## License

Copyright StockCoin. All rights reserved.
