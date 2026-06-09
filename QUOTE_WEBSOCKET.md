# 行情websocket接口

## 访问地址

- protocol：`wss` or `ws`
- uri: /ws/quote/v1



## 数据压缩

- 通过params.binary来开启  见[参数表](#附2. params参数表)

  

## 接口文档

### request格式

| name   | values                                                       | description |
| :----- | :----------------------------------------------------------- | ----------- |
| id     | string                                                       | 唯一标识    |
| topic  | realtimes\|trade\|kline_$interval\|depth\|diffDepth\|broker\|topN\|slowBroker, | 请求主题    |
| event  | sub\|cancel\|cancel_all                                      | 请求类型    |
| symbol | exchangeId.symbol (eg: 301.NVDA-PERP-USDT)                          | 币对。`exchangeId` 为交易所分配的数字标识，通过 [获取合约信息](#realtimes--24小时行情) 接口可获取        |
| params | {}                                                           | 参数        |

#### realtimes  24小时行情

```js
{
  "event": "sub",
  "id": "realtimes",
  "symbol": "301.NVDA-PERP-USDT", // exchangeId.symbol
  "topic": "realtimes",  
  "params": {
    "binary": "true",  // 是否进行压缩，默认为false,
    "realtimeInterval": "24h" //不发默认24h涨幅,其他值1d(北京时间8点),1d+8(北京时间0点)
  }
}
```

#### depth 深度

```js
{
  event: "sub",
  id: "depth301.NVDA-PERP-USDT",
  limit: 100,
  params: {binary: true},
  symbol: "301.NVDA-PERP-USDT",
  topic: "depth" ,
}
```

#### kline_$interval k线

```js
{
  event: "sub",
  id: "kline_301NVDA-PERP-USDT15m",
  params: {binary: true, klineType: "15m", limit: 1500},
  symbol: "301.NVDA-PERP-USDT",
  topic: "kline_15m",
}
```

#### trade 最近成交

```js
{
  event: "sub",
  id: "trade301.NVDA-PERP-USDT",
  limit: 60,
  params: {org: 6001, binary: true},
  symbol: "301.NVDA-PERP-USDT",
  topic: "trade",
}
```

#### mergeDepth 合并深度

```js
{
	event: "cancel",
  id: "301.NVDA-PERP-USDT2",
  limit: 22,
  params: {
    dumpScale: 2, // 合并深度, 2代表2位小数
    binary: true
  },
  symbol: "301.NVDA-PERP-USDT",
  topic: "mergedDepth",
}
```

#### slowBroker 券商全量行情

```js
{
    id: "broker", 
    topic: "slowBroker", 
    event: "sub", 
    params: {
        org: 6001, 
        binary: true,
        realtimeInterval: "24h" //不发默认24h涨幅,其他值1d(北京时间8点),1d+8(北京时间0点)
    }
}
```



### response格式

#### realtimes 24小时行情数据

```javascript
{
  "t": "1736933460000",//time
  "s": "NVDA-PERP-USDT", // symbol
  "c": "183.25",//close price
  "h": "185.50",//high price
  "l": "182.80",//low price
  "o": "183.00", //open price
  "v": "12550.5", //volume 
  "e": "301" //exchange id
}
```

#### depth 深度

```js
{
  "t": 1736933400000, //time
  "s": "NVDA-PERP-USDT", //symbol
  "a": [ //卖单，价格顺序： 从小到大
      ["183.25", "155.92000000"],// price, quantity
      ["183.30", "263.53000000"],
      ["183.35", "41.76000000"],
      ["183.40", "155.92000000"],
      ["183.45", "263.53000000"],
      ["183.50", "41.76000000"]
  ],
  "b": [ //买单，价格顺序： 从大到小
      ["183.20", "155.92000000"],// price, quantity
      ["183.15", "263.53000000"],
      ["183.10", "41.76000000"],
      ["183.05", "155.92000000"],
      ["183.00", "263.53000000"],
      ["182.95", "41.76000000"]
  ]
}
```

#### kline_$interval k线

```javascript
{
  "t": "1736933460000",//time
  "s": "NVDA-PERP-USDT", // symbol
  "c": "183.25",//close price
  "h": "185.50",//high price
  "l": "182.80",//low price
  "o": "183.00", //open price
  "v": "12550.5" //volume
}
```

#### trade 最近成交

```javascript
{
  "t": 1736933520000,//time
  "p": "183.25",//price
  "q": "15.2", //quantity
  "m": true, //true is buy, false is sell
  "v": "123" //version
}
```

#### mergeDepth 合并深度

```json
[
    {
        e: 301, // exchange_id
        s: "NVDA-PERP-USDT",  // symbol
        t: 1736933460000, // time
        v: "237679504_2", // 版本
        b: [ //买单，价格顺序： 从大到小
            [
                "183.20", 
                "26.3425"
            ], 
            [
                "183.15", 
                "20.5"
            ], 
        ], 
        a: [ //卖单，价格顺序： 从小到大
            [
                "183.25", 
                "23.1141"
            ], 
            [
                "183.30", 
                "14.2"
            ], 
 
        ], 
        o: 0 // 发送顺序
    }
]
```

#### slowBroker 币对行情

```json
{
    topic: "slowBroker", 
    params: {
        realtimeInterval: "24h" //不发默认24h涨幅,其他值1d(北京时间8点),1d+8(北京时间0点)
        org: "6001", 
        binary: "true"
    }, 
    data: [
        {
            t: 1736933460000, // time
            s: "NVDA-PERP-USDT",  // 币对
            sn: "NVDA-PERP-USDT",  // 币对名称
            c: "183.25",  // close
            h: "185.50",  // hight
            l: "182.80",  // low
            o: "183.00",  // open
            v: "125500",  // volume
            qv: "22983750",  // quote * volume 
            m: "0.0014",  // margin 涨跌幅
            e: 301 // exchangeId
        }
    ], 
    f: false, 
    sendTime: 1736933459600, 
    shared: false, 
    id: "broker"
}
```



### 正常消息

```js
{
    "symbol": "301.NVDA-PERP-USDT",
    "topic": "realtimes",
    "f": true, //表示是否是第一次的数据，第一次有的类型为全量
    // 不同的topic, data里的数据都为数组，且第0条为最早的数据，最后一条为最新数据
    // 对于realtimes可以直接取数组最后一条就可以
    "data": [{
        "t": "1736933460000",//time
        "s": "NVDA-PERP-USDT", // symbol
        "c": "183.25",//close price
        "h": "185.50",//high price
        "l": "182.80",//low price
        "o": "183.00", //open price
        "v": "12550.5", //volume 
        "e": "301" //exchange id
    },{
        "t": "1736933470000",//time
        "s": "NVDA-PERP-USDT", // symbol
        "c": "183.30",//close price
        "h": "185.50",//high price
        "l": "182.80",//low price
        "o": "183.00", //open price
        "v": "12580.2", //volume 
        "e": "301" //exchange id
    }],
    // 返回的数据也会带上传入的params
    "params": {
    }
}
```

### 异常消息

当发送的消息格式，内容不正确时，会出现以下消息。

```
{
    "code": "-100010",
    "msg": "Invalid Symbols!"
}
```

 



# 接口变更记录

## 2026-01-15

### 新增券商行情topic (broker) 、排序行情topic(topN)

新增`broker`和`topN`两个topic，在params参数里要加入 org，指定券商的id。（可不传，会通过host来判断）。

topN有可选参数limit，为返回前几个行情。

broker首次订阅返回这个券商下的所有币对的realtime，之后每次更新为有变化的币对，每秒最多发送10次。

topN为每秒返回指定券商按涨跌幅排序的前limit个, 默认为5个，每次都为全量。



订阅请求

```javascript
{
    "id": "index",
    "topic": "topN", 
    "event": "sub",
    "params": {
        "limit": "10",
        "org": "6001", // not required
        "binary": "false"
    }
}
```



返回结果

```javascript
{
    "topic": "broker",
    "params": {
        "org": "6001",
        "binary": "false",
        "limit": "10",
        "realtimeInterval": "24h" //不发默认24h涨幅,其他值1d(北京时间8点),1d+8(北京时间0点)
    },
    "data": [
        {
            "t": "1736933460000",
            "s": "NVDA-PERP-USDT",
            "c": "183.25",
            "h": "185.50",
            "l": "182.80",
            "o": "183.00",
            "v": "12550.5",
            "qv": "2298375.125",
            "m": "0.0014",
            "e": 301
        }
    ],
    "f": true,
    "shared": false,
    "id": "index"
}
```

##  2025-12-20 

### 更新指数返回内容

返回消息新加 formula字段

```javascript
{
    "symbol": "NVDA-PERP-USDT",
    "topic": "index",
    "data": [{
        "symbol": "NVDA-PERP-USDT",
        "index": "183.25",
        "edp": "183.30",
        "formula": "(183.20[SOURCE_A]+183.25[SOURCE_B]+183.28[SOURCE_C]+183.27[SOURCE_D])/4"
    }],
    "f": false
}
```



## 2025-11-10

### 新加 mergedDepth/diffMergedDepth 合并深度/增量合并深度

订阅合并深度请求

```javascript
{
    // exchangeId.symbol,exchangeId1.symbol1.....
    "symbol": "301.NVDA-PERP-USDT",
    "topic": "mergedDepth",// 增量为diffMergedDepth
    "event": "sub",
    // 参数
    "params": {
        // 合并深度
        // required!!!
        "dumpScale": "2",
 
        // 是否进行压缩
        // 默认为false
        "binary": "true"
    }
}
```



##  2025-10-25

### 新加指数推送

订阅消息

```js
{
  "event": "sub"
  "symbol": "NVDA-PERP-USDT"
  "topic": "index"
}
```

返回消息

```javascript
{
    "symbol": "NVDA-PERP-USDT",
    "topic": "index",
    "data": [{
        "symbol": "NVDA-PERP-USDT",
        "index": "183.25",
        "edp": "183.30"
    }],
    "f": false
}
```



## 2025-09-15

1. 新增增量推送的订阅类型
2. 增量尝试数据说明

### 新增增量深度数据推送

**说明** 

订阅diffDepth类型的数据之后，返回的第一条数据为全量深度，客户端直接更新全部深度，后续数据为增量数据。

以此深度数据举例:

```js
["183.25", "155.92000000"], // price, quantity (change or add or remove)
```

第二个元素quantity不再只表示数量，需要客户端进行数量的判断。

1. quantity > 0 表示对应价格的数量发生了变化，客户端需要更新此条深度

2. quantity = 0 表示此条深度已经不存在，客户端需要删除此条深度




**处理逻辑**

客户端收到数据后先比较price，

1. 如果当前深度中没有相同price，把此条深度加入到当前深度正确位置中。
2. 如果当前深度中有相同的price，则需要按照上述的quantity比较规则进行处理。



**eg:**

**1  新增**

a. 初始状态为bids [] asks []，买卖均为空

b. 收到数据 bids [1, 2]

c. 结果为 bids [[1,2]] asks []



**2 修改**

a. 初始状态为bids [[1, 2]] asks []

b. 收到数据 bids [1, 4]

c. 结果为 bids [[1, 4]] asks[]



**3 删除**

a. 初始状态为bids [[1, 2]] asks []

b. 收到数据 bids [1, 0]

c. 结果为 bids [] asks[]



------



# 附1. 心跳处理

> WebSocket Client需要周期性的发送心跳，服务器Idle时间为5min，理论上最长周期为5min，考虑到网络等其它原因，建议1~2分钟发送一次心跳。同时也支持发送心跳帧。

-  Request：

```javascript
{
    "ping": 1736933460000
}
```

- Response:

```javascript
{
    "pong": 1736933460000
}
```




# 附2. params参数表

| name             | type   | required | value                                                        | 支持topics                   | description                                                  |
| ---------------- | ------ | -------- | ------------------------------------------------------------ | ---------------------------- | ------------------------------------------------------------ |
| binary           | string | false    | `true|false`                                                 | all                          | 是否返回二进制数据                                           |
| org              | string | false    | orgId （eg:6001）                                            | broker, topN                 | 券商 ID，由平台分配。可不传，服务端会通过请求 host 自动判断                                                       |
| limit            | string | false    |                                                              | kline,topN                   | 可以选择快照的条数，k线最大为2000条;<br/>topN表示前limit个行情;<br/> trade为固定60条;<br/>其它固定为1条; |
| kline_type       | string | true     | `1m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d, 1d+8, 1w, 1w+8, 1M, 1M+8` | kline                        | k线类型                                                      |
| dump_scale       | string | true     |                                                              | mergedDepth, diffMergedDepth | 合并的档位                                                   |
| realtimeInterval | string | false    | `24h, 1d, 1d+8`                                              |                              | 24小时涨幅计算起始时间                                       |




# 附3. 错误码

```javascript
INVALID_REQUEST("-10000", "Invalid request!")
JSON_FORMAT_ERROR("-10001", "Invalid JSON!")
INVALID_EVENT("-10002", "Invalid event")
REQUIRED_EVENT("-10003", "Event required!")
INVALID_TOPIC("-10004", "Invalid topic!")
REQUIRED_TOPIC("-10005", "Topic required!")
PARAM_EMPTY("-10007", "Params required!")
PERIOD_EMPTY("-10008", "Period required!")
PERIOD_ERROR("-10009", "Invalid period!")
SYMBOLS_ERROR("-100010", "Invalid Symbols!")
ORG_ID_REQUIRED("-100007", "OrgId required.")
ORG_ID_INVALID("-100008", "OrgId must be a number.")
```
