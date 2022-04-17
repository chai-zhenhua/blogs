### 先看问题
开发反馈有个接口请求一直是502，运维介入后查看nginx error log 展示信息如下:
```
upstream sent too big header while reading response header from upstream
```
意思是上游服务返回的响应携带的头信息太大了， 超过了配置的缓冲区，导致读取响应超时， nginx直接返回502

### 如何解决
分情况讨论:
- 新上线的Nginx集群，未调整过任何参数
  - 可以在主配置文件中添加proxy buffer 相关参数,如下
```
        proxy_buffer_size 128k；
        proxy_buffers 4 256k；
        proxy_busy_buffers_size 256k；
```
- 已在线运行许久的Nginx集群

此时如果你看到这个报错就直接去百度去谷歌大概率也能解决这个问题，因为网上对这个问题的解决方案无非就是加大缓存区

如果直接加大缓冲区，大概率也可以解决当前的问题，但也许不是最优的解决方案。
此时可以调用异常的接口，查看它的response header 看看是否存在异常, 是否是因为业务代码逻辑问题导致头信息异常增加，可以这样:

- 一般正常的请求如下， buffer通常不会溢出

```
# 请求
♪ localhost ~ $ curl -I www.baidu.com

# 响应头
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
Connection: keep-alive
Content-Length: 277
Content-Type: text/html
Date: Tue, 12 Apr 2022 05:20:07 GMT
Etag: "575e1f71-115"
Last-Modified: Mon, 13 Jun 2016 02:50:25 GMT
Pragma: no-cache
Server: bfe/1.0.8.18
```

- 异常的请求各有各的不同， 比如这个: 由于逻辑错误导致后端返回set-cookie巨多
```
♪ localhost ~ $ curl -I test.chaizh.com/api/v1/get-list

HTTP/1.1 200 OK
Date: Tue, 12 Apr 2022 05:24:57 GMT
Content-Type: application/json
Connection: keep-alive
Server: openresty
X-Powered-By: PHP/7.0.27
Cache-Control: no-cache
Set-Cookie: hahah=xxxxxxxxx; expirse=xxxxxx; Max-Age=xxxxxx; Path=/
...
... 此处省略1000行, 全部是set-cookie
...
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
```
所以，上述的这种问题不应该通过增大缓冲区来处理，而是需要找到对应的业务负责人来修复这种逻辑问题才是根本的解决方案。

### 知其然更要知其所以然
这里我建议了解一下Nginx作为反向代理的数据流向和执行阶段， 比如Nginx从上游接收响应到流程...