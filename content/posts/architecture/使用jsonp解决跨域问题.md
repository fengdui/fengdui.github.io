---
title: "使用jsonp解决跨域问题"
date: "2018-10-25"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

JSONP利用了script标签没有跨域限制的特性
为什么要使用JSONP 因为请求的接口路径不是我们自己的后端 是第三方的接口 我们不能直接请求 所以只能通过JSONP来解决跨域问题
而第三方接口只支持jsonp格式的回调函数 所以我们只能通过jsonp来请求数据

// 服务器需要返回这样的响应，而不是纯 JSON：
callback({data: "value"})
查看服务器响应头是否为 Content-Type: application/javascript 响应内容是否是 函数名(JSON数据) 格式 
你代码中的 jsonp: "callback" 表示使用 callback 参数名 服务器也必须读取 callback 参数

```
$.ajax({
    type: "GET",
    url: result.data.pingUrl,
    data: {indexCode: indexCode},
    dataType: "jsonp",
    jsonp: "callback",
    timeout: 3e3,
    success: function (res) {
        // 处理返回的数据
    },
    error: function (e) {
        // 错误处理
    }
})

```