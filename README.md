
参考：


* [https://i4t.com/19399\.html](https://github.com)
* [https://github.com/apache/apisix/issues/9193](https://github.com)
* [https://github.com/apache/apisix/issues/9830](https://github.com):[veee加速器](https://youhaochi.com)
* [https://apisix.apache.org/docs/apisix/plugins/limit\-conn/](https://github.com)
* [https://blog.frankel.ch/different\-rate\-limits\-apisix/](https://github.com)


在 Apache APISIX 中，限流共提供了三种方式，下面说一下


* limit\-req 插件使用漏桶算法限制单个客户端对服务的请求速率
* limit\-conn 插件用于限制客户端对单个服务的并发请求数。当客户端对路由的并发请求数达到限制时，可以返回自定义的状态码和响应信息。
* limit\-count 件使用固定时间窗口算法，主要用于限制单个客户端在指定的时间范围内对服务的总请求数，并且会在 HTTP 响应头中返回剩余可以请求的个数。该插件原理与 GitHub API 的速率限制类似


# 请求的限制参数


## key\_type


* \["var", "var\_combination"]
* var就是一个变更，不需要加$前缀
* var\_combination是一组变更的组合，如key为：`$http_sub $http_x_forwarded_for`


## key


* \["remote\_addr", "server\_addr", "http\_x\_real\_ip", "http\_x\_forwarded\_for", "consumer\_name"]
* 如果限流标识是在请求头里，如请求头是sub，那么key可以配置为`http_sub`
* 


## var和var\_combination类型对应的配置


* var\_combination



```
"plugins": {
"limit-req": {
  "burst": 10,
  "key": "$consumer_name $remote_addr",
  "key_type": "var_combination",
  "rate": 2,
  "rejected_code": 502
}
}

```

* var



```
"plugins": {
    "limit-req": {
      "burst": 2,
      "key": "remote_addr",
      "key_type": "var",
      "rate": 3,
      "rejected_code": 502
     },
    "limit-count": {
          "allow_degradation": false,
          "count": 5,
          "key": "http_sub",
          "key_type": "var",
          "policy": "local",
          "rejected_code": 429,
          "rejected_msg": "请求太多了",
          "show_limit_quota_header": true,
          "time_window": 60
        }
}

```

## 配置redis


* redis中key的组成`plugin-limit-countroute540160115966214871`:`2389086815`:`347c9e9e-076c-45e3-be74-c482fffcc6e5`
	+ plugin\-插件名route路由id::limit里的key值
* 单节点



```
"plugins": {
    "limit-count": {
        "count": 2,
        "time_window": 60,
        "rejected_code": 503,
        "key": "remote_addr",
        "policy": "redis",
        "redis_host": "127.0.0.1",
        "redis_port": 6379,
        "redis_password": "password",
        "redis_database": 1,
        "redis_timeout": 1001
    }
}

```

* 集群模式



```
"plugins": {
    "limit-count": {
        "count": 2,
        "time_window": 60,
        "rejected_code": 503,
        "key": "remote_addr",
        "policy": "redis-cluster",
        "redis_cluster_nodes": [
          "127.0.0.1:5000",
          "127.0.0.1:5001"
        ]
    }
}

```

## 插件的解释


* limit\-req



```
rate:
    含义：每秒允许的请求数量。
    示例：rate: 3 表示每个客户端（根据配置的 key）每秒最多可以发送 3 个请求。

burst:
    含义：突发请求的最大数量。
    示例：burst: 2 表示在短时间内，客户端可以突发额外的 2 个请求，即在正常速率为 3 的情况下，可以在某些时刻达到 5 个请求（3 + 2）。

```

* limit\-count



```
count:
    含义：在指定的时间窗口内允许的最大请求次数。
    示例：count: 2 表示每个客户端（根据配置的 key）在 time_window 指定的时间段内最多可以发送 2 个请求。

time_window:
    含义：时间窗口的长度，以秒为单位。
    示例：time_window: 60 表示时间窗口为 60 秒。在这 60 秒内，客户端最多可以发送 2 个请求。


```

## 对redis\-key的解释


在 APISIX 中使用 `limit-count` 插件时，生成的 Redis 键通常包含多个部分，这些部分用来标识具体的限流策略和相关信息。你提到的键示例是：



```
plugin-limit-countroute540160115966214871:2389086815:347c9e9e-076c-45e3-be74-c482fffcc6e5

```

我们可以将这个键分解为几个部分以便理解：


### 键的组成部分


1. **`plugin-limit-count`**：


	* 这是固定的前缀，表示该键是由 `limit-count` 插件生成的。
2. **`route540160115966214871`**：


	* 这一部分通常表示与特定路由（route）相关的信息。
	* `540160115966214871` 是路由的唯一标识符（ID），它对应于 APISIX 中配置的某个具体路由。这使得限流策略能够应用于特定的路由。
3. **`2389086815`**：


	* 这一部分通常表示限流的维度或时间窗口的标识符，可能是一个时间戳或者是请求计数的某一部分。
	* 具体含义取决于你的限流配置，比如它可能是用于区分不同时间段内的请求计数。
4. **`347c9e9e-076c-45e3-be74-c482fffcc6e5`**：


	* 这一部分一般是用户标识符（如客户端 ID 或者其他唯一标识符），用于在限流策略中区分不同的用户或客户端。
	* 这种设计允许对不同的用户或客户端分别进行限流控制。


### 总结


整个键的结构是为了确保每个限流实例都是唯一的，并且能够精确地与特定的路由和用户关联。通过这样的设计，APISIX 可以高效地管理和应用限流策略，以保障系统的稳定性和性能。


# redis\-count和redis\-req的使用场景


漏桶算法（Leaky Bucket）和固定窗口限流（Fixed Window）是两种常见的限流策略，它们适用于不同的使用场景。以下是这两种限流策略的特点及其适用场景：


### 漏桶算法（Leaky Bucket）


#### 特点：


* **平滑输出**：漏桶算法允许请求以固定速率被处理，尽管输入请求可以是不规则的。
* **水位限制**：桶内有一个容量限制，当达到上限时，多余的请求将被丢弃或拒绝。
* **时间延迟**：由于请求是以固定速率“漏出”的，因此可能会引入一定的延迟。


#### 使用场景：


1. **网络流量控制**：适合需要平稳流量输出的场景，如视频流、音频流等。
2. **API调用限制**：在需要防止`突发流量`对后端服务造成影响的情况下，可以使用漏桶算法来平衡请求量。
3. **实时数据处理**：适合处理实时数据流的场景，例如消息队列中的消息消费。
4. **用户行为限制**：例如限制用户在一定时间内的操作次数，以防止刷票、刷单等行为。


### 固定窗口限流（Fixed Window）


#### 特点：


* **时间窗口**：将时间划分为固定长度的窗口，在每个窗口内允许一定数量的请求。
* **突发性**：在窗口开始时，所有请求都可以瞬间进入，但在窗口结束前不会再接受新的请求，直到下一个窗口开始。
* **简单易实现**：相对其他限流算法，固定窗口的实现较为简单。


#### 使用场景：


1. **日常访问控制**：适合控制用户在特定时间段内的访问频率，比如限制用户每天的登录次数。
2. **API接口调用**：用于限制外部系统对内部API的调用频率，确保系统稳定性。
3. **短时间高并发场景**：如电商促销活动期间，可以快速处理大量请求，但需要限制每个用户的请求频率。
4. **统计分析**：适合进行简单的统计计算，如记录某一时间窗口内的访问量。


### 总结


* **漏桶算法**更适合需要平滑流量控制的场景，能够有效防止突发流量造成的系统压力。
* **固定窗口限流**则适合短时间内的请求控制，简单易用，适合一些对时间窗口要求不严格的应用。


根据具体的业务需求和系统架构，可以选择合适的限流策略来保障系统的稳定性和可用性。


