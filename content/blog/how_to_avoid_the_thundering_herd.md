+++
title = "ATS零碎笔记(二) - 避免雪崩"
date = "2016-03-17"
categories = "blog"
+++

在应对高并发的请求时，ATS能够顽强的抗住请求压力，高速的输出缓存内容，但是当请求失效的时候，
ATS不得不向回源机发起请求，希望重新获得这块内容并缓存起来。那么这里就有一个问题：

> 当ATS发现缓存失效，向回源机发起请求，在拿回全部内容之前，这段时间内的用户请求要怎么处理？

<!--more-->

如果这里我们不做任何设计和配置，最原始的方案就是放过所有请求，大家都去回源机请求一次新数据，
那么在这段真空期内，就像开闸泄洪一般，一大批请求在极短的时间内被疯狂打在回源机上，会造成一系列
附带的问题，可能会对后端相关联的服务造成巨大影响。这就是我们需要解决的"雪崩"。

那么定位了问题，解决方案很容易可以想到:

1. 在真空期内，让ATS输出旧内容，并且保证只有一个请求被放过去回源机。
2. 在回源机返回新内容的过程中，ATS尽可能早的向客户端返回新的内容。

这样一来，一方面可以保证回源机的安全，另一方面也可以尽可能快的恢复该页面的缓存服务。

为了实现这样两个目标，我们需要对ATS进行一些配置，开启相应的功能。

## open_write_fail_action

`proxy.config.http.cache.open_write_fail_action`是一个很迷的配置，从字面上来看似乎很难理解
`open_write_fail_action`，其实很简单。当某个请求触发到一个已失效的Cache Obejct时，
ATS会去回源机回源，显然ATS不会允许所有请求都去回源一次，这样即浪费客户端时间也会对回源机施加不必要的压力，
因此ATS在这里针对每一个Cache Key加了一个写入锁，只有一个请求可以拿下这个锁，并触发回源机制，
其他请求发现拿不到这个锁的时候，就会采取`proxy.config.http.cache.open_write_fail_action`这里配置的行为，
有以下几种行为可以被配置，当然，这里是在`records.config`当中配置并reload：

    Scope:          CONFIG
    Type:           INT
    Default:        0
    Reloadable:     Yes
    Overridable:    Yes

* 0 : 默认，所有请求都去回源。
* 1 : 如果缓存Miss返回502，其他情况回源。
* 2 : 如果缓存对象的生存时间小于`proxy.config.http.cache.max_stale_age`返回旧版本缓存，否则回源。
* 3 : 如果缓存Miss返回502，如果有缓存但是已过期则和配置2一致。
* 4 : 无论缓存Miss或失效都返回502。

需要强调的是，以上配置被触发的条件是：

1. 缓存Miss或缓存过期。
2. 同时有多个请求，除了回源的那个以外其他请求拿不到写入锁。

其中action = 2的配置实际产生效果的位置是在源码中`proxy/http/HttpTransact.cc`的`HttpTransact::what_is_document_freshness`中。

## Read While Writer

边写边读机制在各类缓存服务器都有采用，大部分的缓存服务器会在收到回源机返回的请求后，立刻向客户端输出新数据，
也就是说从回源机开始返回Header头信息的时候，缓存服务器拿到一点就向客户端返回一点，尽可能的减少客户端排队的时间。
在ATS这里情况有些不同，ATS会等到Header头信息返回完毕之后，再向客户端开始返回内容，这样ATS可以从头信息知道
该缓存是否需要更新，如果缓存并未修改，那么ATS可以很愉快的根据Header的`Cache Control`重置旧缓存的有效期，
如果缓存有修改，那么ATS会向其他缓存服务器那样一边接收回源机内容，一边向客户端返回新内容。

为了开启这个功能，需要在`records.config`中进行配置：

    CONFIG proxy.config.cache.enable_read_while_writer INT 1
    CONFIG proxy.config.cache.max_doc_size INT 0
    CONFIG proxy.config.http.background_fill_active_timeout INT 0
    CONFIG proxy.config.http.background_fill_completed_threshold FLOAT 0.000000

`cache.enable_read_while_writer`功能开关，默认为0关闭。

`cache.max_doc_size`单个缓存的容量上限。

`http.background_fill_active_timeout`配置ATS Background fill回源的超时时间。这里设置为0，
也就是当某客户端请求触发回源之后，这个回源请求永远不会超时。

`http.background_fill_completed_threshold`配置一个百分比，当客户端请求拿到回源机数据百分之多少的时候，
ATS可以介入接管writer。这里设置为百分之零，这样ATS可以立刻介入接管，不用担心触发回源的客户端请求断开。

这样还没有结束，还有两个配置

    CONFIG proxy.config.cache.read_while_writer.max_retries INT 10
    CONFIG proxy.config.cache.read_while_writer_retry.delay INT 50

`read_while_writer.max_retries`设置了ATS尝试获取写入锁的最大重试次数。
`read_while_writer_retry.delay`设置了尝试获取写入锁的间隔时间，需要注意的是ATS采取一种步进的方式，
下一次的间隔时间是本次时间的两倍。

## 模糊回源

ATS可以根据给定的概率在一个缓存失效之前尝试回源。这样虽然大家失效时间一模一样，也会有不同的回源时机。
同样在`records.config`中配置

    CONFIG proxy.config.http.cache.fuzz.time INT 240
    CONFIG proxy.config.http.cache.fuzz.probability FLOAT 0.005
    CONFIG proxy.config.http.cache.fuzz.min_time INT 0

对于一个在某缓存失效前的`fuzz.time`秒内的请求，它有`fuzz.probability` * 100% 的概率触发一次回源，
对于生存时间小于`fuzz.time`的缓存，会根据`fuzz.min_time`的配置，在该时间开始随机触发回源。

**需要注意的是：当回源动作触发之后，这个缓存会被认定为失效，之后的请求都会触发缓存失效逻辑，
结合文章开始时提到的`open_write_fail_action`，即可悄无声息的提前回源掉这个缓存。**

这个配置也可以在`remap.config`或通过插件覆盖，这样可以针对不同的请求单独调整。

对于生存时间很短的缓存来说，务必要开启`fuzz.min_time`，这样在命中概率后，会在 `fuzz.min_time` 与 `fuzz.time`
之间用一个类似指数的函数计算出一个合适的回源时机。这里需要注意看源码`proxy/http/HttpTransact.cc`中的`HttpTransact::calculate_freshness_fuzz`函数，
会先根据概率判断是否命中，再根据是否设定了`fuzz.min_time`采取不同的计算公式。

## 读重试&超时

这个功能主要为了降低某一缓存的并发回源请求数，当某一缓存正在回源过程中，其他同样的请求将会根据配置
间隔`proxy.config.http.cache.open_read_retry_time`毫秒重试`proxy.config.http.cache.max_open_read_retries`次。



### [推荐阅读]

__[Reducing Origin Server Requests (Avoiding the Thundering Herd)](https://docs.trafficserver.apache.org/en/latest/admin-guide/files/records.config.en.html#proxy-config-http-cache-open-write-fail-action)__

__[proxy.config.http.background_fill_active_timeout](https://docs.trafficserver.apache.org/en/latest/admin-guide/files/records.config.en.html#proxy-config-http-background-fill-active-timeout)__

__[proxy.config.http.cache.fuzz.min_time](https://docs.trafficserver.apache.org/en/latest/admin-guide/files/records.config.en.html#proxy-config-http-cache-fuzz-min-time)__
