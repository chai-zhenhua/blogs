# Nginx -t 检测语法时很慢怎么办？

```
[root@prod-front-node-2 ~]# time nginx -t
nginx: the configuration file /usr/local/openresty/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/openresty/nginx/conf/nginx.conf test is successful

real	0m5.219s
user	0m0.023s
sys	0m0.009s
[root@prod-front-node-2 ~]#
```

-   语法检测用了5s的时间，是为什么？
    -   strace 追踪下这个过程它都干了什么：
        -   看到有很多请求指向了DNS，于是查看配置文件发现有很多proxy_pass是指向的域名，如下

```
connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
recvfrom(5, "\306\26\201\200\0\1\0\1\0\0\0\0\7niffler\5wptqc\3com\0\0"..., 2048, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, [28->16]) = 51
recvfrom(5, "X\33\201\200\0\1\0\0\0\0\0\0\7niffler\5wptqc\3com\0\0"..., 65536, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, [28->16]) = 35
connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
recvfrom(5, "\30\357\201\200\0\1\0\2\0\0\0\0\vshop-center\nweipait"..., 2048, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, [28->16]) = 86
recvfrom(5, "\252\363\201\200\0\1\0\1\0\1\0\0\vshop-center\nweipait"..., 65536, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, [28->16]) = 143
connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
recvfrom(5, "\0U\201\200\0\1\0\1\0\0\0\0\10zhongjin\7offical\10yo"..., 2048, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, [28->16]) = 64
```

```
    location /shop-center/api/ {
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_pass https://shop-center.weipaitang.com/;
    }

    location ^~ /api/wechat/ {
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_pass https://api.weipaitang.com/;
    }
```

恍然大悟，原来如此。 那该怎么解决呢？

[resolver指令可以一试](http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver)

```
Syntax:	resolver address ... [valid=time] [ipv6=on|off] [status_zone=zone];
Default:	—
Context:	http, server, location

Syntax:	resolver_timeout time;
Default:
resolver_timeout 30s;
Context:	http, server, location
```

- tips:
  - 禁用ipv6如果你不想网站全崩的话