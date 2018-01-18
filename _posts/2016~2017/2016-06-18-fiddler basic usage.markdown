---
layout: post
title: 使用Fiddler调试http/https协议
categories: Permeation
description: 使用Fiddler调试http/https协议
keywords: Fiddler
---

# 使用Fiddler调试http/https协议

&emsp;&emsp;使用Fiddler调试http/https协议对于渗透测试，Fiddler的强大在于：
* 可以针对http/https协议；可以嗅探环回地址，这样方便测试渗透工具和进行攻防演练

* 可以调试协议；何谓调试？能控制某一项复杂操作中的单个细节，并在这一点控制:暂停/查看/修改/执行即为调试。这样我们就可以任意修改浏览器发给服务器的数据，和服务器发给浏览器的数据了

* 高亮显示文本
```Txt
    ?abc    高亮显示所有包含searchtext的URL
    boldabc 加粗显示包含abc的url
    bold    清除加粗
```

* 会话筛选
```Txt
    按大小[>][<][=]size
        >5k只显示大小大于5k的response包，</=同理
    按状态=status
        =301，只显示301重定向响应包
    按方式=method
        =POST，只显示POST请求包
    按主机@host
        @msn.com，只显示主机包含msn.com的请求
    按Content-Type内容
        selectimag，筛选Content-Type头包含image的会话             image/css/htm/..
    selectui-comments slow，筛选存在于Header和SessionFlag的会话
    select@Request.Accept html，选择所有Accept包含html的请求
    select@Response.Set-Cookie domain，选择所有Set-Cookie包含domain的响应
    allbutstr，保留Content-Type头包含str的会话，xml/java/..
```

* 会话断点
```Txt
    bpafterabc  在收到RequestURI包含abc的响应包时断下
        bpafter /favicon.ico
    bpafter     清除bpafter断点
    bps404      在收到状态为404的响应包时断下
    bps         清除bps断点
    bpv/bpm ?   在发送指定类型(GETPOST ...)的请求包时断下
    bpv         清除bpv断点
    bpu  abc    在发送URI包含abc的请求时断下
        bpu /myservice.asmx
    bpu         清除bpu断点
```

* 替换
```Txt
    urlreplaceseekstr replacestr   替换任何URL中的seekstr为replacestr
    urlreplace        清除urlreplace
```

* 查询域名
```Txt
    !dnswww.example.com 
    !nslookupwww.example.com
    !listen port [certhostname]
    !listen8889
    !listen4443 localhost
    !listen444 secure.example.com   https认证服务器SubjectCN=secure.example.com
```

* 其他功能
```
cls         清除会话列表
dump        保存所有回话到c:\下zip文件
g/go        从断点恢复执行
help        显示此帮助
show/hide   显示/隐藏tray
start /stop 注册/解除作为系统代理
quit        退出
