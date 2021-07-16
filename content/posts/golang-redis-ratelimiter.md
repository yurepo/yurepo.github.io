---
title: "Golang Redis Ratelimiter"
date: 2021-03-16T02:12:14+08:00
draft: false
tags:
  - golang
  - gin
  - ratelimiter
categories:
  - golang
---
# 令牌桶演算法實現

## 前言

這幾天將令牌桶限流演算法使用gin + redis實現了，今天主要要來講整個限流過程是如何運作的。

[Github連結](https://github.com/aitay721822/gin-redis-ratelimiter)

主要定義了四個文件

| 文件名 | 描述 |
| ---- | ---- |
| dto.go | 定義及宣告Ratelimiter的基本結構與核心`Take()`方法 |
| err.go | 定義了一些錯誤 |
| ratelimiter.go | 存放gin middleware中驗證的邏輯 |
| script.go | 存放 lua 腳本及 lua 腳本中輸入變數的結構 |

我們會將上次更新的時間與剩餘令牌的數量儲存在Redis中，而主要更新的邏輯會寫在`script.go`，這裡會發現整個操作Redis資料庫的邏輯是使用lua script去實作的，把多個操作Redis的指令包在lua script中，Redis會保證lua script中的多個操作會以Atomic的方式進行，這樣才可以保證每個操作之間不會有競爭情況(Race Condition)發生。
[參考資料](https://redis.io/commands/eval#atomicity-of-scripts)

## 程式碼部分

### Golang

接下來就進入到程式碼的部分
首先我定義了 `RedisRateLimiter` 結構

`dto.go`

```GO
type RedisRateLimiter struct {
 context      context.Context
 scriptSHA1   string
 client       *redis.Client
}
```

這邊將`RedisRateLimiter`的一些必須用到的變數包裝起來，包裝的變數型別包含了Redis客戶端、LUA Script SHA1(後續會使用evalsha調用已經讀進Redis腳本緩存的Lua script)、Goroutine context。

這個Repository實現的演算法是[Token Bucket](https://en.wikipedia.org/wiki/Token_bucket)演算法，不過也可以利用上述定義的結構去實現不同算法，例如：[Leaky Bucket](https://en.wikipedia.org/wiki/Leaky_bucket)。

`dto.go`

```Go
type TokenBucketRedisRateLimiter struct {
  RedisRateLimiter

  identifier      string
  interval        time.Duration
  maxRequest      int
}

func (r *TokenBucketRedisRateLimiter) Take(request TokenBucketLuaRequest) *LimiterResponse {
 result, err := r.client.EvalSha(
  r.context,
  r.scriptSHA1,
  []string{request.valueKey, request.timestampKey},
  request.limit, request.interval, request.batchSize,
 ).Result()

 if err != nil {
  return &LimiterResponse {
   status: false,
   remain: 0,
   err: err,
  }
 } else {
  data := result.([]interface{})
  if len(data) != 2 {
   return &LimiterResponse{
    status: false,
    remain: 0,
    err:    ErrRedisError,
   }
  }
  return &LimiterResponse{
   status: data[0] == nil,
   remain: data[1].(int64),
   err:    nil,
  }
 }
}

```

這邊的邏輯很簡單，就是透過Redis.Client執行存入的Lua Script，並回傳其執行結果，得益於Redis單線程的優點，使用Lua Script不會發生Race Condition。(因為執行Lua Script時，Redis是將Lua Script視為一個Atomic的操作。)

`ratelimiter.go`

```go

func (r *TokenBucketRedisRateLimiter) Middleware() gin.HandlerFunc {
 return func(context *gin.Context) {
  ip := context.ClientIP()
  if ip == "" {
   _ = context.AbortWithError(http.StatusInternalServerError, ErrIpNotRecognize)
  }
  request := TokenBucketLuaRequest{
   valueKey:    fmt.Sprintf("%v_%v_Token", r.identifier, ip),
   timestampKey: fmt.Sprintf("%v_%v_Update_Time", r.identifier, ip),
   limit:     int64(r.maxRequest),
   interval:     r.interval.Milliseconds(),
   batchSize:    1,
  }
  response := r.Take(request)
  if response.status {
   context.Writer.Header().Set("X-RateLimit-Remaining", strconv.FormatInt(response.remain, 10))
   context.Writer.Header().Set("X-RateLimit-Limit", strconv.Itoa(r.maxRequest))
   context.Next()
  } else {
   _ = context.AbortWithError(http.StatusTooManyRequests, TooManyRequest)
  }
 }
}
func NewRedisRateLimiter(ctx context.Context, identifier string,interval time.Duration, times int, redisClient *redis.Client) *TokenBucketRedisRateLimiter {
 script := TokenBucketLuaScript
 scriptSHA1 := fmt.Sprintf("%x", sha1.Sum([]byte(script)))

 if !redisClient.ScriptExists(ctx, scriptSHA1).Val()[0] {
  redisClient.ScriptLoad(ctx, script).Val()
 }

 return &TokenBucketRedisRateLimiter{
  RedisRateLimiter: RedisRateLimiter{
   context:    ctx,
   scriptSHA1: scriptSHA1,
   client:     redisClient,
  },
  identifier:  identifier,
  interval:       interval,
  maxRequest:     times,
 }
}

```

`TokenBucketRedisRateLimiter`傳入的變數意義
| 文件名 | 描述 |
| ---- | ---- |
| RedisRateLimiter | 傳入Redis客戶端及腳本sha1及Goroutine上下文 |
| identifier | 區別不同middleware的值， |
| interval | 令牌過期時間 |
| maxRequest | 最大請求量 |

Middleware 運作步驟：
  
  1. 取得客戶端IP，若為空回傳500錯誤
  2. 建立要送到Lua腳本的請求.
  3. 調用Lua腳本並返回結果
  4. 若結果為未被拒絕(rejected: false)，設置回傳Header: `X-RateLimit-Remaining`及`X-RateLimit-Limit`，反之則回傳429錯誤。


### Lua Script

最關鍵的Lua Script我們分幾個部分來看它。

第一個部分： 傳入變數

```lua
-- Request value
local valueKey     = KEYS[1]
local timestampKey = KEYS[2]
local limit        = tonumber(ARGV[1])
local interval     = tonumber(ARGV[2]) -- milliseconds
local batchSize    = math.max(tonumber(ARGV[3]), 0)
```

這邊做個表格簡單表示一下變數代表的意思

| 變數名 | 描述 |
| ---- | ---- |
| valueKey | 令牌數量 |
| timestampKey | 上次存取的時間戳(ms) |
| limit | 最大令牌量 |
| interval | 令牌過期時間 |
| batchSize | 執行操作需要消耗的令牌量 |

Token Bucket 演算法是透過剩餘的令牌量去評估請求是否可以被執行，那這邊就會發現一個問題了，Redis只儲存了上次令牌剩餘量為多少，所以必須透過`timestampKey`求得當前剩餘量，計算方式應為：`上次剩餘令牌 + 從上次存取時間到目前時間累積的令牌量`。

```lua
-- Response value
local rejected     = false
local remainToken  = 0
```

這邊就僅附上變數代表的意義，後續說明時會提到這兩個變數。
| 變數名 | 描述 |
| ---- | ---- |
| rejected | 是否被拒絕 |
| remainToken | 剩餘令牌數量 |

```lua

redis.replicate_commands()

local time = redis.call('TIME')
local currentTime = math.floor(time[1] * 1000 + time[2] / 1000)
local modified = false
local lastRemainToken = redis.call('GET', valueKey)
local lastUpdateTime = false

if lastRemainToken == false then
   lastRemainToken = 0
   lastUpdateTime = currentTime - interval
else 
   lastUpdateTime = redis.call('GET', timestampKey)
   if lastUpdateTime == false then
      modified = true
      lastUpdateTime = currentTime - ((lastRemainToken / limit) * interval)
   end
end

```

`redis.replicate_commands()`: 在Redis 3.2以前，Lua腳本的撰寫必須都是確定性的，也就是說假設今天有1個master與2個slave instance，那Lua腳本必須在三個instance中都產生相同的結果，所以就會導致一些非確定性的指令不能使用，像是`redis.call('TIME')`，所以在Redis 3.2後若要使用非確定性指令的話需要調用此函數。[Redis 5後已將腳本效果複製模式設為默認，因此不需要顯式調用)](https://redis.io/commands/EVAL#replicating-commands-instead-of-scripts)

首先我們要取得現在時間(ms)，這邊的`redis.call('TIME')`會回傳兩個值回來，一個是Unix timestamp(以秒為單位)一個是當前時間(微秒)，所以當前時間戳(以毫秒為單位)計算公式是`math.floor(time[1] * 1000 + time[2] / 1000)`。

接下來需要取得上次剩餘的令牌量，那這會有兩種情況

- 無法取得上次剩餘的令牌量: 將上次剩餘的令牌量設為0，並且將上次更新時間設為現在時間減去過期時間(這樣剛好在下一個動作會將其令牌桶補滿)
- 可以取得上次剩餘的令牌量: 取得上次令牌量後再取得上次更新時間，如果無法取得上次更新時間的話，通常是timestampKey過期但valueKey沒過期，因此需要回推上次更新時間，其計算方式為: `現在時間 - ((剩餘令牌 / 令牌桶最大限制) * 時間區段`

```lua

-- feedbackToken: max((現在時間 - 過去時間) / 時間間隔 * 最大限制數量, 0)
local feedbackToken = math.max((currentTime - lastUpdateTime) / interval * limit, 0)
local token = math.min(lastRemainToken + feedbackToken, limit)
remainToken = token - batchSize

if remainToken < 0 then
   rejected = true
   remainToken = token
end

if rejected == false then
   redis.call('PSETEX', valueKey, interval, remainToken)
   if feedbackToken > 0 or modified then
      redis.call('PSETEX', timestampKey, interval, currentTime)
   else 
      redis.call('PEXPIRE', timestampKey, interval)
   end
end

return { rejected, remainToken }

```

接下來要計算回饋令牌，也就是`從上次存取時間到目前時間累積的令牌量`，計算完後需要判斷當前令牌有沒有滿出令牌桶，若滿出則丟棄令牌。

計算完當前令牌剩餘數量後判斷減去`batchSize`後是否還大於等於0，若小於0代表拒絕這此請求，最後再設置當前剩餘令牌數量及其過期時間及延長/設置上次更新時間及其過期時間。

## 測試結果

最後附上測試結果(使用apache bench):
![圖像](/img/gin-redis-ratelimiter/gin-ratelimiter-apache-bench.png)

可以看到傳送了1200個Request，僅有1000個請求被允許(剩餘200個全都拋出429 Too Many Request)。
