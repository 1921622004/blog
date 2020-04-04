# nginx

## http
/etc/nginx/nginx.conf
一个http下面可以配置多个server
```
user  nginx;   设置nginx服务的系统使用用户  
worker_processes  1;  工作进程数,一般和CPU数量相同 

error_log  /var/log/nginx/error.log warn;   nginx的错误日志  
pid        /var/run/nginx.pid;   nginx服务启动时的pid

events {
    worker_connections  1024;每个进程允许的最大连接数 10000
}

http {
    include       /etc/nginx/mime.types;//文件后缀和类型类型的对应关系
    default_type  application/octet-stream;//默认content-type

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';  //日志记录格式

    access_log  /var/log/nginx/access.log  main;//默认访问日志

    sendfile        on;//启用sendfile
    #tcp_nopush     on;//懒发送

    keepalive_timeout  65;//超时时间是65秒

    #gzip  on; # 启用gzip压缩

    include /etc/nginx/conf.d/*.conf;//包含的子配置文件
}
```

## server

/etc/nginx/conf.d/default.conf

```
server{ #每个server对应一个网站
   listen    80; #监听的端口号
   server_name  localhost  192.168.20.159; #域名
   
   #charset koi8-r; #指定字符集
   #access-log  /var/log/nginx/host.access.log  main; #指定访问日志的位置和格式
   location / { #匹配所有的路径
       root  /usr/share/nginx/html  #静态文件根目录
       index  index.html index.htm;  #索引文档    
   }
   
   # error_page  404    /404.html   #错误页面，如果返回的状态404的话会重定向到/404.html
   error_page 500 502 503 504  /50x.html; #把服务端错误状态码重定向到50x.html上
   location = /50x.html {
       root    /usr/share/nginx/html; 
   }
   location ~ /\.ht {
       deny all; 如果路径是/.ht的话，deny all 禁止所有人访问
   }
   
}

```

### location匹配规则


符号 | 含义
---|---
= | 严格匹配。如果这个查询匹配，那么将停止搜索并立刻处理此请求
~ | 区分大小写匹配
!~ | 区分大小写不匹配
~* | 不区分大小写匹配
!~* | 不区分大小写不匹配
^~ | 如果把这个前缀用于一个常规字符串，如果路径匹配那么不测试正则表达式


### 静态服务

#### 1. sendfile
- 不经过用户内核发送文件 [具体解释](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy1/index.html)

#### 2. tcp_push
- sendfile开启后，会合并多个数据包，提高传输效率

#### 3. tcp_nodelay
- 在keepalive连接下，提高网络包的传输实时性

#### 4. gzip
- 压缩文件

#### 5. gzip_comp_level
- 压缩比

#### 6. gzip_http_version
- 压缩版本

#### 7. http_gzip-static_module
- 先查找磁盘上同名的**.gz**文件是否存在，节约CPU的压缩时间和性能损耗

```
 location ~ .*\.(jpg|png|gif)$ {
        gzip off;#关闭压缩
        root /data/www/images;
    }

    location ~ .*\.(html|js|css)$ {
        gzip on; #启用压缩
        gzip_min_length 1k;    #只压缩超过1K的文件
        gzip_http_version 1.1; #启用gzip压缩所需的HTTP最低版本
        gzip_comp_level 9;     #压缩级别，压缩比率越高，文件被压缩的体积越小
        gzip_types  text/css application/javascript;#进行压缩的文件类型
        root /data/www/html;
    }

    location ~ ^/download {
        gzip_static on; #启用压缩
        tcp_nopush on;  # 不要着急发，攒一波再发
        root /data/www; # 注意此处目录是`/data/www`而不是`/data/www/download`
    }
```

#### 缓存
```
location ~ .*\.(jpg|png|gif) {
    expires 24h;
}
```

### 跨域
```
location ~ .*\.json$ {
    add_header Access-Control-Allow-Origin http://localhost:3000;
    add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE,OPTIONS;
    root /data/json;
}
```

### 防盗链
```
location ~ .*\.(jpg|png|gif)$ {
        expires 1h;
        gzip off;
        gzip_http_version 1.1;
        gzip_comp_level 3;
        gzip_types image/jpeg image/png image/gif;
        # none没有refer blocked非正式HTTP请求 特定IP
+       valid_referers none blocked 47.104.184.134;
+       if ($invalid_referer) { # 验证通过为0，不通过为1
+           return 403;
+       }
        root /data/images;
    }
```

### 反向代理

```
location ~^/api {
    proxy_pass  http://localhost:3000;
    proxy_direct default; #重定向
    
    proxy_set_header Host $http_host;  #向后传递头信息
    proxy_set_header X-Real-IP $remote_addr; # 把真实IP传给应用服务器
    
    proxy_connect_timeout  30;  #默认超时时间
    proxy_send_timeout  30;     # 发送超时
    proxy_read_timeout  60;     # 读取超时
    
    proxy_buffering  on;        # proxy_buffering 开启的情况下，nginx会尽可能的读取所有的upstream端传输的数据到buffer，知道proxy_buffers 设置的buffer被写满或者数据读取完
    proxy_buffers 4 128k;  # proxy_buffers由缓冲区数量和缓冲区大小组成。总的大小为number*size
    proxy_busy_buffers_size  256k;  # proxy_busy_buffers_size不是独立的空间，他是proxy_buffers和proxy_buffer_size的一部分。nginx会在没有完全读完后段响应的时候就开始像客户端传送数据，所以它会画出一部分缓冲区来专门像客户端传送数据
    proxy_buffer_size   32k; # 用来存储upsream端response的header
    proxy_max_temp_file_size  256k; #response的内容很大的话，nginx会接受并把他们写入到temp_file中，大小由proxy_max_temp_file_size控制。如果busy的buffer传输完了会从temp_file里面接着读数据，知道传输完毕
}
```

### 负载均衡

1. 后段服务器调试状态

状态 | 描述
---|---
down | 当前的服务器不参与
backup | 当其他节点都无法使用时的备份服务器
max_fails | 允许请求失败的次数，到达最大次数就会休眠
fail_timeout | 到达最大失败次数后，服务暂停的时间，默认10秒
max_conns | 限制每个server最大的接受的连接数，性能高的服务器可以连接多一些

```
upstream awesome {
  server localhost:3000 down;
  server localhost:4000 backup;
  server localhost:5000 max_fails=1 fail_timeout=10s;
}
```

2. 分配方式

类型 | 种类
---|---
轮询 | 每个请求按时间顺序逐一分配到不同的后段服务器，如果后端服务器down掉，能自动剔除
weight | 指定轮询几率，weight和访问比率成正比，用于后段服务器性能不均的情况
ip_hash | 每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题
least_conn | 哪个机器上连接数少就分发给谁
url_hash | 按访问的URL地址来分配，每个url都定向到同一个后端服务器上
fair | 按后端服务器的响应时间来分配请求，响应时间短的优先分配
hash | 自定义hashkey

```
upstream awesome{
    fair;
    server 127.0.0.1:3000;
}
```

### 常用模块

- 内容替换

**ngx_http_sub_module**

```
location {
    sub_filter 'world' 'hello';
    sub_filter_once off; //只替换一次
}
```

- 限制IP访问服务

**ngx_http_access_module**

```
location / {
    deny  192.168.1.1;
    allow 192.168.1.0/24;
    allow 10.1.1.0/16;
    allow 2001:0db8::/32;
    deny  all;
}
```

- 

**ngx_http_limit_req_module**

- gzip压缩

**ngx_http_gzip_module**

- 头部信息

**ngx_http_headers_module**

- 日志

**ngx_http_log_module**