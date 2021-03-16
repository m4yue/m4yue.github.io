---
layout: default
title: New HTTP Status Code 103
---

## HTTP CODE 103

IETF最新批准了新的HTTP状态码[HTTP 103](https://datatracker.ietf.org/doc/draft-ietf-httpbis-early-hints/?include_text=1), 内容为`Early Hints`, 可以让服务器预先发送部分头部, 是一种预加载策略. 草案给出的例子:
```HTTP
HTTP/1.1 103 Early Hints
Link: </main.css>; rel=preload; as=style

HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style
Link: </script.js>; rel=preload; as=script

HTTP/1.1 200 OK
Date: Fri, 26 May 2017 10:02:11 GMT
Content-Length: 1234
Content-Type: text/html; charset=utf-8
Link: </main.css>; rel=preload; as=style
Link: </newstyle.css>; rel=preload; as=style
Link: </script.js>; rel=preload; as=script
```
通过`103`响应, 使用`Link`头部字段通知客户端资源位置.这样的话, 客户端可以在取得正式/标准HTTP头部之前就开始加载部分资源.

但是在HTTP 1.1下, 这种向未经确认的客户端发送多个头部可能存在安全隐患.
```rfc

3.  Security Considerations

   Some clients might have issues handling 103 (Early Hints), since
   informational responses are rarely used in reply to requests not
   including an Expect header field ([RFC7231], Section 5.1.1).

   In particular, an HTTP/1.1 client that mishandles an informational
   response as a final response is likely to consider all responses to
   the succeeding requests sent over the same connection to be part of
   the final response.  Such behavior might constitute a cross-origin
   information disclosure vulnerability in case the client multiplexes
   requests to different origins onto a single persistent connection.

   Therefore, a server might refrain from sending Early Hints over
   HTTP/1.1 unless the client is known to handle informational responses
   correctly.

   HTTP/2 clients are less likely to suffer from incorrect framing since
   handling of the response header fields does not affect how the end of
   the response body is determined.
```

这种机制看起来有点像HTTP 2的`Push`功能, 但是有一些区别:
- `Push`仅能推送服务器被授权的资源
- 根据RFC, `Push`推送的资源会忽略客户端的缓存机制, 导致带宽的浪费, `103`可以利用客户端的缓存.(不过,HTTP 2也有缓存的相关配置`"http2-casper" (cache-aware server-push).`)

----
### 一些评论

```text
As I understand it, the status code would improve performance in situations like the following:
1. Client requests X.html (which requires Y.css and Z.js) from Proxy.
2. Proxy forwards client's X.html request to Server.
3. [Server takes a long time to respond]
4. Server sends a 200 response to Proxy with X.html.
5. Proxy, per some configuration, adds Preload headers to this response indicating that Y.css and Z.js should be immediately loaded once this response is received, and sends it to Client.
6. Client requests Y.css and Z.js.
With the 103 status code, this would look like:
1. Client requests X.html (which requires Y.css and Z.js) from Proxy.
2. Proxy forwards client's X.html request to server and, per some configuration, immediately sends a 103 response with Preload headers for Y.css and Z.js to Client.
3. [Server takes a long time to respond]
4. In parallel, Client requests Y.css and Z.js.
5. Server sends a 200 response to Proxy with X.html.
6. Proxy sends this response to Client.
Note that since Y.css and Z.js are requested sooner, the final load time should be less.
```
