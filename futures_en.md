---
title: Binance API Documentation
language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  #- javascript
  #- jason

toc_footers:
  - <a href='https://testnet.binancefuture.com/'>Binance Future Testnet</a>

includes:

search: true
---

# General Info

## General Rest-API Information
* The base endpoint is: **https://testnet.binancefuture.com**
* All endpoints return either a JSON object or array.
* Data is returned in **ascending** order. Oldest first, newest last.
* All time and timestamp related fields are in milliseconds.
* HTTP `4XX` return codes are used for for malformed requests;
  the issue is on the sender's side.
* HTTP `429` return code is used when breaking a request rate limit.
* HTTP `418` return code is used when an IP has been auto-banned for continuing to send requests after receiving `429` codes.
* HTTP `5XX` return codes are used for internal errors; the issue is on
  Binance's side.
  It is important to **NOT** treat this as a failure operation; the execution status is
  **UNKNOWN** and could have been a success.
* Any endpoint can return an ERROR

> ***the error payload is as follows:***
 
```javascript
{
  "code": -1121,
  "msg": "Invalid symbol."
}
```

* Specific error codes and messages defined in [Error Info](#error-info).
* For `GET` endpoints, parameters must be sent as a `query string`.
* For `POST`, `PUT`, and `DELETE` endpoints, the parameters may be sent as a
  `query string` or in the `request body` with content type
  `application/x-www-form-urlencoded`. You may mix parameters between both the
  `query string` and `request body` if you wish to do so.
* Parameters may be sent in any order.
* If a parameter sent in both the `query string` and `request body`, the
  `query string` parameter will be used.

## LIMITS
* The `/fapi/v1/exchangeInfo` `rateLimits` array contains objects related to the exchange's `RAW_REQUEST`, `REQUEST_WEIGHT`, and `ORDER` rate limits. These are further defined in the `ENUM definitions` section under `Rate limiters (rateLimitType)`.
* A `429` will be returned when either rate limit is violated.
* Each route has a `weight` which determines for the number of requests each endpoint counts for. Heavier endpoints and endpoints that do operations on multiple symbols will have a heavier `weight`.
* Every request will contain a `X-MBX-USED-WEIGHT` header which has the current used weight for the IP for the current minute.
* When a `429` returned, it's your obligation as an API to back off and not spam the API.
* **Repeatedly violating rate limits and/or failing to back off after receiving `429` will result in an automated IP ban (http status `418`).**
* IP bans are tracked and **scale in duration** for repeat offenders, **from 2 minutes to 3 days**.
* A `Retry-After` header is sent with a `418` or `429` responses and will give the **number of seconds** required to wait, in the case of a `418`, to prevent a ban, or, in the case of a `429`, until the ban is over.

## Endpoint security type
* Each endpoint has a security type that determines the how you will
  interact with it.
* API-keys are passed into the Rest API via the `X-MBX-APIKEY`
  header.
* API-keys and secret-keys **are case sensitive**.
* API-keys can be configured to only access certain types of secure endpoints.
 For example, one API-key could be used for TRADE only, while another API-key
 can access everything except for TRADE routes.
* By default, API-keys can access all secure routes.

Security Type | Description
------------ | ------------
NONE | Endpoint can be accessed freely.
TRADE | Endpoint requires sending a valid API-Key and signature.
USER_DATA | Endpoint requires sending a valid API-Key and signature.
USER_STREAM | Endpoint requires sending a valid API-Key.
MARKET_DATA | Endpoint requires sending a valid API-Key.


* `TRADE` and `USER_DATA` endpoints are `SIGNED` endpoints.

## SIGNED (TRADE and USER_DATA) Endpoint security
* `SIGNED` endpoints require an additional parameter, `signature`, to be
  sent in the  `query string` or `request body`.
* Endpoints use `HMAC SHA256` signatures. The `HMAC SHA256 signature` is a keyed `HMAC SHA256` operation.
  Use your `secretKey` as the key and `totalParams` as the value for the HMAC operation.
* The `signature` is **not case sensitive**.
* Please make sure the `signature` is the end part of your `query string` or `request body`.
* `totalParams` is defined as the `query string` concatenated with the
  `request body`.

### Timing security
* A `SIGNED` endpoint also requires a parameter, `timestamp`, to be sent which
  should be the millisecond timestamp of when the request was created and sent.
* An additional parameter, `recvWindow`, may be sent to specify the number of
  milliseconds after `timestamp` the request is valid for. If `recvWindow`
  is not sent, **it defaults to 5000**.
  
> The logic is as follows:

```javascript
  if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow）{
    // process request
  } 
  else {
    // reject request
  }
```

**Serious trading is about timing.** Networks can be unstable and unreliable,
which can lead to requests taking varying amounts of time to reach the
servers. With `recvWindow`, you can specify that the request must be
processed within a certain number of milliseconds or be rejected by the
server.


**It recommended to use a small recvWindow of 5000 or less!**


### SIGNED Endpoint Examples for POST /fapi/v1/order
Here is a step-by-step example of how to send a vaild signed payload from the
Linux command line using `echo`, `openssl`, and `curl`.

Key | Value
------------ | ------------
apiKey | vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A
secretKey | NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j


Parameter | Value
------------ | ------------
symbol | BTCUSDT
side | BUY
type | LIMIT
timeInForce | GTC
quantity | 1
price | 0.1
recvWindow | 5000
timestamp | 1499827319559


#### Example 1: As a query string

> **Example 1**

>  **HMAC SHA256 signature:**

```shell
    $ echo -n "symbol=BTCUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
    
```
> **curl command:**

```shell
    (HMAC SHA256)
    $ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://testnet.binancefuture.com/fapi/v1/order?symbol=BTCUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
    
```
* **queryString:** 

	symbol=BTCUSDT  
	&side=BUY   
	&type=LIMIT  
	&timeInForce=GTC   
	&quantity=1  
	&price=0.1   
	&recvWindow=5000   
	&timestamp=1499827319559




#### Example 2: As a request body

> **Example 2**

> **HMAC SHA256 signature:**

```shell
    $ echo -n "symbol=BTCUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
```


> **curl command:**

```shell
    (HMAC SHA256)
    $ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://testnet.binancefuture.com/fapi/v1/order' -d 'symbol=BTCUSDT&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
```

* **requestBody:**

	symbol=BTCUSDT   
	&side=BUY   
	&type=LIMIT   
	&timeInForce=GTC   
	&quantity=1   
	&price=0.1   
	&recvWindow=5000   
	&timestamp=1499827319559



#### Example 3: Mixed query string and request body

> **Example 3**

> **HMAC SHA256 signature:**

```shell
    $ echo -n "symbol=BTCUSDT&side=BUY&type=LIMIT&timeInForce=GTCquantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= 0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77
```

> **curl command:**

```shell
    (HMAC SHA256)
    $ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://testnet.binancefuture.com/fapi/v1/order?symbol=BTCUSDT&side=BUY&type=LIMIT&timeInForce=GTC' -d 'quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77'
```

* **queryString:** symbol=BTCUSDT&side=BUY&type=LIMIT&timeInForce=GTC
* **requestBody:** quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559


<aside class="notice">
Note that the signature is different in example 3.
There is no & between "GTC" and "quantity=1".
</aside>

## Public Endpoints Info
### Terminology
* `base asset` refers to the asset that is the `quantity` of a symbol.
* `quote asset` refers to the asset that is the `price` of a symbol.


### ENUM definitions

**Symbol type:**

* FUTURE

**Order status (status):**

* NEW
* PARTIALLY_FILLED
* FILLED
* CANCELED
* PENDING_CANCEL (currently unused)
* REJECTED
* EXPIRED

**Order types (orderTypes, type):**

* LIMIT
* MARKET
* STOP

**Order side (side):**

* BUY
* SELL

**Time in force (timeInForce):**

* GTC - Good Till Cancel
* OC - Immediate or Cancel
* FOK - Fill or Kill
* GTX - Good Till Crossing		

**Kline/Candlestick chart intervals:**

m -> minutes; h -> hours; d -> days; w -> weeks; M -> months

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

**Rate limiters (rateLimitType)**

* REQUEST_WEIGHT

```json
  {
  	"rateLimitType": "REQUEST_WEIGHT",
  	"interval": "MINUTE",
  	"intervalNum": 1,
  	"limit": 1200
  }
```

* ORDERS

```json
  {
  	"rateLimitType": "ORDERS",
  	"interval": "SECOND",
  	"intervalNum": 1,
  	"limit": 10
   }
```


**Rate limit intervals (interval)**

* SECOND
* MINUTE
* DAY



## Filters
Filters define trading rules on a symbol or an exchange.

### Symbol filters
#### PRICE_FILTER

> **/exchangeInfo format:**

```javascript
  {
    "filterType": "PRICE_FILTER",
    "minPrice": "0.00000100",
    "maxPrice": "100000.00000000",
    "tickSize": "0.00000100"
  }
```

The `PRICE_FILTER` defines the `price` rules for a symbol. There are 3 parts:

* `minPrice` defines the minimum `price`/`stopPrice` allowed; disabled on `minPrice` == 0.
* `maxPrice` defines the maximum `price`/`stopPrice` allowed; disabled on `maxPrice` == 0.
* `tickSize` defines the intervals that a `price`/`stopPrice` can be increased/decreased by; disabled on `tickSize` == 0.

Any of the above variables can be set to 0, which disables that rule in the `price filter`. In order to pass the `price filter`, the following must be true for `price`/`stopPrice` of the enabled rules:

* `price` >= `minPrice` 
* `price` <= `maxPrice`
* (`price`-`minPrice`) % `tickSize` == 0


#### LOT_SIZE

> **/exchangeInfo format:**

```javascript
  {
    "filterType": "LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

The `LOT_SIZE` filter defines the `quantity` (aka "lots" in auction terms) rules for a symbol. There are 3 parts:

* `minQty` defines the minimum `quantity` allowed.
* `maxQty` defines the maximum `quantity` allowed.
* `stepSize` defines the intervals that a `quantity` can be increased/decreased by.

In order to pass the `lot size`, the following must be true for `quantity`:

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0



#### MARKET_LOT_SIZE


> **/exchangeInfo format:**

```javascript
  {
    "filterType": "MARKET_LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

The `MARKET_LOT_SIZE` filter defines the `quantity` (aka "lots" in auction terms) rules for `MARKET` orders on a symbol. There are 3 parts:

* `minQty` defines the minimum `quantity` allowed.
* `maxQty` defines the maximum `quantity` allowed.
* `stepSize` defines the intervals that a `quantity` can be increased/decreased by.

In order to pass the `market lot size`, the following must be true for `quantity`:

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0


#### MAX_NUM_ORDERS

> **/exchangeInfo format:**

```javascript
  {
    "filterType": "MAX_NUM_ORDERS",
    "limit": 25
  }
```

The `MAX_NUM_ORDERS` filter defines the maximum number of orders an account is allowed to have open on a symbol.
Note that both "algo" orders and normal orders are counted for this filter.




# Market Data Endpoints

## Test connectivity


``
GET /fapi/v1/ping
``

> **Response:**

```javascript
{}
```

Test connectivity to the Rest API.

**Weight:**
1

**Parameters:**
NONE



## Check server time
``
GET /fapi/v1/time
``

> **Response:**

```javascript
{
  "serverTime": 1499827319559
}
```


Test connectivity to the Rest API and get the current server time.

**Weight:**
1

**Parameters:**
NONE


## Exchange information
``
GET /fapi/v1/exchangeInfo
``

> **Response:**

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


Current exchange trading rules and symbol information

**Weight:**
1

**Parameters:**
NONE




## Order book
``
GET /fapi/v1/depth
``

> **Response:**

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

**Weight:**

Adjusted based on the limit:


Limit | Weight
------------ | ------------
5, 10, 20, 50, 100 | 1|
500 | 5
1000 | 10

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 100; max 1000. Valid limits:[5, 10, 20, 50, 100, 500, 1000]




## Recent trades list
``
GET /fapi/v1/trades
``
> **Response:**

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

Get recent trades (up to last 500).

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.



## Old trades lookup
``
GET /fapi/v1/historicalTrades
``
> **Response:**

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

Get older market historical trades.

**Weight:**
5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.
fromId | LONG | NO | TradeId to fetch from. Default gets most recent trades.

* `X-MBX-APIY` required


## Compressed/Aggregate trades list
``
GET /fapi/v1/aggTrades
``
> **Response:**

```javascript
[
  {
    "a": 26129,         // Aggregate tradeId
    "p": "0.01633102",  // Price
    "q": "4.70443515",  // Quantity
    "f": 27781,         // First tradeId
    "l": 27781,         // Last tradeId
    "T": 1498793709153, // Timestamp
    "m": true,          // Was the buyer the maker?
  }
]
```

Get compressed, aggregate trades. Trades that fill at the time, from the same
order, with the same price will have the quantity aggregated.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
fromId | LONG | NO | ID to get aggregate trades from INCLUSIVE.
startTime | LONG | NO | Timestamp in ms to get aggregate trades from INCLUSIVE.
endTime | LONG | NO | Timestamp in ms to get aggregate trades until INCLUSIVE.
limit | INT | NO | Default 500; max 1000.

* If both startTime and endTime are sent, time between startTime and endTime must be less than 1 hour.
* If fromId, startTime, and endTime are not sent, the most recent aggregate trades will be returned.



## Kline/Candlestick data
``
GET /fapi/v1/klines
``

> **Response:**

```javascript
[
  [
    1499040000000,      // Open time
    "0.01634790",       // Open
    "0.80000000",       // High
    "0.01575800",       // Low
    "0.01577100",       // Close
    "148976.11427815",  // Volume
    1499644799999,      // Close time
    "2434.19055334",    // Quote asset volume
    308,                // Number of trades
    "1756.87402397",    // Taker buy base asset volume
    "28.46694368",      // Taker buy quote asset volume
    "17928899.62484339" // Ignore.
  ]
]
```

Kline/candlestick bars for a symbol.
Klines are uniquely identified by their open time.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
interval | ENUM | YES |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.

* If startTime and endTime are not sent, the most recent klines are returned.




## Mark price
``
GET /fapi/v1/premiumIndex
``

> **Response:**

```javascript
{
    "symbol": "BTCUSDT",
    "markPrice": "11012.80409769",
    "lastFundingRate": "-0.03750000",
    "nextFundingTime": 1562569200000,
    "time": 1562566020000
}
```

mark price and funding rate

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |



## 24hr ticker price change statistics
``
GET /fapi/v1/ticker/24hr
``
> **Response:**

```javascript
{
  "symbol": "BTCUSDT",
  "priceChange": "-94.99999800",
  "priceChangePercent": "-95.960",
  "weightedAvgPrice": "0.29628482",
  "prevClosePrice": "0.10002000",
  "lastPrice": "4.00000200",
  "lastQty": "200.00000000",
  "openPrice": "99.00000000",
  "highPrice": "100.00000000",
  "lowPrice": "0.10000000",
  "volume": "8913.30000000",
  "quoteVolume": "15.30000000",
  "openTime": 1499783499040,
  "closeTime": 1499869899040,
  "firstId": 28385,   // First tradeId
  "lastId": 28460,    // Last tradeId
  "count": 76         // Trade count
}
```

> OR

```javascript
[
	{
  		"symbol": "BTCUSDT",
  		"priceChange": "-94.99999800",
  		"priceChangePercent": "-95.960",
  		"weightedAvgPrice": "0.29628482",
  		"prevClosePrice": "0.10002000",
  		"lastPrice": "4.00000200",
  		"lastQty": "200.00000000",
  		"openPrice": "99.00000000",
  		"highPrice": "100.00000000",
  		"lowPrice": "0.10000000",
  		"volume": "8913.30000000",
  		"quoteVolume": "15.30000000",
  		"openTime": 1499783499040,
  		"closeTime": 1499869899040,
  		"firstId": 28385,   // First tradeId
  		"lastId": 28460,    // Last tradeId
  		"count": 76         // Trade count
	}
]
```


24 hour rolling window price change statistics.    
**Careful** when accessing this with no symbol.

**Weight:**   
1 for a single symbol;    
**40** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* If the symbol is not sent, tickers for all symbols will be returned in an array.



## Symbol price ticker
``
GET /fapi/v1/ticker/price
``

> **Response:**

```javascript
{
  "symbol": "BTCUSDT",
  "price": "6000.01"
}
```

> OR


```javascript
[
	{
  		"symbol": "BTCUSDT",
  		"price": "6000.01"
	}
]
```

Latest price for a symbol or symbols.

**Weight:**   
1 for a single symbol;    
**2** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* If the symbol is not sent, prices for all symbols will be returned in an array.



## Symbol order book ticker
``
GET /fapi/v1/ticker/bookTicker
``

> **Response:**

```javascript
{
  "symbol": "BTCUSDT",
  "bidPrice": "4.00000000",
  "bidQty": "431.00000000",
  "askPrice": "4.00000200",
  "askQty": "9.00000000"
}
```

> OR


```javascript
[
	{
  		"symbol": "BTCUSDT",
  		"bidPrice": "4.00000000",
  		"bidQty": "431.00000000",
  		"askPrice": "4.00000200",
  		"askQty": "9.00000000"
	}
]
```

Best price/qty on the order book for a symbol or symbols.

**Weight:**   
1 for a single symbol;    
**2** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |

* If the symbol is not sent, bookTickers for all symbols will be returned in an array.




# Websocket Market Streams

* The base endpoint is: **wss://testnet.binancefuture.com**
* Streams can be access either in a single raw stream or a combined stream
* Combined streams are accessed at **/stream?streams=<streamName1\>/<streamName2\>/<streamName3\>**
* Combined stream events are wrapped as follows: **{"stream":"<streamName\>","data":<rawPayload\>}**
* All symbols for streams are **lowercase**
* The websocket server will send a `ping frame` every 3 minutes. If the websocket server does not receive a `pong frame` back from the connection within a 10 minute period, the connection will be disconnected. Unsolicited `pong frames` are allowed.

## Aggregate Trade Streams
The Aggregate Trade Streams push trade information that is aggregated for a single taker order every 100 milliseconds.

**Stream Name:** 

``<symbol\>@aggTrade``

> **Payload:**

```javascript
{
  "e": "aggTrade",  // Event type
  "E": 123456789,   // Event time
  "s": "BTCUSDT",    // Symbol
  "p": "0.001",     // Price
  "q": "100",       // Quantity
  "f": 100,         // First trade ID
  "l": 105,         // Last trade ID
  "T": 123456785,   // Trade time
  "m": true,        // Is the buyer the market maker?
}
```

## Mark Price Stream
Mark price for a single symbol pushed every 3 secends. 

**Stream Name:** 

``<symbol\>@markPrice``

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


### Kline/Candlestick Streams
The Kline/Candlestick Stream push updates to the current klines/candlestick every 250 milliseconds (if existing).

**Kline/Candlestick chart intervals:**

m -> minutes; h -> hours; d -> days; w -> weeks; M -> months

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

**Stream Name:** 

``<symbol\>@kline_<interval\>``

> **Payload:**

```javascript
{
  "e": "kline",     // Event type
  "E": 123456789,   // Event time
  "s": "BTCUSDT",    // Symbol
  "k": {
    "t": 123400000, // Kline start time
    "T": 123460000, // Kline close time
    "s": "BTCUSDT",  // Symbol
    "i": "1m",      // Interva\
    "f": 100,       // First trade ID
    "L": 200,       // Last trade ID
    "o": "0.0010",  // Open price
    "c": "0.0020",  // Close price
    "h": "0.0025",  // High price
    "l": "0.0015",  // Low price
    "v": "1000",    // Base asset volume
    "n": 100,       // Number of trades
    "x": false,     // Is this kline closed?
    "q": "1.0000",  // Quote asset volume
    "V": "500",     // Taker buy base asset volume
    "Q": "0.500",   // Taker buy quote asset volume
    "B": "123456"   // Ignore
  }
}
```

## Individual Symbol Mini Ticker Stream
24hr rolling window mini-ticker statistics for a single symbol pushed every 3 seconds. These are NOT the statistics of the UTC day, but a 24hr rolling window from requestTime to 24hrs before.

**Stream Name:** 

``<symbol\>@miniTicker``

> **Payload:**

```javascript
  {
    "e": "24hrMiniTicker",  // Event type
    "E": 123456789,         // Event time
    "s": "BTCUSDT",          // Symbol
    "c": "0.0025",          // Close price
    "o": "0.0010",          // Open price
    "h": "0.0025",          // High price
    "l": "0.0010",          // Low price
    "v": "10000",           // Total traded base asset volume
    "q": "18"               // Total traded quote asset volume
  }
```


## Individual Symbol Ticker Streams
24hr rollwing window ticker statistics for a single symbol pushed every 3 seconds. These are NOT the statistics of the UTC day, but a 24hr rolling window from requestTime to 24hrs before.

**Stream Name:** 
``<symbol\>@ticker``

> **Payload:**

```javascript
{
  "e": "24hrTicker",  // Event type
  "E": 123456789,     // Event time
  "s": "BTCUSDT",      // Symbol
  "p": "0.0015",      // Price change
  "P": "250.00",      // Price change percent
  "w": "0.0018",      // Weighted average price
  "c": "0.0025",      // Last price
  "Q": "10",          // Last quantity
  "o": "0.0010",      // Open price
  "h": "0.0025",      // High price
  "l": "0.0010",      // Low price
  "v": "10000",       // Total traded base asset volume
  "q": "18",          // Total traded quote asset volume
  "O": 0,             // Statistics open time
  "C": 86400000,      // Statistics close time
  "F": 0,             // First trade ID
  "L": 18150,         // Last trade Id
  "n": 18151          // Total number of trades
}
```



## Partial Book Depth Streams
Bids and asks, pushed every 250 milliseconds (if existing)

**Stream Name:** 

``<symbol\>@depth``

> **Payload:**

```javascript
{
  "e": "depthUpdate", // Event type
  "E": 123456789,     // Event time
  "s": "BTCUSDT",      // Symbol
  "U": 157,           // first update Id from last stream
  "u": 160,           // last update Id from last stream
  "pu": 149,          // last update Id in last stream（ie ‘u’ in last stream）
  "b": [              // Bids to be updated
    [
      "0.0024",       // Price level to be updated
      "10"            // Quantity
    ]
  ],
  "a": [              // Asks to be updated
    [
      "0.0026",       // Price level to be updated
      "100"          // Quantity
    ]
  ]
}
```


## How to manage a local order book correctly
1. Open a stream to **wss://testnet.binancefuture.com/stream?streams=btcusdt@depth**
2. Buffer the events you receive from the stream
3. Get a depth snapshot from **https://testnet.binancefuture.com/fapi/v1/depth?symbol=BTCUSDT&limit=1000**
4. Drop any event where `u` is <= `lastUpdateId` in the snapshot
5. The first processed should have `U` <= `lastUpdateId`+1 **AND** `u` >= `lastUpdateId`+1
6. While listening to the stream, each new event's `U` should be equal to the previous event's `u`+1
7. The data in each event is the **absolute** quantity for a price level
8. If the quantity is 0, **remove** the price level
9. Receiving an event that removes a price level that is not in your local order book can happen and is normal.






# Account/Trades Endpoints

## New order  (*TRADE*)
``
POST /fapi/v1/order  (HMAC SHA256)
``

> **Response:**

```json
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

Send in a new order.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
side | ENUM | YES |
type | ENUM | YES |
timeInForce | ENUM | NO |
quantity | DECIMAL | YES |
price | DECIMAL | NO |
newClientOrderId | STRING | NO | A unique id for the order. Automatically generated if not sent.
stopPrice | DECIMAL | NO | Used with `STOP` orders.
recvWindow | LONG | NO |
timestamp | LONG | YES |

Additional mandatory parameters based on `type`:

Type | Additional mandatory parameters
------------ | ------------
`LIMIT` | `timeInForce`, `quantity`, `price`
`MARKET` | `quantity`
`STOP` | `quantity`,  `price`, `stopPrice`

* Order with type `MARKET`, parameter `timeInForce` cannot be sent.
* Order with type `STOP`,  parameter `timeInForce` can be sent ( default `GTC`).



## Query order (*USER_DATA*)
``
GET /fapi/v1/order (HMAC SHA256)
``

> **Response:**

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

Check an order's status.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

Notes:

* Either `orderId` or `origClientOrderId` must be sent.



## Cancel order (*TRADE*)
``
DELETE /fapi/v1/order  (HMAC SHA256)
``

> **Response:**

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

Cancel an active order.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
Automatically generated by default.
recvWindow | LONG | NO |
timestamp | LONG | YES |

Either `orderId` or `origClientOrderId` must be sent.



## Current open orders (*USER_DATA*)
``
GET /fapi/v1/openOrders  (HMAC SHA256)
``

> **Response:**

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

Get all open orders on a symbol. **Careful** when accessing this with no symbol.

**Weight:**
1 for a single symbol; **40** when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |




* If the symbol is not sent, orders for all symbols will be returned in an array.

## All orders (*USER_DATA*)
``
GET /fapi/v1/allOrders (HMAC SHA256)
``

> **Response:**

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

Get all account orders; active, canceled, or filled.

**Weight:**
5 with symbol

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO |
timestamp | LONG | YES |

**Notes:**

* If `orderId` is set, it will get orders >= that `orderId`. Otherwise most recent orders are returned.



## Account information (*USER_DATA*)
``
GET /fapi/v1/account (HMAC SHA256)
``

> **Response:**

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

Get current account information.

**Weight:**
5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |



## Position information (*USER_DATA*)
``
GET /fapi/v1/positionRisk (HMAC SHA256)
``
Get current account information.

**Weight:**
5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |

> **Response:**

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

## Account trade list (*USER_DATA*)
``
GET /fapi/v1/userTrades  (HMAC SHA256)
``


> **Response:**

```javascript
[
  {
    "symbol": "BTCUSDT",
    "id": 28457,
    "orderId": 100234,
    "price": "4.00000100",
    "qty": "12.00000000",
    "quoteQty": "48.000012",
    "commission": "10.10000000",
    "commissionAsset": "USDT",
    "time": 1499865549590,
    "isBuyer": true,
    "isMaker": false,
  }
]
```

Get trades for a specific account and symbol.

**Weight:**
5 with symbol

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
startTime | LONG | NO |
endTime | LONG | NO |
fromId | LONG | NO | TradeId to fetch from. Default gets most recent trades.
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO |
timestamp | LONG | YES |

**Notes:**

* If `fromId` is set, it will get orders >= that `fromId`.
Otherwise most recent orders are returned.



# Webscoket User Data Streams

* The base API endpoint is: **https://testnet.binancefuture.com**
* A User Data Stream `listenKey` is valid for 30 minutes after creation.
* Doing a `PUT` on a `listenKey` will extend its validity for 30 minutes.
* Doing a `DELETE` on a `listenKey` will close the stream.
* The base websocket endpoint is: **wss://testnet.binancefuture.com**
* User Data Streams are accessed at **/stream?stream=<listenKey\>**
* User data stream payloads are **not guaranteed** to be in order during heavy periods; **make sure to order your updates using E**


## Start user data stream (USER_STREAM)
``
POST /fapi/v1/listenKey (HMAC SHA256)
``

> **Response:**

```javascript
{
  "listenKey": "pqia91ma19a5s61cv6a81va65sdf19v8a65a1a5s61cv6a81va65sdf19v8a65a1"
}
```

Start a new user data stream. The stream will close after 30 minutes unless a keepalive is sent.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |



## Keepalive user data stream (USER_STREAM)
``
PUT /fapi/v1/listenKey (HMAC SHA256)
``

> **Response:**

```javascript
{}
```

Keepalive a user data stream to prevent a time out. User data streams will close after 30 minutes. It's recommended to send a ping about every 30 minutes.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |



## Close user data stream (USER_STREAM)
``
DELETE /fapi/v1/listenKey (HMAC SHA256)
``

> **Response:**

```javascript
{}
```

Close out a user data stream.

**Weight:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |



##Event: balance and position update
Event type is `ACCOUNT_UPDATE`.   
When balance or position get updated, will push this event.

> **Payload:**

```javascript
{
  "e": "ACCOUNT_UPDATE",        // event type
  "E": 1564745798939            // event time
  "a": [                        
    {
      "B":[                     // balances
        {
          "a":"USDT",           // asset
          "wb":"122624"         // wallet balance
        },
        {
          "a":"BTC",           
          "wb":"0"         
        }
      ],
      "P":[                       // positions 
        {
          "s":"BTCUSDT",          // symbol
          "pa":"1",               // position amount
          "ep":"9000",            // entry price
          "up":"1974.15"          // unrealized profit
        }
      ]
    }
  ]
}
```

##Event: order update
When new order created, order status changed will push such event.   
event type is `ORDER_TRADE_UPDATE`.

> **Payload:**

```javascript
{
  "e": "ORDER_TRADE_UPDATE",     // event type
  "E": 1564745798939             // event time
  "o":
  {
    "s": "BTCUSDT",                 // symbol
    "c": "211",                    // client order id
    "S": "BUY",                    // side
    "o": "LIMIT",                  // order type
    "f": "GTC",                    // Time in force
    "q": "1.00000000",             // Original quantity
    "p": "0.10264410",             // Price
    "ap": "0.10264410",            // average price
    "sp": "0.10264410",            // stop price
    "x": "NEW",                    // execution type
    "X": "NEW",                    // order status
    "i": 4293153,                  // order id 
    "l": "0.00000000",             // order last filled quantity
    "z": "0.00000000",             // order filled accumulated quantity
    "L": "0.00000000",             // last filled price
    "N": "USDT",                   // commission Asset, will not push if no commission
    "n": "0",                      // commission, will not push if no commission
    "T": 1499405658657,            // order trade time
    "t": -1,                       // trade id
    "b": 100,                      // bis notional
    "a": 100                       // ask notional
  }
  
}
```





**Side**

* BUY 
* SELL 

**order type**

* MARKET 
* LIMIT
* STOP

**execution type**

* NEW
* PARTIAL_FILL
* FILL
* CANCELED
* PENDING_CANCEL
* REJECTED
* CALCULATED		 - Liquidation Execution
* EXPIRED
* TRADE
* RESTATED 

**order status**

* NEW
* PARTIALLY_FILLED
* FILLED
* CANCELED
* REPLACED
* PENDING_CANCEL
* STOPPED
* REJECTED
* EXPIRED
* NEW_INSURANCE     - Liquidation with Insurance Fund
* NEW_ADL				- Counterparty Liquidation`

**Time in force**

* GTC
* IOC
* FOK
* GTX


#Error Info

> Here is the error JSON payload:
 
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




