### 第一种: 传统的LNMP方式
这种我称之为最古老的代理方式, 一般是流量入口放置一个Nginx， 然后proxy_pass到 Nginx + PHP的系统(Nginx解析http协议转发给fastcgi_pass交由php处理), 通常的配置如下: 每台PHP机器都会附加一个Nginx
```
# 这里只是php机器的配置, 一般流量会通过前置的负载均衡/Nginx服务转发到此
server {
    listen 80;
    server_name xxx.test.com;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php($|/) {
        # 设置项目根目录 chroot
        set $PROJECT_NAME "代码路径/public";
        include     fastcgi_params;
        fastcgi_pass  127.0.0.1:9000;   # 基于本地回环地址
        fastcgi_index  index.php;
        fastcgi_split_path_info ^(.+\.php)(.*)$;
        fastcgi_param  SCRIPT_FILENAME  $PROJECT_NAME$fastcgi_script_name;
    }
}
```

### 第二种: 远程代理方式
何为远程代理，我给的定义是基于网络的代理方式简单来说是把PHP主机的Nginx拆分出去， PHP主机上就只有PHP服务，通过远程调用请求PHP。
```
upstream test {
    192.168.10.100:9000  weight=100 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name xxx.test.com;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php($|/) {
        # 设置项目根目录
        set $PROJECT_NAME "代码路径/public";
        # 设置upstream_name
        set $fastcgi_name test;

        index index.php index.html index.htm;
        include     fastcgi_params;
        fastcgi_pass  $fastcgi_name;
        fastcgi_index  index.php;
        fastcgi_split_path_info ^(.+\.php)(.*)$;
        fastcgi_param  SCRIPT_FILENAME  $PROJECT_NAME$fastcgi_script_name;  # 指定项目根目录
}
```
```
tips:
    php-fpm的配置需要监听其他地址:
        listen = 127.0.0.1:9000             --> listen = :9000
        listen.allowed_clients = 127.0.0.1  --> ;listen.allowed_clients = 127.0.0.1
```
-   这种方式有什么优势呢？
    -   收敛了Nginx服务和配置， php机器无需再安装Nginx服务， 便于配置统一管理
    -   配置可以托管于gitlab，变更可溯源/可评审，拒绝登录机器改动， 借助ansible+jenkins批量更新配置


### 第三种: 更简单的代理方式
```
upstream test {
    192.168.10.100:9000  weight=100 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name xxx.test.com;

    location /  {
	    set $PROJECT_NAME "代码路径/public/index.php";
        fastcgi_pass  test;
        fastcgi_index  index.php;
        include     fastcgi_params;
        fastcgi_split_path_info ^(.+\.php)(.*)$;
        fastcgi_param  SCRIPT_FILENAME  $PROJECT_NAME;
    }
}
```
这种方式相较于第二种方式，少了一次内部重定向(try_files), 相同的是第二和第三种模式一定是针对前后端完全分离的前提下进行的。