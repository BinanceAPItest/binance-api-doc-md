---
title: Binance API 使用文档

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  #- javascript
  #- jason

toc_footers:
  - <a href='https://testnet.binancefuture.com/'>Binance Future Testnet</a>

includes:

search: true
---


# 基本信息
## Rest 基本信息

* 本篇列出REST接口的baseurl **https://testnet.binancefuture.com**
* 所有接口的响应都是JSON格式
* 响应中如有数组，数组元素以时间升序排列，越早的数据越提前。
* 所有时间、时间戳均为UNIX时间，单位为毫秒
* HTTP `4XX` 错误码用于指示错误的请求内容、行为、格式。
* HTTP `429` 错误码表示警告访问频次超限，即将被封IP
* HTTP `418` 表示收到429后继续访问，于是被封了。
* HTTP `5XX` 错误码用于指示Binance服务侧的问题。 
* HTTP `504` 表示API服务端已经向业务核心提交了请求但未能获取响应，特别需要注意的是`504`代码不代表请求失败，而是未知。很可能已经得到了执行，也有可能执行失败，需要做进一步确认。
* 每个接口都有可能抛出异常

> 异常响应格式如下：

```javascript
{
  "code": -1121,
  "msg": "Invalid symbol."
}
```

* 具体的错误码及其解释在[错误代码](#cf68bca02a)
* `GET`方法的接口, 参数必须在`query string`中发送.
* `POST`, `PUT`, 和 `DELETE` 方法的接口, 参数可以在 `query string`中发送，也可以在 `request body`中发送(content type `application/x-www-form-urlencoded`。允许混合这两种方式发送参数。但如果同一个参数名在query string和request body中都有，query string中的会被优先采用。
* 对参数的顺序不做要求。

## 访问限制
* 在 `/fapi/v1/exchangeInfo`接口中`rateLimits`数组里包含有REST接口(不限于本篇的REST接口)的访问限制。包括带权重的访问频次限制、下单速率限制。本篇`枚举定义`章节有限制类型的进一步说明。
* 违反上述任何一个访问限制都会收到HTTP 429，这是一个警告.
* 每一个接口均有一个相应的权重(weight)，有的接口根据参数不同可能拥有不同的权重。越消耗资源的接口权重就会越大。
* 当收到429告警时，调用者应当降低访问频率或者停止访问。
* **收到429后仍然继续违反访问限制，会被封禁IP，并收到418错误码**
* 频繁违反限制，封禁时间会逐渐延长，**从最短2分钟到最长3天**.

## 接口鉴权类型
* 每个接口都有自己的鉴权类型，鉴权类型决定了访问时应当进行何种鉴权
* 如果需要 API-key，应当在HTTP头中以`X-MBX-APIKEY`字段传递
* API-key 与 API-secret 是大小写敏感的
* 可以在网页用户中心修改API-key 所具有的权限，例如读取账户信息、发送交易指令、发送提现指令

鉴权类型 | 描述
------------ | ------------
NONE | 不需要鉴权的接口
TRADE | 需要有效的API-KEY和签名
USER_DATA | 需要有效的API-KEY和签名
USER_STREAM | 需要有效的API-KEY
MARKET_DATA | 需要有效的API-KEY


## 需要签名的接口 (TRADE 与 USER_DATA)
* 调用这些接口时，除了接口本身所需的参数外，还需要传递`signature`即签名参数。
* 签名使用`HMAC SHA256`算法. API-KEY所对应的API-Secret作为 `HMAC SHA256` 的密钥，其他所有参数作为`HMAC SHA256`的操作对象，得到的输出即为签名。
* 签名大小写不敏感。
* 当同时使用query string和request body时，`HMAC SHA256`的输入query string在前，request body在后

### 时间同步安全
* 签名接口均需要传递`timestamp`参数, 其值应当是请求发送时刻的unix时间戳（毫秒）
* 服务器收到请求时会判断请求中的时间戳，如果是5000毫秒之前发出的，则请求会被认为无效。这个时间窗口值可以通过发送可选参数`recvWindow`来自定义。
* 另外，如果服务器计算得出客户端时间戳在服务器时间的‘未来’一秒以上，也会拒绝请求。

> 逻辑伪代码：
  
  ```javascript
  if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow) {
    // process request
  } else {
    // reject request
  }
  ```

**关于交易时效性** 
互联网状况并不100%可靠，不可完全依赖,因此你的程序本地到币安服务器的时延会有抖动.
这是我们设置`recvWindow`的目的所在，如果你从事高频交易，对交易时效性有较高的要求，可以灵活设置recvWindow以达到你的要求。
**不推荐使用5秒以上的recvWindow**


### POST /fapi/v1/order 的示例

以下是在linux bash环境下使用 echo openssl 和curl工具实现的一个调用接口下单的示例
apikey、secret仅供示范

Key | Value
------------ | ------------
apiKey | vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A
secretKey | NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j


参数 | 取值
------------ | ------------
symbol | LTCBTC
side | BUY
type | LIMIT
timeInForce | GTC
quantity | 1
price | 0.1
recvWindow | 5000
timestamp | 1499827319559


### 示例 1: 所有参数通过 query string 发送
> **示例1:**
> **HMAC SHA256 签名:**

```shell
    $ echo -n "symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
```


> **curl 调用:**

```shell
    (HMAC SHA256)
    $ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.binance.com/fapi/v1/order?symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
```

* **queryString:** 

symbol=LTCBTC    
&side=BUY   
&type=LIMIT   
&timeInForce=GTC   
&quantity=1   
&price=0.1   
&recvWindow=5000   
&timestamp=1499827319559


### 示例 2: 所有参数通过 request body 发送
> **示例2:**
> **HMAC SHA256 签名:**

```shell
    $ echo -n "symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
```


> **curl 调用:**

```shell
    (HMAC SHA256)
    $ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.binance.com/fapi/v1/order' -d 'symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
```

* **requestBody:** 

symbol=LTCBTC   
&side=BUY   
&type=LIMIT   
&timeInForce=GTC   
&quantity=1   
&price=0.1   
&recvWindow=5000   
&timestamp=1499827319559


### 示例 3: 混合使用 query string 与 request body
> **示例3:**
> **HMAC SHA256 签名:**

```shell
    $ echo -n "symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTCquantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= 0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77
```


> **curl 调用:**

```shell
    (HMAC SHA256)
    $ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.binance.com/fapi/v1/order?symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC' -d 'quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77'
```

* **queryString:** 

symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC

* **requestBody:** 

quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559

<aside class="notice">
Note that the signature is different in example 3.
There is no & between "GTC" and "quantity=1".
</aside>


## 公开API参数
### 术语解释
* `base asset` 指一个交易对的交易对象，即写在靠前部分的资产名
* `quote asset` 指一个交易对的定价资产，即写在靠后部分资产名


### 枚举定义

**交易对类型:**

* FUTURE 期货

**订单状态:**

* NEW 新建订单
* PARTIALLY_FILLED  部分成交
* FILLED  全部成交
* CANCELED  已撤销
* PENDING_CANCEL 正在撤销中
* REJECTED 订单被拒绝
* EXPIRED 订单过期(根据timeInForce参数规则)

**订单种类:**

* LIMIT 限价单
* MARKET 市价单
* STOP 止损单

**订单方向:**

* BUY 买入
* SELL 卖出

**Time in force:**

* GTC - Good Till Cancel 成交为止
* IOC - Immediate or Cancel 无法立即成交(吃单)的部分就撤销
* FOK - Fill or Kill 无法全部立即成交就撤销
* GTX - Good Till Crossing 无法成为挂单方就撤销

**K线间隔:**

m -> 分钟; h -> 小时; d -> 天; w -> 周; M -> 月

* 1m
* 3m
* 5m
* 15m
* 30m
* 1h
* 2h
* 4h
* 6h
* 8h
* 12h
* 1d
* 3d
* 1w
* 1M

**限制种类 (rateLimitType)**

* REQUESTS_WEIGHT  单位时间请求权重之和上限

```json
    {
      "rateLimitType": "REQUEST_WEIGHT",
      "interval": "MINUTE",
      "intervalNum": 1,
      "limit": 1200
    }
```
* ORDERS    单位时间下单(撤单)次数上限

```json
    {
      "rateLimitType": "ORDERS",
      "interval": "SECOND",
      "intervalNum": 1,
      "limit": 10
    }
```

**限制间隔**
* MINUTE



## 过滤器
过滤器，即Filter，定义了一系列交易规则。
共有两类，分别是针对交易对的过滤器`symbol filters`，和针对整个交易所的过滤器`exchange filters`(暂不支持)

### 交易对过滤器
#### PRICE_FILTER 价格过滤器

> **/exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "PRICE_FILTER",
    "minPrice": "0.00000100",
    "maxPrice": "100000.00000000",
    "tickSize": "0.00000100"
  }
```

价格过滤器用于检测order订单中price参数的合法性
* `minPrice` 定义了 `price`/`stopPrice` 允许的最小值
* `maxPrice` 定义了 `price`/`stopPrice` 允许的最大值。
* `tickSize` 定义了 `price`/`stopPrice` 的步进间隔，即price必须等于minPrice+(tickSize的整数倍)
以上每一项均可为0，为0时代表这一项不再做限制。

逻辑伪代码如下：

* `price` >= `minPrice`
* `price` <= `maxPrice`
* (`price`-`minPrice`) % `tickSize` == 0



#### LOT_SIZE 订单尺寸

> */exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

lots是拍卖术语，这个过滤器对订单中的`quantity`也就是数量参数进行合法性检查。包含三个部分：

* `minQty` 表示 `quantity` 允许的最小值.
* `maxQty` 表示 `quantity` 允许的最大值
* `stepSize` 表示 `quantity`允许的步进值。

逻辑伪代码如下：

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0


#### MARKET_LOT_SIZE 市价订单尺寸
参考LOT_SIZE，区别仅在于对市价单还是限价单生效

#### MAX_NUM_ORDERS 最多订单数


> **/exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "MAX_NUM_ORDERS",
    "limit": 25
  }
```

定义了某个交易对最多允许的挂单数量（不包括已关闭的订单）
普通订单与条件订单均计算在内




# 行情接口
## 测试服务器连通性 PING
``
GET /fapi/v1/ping
``

> **响应:**

```javascript
pong
```

测试能否联通

**权重:**
1

**参数:**
NONE



## 获取服务器时间
``
GET /fapi/v1/time
``

> **响应:**

```javascript
{
  "serverTime": 1499827319559
}
```

获取服务器时间

**权重:**
1

**参数:**
NONE


## 获取交易规则和交易对
``
GET /fapi/v1/exchangeInfo
``


> **响应:** 

```javascript
{
	"exchangeFilters": [],
 	"rateLimits": [
 		{
 			"interval": "MINUTE",
   			"intervalNum": 1,
   			"limit": 6000,
   			"rateLimitType": "REQUEST_WEIGHT"
   		},
  		{
  			"interval": "MINUTE",
   			"intervalNum": 1,
   			"limit": 6000,
   			"rateLimitType": "ORDERS"
   		}
   	],
 	"serverTime": 1565613908500,
 	"symbols": [
 		{
 			"filters": [
 				{
 					"filterType": "PRICE_FILTER",
     				"maxPrice": "10000000",
     				"minPrice": "0.00000100",
     				"tickSize": "0.00000100"
     			},
    			{
    				"filterType": "LOT_SIZE",
     				"maxQty": "10000000",
     				"minQty": "0.00100000",
     				"stepSize": "0.00100000"
     			},
    			{
    				"filterType": "MARKET_LOT_SIZE",
     				"maxQty": "10000000",
     				"minQty": "0.00100000",
     				"stepSize": "0.00100000"
     			},
     			{
    				"filterType": "MAX_NUM_ORDERS",
    				"limit": 100
  				}
    		],
   			"maintMarginPercent": "2.5000",
   			"pricePrecision": 2,
   			"quantityPrecision": 3,
   			"requiredMarginPercent": "5.0000",
   			"status": "TRADING",
   			"OrderType": [
   				"LIMIT", 
   				"MARKET", 
   				"STOP"
   			],
   			"symbol": "BTCUSDT",
   			"timeInForce": [
   				"GTC",    // Good Till Cancel 
   				"IOC",    // Immediate or Cancel
   				"FOK",    // Fill or Kill
   				"GTX"     // Good Till Crossing
   			]
   		}
   	],
	"timezone": "UTC"
}

```


获取交易规则和交易对

**权重:**
1

**参数:**
NONE



## 深度信息
``
GET /fapi/v1/depth
``

> **响应:**

```javascript
{
  "lastUpdateId": 1027024,
  "bids": [
    [
      "4.00000000",     // PRICE
      "431.00000000"    // QTY
    ]
  ],
  "asks": [
    [
      "4.00000200",
      "12.00000000"
    ]
  ]
}
```

**权重:**

Limit | Weight
------------ | ------------
5, 10, 20, 50, 100 | 1
500 | 5
1000 | 10

**参数:**

名称 | 类型 | 是否必须 | 描述
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | 默认 100; 最大 1000. 可选值:[5, 10, 20, 50, 100, 500, 1000]




## 近期成交
``
GET /fapi/v1/trades
``

> **响应:**

```javascript
[
  {
    "id": 28457,
    "price": "4.00000100",
    "qty": "12.00000000",
    "quoteQty": "8000.00",
    "time": 1499865549590,
    "isBuyerMaker": true,
  }
]
```

获取近期成交

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.


## 查询历史成交(MARKET_DATA)
``
GET /fapi/v1/historicalTrades
``


> **响应:**

```javascript
[
  {
    "id": 28457,
    "price": "4.00000100",
    "qty": "12.00000000",
    "quoteQty": "8000.00",
    "time": 1499865549590,
    "isBuyerMaker": true,
  }
]
```

**权重:**
5

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.
fromId | LONG | NO | 从哪一条成交id开始返回. 缺省返回最近的成交记录

* 需要提交 `X-MBX-APIY` 


## 近期成交(归集)
``
GET /fapi/v1/aggTrades
``

> **响应:**

```javascript
[
  {
    "a": 26129,         // 归集成交ID
    "p": "0.01633102",  // 成交价
    "q": "4.70443515",  // 成交量
    "f": 27781,         // 被归集的首个成交ID
    "l": 27781,         // 被归集的末个成交ID
    "T": 1498793709153, // 成交时间
    "m": true,          // 是否为主动卖出单
  }
]
```

归集交易与逐笔交易的区别在于，同一价格、同一方向、同一时间（按秒计算）的trade会被聚合为一条

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
fromId | LONG | NO | 从包含fromID的成交开始返回结果
startTime | LONG | NO | 从该时刻之后的成交记录开始返回结果
endTime | LONG | NO | 返回该时刻为止的成交记录
limit | INT | NO | 默认 500; 最大 1000.

* 如果同时发送startTime和endTime，间隔必须小于一小时
* 如果没有发送任何筛选参数(fromId, startTime,endTime)，默认返回最近的成交记录



## K线数据
``
GET /fapi/v1/klines
``

> **响应:**

```javascript
[
  [
    1499040000000,      // 开盘时间
    "0.01634790",       // 开盘价
    "0.80000000",       // 最高价
    "0.01575800",       // 最低价
    "0.01577100",       // 收盘价(当前K线未结束的即为最新价)
    "148976.11427815",  // 成交量
    1499644799999,      // 收盘时间
    "2434.19055334",    // 成交额
    308,                // 成交笔数
    "1756.87402397",    // 主动买入成交量
    "28.46694368",      // 主动买入成交额
    "17928899.62484339" // 请忽略该参数
  ]
]
```

每根K线的开盘时间可视为唯一ID

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
interval | ENUM | YES |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.

* 缺省返回最近的数据



## 最新市场价和资金费率
``
GET /fapi/v1/premiumIndex
``

> **响应:**

```javascript
{
    "symbol": "BTCUSDT",
    "markPrice": "11012.80409769",
    "lastFundingRate": "-0.03750000",
    "nextFundingTime": 1562569200000,
    "time": 1562566020000
}
```

采集各大交易所数据加权平均

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |


## 24hr价格变动情况
``
GET /fapi/v1/ticker/24hr
``
请注意，不携带symbol参数会返回全部交易对数据，不仅数据庞大，而且权重极高

> **响应:**

```javascript
{
  "symbol": "BTCUSDT",
  "priceChange": "-94.99999800",    //24小时价格变动
  "priceChangePercent": "-95.960",  //24小时价格变动百分比
  "weightedAvgPrice": "0.29628482", //加权平均价
  "lastPrice": "4.00000200",        //最近一次成交价
  "lastQty": "200.00000000",        //最近一次成交额
  "openPrice": "99.00000000",       //24小时内第一次成交的价格
  "highPrice": "100.00000000",      //24小时最高价
  "lowPrice": "0.10000000",         //24小时成交量
  "volume": "8913.30000000",        //24小时成交额
  "quoteVolume": "15.30000000",     //24小时成交额
  "openTime": 1499783499040,        //24小时内，第一笔交易的发生时间
  "closeTime": 1499869899040,       //24小时内，最后一笔交易的发生时间
  "firstId": 28385,   // 首笔成交id
  "lastId": 28460,    // 末笔成交id
  "count": 76         // 成交笔数
}
```

> 或（当不发送交易对信息）

```javascript
[
	{
  		"symbol": "BTCUSDT",
  		"priceChange": "-94.99999800",    //24小时价格变动
  		"priceChangePercent": "-95.960",  //24小时价格变动百分比
  		"weightedAvgPrice": "0.29628482", //加权平均价
  		"lastPrice": "4.00000200",        //最近一次成交价
  		"lastQty": "200.00000000",        //最近一次成交额
  		"openPrice": "99.00000000",       //24小时内第一次成交的价格
  		"highPrice": "100.00000000",      //24小时最高价
  		"lowPrice": "0.10000000",         //24小时成交量
  		"volume": "8913.30000000",        //24小时成交额
  		"quoteVolume": "15.30000000",     //24小时成交额
  		"openTime": 1499783499040,        //24小时内，第一笔交易的发生时间
  		"closeTime": 1499869899040,       //24小时内，最后一笔交易的发生时间
  		"firstId": 28385,   // 首笔成交id
  		"lastId": 28460,    // 末笔成交id
  		"count": 76         // 成交笔数
}
]
```


**权重:**
带symbol为1   
不带为***40***

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* 不发送交易对参数，则会返回所有交易对信息


## 最新价格
``
GET /fapi/v1/ticker/price
``

> **响应:**

```javascript
{
  "symbol": "LTCBTC",
  "price": "4.00000200"
}
```

> 或（当不发送symbol）

```javascript
[
	{
  		"symbol": "BTCUSDT",
  		"price": "6000.01"
	}
]
```


返回最近价格

**权重:**
单交易对1   
无交易对2


**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* 不发送交易对参数，则会返回所有交易对信息



## 当前最优挂单
``
GET /fapi/v1/ticker/bookTicker
``

> **响应:**

```javascript
{
  "symbol": "BTCUSDT",
  "bidPrice": "4.00000000",//最优买单价
  "bidQty": "431.00000000",//挂单量
  "askPrice": "4.00000200",//最优卖单价
  "askQty": "9.00000000"//挂单量
}
```
> 或（当不发送symbol）

```javascript
[
	{
  		"symbol": "BTCUSDT",
  		"bidPrice": "4.00000000",//最优买单价
  		"bidQty": "431.00000000",//挂单量
  		"askPrice": "4.00000200",//最优卖单价
  		"askQty": "9.00000000"//挂单量
	}
]
```


返回当前最优的挂单(最高买单，最低卖单)

**权重:**
单交易对1   
无交易对2

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* 不发送交易对参数，则会返回所有交易对信息



# Websocket 行情推送

* 本篇所列出的所有wss接口的baseurl为: **wss://testnet.binancefuture.com**
* 组合streams的URL格式为 **/stream?streams=\<streamName1\>/\<streamName2\>/\<streamName3\>**
* 订阅组合streams时，事件payload会以这样的格式封装 **{"stream":"\<streamName\>","data":\<rawPayload\>}**
* stream名称中所有交易对均为**大写**
* 每个到**stream.binance.com**的链接有效期不超过24小时，请妥善处理断线重连。
* 每3分钟，服务端会发送ping帧，客户端应当在10分钟内回复pong帧，否则服务端会主动断开链接。允许客户端发送不成对的pong帧(即客户端可以以高于10分钟每次的频率发送pong帧保持链接)。



## 最新合约价格
aggTrade中的价格'p'或ticker/miniTicker中的价格'c'均可以作为最新成交价。

## 归集交易
归集交易与逐笔交易的区别在于，同一价格、同一方向、同一时间（按秒计算）的trade会被聚合为一条.推送间隔100毫秒。

**Stream 名称:**       
``<symbol>@aggTrade``

> **Payload:**

```javascript
{
  "e": "aggTrade",  // 事件类型
  "E": 123456789,   // 事件时间
  "s": "BNBBTC",    // 交易对
  "p": "0.001",     // 成交价格
  "q": "100",       // 成交数量
  "f": 100,         // 被归集的首个交易ID
  "l": 105,         // 被归集的末次交易ID
  "T": 123456785,   // 成交时间
  "m": true         // 买方是否是做市方。如true，则此次成交是一个主动卖出单，否则是一个主动买入单。
}
```


## 最新市场价
推送间隔3秒

**Stream Name:**    
``<symbol>@markPrice``

> **Payload:**

```javascript
  {
    "e": "markPriceUpdate",  // Event type
    "E": 1562305380000,      // Event time
    "s": "BTCUSDT",          // Symbol
    "p": "11185.87786614",   // Mark price
    "r": "0.00030000",       // Funding rate
    "T": 1562306400000       // Next funding time
  }
```



## K线
K线stream逐秒推送所请求的K线种类(最新一根K线)的更新。推送间隔250毫秒（如有刷新）

**订阅Kline需要提供间隔参数，最短为分钟线，最长为月线。支持以下间隔:**

m -> 分钟; h -> 小时; d -> 天; w -> 周; M -> 月

* 1m
* 3m
* 5m
* 15m
* 30m
* 1h
* 2h
* 4h
* 6h
* 8h
* 12h
* 1d
* 3d
* 1w
* 1M

**Stream 名称:**    
``<symbol>@kline_<interval>``

> **Payload:**

```javascript
{
  "e": "kline",     // 事件类型
  "E": 123456789,   // 事件时间
  "s": "BNBBTC",    // 交易对
  "k": {
    "t": 123400000, // 这根K线的起始时间
    "T": 123460000, // 这根K线的结束时间
    "s": "BNBBTC",  // 交易对
    "i": "1m",      // K线间隔
    "f": 100,       // 这根K线期间第一笔成交ID
    "L": 200,       // 这根K线期间末一笔成交ID
    "o": "0.0010",  // 这根K线期间第一笔成交价
    "c": "0.0020",  // 这根K线期间末一笔成交价
    "h": "0.0025",  // 这根K线期间最高成交价
    "l": "0.0015",  // 这根K线期间最低成交价
    "v": "1000",    // 这根K线期间成交量
    "n": 100,       // 这根K线期间成交数量
    "x": false,     // 这根K线是否完结（是否已经开始下一根K线）
    "q": "1.0000",  // 这根K线期间成交额
    "V": "500",     // 主动买入的成交量
    "Q": "0.500",   // 主动买入的成交额
    "B": "123456"   // 忽略此参数
  }
}
```

## 按Symbol的精简Ticker
按Symbol刷新的24小时精简ticker信息，推送间隔3秒

**Stream 名称:**     
``<symbol>@miniTicker`

> **Payload:**

```javascript
  {
    "e": "24hrMiniTicker",  // 事件类型
    "E": 123456789,         // 事件时间（毫秒）
    "s": "BNBBTC",          // 交易对
    "c": "0.0025",          // 最新成交价格
    "o": "0.0010",          // 24小时前开始第一笔成交价格
    "h": "0.0025",          // 24小时内最高成交价
    "l": "0.0010",          // 24小时内最低成交加
    "v": "10000",           // 成交量
    "q": "18"               // 成交额
  }
```

## 按Symbol的完整Ticker
按Symbol刷新的24小时完整ticker信息，推送间隔3秒

**Stream 名称:**     
``<symbol>@ticker``


> **Payload:**

```javascript
{
  "e": "24hrTicker",  // 事件类型
  "E": 123456789,     // 事件时间
  "s": "BNBBTC",      // 交易对
  "p": "0.0015",      // 24小时价格变化
  "P": "250.00",      // 24小时价格变化（百分比）
  "w": "0.0018",      // 平均价格
  "c": "0.0025",      // 最新成交价格
  "Q": "10",          // 最新成交价格上的成交量
  "o": "0.0010",      // 24小时内第一比成交的价格
  "h": "0.0025",      // 24小时内最高成交价
  "l": "0.0010",      // 24小时内最低成交加
  "v": "10000",       // 24小时内成交量
  "q": "18",          // 24小时内成交额
  "O": 0,             // TODO 0 ??? 统计开始时间？？？
  "C": 86400000,      // TODO 3600*24*1000，24小时的毫秒数 ??? 统计关闭时间？？？
  "F": 0,             // 24小时内第一笔成交交易ID
  "L": 18150,         // 24小时内最后一笔成交交易ID
  "n": 18151          // 24小时内成交数
}
```


## 增量深度信息stream
orderbook的变化部分，推送间隔250毫秒（如有刷新）

**Stream 名称:**     
``<symbol>@depth``

> **Payload:**

```javascript
{
  "e": "depthUpdate", // 事件类型
  "E": 123456789,     // 事件时间
  "s": "BNBBTC",      // 交易对
  "U": 157,           // 从上次推送至今新增的第一个 update Id
  "u": 160,           // 从上次推送至今新增的最后一个 update Id
  "pu": 149,          // 上次推送的最后一个update Id（即上条消息的‘u’）
  "b": [              // 变动的买单深度
    [
      "0.0024",       // 价
      "10"           // 量
    ]
  ],
  "a": [              // 变动的卖单深度
    [
      "0.0026",       // 价
      "100"          // 量
    ]
  ]
}
```

## 如何正确在本地维护一个orderbook副本
1. 订阅 **wss://testnet.binancefuture.com/stream?streams=btcusdt@depth**
2. 开始缓存收到的更新。同一个价位，后收到的更新覆盖前面的。
3. 访问Rest接口 **https://testnet.binancefuture.com/fapi/v1/depth?symbol=BTCUSDT&limit=1000**获得一个1000档的深度快照
4. 将目前缓存到的信息中`u`小于步骤3中获取到的快照中的`lastUpdateId`的部分丢弃(丢弃更早的信息，已经过期)。
5. 将深度快照中的内容更新到本地orderbook副本中，并从websocket接收到的第一个`U` <= `lastUpdateId` **且** `u` >= `lastUpdateId` 的event开始继续更新本地副本。
6. 每一个新event的`pu`应该等于上一个event的`u`，否则可能出现了丢包，请从step3重新进行初始化。
7. 每一个event中的挂单量代表这个价格目前的挂单量**绝对值**，而不是相对变化。
8. 如果某个价格对应的挂单量为0，表示该价位的挂单已经撤单或者被吃，应该移除这个价位。






# 账户和交易接口
## 下单  (TRADE)
``
POST /fapi/v1/order  (HMAC SHA256)
``

> **响应:**

```javascript
{
	"accountId": 10012,
 	"clientOrderId": "testOrder",
 	"cumQuote": "0",
 	"executedQty": "0",
 	"orderId": 22542179,
 	"origQty": "10",
 	"price": "10000",
 	"side": "SELL",
 	"status": "NEW",
 	"stopPrice": "0",
 	"symbol": "BTCUSDT",
 	"timeInForce": "GTC",
 	"type": "LIMIT",
 	"updateTime": 1566818724722
 }
```

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
side | ENUM | YES |
type | ENUM | YES |
quantity | DECIMAL | YES |
price | DECIMAL | NO |
newClientOrderId | STRING | NO | 用户自定义的orderid，如空缺系统会自动赋值
stopPrice | DECIMAL | NO | 仅 `STOP`, `STOP_LIMIT` 需要此参数
timeInForce | ENUM | NO | 仅 `LIMIT` 需要此参数
recvWindow  | LONG | NO
timestamp   | LONG | YES

根据 order `type`的不同，某些参数强制要求，具体如下:

Type | 强制要求的参数
------------ | ------------
`LIMIT` | `timeInForce`, `quantity`, `price`
`MARKET` | `quantity`
`STOP`/`STOP_LIMIT` | `price`, `stopPrice`,`quantity`


条件单的触发价格必须:

* 比下单时当前市价高:
* 比下单时当前市价低: 



## 测试下单接口 (TRADE)
``
POST /fapi/v1/order/test (HMAC SHA256)
``

> **响应:**

```javascript
{}
```

用于测试订单请求，但不会提交到撮合引擎

**权重:**
1

**参数:**

参考 `POST /fapi/v1/order`



## 查询订单 (USER_DATA)
``
GET /fapi/v1/order (HMAC SHA256)
``

> **响应:**

```javascript
{
  "symbol": "BTCUSDT",
  "orderId": 1,
  "clientOrderId": "myOrder1",
  "price": "0.1",
  "origQty": "1.0",
  "executedQty": "0.0",
  "cumQuote": "0.0",
  "status": "NEW",
  "timeInForce": "GTC",
  "type": "LIMIT",
  "side": "BUY",
  "stopPrice": "0.0",
  "time": 1499827319559,
  "updateTime": 1499827319559
}
```

查询订单状态

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

注意:
* 至少需要发送 `orderId` 与 `origClientOrderId`中的一个



## 撤销订单 (TRADE)
``
DELETE /fapi/v1/order  (HMAC SHA256)
``

> **响应:**

```javascript
{
  "symbol": "BTCUSDT",
  "orderId": 28,
  "origClientOrderId": "myOrder1",
  "clientOrderId": "cancelMyOrder1",
  "transactTime": 1507725176595,
  "price": "1.00000000",
  "origQty": "10.00000000",
  "executedQty": "8.00000000",
  "cumQuote": "8.00000000",
  "status": "CANCELED",
  "timeInForce": "GTC",
  "type": "LIMIT",
  "side": "SELL"
}
```

**权重:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
newClientOrderId | STRING | NO |  用户自定义的本次撤销操作的ID(注意不是被撤销的订单的自定义ID)。如无指定会自动赋值。
recvWindow | LONG | NO |
timestamp | LONG | YES |

`orderId` 与 `origClientOrderId` 必须至少发送一个



## 查看账户当前挂单 (USER_DATA)
``
GET /fapi/v1/openOrders  (HMAC SHA256)
``

> **响应:**

```javascript
[
  {
    "symbol": "BTCUSDT",
    "orderId": 1,
    "clientOrderId": "myOrder1",
    "accountId": 1,
    "price": "0.1",
    "origQty": "1.0",
    "cumQty": "1.0",
    "cumQuote": "1.0",
    "status": "NEW",
    "timeInForce": "GTC",
    "type": "LIMIT",
    "side": "BUY",
    "stopPrice": "0.0",
    "updateTime": 1499827319559
  }
]
```


请小心使用不带symbol参数的调用

**权重:**
带symbol 1
不带40

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
recvWindow | LONG | NO |
timestamp | LONG | YES |

* 不带symbol参数，会返回所有交易对的挂单



## 查询所有订单（包括历史订单） (USER_DATA)
``
GET /fapi/v1/allOrders (HMAC SHA256)
``

> **响应:**

```javascript
[
  {
    "symbol": "BTCUSDT",
    "orderId": 1,
    "clientOrderId": "myOrder1",
    "price": "0.1",
    "origQty": "1.0",
    "executedQty": "1.0",
    "cumQuote": "10.0",
    "status": "NEW",
    "timeInForce": "GTC",
    "type": "LIMIT",
    "side": "BUY",
    "stopPrice": "0.0",
    "updateTime": 1499827319559
  }
]
```

**权重:**
5 

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO | 只返回此orderID之后的订单，缺省返回最近的订单
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO |
timestamp | LONG | YES |




## 账户信息 (USER_DATA)
``
GET /fapi/v1/account (HMAC SHA256)
``

> **响应:**

```javascript
{
    "canTrade": true,
    "canDeposit": true,
    "canWithdraw": true,
    "updateTime": 0,
    "totalInitialMargin": "1753.3112796525",
    "totalMaintMargin": "876.65563982625",
    "totalWalletBalance": "74757.0",
    "totalUnrealizedProfit": "18933.77440695",
    "totalMarginBalance": "93690.77440695",
    "assets": [
        {
            "asset": "USDT",
            "walletBalance": "74757.0",
            "unrealizedProfit": "18933.77440695",
            "marginBalance": "93690.77440695",
            "maintMargin": "876.65563982625",
            "initialMargin": "1753.3112796525"
        }
    ]
}
```

**权重:**
5

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |



## 用户持仓风险
``
GET /fapi/v1/positionRisk (HMAC SHA256)
``

> **响应:**

```javascript
[
    {
        "symbol": "BTCUSDT",
        "positionAmt": "-204.0",
        "entryPrice": "18224.2",
        "markPrice": "11593.93170873",
        "unRealizedProfit": "1352574.73141908"
    }
]
```

recvWindow | LONG | NO |
timestamp | LONG | YES |



## 账户成交历史 (USER_DATA)
``
GET /fapi/v1/userTrades  (HMAC SHA256)
``

> **响应:**

```javascript
[
  {
    "symbol": "BTCUSDT",
    "id": 28457,
    "orderId": 100234,
    "price": "4.00000100",
    "qty": "12.00000000",
    "commission": "10.10000000",
    "commissionAsset": "BNB",
    "time": 1499865549590,
    "isBuyer": true,
    "isMaker": false
  }
]
```

获取某交易对的成交历史

**权重:**
5

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
startTime | LONG | NO |
endTime | LONG | NO |
fromId | LONG | NO |返回该fromId之后的成交，缺省返回最近的成交
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO |
timestamp | LONG | YES |





# Websocket账户信息推送


* 本篇所列出REST接口的baseurl **https://testnet.binancefuture.com**
* 用于订阅账户数据的 `listenKey` 从创建时刻起有效期为30分钟
* 可以通过`PUT`一个`listenKey`延长30分钟有效期
* `DELETE`一个 `listenKey` 立即关闭当前数据流
* 本篇所列出的websocket接口baseurl: **wss://testnet.binancefuture.com**
* 订阅账户数据流的stream名称为**/stream?streams=\<listenKey\>**
* 每个链接有效期不超过24小时，请妥善处理断线重连。
* 账户数据流的消息**不保证**严格时间序; **请使用 E 字段进行排序**


## 生成listenKey
``
POST /api/v1/listenKey (HMAC SHA256)
``

> **响应:**

```javascript
{
  "listenKey": "pqia91ma19a5s61cv6a81va65sdf19v8a65a1a5s61cv6a81va65sdf19v8a65a1"
}
```

创建一个新的user data stream，返回值为一个listenKey，即websocket订阅的stream名称。

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |


## 延长listenKey有效期
``
PUT /api/v1/listenKey (HMAC SHA256)
``

> **响应:**

```javascript
{}
```

有效期延长至本次调用后30分钟

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |



### 关闭listenKey
``
DELETE /api/v1/listenKey (HMAC SHA256)
``

> **响应:**

```javascript
{}
```

关闭某账户数据流

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |




## balance和position更新推送
账户更新事件的 event type 固定为 `ACCOUNT_UPDATE`
当账户信息有变动时，会推送此事件

> **Payload:**

```javascript
{
  "e": "ACCOUNT_UPDATE",        // 事件类型
  "E": 1564745798939            // 事件时间
  "a": [                        // 账户更新事件
    {
      "B":[                     // 余额信息
        {
          "a":"USDT",           // 资产名称
          "wb":"122624"         // 可用余额
        },
        {
          "a":"BTC",           
          "wb":"0"         
        }
      ],
      "P":[
        {
          "s":"BTCUSDT",          // 交易对
          "pa":"1",               // 仓位
          "ep":"9000",            // 入仓价格
          "up":"1974.15"          // 未实现利润
        }
      ]
    }
  ]
}
```

## 订单/交易 更新推送
当有新订单创建、订单有新成交或者新的状态变化时会推送此类事件
event type统一为 `ORDER_TRADE_UPDATE`

> **Payload:**

```javascript
{
  "e": "ORDER_TRADE_UPDATE",     // 事件类型
  "E": 1564745798939             // 事件时间
  "o":
  {
      "s": "ETHBTC",                 // 交易对
      "c": "211",                    // 客户端自定订单ID
      "S": "BUY",                    // 订单方向
      "o": "LIMIT",                  // 订单类型
      "f": "GTC",                    // Time in force
      "q": "1.00000000",             // 订单原始数量
      "p": "0.10264410",             // 订单原始价格
      "ap": "0.10264410",            // 订单平均价格
      "sp": "0.10264410",            // 订单停止价格
      "x": "NEW",                    // 本次事件的具体执行类型
      "X": "NEW",                    // 订单的当前状态
      "i": 4293153,                  // 订单ID
      "l": "0.00000000",             // 订单末次成交数量
      "z": "0.00000000",             // 订单累计已成交数量
      "L": "0.00000000",             // 订单末次成交价格
      "n": "0",                      // 手续费数量
      "T": 1499405658657,            // 成交时间
      "t": -1,                       // 成交ID
      "b": 100,                      // 买单净值
      "a": 100                       // 卖单净值
  }
}
```

**订单方向**

* BUY 买入
* SELL 卖出

**订单类型**

* MARKET  市价单
* LIMIT	限价单
* STOP		止损单

**本次事件的具体执行类型**

* NEW
* PARTIAL_FILL	部分成交
* FILL				成交
* CANCELED		已撤
* PENDING_CANCEL 待撤
* REJECTED		拒绝订单
* CALCULATED		强平单
* EXPIRED			订单失效
* TRADE			交易
* RESTATED 		

**订单状态**

* NEW
* PARTIALLY_FILLED    
* FILLED
* CANCELED
* REPLACED
* PENDING_CANCEL
* STOPPED
* REJECTED
* EXPIRED
* NEW_INSURANCE		风险保障基金（强平）
* NEW_ADL				自动减仓序列（强平）

**Time in force:**

* GTC 
* IOC
* FOK
* GTX



#错误代码

> error JSON payload:
 
```javascript
{
  "code":-1121,
  "msg":"Invalid symbol."
}
```

Errors consist of two parts: an error code and a message.    
Codes are universal,but messages can vary. 



## 10xx - General Server or Network issues
### -1000 UNKNOWN
 * An unknown error occured while processing the request.

### -1001 DISCONNECTED
 * Internal error; unable to process your request. Please try again.

### -1002 UNAUTHORIZED
 * You are not authorized to execute this request.

### -1003 TOO_MANY_REQUESTS
 * Too many requests queued.
 * Too many requests; please use the websocket for live updates.
 * Too many requests; current limit is %s requests per minute. Please use the websocket for live updates to avoid polling the API.
 * Way too many requests; IP banned until %s. Please use the websocket for live updates to avoid bans.
 
### -1004 DUPLICATE_IP
 * This IP is already on the white list

### -1005 NO_SUCH_IP
 * No such IP has been white listed
 
### -1006 UNEXPECTED_RESP
 * An unexpected response was received from the message bus. Execution status unknown.

### -1007 TIMEOUT
 * Timeout waiting for response from backend server. Send status unknown; execution status unknown.

### -1010 ERROR_MSG_RECEIVED
 * ERROR_MSG_RECEIVED.  
 
### -1011 NON_WHITE_LIST
 * This IP cannot access this route. 
 
### -1013 ILLEGAL_MESSAGE
* INVALID_MESSAGE.

### -1014 UNKNOWN_ORDER_COMPOSITION
 * Unsupported order combination.

### -1015 TOO_MANY_ORDERS
 * Too many new orders.
 * Too many new orders; current limit is %s orders per %s.

### -1016 SERVICE_SHUTTING_DOWN
 * This service is no longer available.

### -1020 UNSUPPORTED_OPERATION
 * This operation is not supported.

### -1021 INVALID_TIMESTAMP
 * Timestamp for this request is outside of the recvWindow.
 * Timestamp for this request was 1000ms ahead of the server's time.

### -1022 INVALID_SIGNATURE
 * Signature for this request is not valid.


## 11xx - Request issues
### -1100 ILLEGAL_CHARS
 * Illegal characters found in a parameter.
 * Illegal characters found in parameter '%s'; legal range is '%s'.

### -1101 TOO_MANY_PARAMETERS
 * Too many parameters sent for this endpoint.
 * Too many parameters; expected '%s' and received '%s'.
 * Duplicate values for a parameter detected.

### -1102 MANDATORY_PARAM_EMPTY_OR_MALFORMED
 * A mandatory parameter was not sent, was empty/null, or malformed.
 * Mandatory parameter '%s' was not sent, was empty/null, or malformed.
 * Param '%s' or '%s' must be sent, but both were empty/null!

### -1103 UNKNOWN_PARAM
 * An unknown parameter was sent.

### -1104 UNREAD_PARAMETERS
 * Not all sent parameters were read.
 * Not all sent parameters were read; read '%s' parameter(s) but was sent '%s'.

### -1105 PARAM_EMPTY
 * A parameter was empty.
 * Parameter '%s' was empty.

### -1106 PARAM_NOT_REQUIRED
 * A parameter was sent when not required.
 * Parameter '%s' sent when not required.

### -1108 BAD_ASSET
 * Invalid asset.

### -1109 BAD_ACCOUNT
 * Invalid account.

### -1110 BAD_INSTRUMENT_TYPE
 * Invalid symbolType.
 
### -1111 BAD_PRECISION
 * Precision is over the maximum defined for this asset.

### -1112 NO_DEPTH
 * No orders on book for symbol.
 
### -1113 WITHDRAW_NOT_NEGATIVE
 * Withdrawal amount must be negative.
 
### -1114 TIF_NOT_REQUIRED
 * TimeInForce parameter sent when not required.

### -1115 INVALID_TIF
 * Invalid timeInForce.

### -1116 INVALID_ORDER_TYPE
 * Invalid orderType.

### -1117 INVALID_SIDE
 * Invalid side.

### -1118 EMPTY_NEW_CL_ORD_ID
 * New client order ID was empty.

### -1119 EMPTY_ORG_CL_ORD_ID
 * Original client order ID was empty.

### -1120 BAD_INTERVAL
 * Invalid interval.

### -1121 BAD_SYMBOL
 * Invalid symbol.

### -1125 INVALID_LISTEN_KEY
 * This listenKey does not exist.

### -1127 MORE_THAN_XX_HOURS
 * Lookup interval is too big.
 * More than %s hours between startTime and endTime.

### -1128 OPTIONAL_PARAMS_BAD_COMBO
 * Combination of optional parameters invalid.

### -1130 INVALID_PARAMETER
 * Invalid data sent for a parameter.
 * Data sent for paramter '%s' is not valid.

### -2008 BAD_API_ID
 * Invalid Api-Key ID
 
### -2010 NEW_ORDER_REJECTED
 * NEW_ORDER_REJECTED

### -2011 CANCEL_REJECTED
 * CANCEL_REJECTED

### -2013 NO_SUCH_ORDER
 * Order does not exist.

### -2014 BAD_API_KEY_FMT
 * API-key format invalid.

### -2015 REJECTED_MBX_KEY
 * Invalid API-key, IP, or permissions for action.

### -2016 NO_TRADING_WINDOW
 * No trading window could be found for the symbol. Try ticker/24hrs instead.

### -4000 INVALID_ORDER_STATUS
 * Invalid order status.

### -4001 PRICE_LESS_THAN_ZERO
 * Price less than 0.

### -4002 PRICE_GREATER_THAN_MAX_PRICE
 * Price greater than max price.
 
### -4003 QTY_LESS_THAN_ZERO
 * Quantity less than zero.

### -4004 QTY_LESS_THAN_MIN_QTY
 * Quantity less than min quantity.
 
### -4005 QTY_GREATER_THAN_MAX_QTY
 * Quantity greater than max quantity. 

### -4006 STOP_PRICE_LESS_THAN_ZERO
 * Stop price less than zero. 
 
### -4006 STOP_PRICE_GREATER_THAN_MAX_PRICE
 * Stop price greater than max price. 
 
## Messages for -1010 ERROR_MSG_RECEIVED, -2010 NEW_ORDER_REJECTED, and -2011 CANCEL_REJECTED
This code is sent when an error has been returned by the matching engine.
The following messages which will indicate the specific error:


Error message | Description
------------ | ------------
"Unknown order sent." | The order (by either `orderId`, `clientOrderId`, `origClientOrderId`) could not be found
"Duplicate order sent." | The `clientOrderId` is already in use
"Market is closed." | The symbol is not trading
"Account has insufficient balance for requested action." | Not enough funds to complete the action
"Market orders are not supported for this symbol." | `MARKET` is not enabled on the symbol
"Stop loss orders are not supported for this symbol." | `STOP_LOSS` is not enabled on the symbol
"Stop loss limit orders are not supported for this symbol." | `STOP_LOSS_LIMIT` is not enabled on the symbol
"Take profit orders are not supported for this symbol." | `TAKE_PROFIT` is not enabled on the symbol
"Take profit limit orders are not supported for this symbol." | `TAKE_PROFIT_LIMIT` is not enabled on the symbol
"Price * QTY is zero or less." | `price` * `quantity` is too low
"This action disabled is on this account." | Contact customer support; some actions have been disabled on the account.
"Unsupported order combination" | The `orderType`, `timeInForce`, `stopPrice` combination isn't allowed.
"Order would trigger immediately." | The order's stop price is not valid when compared to the last traded price.
"Cancel order is invalid. Check origClientOrderId and orderId." | No `origClientOrderId` or `orderId` was sent in.
"Order would immediately match and take." | `LIMIT_MAKER` order type would immediately match and trade, and not be a pure maker order.

## -9xxx Filter failures
Error message | Description
------------ | ------------
"Filter failure: PRICE_FILTER" | `price` is too high, too low, and/or not following the tick size rule for the symbol.
"Filter failure: LOT_SIZE" | `quantity` is too high, too low, and/or not following the step size rule for the symbol.
"Filter failure: MARKET_LOT_SIZE" | `MARKET` order's `quantity` is too high, too low, and/or not following the step size rule for the symbol.
"Filter failure: MAX_NUM_ORDERS" | Account has too many open orders on the symbol.


