+++
title = "ATS零碎笔记(一) - Lua"
date = "2016-03-14"
categories = "blog"
+++


## tslua配置姿势

这里只介绍从源码编译的方法，当然也有其他的
[安装方式](https://blog.zymlinux.net/index.php/archives/768)，两种都蛮简单。

从源码编译只需要在执行`./configure`的时候加一个参数`--enable-experimental-plugins`，
这样在编译ATS的时候就会把实验性的插件也一起编译了，将来再用到其他插件也会很方便，无需重新编译。

接下来在有三种配置方式：

1. 只想针对某个URL启用某个Lua脚本，在`remap.config`中
    `map http://fuck.off.com/ http://fuck.you.com/ @plugin=/X/tslua.so @pparam=/X/fuck.lua`

2. 只想针对某个URL，但希望能传递参数，在`remap.config`中
    `map http://fuck.off.com/ http://fuck.you.com/ @plugin=/X/tslua.so @pparam=/X/fuck.lua @pparam=yo.man`

3. 全局开启某Lua脚本，在`plugin.config`中
    `tslua.so /X/test_global_hdr.lua`

需要注意的是，在前两种配置方式中，我们在lua脚本中加入`do_remap`函数作为执行入口，其中第二种方式传递的参数，
会在Lua脚本的`__init__`函数中被传递进去。第三种配置方式里，能够作为执行入口的函数很多，从函数名即可知道他们
的执行时机：

* ‘do_global_txn_start’
* ‘do_global_txn_close’
* ‘do_global_os_dns’
* ‘do_global_pre_remap’
* ‘do_global_post_remap’
* ‘do_global_read_request’
* ‘do_global_send_request’
* ‘do_global_read_response’
* ‘do_global_send_response’
* ‘do_global_cache_lookup_complete’
* ‘do_global_read_cache’
* ‘do_global_select_alt’

## 迷之`missing ictx`

在初次使用tslua plugin的时候，你有很大几率会遇到这样的一个报错
`missing ictx`，至少我花费了一个下午的时间才搞明白这个错误是怎么回事。

从错误信息能看出这里缺少了lua上下文`context`，从tslua源码中可以看到确实是这个原因，

那么这个上下文到底是什么呢？

最初我以为是没有在`plugin.config`中添加`libload.so`导致没有引入链接库，但在引入了之后，
lua脚本依旧无法正常运行。

反复的翻找了文档、Google、以及ATS的中文博客之后，无意间在文档中看到这样一段话：

> ts.now
>
> syntax: val = ts.now()
>
> context: global
>
> description: This function returns the time since the Epoch (00:00:00 UTC, January 1, 1970),
> measured in seconds. It includes milliseconds as the decimal part.

finally...

原来每一个tslua函数，都有自己的作用域和上下文限制，遵循这一限制，
在执行到这个函数的时候才会具备执行条件。



## 实现功能 - 区分设备分别缓存内容

配置好了tslua，接下来就要用它来开发功能了，首先我们来实现一个很常见但ATS没有很好支持的功能：

> 根据User-Agent分别缓存不同类型设备的网页

这个需求的主要目的，是针对不同的平台返回不同的静态资源，这样在不同平台上可以分发不同的资源，
提高页面响应速度，完善用户体验。

公司A有一个网站`fuck.you.com`，同时还有一个子页面 `fuck.you.com/shit`。

在默认的工作方式下，ATS针对这一子页面只会缓存一份，Cache Key是`fuck.you.com/shit`，
Cache Object是页面内容。

然而针对这一资源，公司面向不同的平台开发出了不同的页面，并且希望共用一个URL，同时利用ATS缓存起来。
这样原始的ATS配置就无法满足需求了，Cache Key一致的情况下，Cache Object只会被覆盖掉。

为了实现这个功能，我们有两种选择：

1. 使用C编写逻辑
2. 使用tslua插件编写Lua脚本实现逻辑

能用C写业务代码的都是大神...鄙人弱如菜鸡搞不定。于是选择使用Lua脚本实现这部分逻辑。

### 基础版

选择在`remap.config`中通过配置的方式引入Lua脚本，针对某一部分URL开启这一功能。

`map http://fuck.you.com/shit.* http://fuck.you.com/shit @plugin=/X/tslua.so @pparam=/X/ua.lua`

在`ua.lua`中我们写入`do_remap`函数并在其中实现逻辑。

``` lua

    local device = 'PC'

    function do_remap()
        local url = ts.client_request.get_url()
        local ua = ts.client_request.header['User-Agent']
        device = detect_device(ua)
        ts.http.set_cache_url(device .. '|' .. url)

        return 0
    end

    function detect_device(ua)
        if ua == nil or ua == '' then
            return 'PC'
        end

        if string.find(ua, 'Android') then
            return 'Android'
        elseif string.find(ua, 'iPhone') then
            return 'iPhone'
        elseif string.find(ua, 'iOS') then
            return 'iOS'
        elseif string.find(ua, 'iPad') then
            return 'iPad'
        else
            return 'PC'
        end

        -- Defult Device
        return 'PC'
    end

```

从代码中可以看到，除了判断设备类型的`detect_device`函数以外，最重要的函数就是tslua提供的
`ts.http.set_cache_url`了。通过这一函数，我们可以更改Cache Key，ATS将不再使用原始的URL作为Cache Key，
在这里我们将平台名作为前缀加在URL前边，合在一起作为新的Cache Key保存，这样一来ATS就会根据请求，将
同一URL补上平台名再检查缓存是否存在。

需要注意的是，我在这里区分了五种UA，回源机将会收到五次ATS回源请求(当然如果某一平台一直没有人访问，
也不会缓存它的版本，你瞧我就没写Windows Mobile Phone的UA判断对不对)

这样我们就实现了目标需求，在回源机看来，会收到五次代表性的请求，分别具备五类UA，只需要再判断一下平台类型，
就可以输出不同平台的内容了。

然而，问题的解决没有这么简单，这个基础版本的实现方式存在一些缺陷：

1. Purge的时候无法清除被这种方法处理过的URL，因为Cache Key已经不是一个合法的URL。
    当然也可以把平台信息拼在query里边规避这个问题。不过每次Purge都需要清五个URL，
    并且这一逻辑会同时存在于Lua和Purge脚本中，难以维护。

2. ATS不允许Cache Key被重复set，也就是说在tslua中使用`set_cache_url`之后，就无法再使用其他插件
    （比如cacheurl）了

为了解决以上两个问题，诞生了升级版

### 升级版

和基础版的实现方式不同，这里我们避免使用`set_cache_url`函数实现区分存储多份缓存，而是利用ATS本身就具备的
[Caching HTTP Alternates](https://docs.trafficserver.apache.org/en/latest/admin-guide/configuration/cache-basics.en.html#caching-http-alternates)
功能，实现需求。

这里要对上边的Lua脚本进行一些修改：

``` lua

    local device = 'PC'

    function do_remap()
        local ua = ts.client_request.header['User-Agent']
        ts.client_request.header['User-Agent'] = detect_device(ua)

        return 0
    end

    function detect_device(ua)
        if ua == nil or ua == '' then
            return 'PC'
        end

        if string.find(ua, 'Android') then
            return 'Android'
        elseif string.find(ua, 'iPhone') then
            return 'iPhone'
        elseif string.find(ua, 'iOS') then
            return 'iOS'
        elseif string.find(ua, 'iPad') then
            return 'iPad'
        else
            return 'PC'
        end

        -- Defult Device
        return 'PC'
    end

```

可以看到，在Lua脚本中仅仅是将原本的Client Requst的`User-Agent`替换为了平台名，
那么ATS是怎么知道根据`User-Agent`缓存多个HTTP内容呢？

答案在ATS的`records.config`中，内置了几个配置项，可以指定以某一Header字段为版本依据，
在这里我们指定`User-Agent`为依据。

同时又由于我们通过Lua脚本将格式相当不统一的`User-Agent`内容规范为了五种，因此在ATS中只会缓存五份。

在`records.config`中添加如下配置（默认是没有的）：

    # 开启vary检查
    CONFIG proxy.config.http.cache.enable_default_vary_headers INT 1
    # 对text类型的http请求开启多份存储，以User-agent为依据
    CONFIG proxy.config.http.cache.vary_default_text STRING User-agent
    # 同理还有
    # CONFIG proxy.config.http.cache.vary_default_images STRING User-agent
    # CONFIG proxy.config.http.cache.vary_default_other STRING User-agent
    # 这一条配置限制了最大的副本数量
    # CONFIG proxy.config.cache.limits.http.max_alts INT 5


其中每一项的配置内容，就是配置校验的HTTP Header字段，这里我们填写了`User-agent`。

如果要根据`Cookies`区分版本，记得在配置中开启这一项：

`proxy.config.http.cache.cache_responses_to_cookies`

如此一来，我们就可以根据不同的平台区分同一URL的不同版本了，并且可以统一使用这一个URL进行Purge操作。

在回源机看来，原本纷乱的`User-Agent`被规范成了少数几类，可以更方便的做处理。

### 终极版

不知道你有没有发现高级版中的一个问题，那就是当我们在`records.config`里开启了开关之后，这是一个全局开关，
ATS会对每个URL都进行Vary校验，那么如果某些URL没有被配置该Lua脚本，那么这些URL的命中率将会极大的降低，
毕竟`User-Agent`是一个很随意的值，各家公司都在里边发挥聪明才智，而ATS保存的副本越多，性能也就越差。
所以我们需要对上一方案进一步改进，解决这个问题。

那么解决方案其实很简单，就是配置全局Lua脚本，对每一次请求都处理一次`User-Agent`，这样就过滤了所有`User-Agent`，
也就能解决刚才那个问题了。

那么我们需要修改两个地方。

1. 修改`ua.lua`脚本，将`do_remap`函数换为`do_global_read_request`
2. 在`plugin.config`中添加这样的配置`tslua.so /X/ua.lua`

这样，我们就加入了一个tslua全局函数，当收到请求后就会被执行，从而对所有的`User-Agent`都过滤处理。


