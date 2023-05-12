---
title: nginx架构及源码阅读
date: 2021-09-30 15:39:42
tags: nginx
categories: Web服务
mathjax:
    true
description: Nginx源码解读，正在阅读中。
---

# nginx安装

从官方网站下载源码包（http://nginx.org/en/download.html)。

解压

```
tar -zxvf nginx-1.*.tar.gz
```

编译安装

```
cd nginx
./config
make
make install
```

可以使用

```
./config --help
```

查看config包含的参数。

这里主要介绍几个参数：

## --prefix=PATH

nginx安装部署后的根目录。默认在/usr/local/nginx目录。该目录影响其他目录的相对目录。

## --sbin-path=PATH

该目录是可执行文件放置的目录。默认为\<prefix>/sbin/nginx。

## --conf-path=PATH

配置文件目录。默认为\<prefix>/conf/nginx.conf。执行时读取的默认配置。

## --error-log-path=PATH

error日志的放置路径。默认为\<prefix>/log/error.log。可以打印不同等级的log日志，在nginx.conf中控制。

## --pid-path=PATH

pid文件存放路径。默认为\<prefix>/log/nginx.pid。该文件以ASCII码存放着nginx master的进程ID。

# nginx的命令行控制

## 直接执行Nginx二进制文件

```
/usr/local/nginx/sbin/nginx
```

读取默认配置文件

## 指定配置文件

```
/usr/local/nginx/sbin/nginx -c 配置文件路径
```

## 另行指定全局配置项

-g

```
/usr/local/nginx/sbin/nginx -g "pid /tem/test.pid;"
```

这时，pid文件将写到`/tem/test.pid`。-g的配置不能与nginx.conf不一致。

对于以-g启动的nginx来说，要执行其他操作，也应该包含-g参数，如

```
/usr/local/nginx/sbin/nginx -g "pid /tem/test.pid;" -s stop
```

## 测试配置信息是否错误

```
/usr/local/nginx/sbin/nginx -t
```

## 显示版本

```
/usr/local/nginx/sbin/nginx -v
```

## 显示编译阶段参数

```
/usr/local/nginx/sbin/nginx -V
```

## 快速停止服务

-s

```
/usr/local/nginx/sbin/nginx -s stop
```

-s参数是告诉Nginx向正在运行的服务发送信号。直接使用kill也是可以的。

## 优雅的停止服务

关闭监听端口，处理完当前所有请求。

```
/usr/local/nginx/sbin/nginx -s quit
```

等价于

```
kill -s SIGQIT masterpid
```

停止某个worker进程

```
kill -s SIGQIT workerpid
```

# Nginx的配置

部署Nginx是，都是使用一个master进程来管理多个worker进程，一般worker进程数量与服务器中的CPU核心数一致。

## 配置demo

```nginx
user nobody;

worker_process 8;
error_log /var/log/nginx/error.log error;

#pid logs/nginx.pid

events {
  use epoll;
  worker_connections 500000;
}

http {
  include mime.types;
  default_type application/octet-stream;
  
  log_format main '$remote_addr [$time_local] "$request" '
    '$status $bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
  access_log logs/access.log main buffer=32k;
   
  ....
}
```

## 块配置

块配置项由一个块配置名和一对大括号组成。

```nginx
events {
  ...
}

http {
  upstream backend {
    server 127.0.0.1:8080;
  }
  
  gzip on;
  
  server {
    ...;
    location /webstatic {
      gzip off;
    }
}
```

上面的events、http、server、location、upstream都是块配置项，块配置项可以嵌套。

## 配置项语法

基本的配置项语法为

```
配置项名 配置项值1 配置项值2 ... ;
```

配置项值可以是数字，字符串，正则表达式，这取决于该配置项名实际需求。（这个有点像是函数名和参数的关系）。

## 配置项单位

指定大小时，可以使用的单位包括：

1. K或者k字节（KiB）
2. M或者m字节（MiB）

指定时间时，可以使用：

1. ms（毫秒）
2. s（秒）
3. m（分钟）
4. h（小时）
5. d（天）
6. w（周）
7. M（月，30天）
8. y（年，365天）

## 配置中使用变量

部分模块可以使用变量，如

```
log_format main '$remote_addr [$time_local] "$request" '
    '$status $bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
```

变量前加上`$`。

## nginx提供的基础服务

### 用语调试和定位问题的配置

#### 是否以守护进程来执行

```
daemon on|off

#default
daemon on
```

#### 是否以master/worker方式进行工作

```
master_process on|off

#deault
master_process on
```

#### error日志设置

```
error_log /path/file level

#default
error_log logs/error.log error
```

当指定`/path/file`为`/dev/null`时即关闭日志。

`level`为日志级别。取值为`debug` `info` `notice` `warn` `error` `crit` `alert` `emerg`。从左到右等级依次升高。指定的等级时，大于等于该等级的日志都会输出。

#### 设置特殊的调试点

```
debug_points [stop|abort]
```

nginx在关键逻辑中设置了调试点。如果设置为stop，则nginx代码运行到这些调试点就会发出SIGSTOP信号以用于调试。如果设置为abort，则会产生一个coredump文件，可以使用gdb来查看。

#### 对指定客户端输出debug级别日志

该配置属于事件类型配置，要放在events中才有效，起值可以是IP地址或者CIDR地址，如

```nginx
events {
  debug_connection 10.224.66.14;
  debug_connection 10.224.57.0/24;
}
```

此时，只有配置的IP才会打印debug级别日志，其他请求仍然沿用error_log中配置的日志级别。

#### 限制coredump核心转储文件大小

```
worker_rlimit_core size;
```

#### 指定coredump文件生成目录

```
working_directory path;
```

### 正常运行配置

#### 定义环境变量

```
env VAR|VAR=VALUE
```

该配置可以让用户直接设置操作系统上的环境变量。仅是变更运行时使用的环境变量。

#### 嵌入其他配置

```
include /path/file
```

将其他配置文件嵌入到当前配置中，参数可以是绝对路径，也可以是相对路径（相对于当前配置文件的目录）。路径可以是字符串和正则表达式。

#### pid文件路径

```
pid path/file

#default
pid logs/nginx.pid
```

master进程的pid

#### nginx进程运行的用户及用户组

```
user username [groupname]

#default
user nobody nobody
```

#### worker进程最大句柄描述符个数

```
worker_rlimit_nofile limit;
```

#### 限制信号队列

```
worker_rlimit_sigpending limit;
```

设置每个用户发往nginx的信号队列大小，当超过该值时，该用户发送的信号将被丢弃。

### 性能优化配置

#### nginx的worker进程数

```
worker_processes number;

#default
worker_processes 1;
```

#### 绑定nginx的worker进程到指定的CPU内核

```
worker_cpu_affinity cpumask [cpumask, ...]
```

避免多个worker进程抢占同一个cpu，设置每个进程绑定的cpu内核来加速。如

```
worker_processes  8;
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
```

### 事件类配置(都在event块中)

#### 是否打开负载均衡锁

```
accept_mutex [on|off]

#default
accept_mutex on;
```

Accept_mutex是nginx的负载均衡锁。该锁来让各个worker进程直接负载均衡。关闭该锁可以加快TCP连接速度，但会导致负载不均衡。

#### lock文件路径

```
lock_file path/file;

#default
lock_file logs/nginx.lock;
```

存储负载均衡锁所需文件路径。

#### 使用accept锁后未建立连接后等待时间

```
accept_mutex_delay Nms;

#default
accept_mutex_delay 500ms;
```

使用accept锁后，每个时刻，只能有一个worker进程能够获取accept锁。该accept不是阻塞锁，没有获取就会立即返回。如果一个worker进程尝试获取accept锁没有获得时，至少要等到accept_mutex_delay时间才能再次获取。

#### 批量建立新连接

```
multi_accept [on|off];

#default
multi_accept off;
```

当事件模型通知有新连接时，尽可能地对本次调度中客户端发起的所有TCP请求都建立连接。

#### 选择事件模型

```
use [kqueue|rtsig|epoll|/dev/poll|select|poll|eventport]

#default
use 最适合的（nginx自己寻找）。
```

#### 每个master的最大连接数

```
worker_connections number;
```



## 虚拟主机与请求的分发

由于IP地址限制，存在多个主机域名对应同一IP地址的情况，这时在nginx.conf中按照server_name(对应用户请求中的主机域名）并通过server块来定义虚拟机，每个server块就是一个虚拟机，它只处理与之相对应的主机域名请求。这样，一台服务器上的nginx就能够以不同方式处理访问不同主机域名的HTTP的请求了。

### 监听端口

```
listen address:port [default(deprecated in 0.8.21) | default_server | [backlog=num | rcvbuf=size | sndbuf=size | accept_filter=filter | deferred | bind | ipv6only=[on|off] ssl]];

#default 
listion 80;

#conf block
server
```

listen决定nginx服务监听端口。listen后可以加IP地址、端口号或者主机名。如

```
listen 127.0.0.1:8002;
listen 127.0.0.1; //default port 80
listen *:8000;
```

地址后可以加其他参数。

| 参数          | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| default       | 将所在server块作为整个web服务的默认server块。当一个请求无法匹配配置文件中的所有主机域名时，就会选择默认虚拟主机。如果所有server都未指定，则默认选第一个。 |
| backlog=num   | TCP中backlog大小。（默认-1，无限制）在TCP建立三次连接时，进程还未监听句柄，此时backlog队列将会放置这些连接。如果backlog已满，还有客户端企图建立连接，则会失败。 |
| rcvbuf=size   | 设置监听句柄的SO_RCVBUF参数。                                |
| sndbuf=size   | 设置监听句柄的SO_SNDBUF参数。                                |
| Accept_filter | 设置accept过滤，只对FreeBSD系统有用。                        |
| deferred      | 设置该参数时，若用户建立了TCP连接（三次握手），内核也不会对该连接调度worker进程来处理，只会在用户真正发送请求时才会分配worker进程。使用于大并发情况下。 |
| bind          | 绑定当前端口/地址对，如127.0.0.1:8000。                      |
| ssl           | 在当前监听的端口上建立的连接必须基于SSL。                    |

### 主机名称

```
server_name name [...];

#default
server_name "";

#conf block
server
```

Server_name后面可以有多个主机名称，如

```
server_name www.testweb.com download.testweb.com
```

在开始处理一个HTTP请求时，nginx会取出header头中的Host，与每个server中的server_name进行比对，以决定由哪个server块来处理该请求。server_name与Host匹配优先级为：

1. 最先选择完全匹配的
2. 其次选择通配符在前面的，如*.testweb.com
3. 再次选择通配符在后面的，如www.testweb.*
4. 最后选择使用正则表达式的，如~^\\.testweb.com$

Server_name后面是空字符串时，表示匹配没有这个host这个http头的请求。

### server_name_hash_bucket_size

```
server_name_hash_bucket_size size;

#default
server_name_hash_bucket_size 32|64|128;

#conf block
http server location
```

为快速寻找对应server_name能力，nginx使用散列来存储server name。server_name_hash_bucket_size设置每个散列桶占用内存大小。

### 重定向主机名称处理

```
server_name_in_redirect on|off;

#default
server_name_in_redirect on|off;

# conf block
http server location
```

该配置配合server_name使用。在打开on时，表示重定向请求时会使用server_name的配置的第一个主机名代替原先请求的Host，使用off时，表示重定向时使用请求本身的Host。

### location

```
location [=|~|~*|^~|@] /uri/ {...}

#conf block
server
```

location尝试根据用户请求中的URI来匹配上面的/uri表达式，如果可以匹配，就选择location块中的请求来处理。

| 参数 | 含义                                                         | 举例            |
| ---- | ------------------------------------------------------------ | --------------- |
| `=`  | 把URI作为字符串，与参数的uri做完全匹配                       | Location = / {} |
| `~`  | 匹配URI时是大小写敏感的                                      |                 |
| `~*` | 忽略大小写                                                   |                 |
| `^~` | 匹配前半部分uri即可。                                        |                 |
| `@`  | 表示仅用于nginx服务内部请求之间的重定向，带有@的请求不直接处理用户请求。 |                 |

最常用的是uri为正则表达式。

## 文件路径的定义

### 以root方式设置资源路径

```
root path

# default
root html

#conf block
http, server, location, if
```

例如：

```nginx
location /download/ {
  root /opt/web/html/;
}
```

对于上面的配置，如果有一个请求的URI是/download/index/test.html，那么web服务器返回服务器上/opt/web/html/download/index/test.html文件的内容。

### alias方式设置资源路径

语法

```
alias path;

#conf block
location
```

alias也是用来设置文件资源路径，与root不同点主要在于如何解读跟在location后面的uri参数，这使得alias与root以不同的方式将用户请求映射到真正的磁盘文件上。例如，如果一个请求的URI是/conf/nginx.conf，而用户实际希望访问的文件在/usr/location/nginx/conf/nginx.conf。中，如果想要使用alias来进行设置的话，可以采用如下形式：

```nginx
location /conf {
  alias /usr/local/nginx/conf/;
}
```

如果使用root，则语句如下：

```nginx
location /conf {
  root /usr/location/nginx/;
}
```

使用alias时，在URI向实际路径的映射过程中，已经将location后配置的/conf部分去除了，因此/conf/nginx.conf请求将根据alias path映射为path/nginx.conf。root则会根据完整的URI请求来映射，因此root会根据root path映射为path/conf/nginx.conf。

alias后面还可以添加正则表达式，例如：

```nginx
location ~^/test/(\w+)\.(w+)$ {
  alias /usr/location/nginx/$2/$1.$2;
}
```

### 访问首页

语法：

```
index file

#default
index index.html

#conf block
http, server, location
```

有时，访问站点的URI为/，这一般是网页站点，这与root和alias都不同，使用nginx_http_index_module模块提供的index配置实现。index后面可以更多个文件参数，nginx会按顺序访问这些文件，例如：

```nginx
location / {
  root path;
  index index.html, /html/index.html;
}
```



### 根据http返回码重定向页面

语法：

```
error_page code[code...][=|=answer-code] uri | @named_location

#conf block
http,server,location,if
```

对于某个请求返回错误码时，如果匹配上了error_page中设置的code，则重定向到新的URI中，例如

```nginx
error_page 404  /404.html
error_page 502 504 504 /50x.html
error_page 403 http://example.com/forbidden.html
error_page 404 = @fetch
```

注意，虽然重定向了URI，但返回的HTTP错误码还是原来的错误码。可以通过'='来更改返回的错误码，例如

```
error_page 404 = 200  /empty.html;
error_page 404 = 403  /forbidden.gif;
```

也可以不指定确切的错误码，而是由重定向后时间处理的真实结果来决定，这时只要把=后面的错误码去掉即可，例如

```
error_page 404  /empty.html;
```

如果不想修改URI，只是想让请求重定向到另一个location中进行处理，那么可以设置为：

```nginx
location / {
  error_page 404 @fallback;
}

location @fallback {
  proxy_pass http://backend;
}
```

此时，出现404错误时，请求转发到反向代理http://backend服务上。

### 是否允许递归使用error_page

语法：

```
recursive_error_pages [on|off];

#default
recursive_error_pages off;

#conf block
http,server,location
```

### try_file

语法：

```
try_file path1[path2] uri;

#conf bloak
server,location
```

try_file后跟若果路径，如path1，path2...，最后必须要有uri参数，意义为：尝试顺序访问每一个path，如果可以有效获取，就直接向用户返回对于的文件请求，并结束，否则接着向下访问，如果所有path都找不到，则重定向到最后的uri上。例如

```nginx
try_file /sys/main.html $uri $uri/index/html $uri.html @other;
location @other {
  proxy_pass http://backend;
}
```

### 示例

一个网页，要访问服务器上3个页面，为路径$path下的index.html,master.html和slave.html，其中index.html是首页，其请求的uri为/。后面两个文件是index.html返回后，请求的两个页面，请求的uri分别为/showmaster/和/showslave/。此时，nginx配置可设置为：

```
error_log /home/work/error.log debug;
events {
    worker_connections 1024;
}
http {
    log_format main '"$request"';
    access_log /home/work/debug.log main;

    server {
        listen 8894;
        location ~* ^/show([a-z]+)\/$ {
           root path/$1.html;
        }
       location / {
           root path;
           index index.html;
       }
    }
}
```



## 内存及磁盘资源的分配

### HTTP包体只存储在磁盘文件中

语法

```
client_body_in_file_only on | clean | off

#default
client_body_in_file_only off

#conf block
http,server,location
```

当设置为非off时，用户请求的HTTP包体一律存储到磁盘文件中，即使只有0字节也会存储为文件。当配置结束时，如果设置为on，则该文件不会被删除（常用于调试），如果配置为clean，则会在请求结束，清除文件。

### HTTP包体尽量写到一个内存buffer中

语法：

```
client_body_in_single_buffer on | off

#default
client_body_in_single_buffer off

#conf block
http,server,location
```

配置on时，HTTP包体一律存储到内存buffer。如果大小超过了client_body_buffer_size值，包体依然会写入磁盘。

### 存储HTTP请求头部的内存大小

语法：

```
client_header_buffer_size size;

#default
client_header_buffer_size 1k

#conf block
http,server
```

该配置定义了正常情况下Nginx接收用户请求中HTTP header部分（包括HTTP行和HTTP头部）分配的内存buffer大小。当HTTP header部分超过该值时，定义的buffer大小将失效。

### 存储超大HTTP头部的内存buffer大小

语法：

```
lager_client_header_buffer number size;

#defult
lager_client_header_buffer 4 8k;

#conf block
http,server
```

订阅了Nginx接收一个超大HTTP头部请求的buffer个数和每个buffer大小。如果请求行（如GET /index HTTP/1.1）大小超过上面单个大小限制时，返回Request URI too large（414）。请求中有很多header，每个header的大小也不能超过单个buffer大小，否则会返回Bad request（400）。请求行和请求头总数也不能超过buffer个数*buffer大小。

### 存储HTTP包体的内存buffer大小

语法：

```
client_body_buffer_size size;

#default
client_body_buffer_size 8k/16k;

#conf block
http,server,location
```

定义了Nginx接收http请求包体的内存缓冲区大小。即HTTP包体会先接收到指定的缓存中，再决定是否写入磁盘。

### HTTP包体临时存放目录

语法

```
client_body_temp_path dir-path [level1 [level2 [level3]]]

#default
client_body_temp_path client_body_temp

#conf block
http,server,location
```

配置包体存放的临时目录。包体大小超过client_body_buffer_size指定大小时，会以递增的整数命名并存放到client_body_temp_path指定目录。后面的level1，level2，level3是防止一个目录文件过多，影响性能，使用level参数，可以按文件临时名再多加3层目录。

### connection_pool_size

语法

```
connection_pool_size size;

#default
connection_pool_size 256;

#conf block
http,server
```

Nginx对每一个建立的成功的TCP连接会预先分配一个内存池，这里size指定内存池初始大小，用于减少对小块内存的分配次数。过大的size导致内存资源浪费，过小的size，导致重复分配次数增加，影响性能，谨慎设置。

### request_pool_size

语法

```
request_pool_size size;

#default
request_pool_size 4k

#conf block
http,server
```

Nginx开始处理HTTP请求时，会为每个请求都分配一个内存池。这里size指定内存池初始大小，用于减少对小块内存的分配次数。TCP连接关闭时会销毁connection_pool_size指定的内存池，HTTP请求结束会销毁request_pool_size指定的内存池，但其创建和销毁时间并不一致，因为一个TCP连接可能被复用于多个HTTP请求。

## 网络连接设置

### 读取HTTP头部超时时间

语法

```
client_header_timeout time(s)

#default
client_header_timeout 60;

#conf block
http，server，location
```

客户端与服务器建立TCP连接后将开始接收HTTP头部，在这个过程中，如果在一个时间间隔内没有读取到客户端发来的字节，则认为超时，向客户端返回408（Request timed out）响应。

### 读取HTTP包体超时时间

语法：

```
client_body_timeout time(s);

#default
client_body_timeout 60;

#conf block
http,server,location
```

与client_header_timeout类似，不过使用在读取http包体。

### 发送响应超时时间

语法：

```
send_timeout time;

#default
send_timeout 60;

#conf block
http,server,location
```

Nginx向客户端发送数据包，客户端一直没有接收数据包，超过超时响应时间后，Nginx关闭连接。

还有很多网络连接配置，看书。

## client_body_in_file_only

此指令禁用NGINX缓冲区并将请求体存储在临时文件中。 文件包含纯文本数据。 该指令在NGINX配置的http，server和location区块使用。 

```
client_body_in_file_only off|on|clean

default
off
```

不同值含义如下：

1. off 该值将禁用文件写入
2. clean:请求body将被写入文件。 该文件将在处理请求后删除。
3. on:请求正文将被写入文件。 处理请求后，将不会删除该文件。

## client_body_in_single_buffer

该指令设置NGINX将完整的请求主体存储在单个缓冲区中。 默认情况下，指令值为off。如果启用，它将优化读取$request_body变量时涉及的I/O操作。

## 长连接

http为了避免每次都需要进行连接的三次握手和四次挥手，支持连接的复用，报错连接为长连接。nginx也支持该功能。对应的配置包括。

### keepalive_requests

长连接最多接收的请求数量。语法为：

```c
Syntax:	keepalive_requests number;
Default:	
keepalive_requests 1000;
Context:	http, server, location
```



### keepalive_timeout

对于保持长连接的请求来说，如果指定事件未收到请求，则关闭连接。配置如下：

```
Syntax:	keepalive_timeout timeout [header_timeout];
Default:	
keepalive_timeout 75s;
Context:	http, server, location
```

如果配置的事件为0，则表示不支持长连接。可选的第二个参数在响应的header域中设置一个值“Keep-Alive: timeout=time”。这两个参数可以不一样。

除了长连接以外，还有一个特性为pipeline。

在http1.1中，引入了一种新的特性，即pipeline。那么什么是pipeline呢？pipeline其实就是流水线作业，它可以看作为keepalive的一种升华，因为pipeline也是基于长连接的，目的就是利用一个连接做多次请求。如果客户端要提交多个请求，对于keepalive来说，那么第二个请求，必须要等到第一个请求的响应接收完全后，才能发起，这和TCP的停止等待协议是一样的，得到两个响应的时间至少为2*RTT。而对pipeline来说，客户端不必等到第一个请求处理完后，就可以马上发起第二个请求。得到两个响应的时间可能能够达到1*RTT。nginx是直接支持pipeline的，但是，nginx对pipeline中的多个请求的处理却不是并行的，依然是一个请求接一个请求的处理，只是在处理第一个请求的时候，客户端就可以发起第二个请求。这样，nginx利用pipeline减少了处理完一个请求后，等待第二个请求的请求头数据的时间。其实nginx的做法很简单，前面说到，nginx在读取数据时，会将读取的数据放到一个buffer里面，所以，如果nginx在处理完前一个请求后，如果发现buffer里面还有数据，就认为剩下的数据是下一个请求的开始，然后就接下来处理下一个请求，否则就设置keepalive。

## tcp_nopush/tcp_nodelay

对于tcp连接来说，缓存中数据发送方式会对传输存在一定的影响。

如果是缓存中有数据就立即发出，将导致负载过高的问题。例如可能传输的数据只有一个字节，但tcp头本身就有40字节长度，效率过低。因此当前tcp默认传输方式是使用Nagle算法，TCP堆栈实现了等待数据 0.2秒钟，因此操作后它不会发送一个数据包，而是将这段时间内的数据打成一个大的包。

但等待0.2秒不一定适合所有场景，因此有了tcp_nopush和tcp_nodelay方式，这些都是套接字属性，通过`setsockopt`系统调用设置套接字属性即可。

在 nginx 中，tcp_nopush 配置和 tcp_nodelay "互斥"。它可以配置一次发送数据的包大小。也就是说，它不是按时间累计 0.2 秒后发送包，而是当包累计到一定大小后就发送。在 nginx 中，tcp_nopush 必须和 sendfile 搭配使用。

tcp_nodelay表示不使用等待，立即发送。

默认配置为：

```
tcp_nopush : on;
```

配置块在http、server、location中。

## 延迟关闭（lingering_close）

lingering_close，字面意思就是延迟关闭，也就是说，当nginx要关闭连接时，并非立即关闭连接，而是先关闭tcp连接的写，再等待一段时间后再关掉连接的读。为什么要这样呢？我们先来看看这样一个场景。nginx在接收客户端的请求时，可能由于客户端或服务端出错了，要立即响应错误信息给客户端，而nginx在响应错误信息后，大分部情况下是需要关闭当前连接。nginx执行完write()系统调用把错误信息发送给客户端，write()系统调用返回成功并不表示数据已经发送到客户端，有可能还在tcp连接的write buffer里。接着如果直接执行close()系统调用关闭tcp连接，内核会首先检查tcp的read buffer里有没有客户端发送过来的数据留在内核态没有被用户态进程读取，如果有则发送给客户端RST报文来关闭tcp连接丢弃write buffer里的数据，如果没有则等待write buffer里的数据发送完毕，然后再经过正常的4次分手报文断开连接。所以,当在某些场景下出现tcp write buffer里的数据在write()系统调用之后到close()系统调用执行之前没有发送完毕，且tcp read buffer里面还有数据没有读，close()系统调用会导致客户端收到RST报文且不会拿到服务端发送过来的错误信息数据。那客户端肯定会想，这服务器好霸道，动不动就reset我的连接，连个错误信息都没有。

在上面这个场景中，我们可以看到，关键点是服务端给客户端发送了RST包，导致自己发送的数据在客户端忽略掉了。所以，解决问题的重点是，让服务端别发RST包。再想想，我们发送RST是因为我们关掉了连接，关掉连接是因为我们不想再处理此连接了，也不会有任何数据产生了。对于全双工的TCP连接来说，我们只需要关掉写就行了，读可以继续进行，我们只需要丢掉读到的任何数据就行了，这样的话，当我们关掉连接后，客户端再发过来的数据，就不会再收到RST了。当然最终我们还是需要关掉这个读端的，所以我们会设置一个超时时间，在这个时间过后，就关掉读，客户端再发送数据来就不管了，作为服务端我会认为，都这么长时间了，发给你的错误信息也应该读到了，再慢就不关我事了，要怪就怪你RP不好了。当然，正常的客户端，在读取到数据后，会关掉连接，此时服务端就会在超时时间内关掉读端。这些正是lingering_close所做的事情。协议栈提供 SO_LINGER 这个选项，它的一种配置情况就是来处理lingering_close的情况的，不过nginx是自己实现的lingering_close。lingering_close存在的意义就是来读取剩下的客户端发来的数据，所以nginx会有一个读超时时间，通过lingering_timeout选项来设置，如果在lingering_timeout时间内还没有收到数据，则直接关掉连接。nginx还支持设置一个总的读取时间，通过lingering_time来设置，这个时间也就是nginx在关闭写之后，保留socket的时间，客户端需要在这个时间内发送完所有的数据，否则nginx在这个时间过后，会直接关掉连接。

### lingering_close

设置是否需要延迟关闭。语法如下：

```
Syntax:	lingering_close off | on | always;
Default:	
lingering_close on;
Context:	http, server, location
```

默认值“on”指示 nginx 在完全关闭连接之前等待并处理来自客户端的额外数据，但前提是启发式表明客户端可能正在发送更多数据。

值“always”将导致 nginx 无条件等待和处理额外的客户端数据。

值“off”告诉nginx永远不要等待更多数据并立即关闭连接。 这种行为违反了协议，不应在正常情况下使用。

### lingering_time

设置总的延迟时间。当 lingering_close 生效时，该指令指定 nginx 处理（读取和忽略）来自客户端的附加数据的最长时间。 之后，即使会有更多数据，连接也会关闭。语法如下：

```
Syntax:	lingering_time time;
Default:	
lingering_time 30s;
Context:	http, server, location
```

### lingering_timeout

当 lingering_close 生效时，该指令指定更多客户端数据到达的最大等待时间。 如果在此期间未收到数据，则关闭连接。 否则，数据被读取并忽略，nginx再次开始等待更多数据。 重复“等待-读取-忽略”循环，但不会超过 lingering_time 指令指定的时间。

语法如下：

```
Syntax:	lingering_timeout time;
Default:	
lingering_timeout 5s;
Context:	http, server, location
```

## 加速close

在某个请求超时时，nginx会主动关闭该请求，这时会主动调用close函数来关闭套接字，但主动关闭套接字会导致套接字处于TIME_WAIT状态，这将导致大量的超时连接无法被立即释放，造成大量浪费。

tcp通过了`SO_LINGER`选项来设置关闭方式。其中参数为`linger`结构：

```c
struct linger
  {
    int l_onoff;		/* Nonzero to linger on close.  */
    int l_linger;		/* Time to linger.  */
  };
```

当`l_onoff`为正数，`l_linger`为0时，会立即在执行close时，立即向客户端发送`RST`报文，立即关闭连接，并释放此套接字占用的所有内存。

nginx中使用`reset_timedout_connection`配置来支持该功能：

```
Syntax:	reset_timedout_connection on | off;
Default:	
reset_timedout_connection off;
Context:	http, server, location
```

当配置为`on`时，会快速关闭。

# Nginx常用数据结构

## ngx_buf_t

```c
typedef struct ngx_buf_s  ngx_buf_t;

struct ngx_buf_s {
    // pos通常表示从该位置处理内存中的数据
    u_char          *pos;
    // last表示有效内容的结尾。ps到last的内存是nginx需要处理的内容
    u_char          *last;
    // 处理文件时，file_post与file_last和处理内存时的pos与last相同
    off_t            file_pos;
    off_t            file_last;
    
    // 如果ngx_buf_t缓冲区用于内存，那么start指向这段内存的起始地址
    u_char          *start;         /* start of buffer */
    // 如果ngx_buf_t缓冲区用于内存，那么start指向这段内存的结束地址
    u_char          *end;           /* end of buffer */
    
    // 表示当前缓冲区类型，例如由哪个模块使用，就执行该模块的ngx_module_t变量的地址
    ngx_buf_tag_t    tag;
    // 引用的文件
    ngx_file_t      *file;
    // 当前缓冲区的影子缓冲区，使用较少。
    ngx_buf_t       *shadow;


    /* the buf's content could be changed */
    unsigned         temporary:1;

    /*
     * the buf's content is in a memory cache or in a read only memory
     * and must not be changed
     */
    unsigned         memory:1;

    /* the buf's content is mmap()ed and must not be changed */
    unsigned         mmap:1;
    // 是否可回收
    unsigned         recycled:1;
    // 处理的是否为文件
    unsigned         in_file:1;
    // 是否需要flush
    unsigned         flush:1;
    // 是否使用同步方式，需谨慎考虑，可能会阻塞nginx。sync为1时可能会以阻塞的方式进行I/O
    unsigned         sync:1;
    // 是否为最后一个缓冲区。因为ngx_buf_t可以由ngx_chain_t链表串联起来。
    unsigned         last_buf:1;
    // 是否为ngx_chain_t中最后一个缓冲区
    unsigned         last_in_chain:1;
    
    // 省份为最后一个影子缓冲区，与shadow配合使用
    unsigned         last_shadow:1;
    // 当前缓冲区是否为临时文件
    unsigned         temp_file:1;

    /* STUB */ int   num;
};
```

ngx_buf_t为缓存区结构。

## ngx_chain_s

该结构为配合`ngx_buf_t`使用的链表，定义如下：

```c
struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

## ngx_list_t

`ngx_list_t`为nginx封装的链表容器，其定义如下：

```c
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts; // 数组元素的起始地址
    ngx_uint_t        nelts; // 当前已经使用的元素数量
    ngx_list_part_t  *next; // 下一个链表元素地址
};

typedef struct {
    ngx_list_part_t  *last; // 指向链表中最后一个元素的地址
    ngx_list_part_t   part; // 链表的首个数组元素
    size_t            size; // 每个ngx_list_part_s中的元素大小
    ngx_uint_t        nalloc; // 每个ngx_list_part_s可以容纳的元素数量
    ngx_pool_t       *pool; // 链表中管理内存分配的内存池
} ngx_list_t;
```

`ngx_list_t`描述整个链表，`ngx_list_part_s`，描述每个链表的元素。每个链表的元素`ngx_list_part_s`是一个数组，拥有连续的内存，依赖`ngx_list_t`的size和nalloc来表示数组容量，又依赖每个`ngx_list_part_s`成员的nelts来表示已经使用的容量。

[![IPik9I.png](https://z3.ax1x.com/2021/11/01/IPik9I.png)](https://imgtu.com/i/IPik9I)

### 初始化链表ngx_list_init

```
static ngx_inline ngx_int_t
ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    // 为第一个元素分配指定的空间
    list->part.elts = ngx_palloc(pool, n * size);
    if (list->part.elts == NULL) {
        return NGX_ERROR;
    }

    list->part.nelts = 0;
    list->part.next = NULL;
    list->last = &list->part;
    list->size = size;
    list->nalloc = n;
    list->pool = pool;

    return NGX_OK;
}
```

### 创建list ngx_list_create

```c
ngx_list_t *
ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    ngx_list_t  *list;

    list = ngx_palloc(pool, sizeof(ngx_list_t));
    if (list == NULL) {
        return NULL;
    }

    if (ngx_list_init(list, pool, n, size) != NGX_OK) {
        return NULL;
    }

    return list;
}
```

### 添加数据ngx_list_push

该方法返回添加元素对应的地址，返回地址后由调用方对地址赋值。

```c
void *
ngx_list_push(ngx_list_t *l)
{
    void             *elt;
    ngx_list_part_t  *last;

    last = l->last;
    // 如果最后的地址已经使用完了，新建一个元素
    if (last->nelts == l->nalloc) {

        /* the last part is full, allocate a new list part */

        last = ngx_palloc(l->pool, sizeof(ngx_list_part_t));
        if (last == NULL) {
            return NULL;
        }

        last->elts = ngx_palloc(l->pool, l->nalloc * l->size);
        if (last->elts == NULL) {
            return NULL;
        }

        last->nelts = 0;
        last->next = NULL;

        l->last->next = last;
        l->last = last;
    }
    // 分配一个地址
    elt = (char *) last->elts + l->size * last->nelts;
    // 已使用地址+1
    last->nelts++;

    return elt;
}
```

## ngx_table_elt_t

```c
typedef struct {
    ngx_uint_t        hash; // 某个散列表中执行对应的hash算法计算的hash值
    ngx_str_t         key;
    ngx_str_t         value;
    u_char           *lowcase_key; // 全小写的key值
} ngx_table_elt_t;
```

`ngx_table_elt_t`是一个Key/Value对。主要使用于解析http头部。

## 散列表（ngx_hash_t）

存储散列表的类有如下三个相关结构：

```c
// hash基础数据类
typedef struct {
    ngx_str_t         key;  // key值
    ngx_uint_t        key_hash; // key经过ngx_hash_key_pt转换的类
    void             *value; // key对应的值
} ngx_hash_key_t;

// 散列表中存储的每个数据
typedef struct {
    void             *value;  // 对应存储的数据
    u_short           len;    // name长度
    u_char            name[1];  // key值，name
} ngx_hash_elt_t;

// 实际存储散列表的结构
typedef struct {
    ngx_hash_elt_t  **buckets; // 存储数据内容。为了解决冲突，因此是一个二维数组。
    ngx_uint_t        size;    // 桶大小，多少个桶
} ngx_hash_t;

// 用来构建散列表的结构
typedef struct {
    ngx_hash_t       *hash;
    ngx_hash_key_pt   key; // 将数据的key进行转换的函数

    ngx_uint_t        max_size; // hash表中的桶的个数。该字段越大，元素存储时冲突的可能性越小，每个桶中存储的元素会更少，则查询起来的速度更快。当然，这个值越大，越造成内存的浪费也越大，(实际上也浪费不了多少)。
    ngx_uint_t        bucket_size; // 每个桶的最大限制大小，单位是字节。如果在初始化一个hash表的时候，发现某个桶里面无法存的下所有属于该桶的元素，则hash表初始化失败。

    char             *name;  // 散列表的名称
    ngx_pool_t       *pool; // 内存池
    ngx_pool_t       *temp_pool; // 临时内存池
} ngx_hash_init_t;

typedef ngx_uint_t (*ngx_hash_key_pt) (u_char *data, size_t len);
```

### 散列表构建

构建hash表使用函数ngx_hash_init:

```c
#define NGX_HASH_ELT_SIZE(name)                                               \
    (sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))
/* 该宏返回每个key生成ngx_hash_elt_t大小（字节）的sizeof(void *)的整数倍，这即使用其占用内存大小（虽然可能实际需要内存小于该值）。这里sizeof(void *)对应于ngx_hash_elt_t的value指针，(name)->key.len对应与name占用空间，2对应于len字节。这里使用ngx_align方法不是为了内存对其，而是为了将每个ngx_hash_elt_t进行划分，为了方便查询时更方快速获取。由于ngx_hash_t中的buckets申请的连续内存，name字段是要存储在该连续内存中，因此每个ngx_hash_elt_t所占用空间大小不一致，不能直接使用下标进行获取，使用该方法是为了内存对其，加速获取速度，具体查找时的方法参加下一小*/

ngx_int_t
ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts)
{
    // 这里参数ngx_hash_key_t一般是存储在ngx_array_t中第一个元素，nelts是元素数量
    u_char          *elts;
    size_t           len;
    u_short         *test;
    ngx_uint_t       i, n, key, size, start, bucket_size;
    ngx_hash_elt_t  *elt, **buckets;
    
    // max_size为0表示不能分配一个桶
    if (hinit->max_size == 0) {
        ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                      "could not build %s, you should "
                      "increase %s_max_size: %i",
                      hinit->name, hinit->name, hinit->max_size);
        return NGX_ERROR;
    }
    
    // 每个桶的容量（字节数量）过大
    if (hinit->bucket_size > 65536 - ngx_cacheline_size) {
        ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                      "could not build %s, too large "
                      "%s_bucket_size: %i",
                      hinit->name, hinit->name, hinit->bucket_size);
        return NGX_ERROR;
    }

    for (n = 0; n < nelts; n++) {
        /* 如果某个key构建的ngx_hash_elt_t所需空间加上一个void*所用空间大于bucket_siz，则报错。这里增加void*指针，是因为在ngx_hash_t中的buckets中，对于每一个桶，并没有指名末尾是什么，因此需要在最后增加一个void*的空间，对应获取到ngx_hash_elt_t的value字段，该字段设置为空，表示桶的末尾。*/
        if (hinit->bucket_size < NGX_HASH_ELT_SIZE(&names[n]) + sizeof(void *))
        {
            ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build %s, you should "
                          "increase %s_bucket_size: %i",
                          hinit->name, hinit->name, hinit->bucket_size);
            return NGX_ERROR;
        }
    }
    // 分配 存储每个hash元素所需要的空间大小 数组
    test = ngx_alloc(hinit->max_size * sizeof(u_short), hinit->pool->log);
    if (test == NULL) {
        return NGX_ERROR;
    }
    // bucket_size为允许桶存储的大小减去末尾记录为结尾的空间
    bucket_size = hinit->bucket_size - sizeof(void *);
    
    /* 初始分配需要多少个桶，使用元素数量/(每个桶允许存储数据的字节大小/(2 * sizeof(void *))，这里2 * sizeof(void *)是一个ngx_hash_elt_t预估占用内存大小，因此实际为元素数量除以每个桶允许存放的元素个数 */
    start = nelts / (bucket_size / (2 * sizeof(void *)));
    start = start ? start : 1;

    if (hinit->max_size > 10000 && nelts && hinit->max_size / nelts < 100) {
        start = hinit->max_size - 1000;
    }
    
    // 查找实际需要多少个桶才能满足每个桶中最多分配bucket_size内存，实际需要的桶数量不能超过限制的最大值max_size
    for (size = start; size <= hinit->max_size; size++) {

        ngx_memzero(test, size * sizeof(u_short));
        // 遍历每一个元素
        for (n = 0; n < nelts; n++) {
            if (names[n].key.data == NULL) {
                continue;
            }
            // 找到以当前分配桶的数量时，其应该存放到哪个桶中
            key = names[n].key_hash % size;
            // 计算当前分配到这个桶中所以元素所占用的内存大小
            len = test[key] + NGX_HASH_ELT_SIZE(&names[n]);

#if 0
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: %ui %uz \"%V\"",
                          size, key, len, &names[n].key);
#endif
            // 如果当前该桶占用的内存已经超出限制，则表示当前分配的桶数量不足，增加桶的数量，再查找验证
            if (len > bucket_size) {
                goto next;
            }
            // 如果循环结束，依然位发现有某个桶分配的元素所占用内存过大，则说明找到了桶大小
            test[key] = (u_short) len;
        }

        goto found;

    next:

        continue;
    }
    // 未找到，则表示hash表初始化失败，报错
    size = hinit->max_size;

    ngx_log_error(NGX_LOG_WARN, hinit->pool->log, 0,
                  "could not build optimal %s, you should increase "
                  "either %s_max_size: %i or %s_bucket_size: %i; "
                  "ignoring %s_bucket_size",
                  hinit->name, hinit->name, hinit->max_size,
                  hinit->name, hinit->bucket_size, hinit->name);

found:
// 查找到需要建立多少个桶的处理
    // 每一个桶中先分配一个作为末尾表示的空指针对应的存储空间大小
    for (i = 0; i < size; i++) {
        test[i] = sizeof(void *);
    }
    // 遍历每个元素
    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }
        // 计算每个元素在该桶大小下分配到哪个桶中
        key = names[n].key_hash % size;
        // 记录该桶中当前已占用内存大小
        len = test[key] + NGX_HASH_ELT_SIZE(&names[n]);
        // 超过该值报错
        if (len > 65536 - ngx_cacheline_size) {
            ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build %s, you should "
                          "increase %s_max_size: %i",
                          hinit->name, hinit->name, hinit->max_size);
            ngx_free(test);
            return NGX_ERROR;
        }

        test[key] = (u_short) len;
    }

    len = 0;
    // 变量每一个桶，计算总共需要创建的内存大小
    for (i = 0; i < size; i++) {
        // 空的桶不需要创建内存
        if (test[i] == sizeof(void *)) {
            continue;
        }
        // 为了加速请求速度，每个桶分配内存都为cache line的整数倍 
        test[i] = (u_short) (ngx_align(test[i], ngx_cacheline_size));

        len += test[i];
    }
    // 分配hash需要的内存大小，分配buckets空间
    if (hinit->hash == NULL) {
        hinit->hash = ngx_pcalloc(hinit->pool, sizeof(ngx_hash_wildcard_t)
                                             + size * sizeof(ngx_hash_elt_t *));
        if (hinit->hash == NULL) {
            ngx_free(test);
            return NGX_ERROR;
        }

        buckets = (ngx_hash_elt_t **)
                      ((u_char *) hinit->hash + sizeof(ngx_hash_wildcard_t));

    } else {
        buckets = ngx_pcalloc(hinit->pool, size * sizeof(ngx_hash_elt_t *));
        if (buckets == NULL) {
            ngx_free(test);
            return NGX_ERROR;
        }
    }
    
    // 分配存储所有元素需要的空间大小
    elts = ngx_palloc(hinit->pool, len + ngx_cacheline_size);
    if (elts == NULL) {
        ngx_free(test);
        return NGX_ERROR;
    }
    
    // 为加速，进行内存行对其
    elts = ngx_align_ptr(elts, ngx_cacheline_size);
    
    // 遍历每个桶
    for (i = 0; i < size; i++) {
        if (test[i] == sizeof(void *)) {
            continue;
        }
        // 使用分配的内存，初始化每个桶的起始位置
        buckets[i] = (ngx_hash_elt_t *) elts;
        elts += test[i];
    }

    for (i = 0; i < size; i++) {
        test[i] = 0;
    }
    
    // 对每一个元素赋值
    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }
        // 找到该元素分配到每个哪个桶中
        key = names[n].key_hash % size;
        // 找到该元素应该分配到桶中的内存起始位置
        elt = (ngx_hash_elt_t *) ((u_char *) buckets[key] + test[key]);
        // 对元素赋值
        elt->value = names[n].value;
        elt->len = (u_short) names[n].key.len;

        ngx_strlow(elt->name, names[n].key.data, names[n].key.len);
        // 增加该桶中使用了的内存大小
        test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    }
    
    // 对每一个桶中，增加结束标识
    for (i = 0; i < size; i++) {
        if (buckets[i] == NULL) {
            continue;
        }
        // 获取到当前桶已经使用到的内存位置
        elt = (ngx_hash_elt_t *) ((u_char *) buckets[i] + test[i]);
        // 对该位置的value设置为null，查找时作为结束标识
        elt->value = NULL;
    }

    ngx_free(test);
    // 赋值桶和桶大小
    hinit->hash->buckets = buckets;
    hinit->hash->size = size;

#if 0

    for (i = 0; i < size; i++) {
        ngx_str_t   val;
        ngx_uint_t  key;

        elt = buckets[i];

        if (elt == NULL) {
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: NULL", i);
            continue;
        }

        while (elt->value) {
            val.len = elt->len;
            val.data = &elt->name[0];

            key = hinit->key(val.data, val.len);

            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: %p \"%V\" %ui", i, elt, &val, key);

            elt = (ngx_hash_elt_t *) ngx_align_ptr(&elt->name[0] + elt->len,
                                                   sizeof(void *));
        }
    }

#endif

    return NGX_OK;
}
```

### 散列表查找

```c
void *
ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len)
{
    ngx_uint_t       i;
    ngx_hash_elt_t  *elt;

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "hf:\"%*s\"", len, name);
#endif
    // 找到该元素属于哪个桶
    elt = hash->buckets[key % hash->size];

    if (elt == NULL) {
        return NULL;
    }
    
    // 查找桶中元素，直到末尾
    while (elt->value) {
        if (len != (size_t) elt->len) {
            goto next;
        }

        for (i = 0; i < len; i++) {
            if (name[i] != elt->name[i]) {
                goto next;
            }
        }
        // 找到后返回
        return elt->value;

    next:

        elt = (ngx_hash_elt_t *) ngx_align_ptr(&elt->name[0] + elt->len,
                                               sizeof(void *));
        continue;
    }

    return NULL;
}
```

## 管理散列表（ngx_hash_keys_arrays_t）

nginx为了让大家方便的构造hash表，提供给了此辅助类型，如下：

```c
typedef struct {
    // 将要构建的hash表的桶的个数。对于使用这个结构中包含的信息构建的三种类型的hash表都会使用此参数。
    ngx_uint_t        hsize;
    // 构建这些hash表使用的pool。
    ngx_pool_t       *pool;
    // 在构建这个类型以及最终的三个hash表过程中可能用到临时pool。该temp_pool可以在构建完成以后，被销毁掉。这里只是存放临时的一些内存消耗。
    ngx_pool_t       *temp_pool;

    // 存放所有非通配符key的数组。
    ngx_array_t       keys;
    /*这是个二维数组，第一个维度代表的是bucket的编号，那么keys_hash[i]中存放的是所有的key算出来的hash值对hsize取模以后的值为i的key。假设有3个key,分别是key1,key2和key3假设hash值算出来以后对hsize取模的值都是i，那么这三个key的值就顺序存放在keys_hash[i][0],keys_hash[i][1], keys_hash[i][2]。该值在调用的过程中用来保存和检测是否有冲突的key值，也就是是否有重复。用于存放非通配符数据*/
    ngx_array_t      *keys_hash;
    // 放前向通配符key被处理完成以后的值。比如：“*.abc.com” 被处理完成以后，变成 “com.abc.” 被存放在此数组中。
    ngx_array_t       dns_wc_head;
    // 该值在调用的过程中用来保存和检测是否有冲突的前向通配符的key值，也就是是否有重复。
    ngx_array_t      *dns_wc_head_hash;
    // 存放后向通配符key被处理完成以后的值。比如：“mail.xxx.*” 被处理完成以后，变成 “mail.xxx.” 被存放在此数组中。
    ngx_array_t       dns_wc_tail;
    // 该值在调用的过程中用来保存和检测是否有冲突的后向通配符的key值，也就是是否有重复。
    ngx_array_t      *dns_wc_tail_hash;
} ngx_hash_keys_arrays_t;
// 一维数组存储数据一般ngx_hash_key_t，二维数组存储数据一般ngx_str_t
```

### 构建ngx_hash_keys_arrays_t

使用ngx_hash_keys_array_init构建该结构：

```c
ngx_int_t
// type指名构建的大小。其中NGX_HASH_SMALL表示待初始化的元素较少，NGX_HASH_LARGE表示待初始化的元素较多
ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type)
{
    // 数组预分配空间大小 
    ngx_uint_t  asize;
    // 初始化较少时，预分配大小为4，桶数量为107
    if (type == NGX_HASH_SMALL) {
        asize = 4;
        ha->hsize = 107;
    // 初始化较多时，预分配大小为16384，桶数量为10007
    } else {
        asize = NGX_HASH_LARGE_ASIZE;
        ha->hsize = NGX_HASH_LARGE_HSIZE;
    }
    // 初始化存储数据的三个数组
    if (ngx_array_init(&ha->keys, ha->temp_pool, asize, sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    if (ngx_array_init(&ha->dns_wc_head, ha->temp_pool, asize,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    if (ngx_array_init(&ha->dns_wc_tail, ha->temp_pool, asize,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    
    // 初始化用于查看是否有重复的简易hash表
    ha->keys_hash = ngx_pcalloc(ha->temp_pool, sizeof(ngx_array_t) * ha->hsize);
    if (ha->keys_hash == NULL) {
        return NGX_ERROR;
    }

    ha->dns_wc_head_hash = ngx_pcalloc(ha->temp_pool,
                                       sizeof(ngx_array_t) * ha->hsize);
    if (ha->dns_wc_head_hash == NULL) {
        return NGX_ERROR;
    }

    ha->dns_wc_tail_hash = ngx_pcalloc(ha->temp_pool,
                                       sizeof(ngx_array_t) * ha->hsize);
    if (ha->dns_wc_tail_hash == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

### 添加元素

使用如下函数增加元素：

```c
/* flag取值有三种：NGX_HASH_WILDCARD_KEY表示需要处理通配符。 NGX_HASH_READONLY_KEY表示不能对关键字做变更，即不能通过全小写关键字来获取散列码，其他值：即不处理通配符，又允许把关键字全小写获取散列码*/
ngx_int_t
ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key, void *value,
    ngx_uint_t flags)
{
    size_t           len;
    u_char          *p;
    ngx_str_t       *name;
    // skip表示起始位置要跳过字符数量，last表示到末尾的长度
    ngx_uint_t       i, k, n, skip, last;
    ngx_array_t     *keys, *hwc;
    ngx_hash_key_t  *hk;

    last = key->len;
    // 处理通配符
    if (flags & NGX_HASH_WILDCARD_KEY) {

        /*
         * supported wildcards:
         *     "*.example.com", ".example.com", and "www.example.*"
         */

        n = 0;
        // 校验字符串正确性
        for (i = 0; i < key->len; i++) {
            // 如果出现多个*表示为nginx不正常的通配符格式（只支持首尾一个通配符）
            if (key->data[i] == '*') {
                if (++n > 1) {
                    return NGX_DECLINED;
                }
            }
            // 错误通配符数据
            if (key->data[i] == '.' && key->data[i + 1] == '.') {
                return NGX_DECLINED;
            }
            // 字符串以\0结尾
            if (key->data[i] == '\0') {
                return NGX_DECLINED;
            }
        }
        // ".example.com"格式数据,存储数据时去带.
        if (key->len > 1 && key->data[0] == '.') {
            skip = 1;
            goto wildcard;
        }

        if (key->len > 2) {
            // "*.example.com"格式数据,存储数据时去带.*
            if (key->data[0] == '*' && key->data[1] == '.') {
                skip = 2;
                goto wildcard;
            }
            // "www.example.*"格式数据,存储数据时去带.*
            if (key->data[i - 2] == '.' && key->data[i - 1] == '*') {
                skip = 0;
                last -= 2;
                goto wildcard;
            }
        }
        // 存在通配符*,但不在首尾
        if (n) {
            return NGX_DECLINED;
        }
    }

    /* exact hash */
    // 未找到通配符或不查看通配符
    k = 0;
    // 根据是否可以转换为小写计算哈希值
    for (i = 0; i < last; i++) {
        if (!(flags & NGX_HASH_READONLY_KEY)) {
            key->data[i] = ngx_tolower(key->data[i]);
        }
        k = ngx_hash(k, key->data[i]);
    }
    
    k %= ha->hsize;

    /* check conflicts in exact hash */
    // 找到在keys_hash精准匹配的哪个桶中
    name = ha->keys_hash[k].elts;
    // 如果桶不为空，查看桶中是否有当前值
    if (name) {
        for (i = 0; i < ha->keys_hash[k].nelts; i++) {
            if (last != name[i].len) {
                continue;
            }

            if (ngx_strncmp(key->data, name[i].data, last) == 0) {
                return NGX_BUSY;
            }
        }
    // 初始化桶
    } else {
        if (ngx_array_init(&ha->keys_hash[k], ha->temp_pool, 4,
                           sizeof(ngx_str_t))
            != NGX_OK)
        {
            return NGX_ERROR;
        }
    }
    // 添加元素到桶中
    name = ngx_array_push(&ha->keys_hash[k]);
    if (name == NULL) {
        return NGX_ERROR;
    }

    *name = *key;

    hk = ngx_array_push(&ha->keys);
    if (hk == NULL) {
        return NGX_ERROR;
    }

    hk->key = *key;
    hk->key_hash = ngx_hash_key(key->data, last);
    hk->value = value;

    return NGX_OK;

// 通配符处理逻辑
wildcard:

    /* wildcard hash */
    // 去除通配符相关字段*.,.，转换小写计算hash值
    k = ngx_hash_strlow(&key->data[skip], &key->data[skip], last - skip);

    k %= ha->hsize;
    // 对于是.example.com这种形式的key，其只能和精准的example.com两者直接存在一个。或者在精准中存在一个example.com，或者在前缀匹配中，存在一个.example.com
    if (skip == 1) {

        /* check conflicts in exact hash for ".example.com" */

        name = ha->keys_hash[k].elts;

        if (name) {
            len = last - skip;

            for (i = 0; i < ha->keys_hash[k].nelts; i++) {
                if (len != name[i].len) {
                    continue;
                }
                // 如果精准匹配中已存在，则返回
                if (ngx_strncmp(&key->data[1], name[i].data, len) == 0) {
                    return NGX_BUSY;
                }
            }

        } else {
            if (ngx_array_init(&ha->keys_hash[k], ha->temp_pool, 4,
                               sizeof(ngx_str_t))
                != NGX_OK)
            {
                return NGX_ERROR;
            }
        }
        // 如果精准匹配中不存在，则添加一个example.com形式的key到精准匹配中，避免精准匹配中再增加
        name = ngx_array_push(&ha->keys_hash[k]);
        if (name == NULL) {
            return NGX_ERROR;
        }

        name->len = last - 1;
        name->data = ngx_pnalloc(ha->temp_pool, name->len);
        if (name->data == NULL) {
            return NGX_ERROR;
        }

        ngx_memcpy(name->data, &key->data[1], name->len);
    }

    // 对于前置通配符
    if (skip) {

        /*
         * convert "*.example.com" to "com.example.\0"
         *      and ".example.com" to "com.example\0"
         */

        p = ngx_pnalloc(ha->temp_pool, last);
        if (p == NULL) {
            return NGX_ERROR;
        }

        len = 0;
        n = 0;
        // 转换*.example.com到com.example.\0，.example.com到com.example\0，注意这里的i未到0
        for (i = last - 1; i; i--) {
            if (key->data[i] == '.') {
                ngx_memcpy(&p[n], &key->data[i + 1], len);
                n += len;
                p[n++] = '.';
                len = 0;
                continue;
            }

            len++;
        }
        // 对于.example.com的来说，将example拼接上
        if (len) {
            ngx_memcpy(&p[n], &key->data[1], len);
            n += len;
        }

        p[n] = '\0';

        hwc = &ha->dns_wc_head;
        keys = &ha->dns_wc_head_hash[k];

    } else {

        /* convert "www.example.*" to "www.example\0" */

        last++;

        p = ngx_pnalloc(ha->temp_pool, last);
        if (p == NULL) {
            return NGX_ERROR;
        }

        ngx_cpystrn(p, key->data, last);

        hwc = &ha->dns_wc_tail;
        keys = &ha->dns_wc_tail_hash[k];
    }


    /* check conflicts in wildcard hash */
    // 在对应的通配符表中查找是否已经存在
    name = keys->elts;

    if (name) {
        len = last - skip;

        for (i = 0; i < keys->nelts; i++) {
            if (len != name[i].len) {
                continue;
            }

            if (ngx_strncmp(key->data + skip, name[i].data, len) == 0) {
                return NGX_BUSY;
            }
        }

    } else {
        if (ngx_array_init(keys, ha->temp_pool, 4, sizeof(ngx_str_t)) != NGX_OK)
        {
            return NGX_ERROR;
        }
    }
    // 向对应的简易hash表中增加数据，注意，这里增加的是去除通配符字段后原始数据，并未将首通配符样式字段倒排
    name = ngx_array_push(keys);
    if (name == NULL) {
        return NGX_ERROR;
    }

    name->len = last - skip;
    name->data = ngx_pnalloc(ha->temp_pool, name->len);
    if (name->data == NULL) {
        return NGX_ERROR;
    }

    ngx_memcpy(name->data, key->data + skip, name->len);


    /* add to wildcard hash */
    // 添加数据到对于的数组中，这里添加的是处理后的数据，即将首通配符样式字段倒排
    hk = ngx_array_push(hwc);
    if (hk == NULL) {
        return NGX_ERROR;
    }
    // *.baidu.com被存储为com.baidu.;.baidu.com被存储为com.baidu; www.baidu.*被存储为www.baidu
    hk->key.len = last - 1;
    hk->key.data = p;
    hk->key_hash = 0;
    hk->value = value;

    return NGX_OK;
}
```

对于构建好的通配符数组，不能直接调用ngx_hash_wildcard_init来构建包含通配符的散列表，而是要先对数组进行排序，这就是为什么要在字节后面添加\0的作用，用于判断字符串结尾。排序时，`.`被认为是最低顺序的字节，即：`example.com`小于（排在前面）`example1.com`。

## 带有通配符的散列表（ngx_hash_wildcard_t）

```
typedef struct {
    // 基本散列表
    ngx_hash_t        hash;
    // 当使用ngx_hash_wildcard_t通配符散列表作为某容器的元素时，可以使用value指向用户数据
    void             *value;
} ngx_hash_wildcard_t;
```

nginx支持的带通配符的散列表，仅支持在起始或末尾带有散列表，即如下：

```
www.baidu.*
*.baidu.com
.baidu.com // 有可能被记录为不带通配符的baidu.com
```

### 构建带有通配符的散列表

ngx_hash_wildcard_init函数用来构建带有通配符的散列表。这里需要注意，不论是构建前置为通配符还是后置为通配符，其中的参数，即ngx_hash_key_t列表，均是z已被排序好的，即key均被排序完成，这时为了方便后续的构建。对于含有通配符的hash表来说，其是一个层级结构。例如对于如下两个key：

```
www.baidu.com
www.tencent.com
```

则构建的hash表为：

```
第一级   第二级    第三极
        baidu    com
www 
        tencent  com
```

查找时也是同样的逻辑，先按照`.`划分块，再一层一层查找。这也是为啥要将前置的通配符反转，变更成类似于后置的结构。

具体逻辑如下：

```c
ngx_int_t
ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,
    ngx_uint_t nelts)
{
    size_t                len, dot_len;
    ngx_uint_t            i, n, dot;
    ngx_array_t           curr_names, next_names;
    ngx_hash_key_t       *name, *next_name;
    ngx_hash_init_t       h;
    ngx_hash_wildcard_t  *wdc;
    // 分配当前层级数据空间
    if (ngx_array_init(&curr_names, hinit->temp_pool, nelts,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // 分配当下一层级数据空间
    if (ngx_array_init(&next_names, hinit->temp_pool, nelts,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // 从第一个开始查看，并不是简单的遍历
    for (n = 0; n < nelts; n = i) {

#if 0
        ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                      "wc0: \"%V\"", &names[n].key);
#endif
        // 记录是否查找到逗号，并使用len记录逗号所在下标
        dot = 0;

        for (len = 0; len < names[n].key.len; len++) {
            if (names[n].key.data[len] == '.') {
                dot = 1;
                break;
            }
        }
        // 添加当前的数据到当前层级数据中，对于查找到逗号的情况，其key变更为逗号前的部分，value为当前key对应数据的value
        name = ngx_array_push(&curr_names);
        if (name == NULL) {
            return NGX_ERROR;
        }

        name->key.len = len;
        name->key.data = names[n].key.data;
        name->key_hash = hinit->key(name->key.data, name->key.len);
        name->value = names[n].value;

#if 0
        ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                      "wc1: \"%V\" %ui", &name->key, dot);
#endif
        // 记录从开始到当前位置的长度，用于切割key
        dot_len = len + 1;
        // 如果发现了.，则len加一，对比时，.字符也会用来比较相同前缀的key
        if (dot) {
            len++;
        }

        next_names.nelts = 0;
        // 表示并为遍历到结尾，后面还有待切割字段，则将后面部分数据添加到构建下一层hash表的列表中
        if (names[n].key.len != len) {
            next_name = ngx_array_push(&next_names);
            if (next_name == NULL) {
                return NGX_ERROR;
            }

            next_name->key.len = names[n].key.len - len;
            next_name->key.data = names[n].key.data + len;
            next_name->key_hash = 0;
            // value存储对于的数据，后续可能会变更
            next_name->value = names[n].value;

#if 0
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "wc2: \"%V\"", &next_name->key);
#endif
        }
        // 遍历后面的key，找到第一个和当前查找到的前缀不匹配的key，注意 . 也会被匹配，即，查看的应该是 example.或者com这两种形式的查找匹配。
        for (i = n + 1; i < nelts; i++) {
            if (ngx_strncmp(names[n].key.data, names[i].key.data, len) != 0) {
                break;
            }
            // 如果没有查找到逗号，即当前的n是切割的末尾了，并且当前的i长度要大于现在的n的长度，且i的len不是.就结束查找。这时由于，com.ex也要是com的子一层hash表。但comm不是com子一层。
            if (!dot
                && names[i].key.len > len
                && names[i].key.data[len] != '.')
            {
                break;
            }
            // 将查找到属于当前n的子一层hash元素的数据添加到用来构建子一层hash表的数组中。
            next_name = ngx_array_push(&next_names);
            if (next_name == NULL) {
                return NGX_ERROR;
            }
            // 同样是要把当前一层的key去掉，子一层只用剩下的字符串
            next_name->key.len = names[i].key.len - dot_len;
            next_name->key.data = names[i].key.data + dot_len;
            next_name->key_hash = 0;
            next_name->value = names[i].value;

#if 0
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "wc3: \"%V\"", &next_name->key);
#endif
        }

        // 如果存在需要构建子一层hash表的key
        if (next_names.nelts) {

            h = *hinit;
            h.hash = NULL;
            // 对子数组递归调用ngx_hash_wildcard_init构建函数。这里会调用ngx_hash_init，对于hash等于null来说，构建的hash是ngx_hash_wildcard_t结构
            if (ngx_hash_wildcard_init(&h, (ngx_hash_key_t *) next_names.elts,
                                       next_names.nelts)
                != NGX_OK)
            {
                return NGX_ERROR;
            }
            // 转换为ngx_hash_wildcard_t
            wdc = (ngx_hash_wildcard_t *) h.hash;
            // 如果当前的n是查询的末尾，那么用ngx_hash_wildcard_t存储对应的value，如果不是查询末尾，则ngx_hash_wildcard_t存储对应的value对应为NULL
            if (names[n].key.len == len) {
                wdc->value = names[n].value;
            }
            // 当前ngx_hash_key_t的value指向下一层的hash表。同时利用指针最后两位来标识当前节点状况。10表示当前的value指向下一层hash表，11表示当前value指向下一层hash并且该字符串后面跟随着 .
            name->value = (void *) ((uintptr_t) wdc | (dot ? 3 : 2));
        // 如果不需要建下一层的hash表，并且发现了.，即com.example.这种情况，使用01记录。00则表示是最后的节点，其下层没有hash表，并且字段后面没有. 当前的value指向对应的数据
        } else if (dot) {
            name->value = (void *) ((uintptr_t) name->value | 1);
        }
    }
    // 构建当前层级的hash表
    if (ngx_hash_init(hinit, (ngx_hash_key_t *) curr_names.elts,
                      curr_names.nelts)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

对于如下数据：  

```
*.baidu.com               value1
*.baidu.cn                value2
*.tencent.com             value3
*.example.baidu.com       value4 
*.example.baidu-int.com   value5
```

首先被转换并排序为：

```
cn.baidu.                value2
com.baidu.               value1
com.baidu.example.       value4 
com.baidu-int.example.   value5
com.tencent.             value3
```

构建多层hash表如下：

[![WItYGR.png](https://z3.ax1x.com/2021/07/27/WItYGR.png)](https://imgtu.com/i/WItYGR)

### 查找后缀通配符

使用ngx_hash_find_wc_tail在后缀通配符中查找：

```c
void *
ngx_hash_find_wc_tail(ngx_hash_wildcard_t *hwc, u_char *name, size_t len)
{
    void        *value;
    ngx_uint_t   i, key;

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "wct:\"%*s\"", len, name);
#endif

    key = 0;
    // 使用.进行切割。如果查找到末尾还未找到，则返回null。由于是后缀通配符，因此查找到了末尾还未找到对应数据，则应该在精准匹配里面查找，而不应该在后置表中查找。
    for (i = 0; i < len; i++) {
        if (name[i] == '.') {
            break;
        }
        // 计算切割后的hash值
        key = ngx_hash(key, name[i]);
    }

    if (i == len) {
        return NULL;
    }

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "key:\"%ui\"", key);
#endif
    // 查找当前的hash表
    value = ngx_hash_find(&hwc->hash, key, name, i);

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "value:\"%p\"", value);
#endif

    if (value) {

        /*
         * the 2 low bits of value have the special meaning:
         *     00 - value is data pointer;
         *     11 - value is pointer to wildcard hash allowing "example.*".
         */
        // 对于指针倒数第二位来说，如果是1，则表明value值是下一层的hash数据
        if ((uintptr_t) value & 2) {
            // 加1，去掉 . 继续查找
            i++;
            // 恢复指针为正常指针
            hwc = (ngx_hash_wildcard_t *) ((uintptr_t) value & (uintptr_t) ~3);
            // 查找下一层的hash数据
            value = ngx_hash_find_wc_tail(hwc, &name[i], len - i);
            // 如果找到，则返回
            if (value) {
                return value;
            }
            // 未找到，则查找到的value对应的hash表结构ngx_hash_wildcard_t的value值，有可能为null（取决于构建时是否构建该层级的数据）
            return hwc->value;
        }

        return value;
    }
    // 如果hash表中没有对应切割的数据，那么返回当前层级hash表结构ngx_hash_wildcard_t的value值。有可能为空，因为查找时找最长匹配，如果后面没有对应的匹配，则使用当前层级的hash对应数据（有可能为null）。
    return hwc->value;
}
```

### 查找前缀通配符

```c
void *
ngx_hash_find_wc_head(ngx_hash_wildcard_t *hwc, u_char *name, size_t len)
{
    void        *value;
    ngx_uint_t   i, n, key;

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "wch:\"%*s\"", len, name);
#endif

    n = len;
    // 从后向前切词
    while (n) {
        if (name[n - 1] == '.') {
            break;
        }

        n--;
    }

    key = 0;
    // 计算切词的hash值
    for (i = n; i < len; i++) {
        key = ngx_hash(key, name[i]);
    }

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "key:\"%ui\"", key);
#endif
    // 查找当前的前缀表
    value = ngx_hash_find(&hwc->hash, key, &name[n], len - n);

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "value:\"%p\"", value);
#endif
    // 在当前hash表中查找到了对应数据
    if (value) {

        /*
         * the 2 low bits of value have the special meaning:
         *     00 - value is data pointer for both "example.com"
         *          and "*.example.com";
         *     01 - value is data pointer for "*.example.com" only;
         *     10 - value is pointer to wildcard hash allowing
         *          both "example.com" and "*.example.com";
         *     11 - value is pointer to wildcard hash allowing
         *          "*.example.com" only.
         */
        // 指针的倒数第二位是1，表示当前value指向的是下一层hash表
        if ((uintptr_t) value & 2) {
            // 查找到了key的末尾
            if (n == 0) {

                /* "example.com" */
                // 11表示构建hash时，是*.example.com，并且其后还有下一层hash。而当前搜索的key是example.com，返回空
                if ((uintptr_t) value & 1) {
                    return NULL;
                }
                // 10表示构建hash时，是.example.com，并且其后还有下一层hash。其允许搜索的是example.com，因此返回value对应的ngx_hash_wildcard_t中的value数据
                hwc = (ngx_hash_wildcard_t *)
                                          ((uintptr_t) value & (uintptr_t) ~3);
                return hwc->value;
            }
            // 恢复原指针，进行向下查找
            hwc = (ngx_hash_wildcard_t *) ((uintptr_t) value & (uintptr_t) ~3);

            value = ngx_hash_find_wc_head(hwc, name, n - 1);
 
            if (value) {
                return value;
            }
            // 如果没找到，则返回value对应的ngx_hash_wildcard_t中的value数据（有可能为空）
            return hwc->value;
        }
        // 01表示构建表的时候，是*.example.com，且其后没有下一层hash表了
        if ((uintptr_t) value & 1) {
            // 而当前搜索的key是example.com，返回空
            if (n == 0) {

                /* "example.com" */

                return NULL;
            }
            // 01表示构建表的时候，是.example.com，且其后没有下一层hash表了，其存储的是普通数据
            return (void *) ((uintptr_t) value & (uintptr_t) ~3);
        }
        // 如果是00，表示就是普通数据
        return value;
    }
    // 如果hash表中没有对应切割的数据，那么返回当前层级hash表结构ngx_hash_wildcard_t的value值。有可能为空，因为查找时找最长匹配，如果后面没有对应的匹配，则使用当前层级的hash对应数据（有可能为null）。
    return hwc->value;
}
```

## ngx_queue_t双向循环链表

### 数据结构

```c
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev; // 指向前一个元素的指针
    ngx_queue_t  *next; // 指向后一个元素的指针
};
```

ngx_queue_s维持了双向链表，忽略了其实际元素的内容，不负责链表元素的内存分配。要想使用双向链表，则要求我们定义的类能够通过指针转换为ngx_queue_s，因此其必须包含`*prev`和`*next`。此时可以直接通过指针来进行转换。例如：

```c
struct my_test {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
    int num;
}

my_test a;
a.prev = &a;
a.next = &a;
ngx_queue_s *q = (ngx_queue_s *)&a
```

注意双向循环链表存在一个维护双向链表结构的节点，该节点不存储数据，只是作为维护结构的节点。后续参数中h，表示维护链表的节点，参数为q表示存储数据的节点。

### 初始化双向循环链表：ngx_queue_init(q)

```
#define ngx_queue_init(q)                                                     \
    (q)->prev = q;                                                            \
    (q)->next = q
```

q为链表容器结构ngx_queue_s。

### 判空：ngx_queue_empty(h)

```
#define ngx_queue_empty(h)                                                    \
    (h == (h)->prev)
```

由于是循环链表，因此只需要判断任意一个链表元素的前驱是否为自身即可。这里使用的是维护结构的节点。

### 在头部插入元素：ngx_queue_insert_head（h，x）

在维护链表的节点后增加一个元素，即为在链表头插入元素。：

```
#define ngx_queue_insert_head(h, x)                                           \
    (x)->next = (h)->next;                                                    \
    (x)->next->prev = x;                                                      \
    (x)->prev = h;                                                            \
    (h)->next = x
```

### 在尾部插入节点：ngx_queue_insert_tail（h，x）

在维护链表的节点前插入一个元素，即为在链表尾部插入元素（双向的作用）

```
#define ngx_queue_insert_tail(h, x)                                           \
    (x)->prev = (h)->prev;                                                    \
    (x)->prev->next = x;                                                      \
    (x)->next = h;                                                            \
    (h)->prev = x
```

### 获取列表第一个元素：ngx_queue_head(h) 

维护链表节点的后一个元素即为首尾元素：

```
#define ngx_queue_head(h)                                                     \
    (h)->next
```

### 获取列表中最后一个元素：ngx_queue_last(h)

维护链表节点的前一个元素即为首尾元素：

```
#define ngx_queue_last(h)                                                     \
    (h)->prev
```

### 返回链表容器结构体指针：ngx_queue_sentinel(h)

```c
#define ngx_queue_sentinel(h)                                                 \
    (h)
```

常在循环中判断终止。

### 获取当前元素的下一个元素：ngx_queue_next(q) 

```
#define ngx_queue_next(q)                                                     \
    (q)->next
```

### 获取当前元素的上一个元素：ngx_queue_prev(q)

```
#define ngx_queue_prev(q)                                                     \
    (q)->prev
```

### 移出元素：ngx_queue_remove(x)

```
#define ngx_queue_remove(x)                                                   \
    (x)->next->prev = (x)->prev;                                              \
    (x)->prev->next = (x)->next
```

### 拆分链表：ngx_queue_split(h, q, n)

h为维护链表节点，q为其中一个元素，该方法将链表切割成两部分，通过q进行切割成两个链表h和n，前半部分h由原链表的前半部分构成（不包含q），n链表由原链表的后半部分组成，q为其首元素。

```
#define ngx_queue_split(h, q, n)                                              \
    (n)->prev = (h)->prev;                                                    \
    (n)->prev->next = n;                                                      \
    (n)->next = q;                                                            \
    (h)->prev = (q)->prev;                                                    \
    (h)->prev->next = h;                                                      \
    (q)->prev = n;
```

### 合并链表：ngx_queue_add(h, n)

将n链表添加到h链表的末尾。

```
#define ngx_queue_add(h, n)                                                   \
    (h)->prev->next = (n)->next;                                              \
    (n)->next->prev = (h)->prev;                                              \
    (h)->prev = (n)->prev;                                                    \
    (h)->prev->next = h;
```

### 返回元素对应的地址：ngx_queue_data(q, type, link)

对应能够转换为ngx_queue_t的元素，使用该方法，通过元素内容，获取ngx_queue_t的起始地址：

```
#define ngx_queue_data(q, type, link)                                         \
    (type *) ((u_char *) q - offsetof(type, link))
```

### 在指定元素后插入内容：ngx_queue_insert_after(h, x)

与ngx_queue_insert_head类似，不过一个是维护链表的节点，一个是存储数据的节点：

```
#define ngx_queue_insert_after   ngx_queue_insert_head
```

### 返回链表中心元素：ngx_queue_middle(ngx_queue_t *queue)

其处理方式为，分配两个指针middle和next，都从头开始遍历链表，让next一次走两步，middle一个走一步，当next走到终点时，milddle就是中间节点。具体逻辑如下：

```c
ngx_queue_t *
ngx_queue_middle(ngx_queue_t *queue)
{
    ngx_queue_t  *middle, *next;

    middle = ngx_queue_head(queue);
    // 判断是否只有一个元素
    if (middle == ngx_queue_last(queue)) {
        return middle;
    }

    next = ngx_queue_head(queue);

    for ( ;; ) {
        // middle先走一步
        middle = ngx_queue_next(middle);
        // next走第一步
        next = ngx_queue_next(next);
        // 判断next是否到末尾
        if (next == ngx_queue_last(queue)) {
            return middle;
        }
        // next再走一步，并判断是否到末尾
        next = ngx_queue_next(next);

        if (next == ngx_queue_last(queue)) {
            return middle;
        }
    }
}
```

### 对链表排序：ngx_queue_sort

按照指定排序方式对链表排序：采用稳定的插入排序算法

```c
void
ngx_queue_sort(ngx_queue_t *queue,
    ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *))
{
    ngx_queue_t  *q, *prev, *next;

    q = ngx_queue_head(queue);

    if (q == ngx_queue_last(queue)) {
        return;
    }
    // 遍历链表
    for (q = ngx_queue_next(q); q != ngx_queue_sentinel(queue); q = next) {

        prev = ngx_queue_prev(q);
        // 记录下一个要遍历的节点
        next = ngx_queue_next(q);
        // 先移出节点q
        ngx_queue_remove(q);

        do {
            // 找到在q前面第一个无序的节点
            if (cmp(prev, q) <= 0) {
                break;
            }

            prev = ngx_queue_prev(prev);

        } while (prev != ngx_queue_sentinel(queue));
        // 在第一个无序节点后添加q
        ngx_queue_insert_after(prev, q);
    }
}
```

## 红黑树ngx_rbtree_node_t

关于红黑树具体介绍及代码实现可以参考如下文档：[红黑树](http://www.yinkuiwang.cn/2019/06/13/%E7%BA%A2%E9%BB%91%E6%A0%91/)。

### 数据结构

```c
typedef ngx_uint_t  ngx_rbtree_key_t;
typedef ngx_int_t   ngx_rbtree_key_int_t;


typedef struct ngx_rbtree_node_s  ngx_rbtree_node_t;
// 节点类
struct ngx_rbtree_node_s {
    // 无符合整型关键字，排序依据
    ngx_rbtree_key_t       key;
    // 左子树
    ngx_rbtree_node_t     *left;
    // 右子树
    ngx_rbtree_node_t     *right;
    // 父节点
    ngx_rbtree_node_t     *parent;
    // 节点颜色 0表示红色，1表示黑色
    u_char                 color;
    // 仅一个字节的节点数据。由于表示空间过小，很少使用
    u_char                 data;
};


typedef struct ngx_rbtree_s  ngx_rbtree_t;
// 为解决不同节点含有相同关键字的元素冲突问题，红黑树设置了ngx_rbtree_insert_pt指针来灵活地添加冲突元素
typedef void (*ngx_rbtree_insert_pt) (ngx_rbtree_node_t *root,
    ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel);
// 红黑树容器
struct ngx_rbtree_s {
    // 根节点
    ngx_rbtree_node_t     *root;
    // 哨兵节点
    ngx_rbtree_node_t     *sentinel;
    // 指示红黑树添加元素的指针函数，其决定在添加新节点时的行为是替换还是新增
    ngx_rbtree_insert_pt   insert;
};
```

对应节点来说，与之前的结构类似，存储数据要依托于ngx_rbtree_node_s结构，因此可以自定义节点元素，但是必须包含ngx_rbtree_node_s结构，以使得方便结构体强制转换为ngx_rbtree_node_s结构。例如：

```c
typedef struct {
    ngx_rbtree_node_t node;
    ngx_uint_t num;
}
```

### 红黑树节点提供的方法

#### 设置节点颜色

```
#define ngx_rbt_red(node)               ((node)->color = 1)
#define ngx_rbt_black(node)             ((node)->color = 0)
```

#### 判断节点颜色

```
#define ngx_rbt_is_red(node)            ((node)->color)
#define ngx_rbt_is_black(node)          (!ngx_rbt_is_red(node))
```

#### 拷贝节点颜色

```
#define ngx_rbt_copy_color(n1, n2)      (n1->color = n2->color)
```

将n1节点变更为与n2一样。

#### 初始化哨兵节点

```
#define ngx_rbtree_sentinel_init(node)  ngx_rbt_black(node)
```

哨兵节点为黑色节点，因此设置节点为黑色即可。

#### 查找子树中最小节点

```c
static ngx_inline ngx_rbtree_node_t *
ngx_rbtree_min(ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel)
{
    while (node->left != sentinel) {
        node = node->left;
    }

    return node;
}
```

以key为关键字，查找最小的一个节点。

### 红黑树容器提供的方法

#### 初始化红黑树ngx_rbtree_init

```
#define ngx_rbtree_init(tree, s, i)                                           \
    ngx_rbtree_sentinel_init(s);                                              \
    (tree)->root = s;                                                         \
    (tree)->sentinel = s;                                                     \
    (tree)->insert = i
```

参数tree是红黑树容器指针，s是哨兵节点指针，i为ngx_rbtree_insert_pt类型的节点添加方法。

#### 寻找下一个元素ngx_rbtree_next

```c
// 在tree容器中，找到第一个比node大的元素
ngx_rbtree_node_t *
ngx_rbtree_next(ngx_rbtree_t *tree, ngx_rbtree_node_t *node)
{
    ngx_rbtree_node_t  *root, *sentinel, *parent;

    sentinel = tree->sentinel;
    // node节点如果存在又子树，则比node节点大的第一个元素为其右子树的最小值
    if (node->right != sentinel) {
        return ngx_rbtree_min(node->right, sentinel);
    }
    // 如果node节点不存在右子树，则向上查找其父辈节点，找到第一个使node节点处于其左子树的父节点。如果没有，则说明node自身为最大的节点，不存在下一个节点
    root = tree->root;

    for ( ;; ) {
        parent = node->parent;

        if (node == root) {
            return NULL;
        }

        if (node == parent->left) {
            return parent;
        }

        node = parent;
    }
}
```

#### 旋转节点

```c
// 左旋
static ngx_inline void
ngx_rbtree_left_rotate(ngx_rbtree_node_t **root, ngx_rbtree_node_t *sentinel,
    ngx_rbtree_node_t *node)
{
    ngx_rbtree_node_t  *temp;

    temp = node->right;
    node->right = temp->left;

    if (temp->left != sentinel) {
        temp->left->parent = node;
    }

    temp->parent = node->parent;

    if (node == *root) {
        *root = temp;

    } else if (node == node->parent->left) {
        node->parent->left = temp;

    } else {
        node->parent->right = temp;
    }

    temp->left = node;
    node->parent = temp;
}

// 右旋
static ngx_inline void
ngx_rbtree_right_rotate(ngx_rbtree_node_t **root, ngx_rbtree_node_t *sentinel,
    ngx_rbtree_node_t *node)
{
    ngx_rbtree_node_t  *temp;

    temp = node->left;
    node->left = temp->right;

    if (temp->right != sentinel) {
        temp->right->parent = node;
    }

    temp->parent = node->parent;

    if (node == *root) {
        *root = temp;

    } else if (node == node->parent->right) {
        node->parent->right = temp;

    } else {
        node->parent->left = temp;
    }

    temp->right = node;
    node->parent = temp;
}
```

关于节点旋转，这里不详细介绍，具体参考文档：[红黑树](http://www.yinkuiwang.cn/2019/06/13/%E7%BA%A2%E9%BB%91%E6%A0%91/)。

#### 添加节点ngx_rbtree_insert

向红黑树容器中增加节点：

```c
void
ngx_rbtree_insert(ngx_rbtree_t *tree, ngx_rbtree_node_t *node)
{
    ngx_rbtree_node_t  **root, *temp, *sentinel;

    /* a binary tree insert */

    root = &tree->root;
    sentinel = tree->sentinel;
    // 如果红黑树为空，则直接设置root节点即可
    if (*root == sentinel) {
        node->parent = NULL;
        node->left = sentinel;
        node->right = sentinel;
        ngx_rbt_black(node);
        *root = node;

        return;
    }
    // 否则，调用容器的ngx_rbtree_insert_pt函数，向其中增加节点
    tree->insert(*root, node, sentinel);

    /* re-balance tree */
    // 增加节点后导致红黑树无法满足原性质，进行调整。
    while (node != *root && ngx_rbt_is_red(node->parent)) {

        if (node->parent == node->parent->parent->left) {
            temp = node->parent->parent->right;

            if (ngx_rbt_is_red(temp)) {
                ngx_rbt_black(node->parent);
                ngx_rbt_black(temp);
                ngx_rbt_red(node->parent->parent);
                node = node->parent->parent;

            } else {
                if (node == node->parent->right) {
                    node = node->parent;
                    ngx_rbtree_left_rotate(root, sentinel, node);
                }

                ngx_rbt_black(node->parent);
                ngx_rbt_red(node->parent->parent);
                ngx_rbtree_right_rotate(root, sentinel, node->parent->parent);
            }

        } else {
            temp = node->parent->parent->left;

            if (ngx_rbt_is_red(temp)) {
                ngx_rbt_black(node->parent);
                ngx_rbt_black(temp);
                ngx_rbt_red(node->parent->parent);
                node = node->parent->parent;

            } else {
                if (node == node->parent->left) {
                    node = node->parent;
                    ngx_rbtree_right_rotate(root, sentinel, node);
                }

                ngx_rbt_black(node->parent);
                ngx_rbt_red(node->parent->parent);
                ngx_rbtree_left_rotate(root, sentinel, node->parent->parent);
            }
        }
    }

    ngx_rbt_black(*root);
}
```

nginx提供了三种ngx_rbtree_insert_pt方法来增加元素，后续会详细介绍，关于如果重新平衡二叉树，也参考文档即可：[红黑树](http://www.yinkuiwang.cn/2019/06/13/%E7%BA%A2%E9%BB%91%E6%A0%91/)。

#### 删除元素

```c
void
ngx_rbtree_delete(ngx_rbtree_t *tree, ngx_rbtree_node_t *node)
{
    ngx_uint_t           red;
    ngx_rbtree_node_t  **root, *sentinel, *subst, *temp, *w;

    /* a binary tree delete */

    root = &tree->root;
    sentinel = tree->sentinel;

    if (node->left == sentinel) {
        temp = node->right;
        subst = node;

    } else if (node->right == sentinel) {
        temp = node->left;
        subst = node;

    } else {
        subst = ngx_rbtree_min(node->right, sentinel);
        temp = subst->right;
    }

    if (subst == *root) {
        *root = temp;
        ngx_rbt_black(temp);

        /* DEBUG stuff */
        node->left = NULL;
        node->right = NULL;
        node->parent = NULL;
        node->key = 0;

        return;
    }

    red = ngx_rbt_is_red(subst);

    if (subst == subst->parent->left) {
        subst->parent->left = temp;

    } else {
        subst->parent->right = temp;
    }

    if (subst == node) {

        temp->parent = subst->parent;

    } else {

        if (subst->parent == node) {
            temp->parent = subst;

        } else {
            temp->parent = subst->parent;
        }

        subst->left = node->left;
        subst->right = node->right;
        subst->parent = node->parent;
        ngx_rbt_copy_color(subst, node);

        if (node == *root) {
            *root = subst;

        } else {
            if (node == node->parent->left) {
                node->parent->left = subst;
            } else {
                node->parent->right = subst;
            }
        }

        if (subst->left != sentinel) {
            subst->left->parent = subst;
        }

        if (subst->right != sentinel) {
            subst->right->parent = subst;
        }
    }

    /* DEBUG stuff */
    node->left = NULL;
    node->right = NULL;
    node->parent = NULL;
    node->key = 0;

    if (red) {
        return;
    }

    /* a delete fixup */

    while (temp != *root && ngx_rbt_is_black(temp)) {

        if (temp == temp->parent->left) {
            w = temp->parent->right;

            if (ngx_rbt_is_red(w)) {
                ngx_rbt_black(w);
                ngx_rbt_red(temp->parent);
                ngx_rbtree_left_rotate(root, sentinel, temp->parent);
                w = temp->parent->right;
            }

            if (ngx_rbt_is_black(w->left) && ngx_rbt_is_black(w->right)) {
                ngx_rbt_red(w);
                temp = temp->parent;

            } else {
                if (ngx_rbt_is_black(w->right)) {
                    ngx_rbt_black(w->left);
                    ngx_rbt_red(w);
                    ngx_rbtree_right_rotate(root, sentinel, w);
                    w = temp->parent->right;
                }

                ngx_rbt_copy_color(w, temp->parent);
                ngx_rbt_black(temp->parent);
                ngx_rbt_black(w->right);
                ngx_rbtree_left_rotate(root, sentinel, temp->parent);
                temp = *root;
            }

        } else {
            w = temp->parent->left;

            if (ngx_rbt_is_red(w)) {
                ngx_rbt_black(w);
                ngx_rbt_red(temp->parent);
                ngx_rbtree_right_rotate(root, sentinel, temp->parent);
                w = temp->parent->left;
            }

            if (ngx_rbt_is_black(w->left) && ngx_rbt_is_black(w->right)) {
                ngx_rbt_red(w);
                temp = temp->parent;

            } else {
                if (ngx_rbt_is_black(w->left)) {
                    ngx_rbt_black(w->right);
                    ngx_rbt_red(w);
                    ngx_rbtree_left_rotate(root, sentinel, w);
                    w = temp->parent->left;
                }

                ngx_rbt_copy_color(w, temp->parent);
                ngx_rbt_black(temp->parent);
                ngx_rbt_black(w->left);
                ngx_rbtree_right_rotate(root, sentinel, temp->parent);
                temp = *root;
            }
        }
    }

    ngx_rbt_black(temp);
}
```

删除元素这里也不过多介绍，阅读文档即可。

#### 架构提供的三个ngx_rbtree_insert_pt增加节点函数。

##### ngx_rbtree_insert_value

使用场景：向红黑树中增加数据节点，每个数据节点的关键字都是唯一的，不存在同一个关键字有多个节点的情况。

逻辑如下：

```c
void
ngx_rbtree_insert_value(ngx_rbtree_node_t *temp, ngx_rbtree_node_t *node,
    ngx_rbtree_node_t *sentinel)
{
    ngx_rbtree_node_t  **p;
    // 自订向下查找，遇到比自己的key大的节点就进入左子树继续查找，遇到比自己小的节点就到右子树继续查找，直到哨兵节点，决定添加位置。
    for ( ;; ) {

        p = (node->key < temp->key) ? &temp->left : &temp->right;

        if (*p == sentinel) {
            break;
        }

        temp = *p;
    }

    *p = node;
    // 添加节点，设置节点为红色
    node->parent = temp;
    node->left = sentinel;
    node->right = sentinel;
    ngx_rbt_red(node);
}
```

##### ngx_rbtree_insert_timer_value

该函数向红黑树添加数据节点，每个节点的关键字表示时间或者时间差。因此其中的key可能为负数。这是并不关心是否有重复的key。其逻辑如下：

```c
void
ngx_rbtree_insert_timer_value(ngx_rbtree_node_t *temp, ngx_rbtree_node_t *node,
    ngx_rbtree_node_t *sentinel)
{
    ngx_rbtree_node_t  **p;

    for ( ;; ) {

        /*
         * Timer values
         * 1) are spread in small range, usually several minutes,
         * 2) and overflow each 49 days, if milliseconds are stored in 32 bits.
         * The comparison takes into account that overflow.
         */

        /*  node->key < temp->key */

        p = ((ngx_rbtree_key_int_t) (node->key - temp->key) < 0)
            ? &temp->left : &temp->right;

        if (*p == sentinel) {
            break;
        }

        temp = *p;
    }

    *p = node;
    node->parent = temp;
    node->left = sentinel;
    node->right = sentinel;
    ngx_rbt_red(node);
}
```

与ngx_rbtree_insert_value逻辑一致，只是做了一个ngx_rbtree_key_int_t的强制类型转换，支持负数。

##### ngx_str_rbtree_insert_value

向红黑树中增加节点，每个数据节点的关键字可以不唯一，但是以字符串作为唯一标识，存放在ngx_str_node_t的str中。ngx_str_node_t定义如下：

```c
typedef struct {
    ngx_rbtree_node_t         node;
    ngx_str_t                 str;
} ngx_str_node_t;
```

对应的插入方法为：

```c
void
ngx_str_rbtree_insert_value(ngx_rbtree_node_t *temp,
    ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel)
{
    ngx_str_node_t      *n, *t;
    ngx_rbtree_node_t  **p;

    for ( ;; ) {

        n = (ngx_str_node_t *) node;
        t = (ngx_str_node_t *) temp;

        if (node->key != temp->key) {

            p = (node->key < temp->key) ? &temp->left : &temp->right;

        } else if (n->str.len != t->str.len) {

            p = (n->str.len < t->str.len) ? &temp->left : &temp->right;

        } else {
            p = (ngx_memcmp(n->str.data, t->str.data, n->str.len) < 0)
                 ? &temp->left : &temp->right;
        }

        if (*p == sentinel) {
            break;
        }

        temp = *p;
    }

    *p = node;
    node->parent = temp;
    node->left = sentinel;
    node->right = sentinel;
    ngx_rbt_red(node);
}
```

与前两个类似，但是增加了对str的比较。

# Nginx特殊技巧

## ngx_align 值对齐宏

ngx_align 为nginx中的一个值对齐宏。主要在需要内存申请的地方使用，为了减少在不同的 cache line 中内存而生。

```c
// d 为需要对齐的
// a 为对齐宽度，必须为 2 的幂
// 返回对齐值
#define ngx_align(d, a)     (((d) + (a - 1)) & ~(a - 1))
```

原理简单，利用 `~(a - 1)` 的低位全为 0。在与 `~(a - 1)` 做 `&` 操作时，低位的1被丢弃，就得到了a倍数的值（对齐）。

如果使用原始值直接与 `~(a - 1)` 做 `&` 操作，那么得到的对齐值是会小于等于原始值的，这样会造成内存重叠，而期望的对齐值是一个大于等于原始值的，所以需要加上一个数来补上至对齐值这中间的差，这个数为 `(a - 1)` ，选择这个数的原因是 `(a - 1) & ~(a - 1)` 的结果为0。

该操作含义为：取大于d且为a整数倍的第一个数。

## ngx_align_ptr内存对其

```c
#define ngx_align_ptr(p, a)                                                   \
    (u_char *) (((uintptr_t) (p) + ((uintptr_t) a - 1)) & ~((uintptr_t) a - 1))
```

这里和上面都存储对其类似，a一般是cache line大小。对其后，指针指向cache line的整数倍的地方，加快读取速度。具体参考下面文章。

https://oopschen.github.io/posts/2013/cpu-cacheline/

## 指针最后两位一定是0

字节对齐和系统有关，也和编译器有关，具体到底是几字节对齐和本问题关系不大。主要是因为无论如何对齐，都是字节对齐，而不是bit对齐。也就是说指针的起始存放地址只可能是8的倍数，比如0x00,0x08,0x10,转换成二进制后三位永远是0。不可能出现0x02到0x22作为一个四字节的指针。

ps1：有些资料中将之描述为指针的后2位永远是0，猜想是汇编语言中好像有的语句可以把4bit看作一个基本的操作单位，所以只能保证指针的后2位永远是0。

ps2：是否可以申请一个数组，然后对其中的位进行操作，使之出现一个类似0x02到0x22作为一个四字节指针的情况，将这个畸形的指针传入指针操作API中会导致什么后果，这个问题还有待考证。

出处：https://www.zhihu.com/question/40636241/answer/311889614

## 内联汇编

内联汇编语言可以直接操作硬件。可以用来在nginx源码中实现对整数的原子操作。

使用GCC编译器在C语言中嵌入汇编语言的方式是使用\__asm__关键字，语法如下：

```c
__asm__ volatile ( 汇编语句部分
     : 输出部分    /*可选*/
     : 输入部分    /*可选*/
     : 破坏描述部分 /*可选*/
）;
```

加入volatile关键字用于限制GCC编译器对这段代码做优化。

内联汇编语言包含四部分：

1. 汇编语句部分

   引号中所包含的汇编语句可以直接用占位符%来引用C语言中的变量（最多10个，%0-%9）。

2. 输出部分

   将寄存器中的值设置到C语言的变量中

3. 输入部分

   将C语言中的变量设置到寄存器中。

4. 破坏描述部分

   通知编译器使用了哪些寄存器、内存。

以如下语句举例：

```C
static ngx_inline ngx_atomic_uint_t
ngx_atomic_cmp_set(ngx_atomic_t *lock, ngx_atomic_uint_t old,
    ngx_atomic_uint_t set)
{
    u_char  res;

    __asm__ volatile (

    "    lock                "
    "    cmpxchgl  %3, %1;   "
    "    sete      %0;       "

    : "=a" (res) : "m" (*lock), "a" (old), "r" (set) : "cc", "memory");

    return res;
}

```

先看输入部分："m" (\*lock)表示*lock变量是在内存中，操作\*lock直接通过内存（不使用寄存器处理）,而"a" (old)表示把old变量写入eax寄存器中，"r" (set)表示把set变量写入通用寄存器中。这些都是为cmpxchgl做准备。

再来看汇编语句部分："lock"表示在多核架构上首先锁住总线。

cmpxchgl语句含义为如下代码表示：

```
/*
 * "cmpxchgl  r, [m]":
 *
 *     if (eax == [m]) {
 *         zf = 1;
 *         [m] = r;
 *     } else {
 *         zf = 0;
 *         eax = [m];
 *     }
 *
 */
```

即判断寄存器中的值是否等于[m]，如果相等，则设置[m]为r，并且设置sf为1。否则，设置zf为0，并且设置寄存器中值为[m]。

在语句cmpxchgl  %3, %1;中，寄存器变量为old。%3表示set，%1表示*loct。先比较\*lock是否等于old，如果相等设置\*lock为set。并进行zf设置。

"sete      %0;"表示设置zf值到寄存器变量中。

返回部分"=a" (res)将寄存器中值写入res变量中，返回。

# Nginx架构及启动流程

## Nginx的架构设计

### 优秀的模块化设计

高度模块化的设计是Nginx的架构基础。在nginx中，除了少量的核心代码，其他一切皆为模块。其具有如下特点：

1. 高度抽象的模块接口

   所有模块都遵循同样的ngx_module_t接口设计规范，减少了系统变数。

2. 模块接口非常简单，具有高度灵活性

   模块的基本接口nginx_module_t足够简单，只设计模块的初始化、退出以及对配置项的处理，同时带来了足够的灵活性。

   [![2pk4cn.md.png](https://z3.ax1x.com/2021/05/26/2pk4cn.md.png)](https://imgtu.com/i/2pk4cn)

<center/>图8-1</center>
如图所示，nginx_module_t结构体作为所有模块的通用接口，其只定义了`init_master` `init_module` `init_process` `init_thread` `exit_thread` `exit_process` `exit_master`这七个回调方法，他们负责模块的初始化与退出，同时权限非常高，可以处理核心结构体nginx_cycle_t。

#### nginx_module_t类

nginx_module_t结构如下：

```c
struct ngx_module_s {
  /* 下面的ctx_index、index、spare0、spare1、spare2、spare3、version不需要在定义时赋值，可以用Nginx准备好的宏NGINX_MODULE_V1来定义，其已经定义好了这7个值。 #define NGINX_MODULE_V1 0,0,0,0,0,0,1 */
  
  
  /* 对于一类模块（同一type）而言，ctx_index表示当前模块在这类模块中的序号，改成员变量通常由管理这类模块的一个nginx核心模块设置的，对于http而言，由核心模块nginx_http_module设置。ctx_index十分重要，nginx模块化依赖各个模块顺序，其既表达优先级，也用于表面每个模块的位置*/
    ngx_uint_t            ctx_index;
  
  /* index表示当前模块在ngx_modules数组中序号，即为当前模块在所有模块中的序号，nginx启动时，依据ngx_modules数组设置该值*/
    ngx_uint_t            index;

    char                 *name;

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;
    const char           *signature;

  /* 指向一类模块的上下文结构体，例如在http模块中，ctx指向ngx_http_module_t结构体*/
    void                 *ctx;
  
  /* commands将处理nginx.conf中的配置项 */
    ngx_command_t        *commands;
    ngx_uint_t            type;

  /* master进程启动时回调init_master，但目前未使用，设置为NULL*/
    ngx_int_t           (*init_master)(ngx_log_t *log);

  /* 初始化所有模块是调用init_module，在master/worker模式下，该阶段在启动worker子进程前完成，具体执行位置在ngx_init_cycle中，执行完成配置解析，并打开监听端口号后，执行每个模块的init_module */
    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

  /*init_process在正常服务前被调用，在master/worker模式下，多个worker子进程已产生，在每个woker进程的初始化过程中会调用所有模块的该函数 */
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
  
  /*exit_process在服务停止前被调用，在master/worker模式下，worker进程退出前调用 */
    void                (*exit_process)(ngx_cycle_t *cycle);

  /*exit_master在master进程退出前调用*/
    void                (*exit_master)(ngx_cycle_t *cycle);

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
```

ctx是void指针，可以指向任何数据，这改模块提供了极大的灵活性。

3. 配置模块的设计

   ngx_module_t接口中type类型指名了nginx在设计模块时定义模块类型，允许专注于不同领域的模块按照类型来区别。配置类型模块是唯一一个只有一个模块的模块类型。配置模块的类型叫做NGX_CONF_MODULE，其仅有的模块为ngx_conf_module，其为底层模块，指导所有模块以配置项为核心来提供功能。

4. 核心模块的简单化

5. 多层次，多类别的模块设计

   所以模块间是分层次、分类别的，官方Nginx共用5大类型模块：核心模块、配置模块、时间模块、HTTP模块、mail模块。虽然都具备相同的ngx_module_t接口，但在请求处理中的层级并不相同。

   [![2ELokD.png](https://z3.ax1x.com/2021/05/30/2ELokD.png)](https://imgtu.com/i/2ELokD)

<center/>图8-2</center>
如上图，配置模块和核心模块由Nginx的框架代码定义，配置模块是所有模块的基础，其实现了最基础的配置项解析功能（解析nginx.conf）。Nginx框架还会调用核心模块，但其他3种模块都不会与框架产生直接关系。事件模块、HTTP模块、mail模块的共性为：它们在核心模块中各有一个模块，作为其代言人，并在同类模块中有一个作为核心业务与管理功能的模块。



### 事件驱动框架

事件驱动指：由一些事件发送源来产生事件，由一个或多个时间收集器来收集、分发时间，然后许多事件处理器会注册自己感兴趣的，同时会消费这些事件。

对于nginx来说，一般会由网卡、磁盘产生事件，事件模块负责收集、分发操作，所有模块都可能是消费者，其首选向事件模块注册感兴趣的事件类型，这样，有事件产生时，事件模块会把事件分发到响应模块中进行处理。

传统Web服务器，采用的事件驱动往往局限于在TCP连接、关闭事件上，一个连接建立后，在其关闭前所有操作都不再是事件驱动，此时会退化为按需执行每个操作的批处理模式，这样，每个请求在连接后都将始终占有系统资源，直到连接关闭才会释放。

Nginx则不然，他不会使用进程或线程作为事件消费者，所谓事件消费者只能是某个模块。只有事件收集器、分发器才有资格占用进程资源，它们会在分发某个事件时调用事件消费模块使用当前占用进程资源。

[![2EXtrq.png](https://z3.ax1x.com/2021/05/30/2EXtrq.png)](https://imgtu.com/i/2EXtrq)

<center/>图8-4</center>
如上图，在事件收集、分发者进程的一次处理过程中，5个事件按序被收集后，将开始使用当前进程分发事件，从而调用响应的事件消费者模块来处理事件。事件消费者只是被事件分发者进程短期调用而已。

### 请求的多阶段异步处理

请求的多阶段异步处理是指：把一个请求过程按照事件的触发方式划分为多个阶段，每个阶段都可以由事件收集、分发来触发。

请求的多阶段异步处理优势：这种设计配合事件驱动架构，将极地提高网络性能，同时使得每个进程都能全力运转，不会或者尽量少的出现进程休眠状况。

划分请求阶段原则为：找到请求处理流程中阻塞方法，在阻塞代码段上按照下面四个方法来划分阶段：

1. 将阻塞进程的方法按照相关的触发事件分解为两个阶段：

   一个本身可能导致进程休眠的方法或系统调用，一般可以分解为多个更小的方法或者系统调用，这些调用间可以通过事件触发关联起来。大部分情况，一个阻塞的方法调用可以划分为两个阶段：第一阶段为，将阻塞方法改为非阻塞方法，并将进程归还给事件分发器；第二阶段，用于处理非阻塞方法最终返回结果，这里的返回结果就是第二阶段触发事件。

   例如使用send调用时，如果使用阻塞socket句柄，send向内核发送数据后将使当前进程休眠，直到成功发出数据。可以将send调用划分为两个阶段：使用非阻塞socket句柄，发送后进程不休眠，再将socket句柄加入事件收集器中就可以等待相应事件触发下一阶段，send发送数据被对方接收后会触发send结果返回阶段。

2. 将阻塞方法调用按照时间分解为多个阶段的方法调用

   系统中事件收集器、分发器并非可以处理任何事件。例如读取文件调用（非异步I/O），如果我们读取10MB文件，这些文件在磁盘块未必是连续的，此时可能需要多次驱动硬盘寻址，寻址时，进程多半会休眠或等待。如果内核不支持异步I/O时（或未打开），就不能采用第一个方案。此时可以分解读取文件调用：把10MB的文件划分为1000份，每次读取10KB。每次读取10KB的时间是可控的，意味着该事件不会占用进程太久，整个系统可以及时处理其他请求。

   在读取0KB-10KB后如何进入10KB-20KB呢，可以有多种方式：如读取完10KB要使用网络进行发送，可以由网络事件进行触发。或者没有网络事件，可以设置一个简单的定时器。

3. 在"无所事事"且必须等待系统响应时，使用定时器划分阶段

   有时阻塞代码可以是这样的：进行某个无阻塞的系统调用后，必须通过持续检查标志位来确定是否继续向下执行，当标志位没有获得满足时就循环地检查。此时，应该使用定时器来代替循环检查标志，这样定时器事件发送时就会先检查标志，如果标志不满足，就立即归还进程控制权，同时继续加入期望的下一个定时器事件。

4. 如果阻塞方法完全无法划分，则必须使用独立的进程执行这个阻塞方法

   如果某个方法的调用时可能导致进程休眠，或者占用进程时间过长，开始又无法将该方法分级为非阻塞的方法，那么，这与事件驱动框架是相违背的。通常是由于方法实现者未开放非阻塞接口所导致，这时必须通过产生新的进程或者指定某个非事件分发者进程来执行阻塞方法，并在阻塞方法执行完毕时向事件收集、分发者进程发送事件通知继续执行。因此，至少要拆分为两个阶段：阻塞方法执行前阶段、阻塞方法执行后阶段，阻塞方法由单独的进程取调度，并在方法返回后发送事件通知。一旦出现这种情况，应该考虑这样的事件消费者是否合理，有没有必要使用这种违反事件驱动的方式来解决阻塞问题。

### 管理进程、多工作进程设计

Nginx采用一个master管理进程，多个worker工作进程的设计方式，如下图：

[![2npYee.png](https://z3.ax1x.com/2021/06/01/2npYee.png)](https://imgtu.com/i/2npYee)

<center/>图8-5</center>
该设计的优点为：

1. 利用多核系统的并发处理能力

2. 负载均衡

   每个worker工作进程通过进程间通信来实现负载均衡，即一个请求到达时会更容易地被分配到负载较轻的进程中。

3. 管理进程负责监控工作进程的状态，并负责其行为

   管理进程不会占用太多系统资源，其只用来启动、停止、建库或使用其他行为来控制工作进程。首选，这提高了系统的可靠性，当工作进程出现问题时，管理进程可以启动新的工作进程来避免系统性能下降。其次，管理进程支持nginx服务运行中的程序升级、配置项的修改等操作。这种设计使得动态可扩展性、动态定制性、动态可进化性较容易实现。

### 内存池的设计

为了避免出现内存碎片、减少向操作系统申请内存的次数、降低各个模块的开发复杂度，Nginx设计了简单的内存池。内存池没有很复杂的功能：其通常不负责回收内存池中已经分配的内存。内存池最大的优点在于：把多次向系统申请内存的操作整合到一次，这大大减少了CPU资源消耗，同时减少了内存碎片。

## Nginx框架中的核心结构体ngx_cycle_t

Nginx核心的框架代码围绕ngx_cycle_t结构体展开。

### ngx_listening_t结构体

作为web服务器，nginx首先需要监听端口并处理其中的网络事件。ngx_cycle_t对象有一个动态数组成员listening，其每个元素都是ngx_listening_t结构体，每个ngx_listening_t结构体代表nginx服务器监听的一个端口。

```c
typedef struct ngx_listening_s  ngx_listening_t;

struct ngx_listening_s {
    ngx_socket_t        fd;

    struct sockaddr    *sockaddr;
    socklen_t           socklen;    /* size of sockaddr */
    size_t              addr_text_max_len;
    ngx_str_t           addr_text;

    int                 type;

  /* TCP实现监听时的backlog队列，它表示允许正在通过三次握手建立TCP连接但还没有任何进程开始处理的连接的最大个数 */
    int                 backlog;
  /* 内核中对该套接字的接收缓冲区大小 */
    int                 rcvbuf;
  /* 内核中对该套接字的发送缓冲区大小 */
    int                 sndbuf;
#if (NGX_HAVE_KEEPALIVE_TUNABLE)
    int                 keepidle;
    int                 keepintvl;
    int                 keepcnt;
#endif

    /* handler of accepted connection */
    /* 当新的TCP连接成功建立后的处理方法 */
    ngx_connection_handler_pt   handler;

    void               *servers;  /* array of ngx_http_in_addr_t, for example */

    /* log和kogp都是可用的日志对象的指针 */
    ngx_log_t           log;
    ngx_log_t          *logp;

    /* 如果为新的TCP连接创建内存池，则内存池从初始大小 */
    size_t              pool_size;
    /* should be here because of the AcceptEx() preread */
    size_t              post_accept_buffer_size;
    /* should be here because of the deferred accept */
    /* TCP_DEFER_ACCEPT选项将在建立TCP连接成功并且接受到用户请求数据后，才向对监听套接字感兴趣的进程发送事件通知，而连接成功后，如果post_accept_timeout秒后仍未收到用户数据，则内核直接丢弃连接 */
    ngx_msec_t          post_accept_timeout;

    /* 前一个ngx_listening_t的指针，多个ngx_listening_t结构体之间由previous指针组成单链表 */
    ngx_listening_t    *previous;
    
    /* 当前监听句柄对应着的ngx_connection_t结构体，为一个单链表 */
    ngx_connection_t   *connection;

    ngx_rbtree_t        rbtree;
    ngx_rbtree_node_t   sentinel;

    ngx_uint_t          worker;

    /* 标志位，为1表示当前监听句柄有效，且执行ngx_init_cycle时不关闭监听端口，为0正常关闭。该标志由架构代码自动设置 */
    unsigned            open:1;
    /* 为1表示已有ngx_cycle_t来初始化新的ngx_cycle_t结构体时，不关闭原先打开的监听端口，对运行中升级程序很有效。为0时，表示正常关闭曾经打开的监听端口。该标志由架构代码自动设置 */
    unsigned            remain:1;
    /* 1表示跳过设置当前ngx_listening_t结构体中套接字，0正常初始化套接字。 该标志由架构代码自动设置*/
    unsigned            ignore:1;

    unsigned            bound:1;       /* already bound */
    /* 当前监听句柄是否来着于前一个进程 */
    unsigned            inherited:1;   /* inherited from previous process */
    unsigned            nonblocking_accept:1;
    /* （书中说是当前套接字是否已监听），看代码似乎是说要重新执行listing函数来设置监听的第二个参数 */
    unsigned            listen:1;
    /* 套接字是否阻塞，目前无意义 */
    unsigned            nonblocking:1;
    unsigned            shared:1;    /* shared between threads or processes */
  
    /* 1时nginx将网络地址转变为字符串形式地址 */
    unsigned            addr_ntop:1;
    unsigned            wildcard:1;

#if (NGX_HAVE_INET6)
    unsigned            ipv6only:1;
#endif
    unsigned            reuseport:1;
    unsigned            add_reuseport:1;
    unsigned            keepalive:2;

    unsigned            deferred_accept:1;
    unsigned            delete_deferred:1;
    unsigned            add_deferred:1;
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
    char               *accept_filter;
#endif
#if (NGX_HAVE_SETFIB)
    int                 setfib;
#endif

#if (NGX_HAVE_TCP_FASTOPEN)
    int                 fastopen;
#endif

};
```

ngx_connection_handler_pt类型的handler成员表示在这个监听端口上成功建立新的tcp连接后，就会回调handler方法，其定义为：

```
typedef void (*ngx_connection_handler_pt)(ngx_connect_t *c);
```

### ngx_cycle_t结构体

首先来介绍一下ngx_cycle_t中的成员（其中connectins、read_events、write_events、files、free_connection成员与事件模块强相关，在事件模块中详细介绍）。

```c
struct ngx_cycle_s {
    /* 保存着所有模块（modules）存储配置项的结构体的指针。其首先是一个数组，每个数组成员又是一个指针，这个指针指向另一个存储着指针的数组 */
    void                  ****conf_ctx;
  
    /* 内存池 */
    ngx_pool_t               *pool;
    /* 在未执行ngx_init_cycle方法前，未确定日志输出位置时，暂时使用log，将内容输出到屏幕上，在执行了ngx_init_cycle方法后，确定日志输出位置时，会替换该值 */
    ngx_log_t                *log;
    /* 由nginx.conf配置文件读取到日志后，将开始初始化error_log日志文件，但log依然在向屏幕输出日志。这时会使用new_log对象暂时替代log日志，待初始化成功，使用new_log的地址替换log指针 */
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    /* 对于poll、rtsig的事件模块，会以有效文件句柄来预先建立这些ngx_connection_t结构体，以加速事件的收集分发，此时file保存了ngx_connection_t的指针数组 */
    ngx_connection_t        **files;
  
    /* 可用连接池，与free_connection_n配合使用 */
    ngx_connection_t         *free_connections;
    /* 可用连接池中连接总数 */
    ngx_uint_t                free_connection_n;

    ngx_module_t            **modules;
    ngx_uint_t                modules_n;
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    /* 双向链表容器，元素类型为ngx_connection_t，表示可重复使用连接队列 */
    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;

    /* 动态数组，每个数组元素中存储着ngx_listening_t成员，表示监听端口及相关参数 */
    ngx_array_t               listening;
  
    /* 动态数组容器，其存储着Nginx所有要操作的目录。如果目录不存在则会试图创建，创建失败将会导致nginx启动失败.比如配置了client_body_temp_path，此时就需要创建一个路径临时存放请求用户http包体*/
    ngx_array_t               paths;

    ngx_array_t               config_dump;
    ngx_rbtree_t              config_dump_rbtree;
    ngx_rbtree_node_t         config_dump_sentinel;
    
    /* 单链表容器，元素类型为ngx_open_file_t结构体，其表示nginx已经打开的所有文件 */
    ngx_list_t                open_files;
  
    /* 单链表容器，元素类型为ngx_shm_zone_y，每一个元素表示一块共享内存 */
    ngx_list_t                shared_memory;

    /* 当前进程中连接总数 */
    ngx_uint_t                connection_n;
    ngx_uint_t                files_n;

    /* 指向当前进程中所有链接对象，与connection_n配合使用 */ 
    ngx_connection_t         *connections;
    /* 指向当前进程中所有读事件 connection_n同时表示读事件总数*/ 
    ngx_event_t              *read_events;
    /* 指向当前进程中所有写事件 connection_n同时表示写事件总数*/ 
    ngx_event_t              *write_events;
    
    /* 旧的ngx_cycle_t对象引用上一个ngx_cycle_t对象的成员。在调用ngx_init_cycle方法前会建立一个临时ngx_cycle_t对象保存一些变量，再调用ngx_init_cycle方法时将旧的ngx_cycle_t传递进去，此时old_cycle将保存旧的ngx_cycle_t */
    ngx_cycle_t              *old_cycle;
    
    /* 配置文件相对按照目录的路径名 */
    ngx_str_t                 conf_file;
    /* nginx处理配置文件时需要特殊处理的在命令行携带的参数，一般是-g选项携带的参数 */
    ngx_str_t                 conf_param;
    /* nginx配置文件所在目录的路径 */
    ngx_str_t                 conf_prefix;
    /* nginx安装目录 */
    ngx_str_t                 prefix;
    /* 用于进程间同步的锁的名称 */
    ngx_str_t                 lock_file;
    /* gethostname获取到的主机名 */
    ngx_str_t                 hostname;
};
```



### ngx_cycle_t支持的方法

每个模块都可以通过init_module、init_process、exit_process、exit_master等方法操作进程的单独的ngx_cycle_t结构体。nginx框架关于ngx_cycle_t结构体方法如下：





| 方法名                                                       | 参数含义                                                     | 执行含义                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ngx_cycle_t *ngx_init_cycle(ngx_cycle_t *old_cycle)          | old_cycle表示临时的ngx_cycle_t指针，一般用来传递配置文件路径等参数。 | 返回初始化完成的结构体，该函数将会负责初始化ngx_cycle_t中的数据结构、解析配置文件、加载所有模块、打开监听端口、初始化进程间通讯方式等工作。失败返回null。 |
| ngx_init_t *ngx_process_options(ngx_cycle_t *cycle)          | 与上一个参数一致                                             | 用运行Nginx时可能携带的目录参数来初始化cycle，包括初始化运行目录、配置目录，并生成完整的nginx.conf配置文件路径 |
| ngx_init_t *ngx_add_inherited_sockets(ngx_cycle_t *cycle)    | cycle是当前进程的ngx_cycle_t结构体指针                       | 在不重启服务器升级时，老的nginx进程会通过环境变量NGINX来传递需要打开的监听端口，新的nginx进程会通过ngx_add_ingerited_sockets方法来使用已经打开的TCP监听端口 |
| ngx_int_t ngx_open_listening_sockets(ngx_cycle_t *cycle)     | cycle是当前进程的ngx_cycle_t结构体指针                       | 监听、绑定cycle中listening动态数组指定的相应端口             |
| void *ngx_configure_listening_sockets(ngx_cycle_t *cycle)    | cycle是当前进程的ngx_cycle_t结构体指针                       | 根据nginx.conf中的配置项设置已经监听的句柄                   |
| void ngx_close_listening_sockets(ngx_cycle_t *cycle)         | cycle是当前进程的ngx_cycle_t结构体指针                       | 关闭cycle中listening动态数组中已经监听的句柄                 |
| void *ngx_master_process_cycle(ngx_cycle_t *cycle)           | cycle是当前进程的ngx_cycle_t结构体指针                       | 进入master进程主循环                                         |
| void ngx_master_single_cycle(ngx_cycle_t *cycle)             | cycle是当前进程的ngx_cycle_t结构体指针                       | 进入单进程模式的工作循环                                     |
| void ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type) | cycle是当前进程的ngx_cycle_t结构体指针，n是启动进程数量，type是启动方式，其取值为如下5个；1）NGX_PROCESS_RESPAWN；2)NGX_PROCESS_NORESPAWN；3)NGX_PROCESS_JUST_SPAWN；4)NGX_PROCESS_JUST_RESPAWN；5)NGX_PROCESS_DEFACHED。type值影响ngx_process_t中respawn，detached，just_spawn标志位值 | 启动n个work子进程，并设置好每个子进程与父进程之间使用socketpair系统调用建立起来的socket句柄通信机制 |
| void ngx_start_cache_manger_processes(ngx_cycle_t *cycle, ngx_nint_t respawn) | cycle是当前进程的ngx_cycle_t结构体指针,respawn与ngx_start_worker_processes的type一致 | 根据是否使用文件缓存模块，即cycle中存储路径的动态数组中是否有路径的manage标志打开，来决定是否启动cache manage子进程，根据loader标志位来决定是否启动cache loader子进程 |
| void ngx_pass_open_channel(ngx_cycle_t *cycle, ngx_channel_t *ch) | cycle是当前进程的ngx_cycle_t结构体指针，ch是将要发送的信息   | 向所有已经打开的channel（通过socketpair生成的句柄进行通信）发送ch信息 |
| void ngx_single_worker_processes(ngx_cycle_t *cycle, int signo) | cycle是当前进程的ngx_cycle_t结构体指针，signo是信号          | 处理worker进程接受到的信号                                   |
| ngx_uint_t ngx_reap_children(ngx_cycle_t *cycle)             | cycle是当前进程的ngx_cycle_t结构体指针                       | 检查master进程的所有子进程，根据每个子进程的状态(ngx_process_t结构体中标志位)判断是否要启动子进程、更改pid文件等 |
| void ngx_master_process_exit(ngx_cycle_t *cycle)             | cycle是当前进程的ngx_cycle_t结构体指针                       | 退出master进程主循环                                         |
| void ngx_work_process_cycle(ngx_cycle_t *cycle, void *data)  | cycle是当前进程的ngx_cycle_t结构体指针,data目前还未使用，null | 进入worker进程主循环                                         |
| void ngx_work_process_init(ngx_cycle_t *cycle, ngx_uint_t priority) | cycle是当前进程的ngx_cycle_t结构体指针,priority是当前worker进程的优先级 | 进入worker进程主循环之前的初始化工作                         |
| void ngx_work_process_exit(ngx_cycle_t *cycle)               | cycle是当前进程的ngx_cycle_t结构体指针                       | 退出worker进程主循环                                         |
| void ngx_cache_manager_process_cycle(ngx_cycle_t *cycle, void *data) | cycle是当前进程的ngx_cycle_t结构体指针,data是传入的ngx_cache_manager_ctx_t结构体指针 | 执行缓存管理工作的循环方法。                                 |
| void ngx_process_events_and_timers(ngx_cycle_t *cycle)       | cycle是当前进程的ngx_cycle_t结构体指针                       | 使用事件管理模块处理截止到现在已经收集到的事件               |

### nginx启动时框架处理流程

[![21Zga9.png](https://z3.ax1x.com/2021/06/03/21Zga9.png)](https://imgtu.com/i/21Zga9)

<center/>图8-6</center>
## nginx启动过程详解

```c
int ngx_cdecl
main(int argc, char *const *argv)
{
    ngx_buf_t        *b;
    ngx_log_t        *log;
    ngx_uint_t        i;
    ngx_cycle_t      *cycle, init_cycle;
    ngx_conf_dump_t  *cd;
    ngx_core_conf_t  *ccf;

    ngx_debug_init();

    if (ngx_strerror_init() != NGX_OK) {
        return 1;
    }

    if (ngx_get_options(argc, argv) != NGX_OK) {
        return 1;
    }

    if (ngx_show_version) {
        ngx_show_version_info();

        if (!ngx_test_config) {
            return 0;
        }
    }

    /* TODO */ ngx_max_sockets = -1;

    ngx_time_init();

#if (NGX_PCRE)
    ngx_regex_init();
#endif
    // 获取进程id和父进程id
    ngx_pid = ngx_getpid();
    ngx_parent = ngx_getppid();

    log = ngx_log_init(ngx_prefix);
    if (log == NULL) {
        return 1;
    }

    /* STUB */
#if (NGX_OPENSSL)
    ngx_ssl_init(log);
#endif

    /*
     * init_cycle->log is required for signal handlers and
     * ngx_process_options()
     */

    ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;

    init_cycle.pool = ngx_create_pool(1024, log);
    if (init_cycle.pool == NULL) {
        return 1;
    }

    if (ngx_save_argv(&init_cycle, argc, argv) != NGX_OK) {
        return 1;
    }

    if (ngx_process_options(&init_cycle) != NGX_OK) {
        return 1;
    }

    if (ngx_os_init(log) != NGX_OK) {
        return 1;
    }

    /*
     * ngx_crc32_table_init() requires ngx_cacheline_size set in ngx_os_init()
     */

    if (ngx_crc32_table_init() != NGX_OK) {
        return 1;
    }

    /*
     * ngx_slab_sizes_init() requires ngx_pagesize set in ngx_os_init()
     */

    ngx_slab_sizes_init();

    if (ngx_add_inherited_sockets(&init_cycle) != NGX_OK) {
        return 1;
    }

    if (ngx_preinit_modules() != NGX_OK) {
        return 1;
    }

    cycle = ngx_init_cycle(&init_cycle);
    if (cycle == NULL) {
        if (ngx_test_config) {
            ngx_log_stderr(0, "configuration file %s test failed",
                           init_cycle.conf_file.data);
        }

        return 1;
    }

    if (ngx_test_config) {
        if (!ngx_quiet_mode) {
            ngx_log_stderr(0, "configuration file %s test is successful",
                           cycle->conf_file.data);
        }

        if (ngx_dump_config) {
            cd = cycle->config_dump.elts;

            for (i = 0; i < cycle->config_dump.nelts; i++) {

                ngx_write_stdout("# configuration file ");
                (void) ngx_write_fd(ngx_stdout, cd[i].name.data,
                                    cd[i].name.len);
                ngx_write_stdout(":" NGX_LINEFEED);

                b = cd[i].buffer;

                (void) ngx_write_fd(ngx_stdout, b->pos, b->last - b->pos);
                ngx_write_stdout(NGX_LINEFEED);
            }
        }

        return 0;
    }

    if (ngx_signal) {
        return ngx_signal_process(cycle, ngx_signal);
    }

    ngx_os_status(cycle->log);

    ngx_cycle = cycle;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    if (ccf->master && ngx_process == NGX_PROCESS_SINGLE) {
        ngx_process = NGX_PROCESS_MASTER;
    }

#if !(NGX_WIN32)

    if (ngx_init_signals(cycle->log) != NGX_OK) {
        return 1;
    }

    if (!ngx_inherited && ccf->daemon) {
        if (ngx_daemon(cycle->log) != NGX_OK) {
            return 1;
        }

        ngx_daemonized = 1;
    }

    if (ngx_inherited) {
        ngx_daemonized = 1;
    }

#endif

    if (ngx_create_pidfile(&ccf->pid, cycle->log) != NGX_OK) {
        return 1;
    }

    if (ngx_log_redirect_stderr(cycle) != NGX_OK) {
        return 1;
    }

    if (log->file->fd != ngx_stderr) {
        if (ngx_close_file(log->file->fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_close_file_n " built-in log failed");
        }
    }

    ngx_use_stderr = 0;

    if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);

    } else {
        ngx_master_process_cycle(cycle);
    }

    return 0;
}

```



### 解析启动参数

```c
if (ngx_get_options(argc, argv) != NGX_OK) {
        return 1;
    }
```

逐字符解析启动请求参数，根据解析参数设置全局变量。

```c
static ngx_int_t
ngx_get_options(int argc, char *const *argv)
{
    u_char     *p;
    ngx_int_t   i;

    for (i = 1; i < argc; i++) {

        p = (u_char *) argv[i];
        // 启动参数，必须使用-开始
        if (*p++ != '-') {
            ngx_log_stderr(0, "invalid option: \"%s\"", argv[i]);
            return NGX_ERROR;
        }
        
        while (*p) {
            // 检查参数
            switch (*p++) {

            case '?':
            case 'h':
                // 打印help信息标识
                ngx_show_version = 1;
                ngx_show_help = 1;
                break;

            case 'v':
                // 打印版本标识
                ngx_show_version = 1;
                break;

            case 'V':
                // 打印版本标识和显示配置编译阶段信息
                ngx_show_version = 1;
                ngx_show_configure = 1;
                break;

            case 't':
                // 测试配置正确性标识
                ngx_test_config = 1;
                break;

            case 'T':
                // 测速配置正确性标识，dump配置并输出标识
                ngx_test_config = 1;
                ngx_dump_config = 1;
                break;

            case 'q':
                // 测试配置选项时，使用-q使得error级别以下的信息不输出
                ngx_quiet_mode = 1;
                break;

            case 'p':
                // 另行指定nginx安装目录。决定了，error日志，conf配置和pid文件存储的默认位置
                if (*p) {
                    // 使用ngx_prefix存储值，-p直接接目录，如 -p/home/work/nginx
                    ngx_prefix = p;
                    goto next;
                }
                // -p /home/work/nginx
                if (argv[++i]) {
                    ngx_prefix = (u_char *) argv[i];
                    goto next;
                }

                ngx_log_stderr(0, "option \"-p\" requires directory name");
                return NGX_ERROR;

            case 'c':
                // 配置文件目录，使用ngx_conf_file存储
                if (*p) {
                    ngx_conf_file = p;
                    goto next;
                }

                if (argv[++i]) {
                    ngx_conf_file = (u_char *) argv[i];
                    goto next;
                }

                ngx_log_stderr(0, "option \"-c\" requires file name");
                return NGX_ERROR;

            case 'g':
                // 指定一些全局变量，使用ngx_conf_params存储，后续会被与解析配置方式一样进行解析，使用ngx_conf_params存储
                if (*p) {
                    ngx_conf_params = p;
                    goto next;
                }

                if (argv[++i]) {
                    ngx_conf_params = (u_char *) argv[i];
                    goto next;
                }

                ngx_log_stderr(0, "option \"-g\" requires parameter");
                return NGX_ERROR;

            case 's':
                // 发送信号，使用ngx_signal存储
                if (*p) {
                    ngx_signal = (char *) p;

                } else if (argv[++i]) {
                    ngx_signal = argv[i];

                } else {
                    ngx_log_stderr(0, "option \"-s\" requires parameter");
                    return NGX_ERROR;
                }

                if (ngx_strcmp(ngx_signal, "stop") == 0
                    || ngx_strcmp(ngx_signal, "quit") == 0
                    || ngx_strcmp(ngx_signal, "reopen") == 0
                    || ngx_strcmp(ngx_signal, "reload") == 0)
                {
                    // 如果是上述四种之一，设置ngx_process为NGX_PROCESS_SIGNALLER
                    ngx_process = NGX_PROCESS_SIGNALLER;
                    goto next;
                }

                ngx_log_stderr(0, "invalid option: \"-s %s\"", ngx_signal);
                return NGX_ERROR;

            default:
                ngx_log_stderr(0, "invalid option: \"%c\"", *(p - 1));
                return NGX_ERROR;
            }
        }

    next:

        continue;
    }

    return NGX_OK;
}
```

### 初始化信息

```
ngx_time_init();
```

详见事件处理部分。

### 初始化log打印描述符

```
log = ngx_log_init(ngx_prefix);
```

更加启动参数或默认log路径，初始化log信息，包括描述符和等级等信息。

### 申请内存池空间Pool

```
ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
init_cycle.log = log;
ngx_cycle = &init_cycle;

init_cycle.pool = ngx_create_pool(1024, log);
```

### 存储命令行参数

```c
if (ngx_save_argv(&init_cycle, argc, argv) != NGX_OK) {
    return 1;
}
```

将命令行存储到全国变量中：

```c
ngx_argv
ngx_argc
```

 ### 设置相关路径

```
if (ngx_process_options(&init_cycle) != NGX_OK) {
    return 1;
}
```

通过启动命令行参数或默认值设置cycle中的参数

```c
static ngx_int_t
ngx_process_options(ngx_cycle_t *cycle)
{
    u_char  *p;
    size_t   len;
    // 如果存在-p参数，即设置了nginx环境目录
    if (ngx_prefix) {
        len = ngx_strlen(ngx_prefix);
        p = ngx_prefix;
        // 如果设置的目录最后不是/，则添加一个/
        if (len && !ngx_path_separator(p[len - 1])) {
            p = ngx_pnalloc(cycle->pool, len + 1);
            if (p == NULL) {
                return NGX_ERROR;
            }

            ngx_memcpy(p, ngx_prefix, len);
            p[len++] = '/';
        }
        // 设置conf_prefix配置文件所在目录路径和prefix安装路径
        cycle->conf_prefix.len = len;
        cycle->conf_prefix.data = p;
        cycle->prefix.len = len;
        cycle->prefix.data = p;

    } else {
    // 如果未设置，则使用默认路径

#ifndef NGX_PREFIX

        p = ngx_pnalloc(cycle->pool, NGX_MAX_PATH);
        if (p == NULL) {
            return NGX_ERROR;
        }

        if (ngx_getcwd(p, NGX_MAX_PATH) == 0) {
            ngx_log_stderr(ngx_errno, "[emerg]: " ngx_getcwd_n " failed");
            return NGX_ERROR;
        }

        len = ngx_strlen(p);

        p[len++] = '/';

        cycle->conf_prefix.len = len;
        cycle->conf_prefix.data = p;
        cycle->prefix.len = len;
        cycle->prefix.data = p;

#else

#ifdef NGX_CONF_PREFIX
        // 设置conf_prefix为conf/，为相对于prefix路径
        ngx_str_set(&cycle->conf_prefix, NGX_CONF_PREFIX);
#else
        ngx_str_set(&cycle->conf_prefix, NGX_PREFIX);
#endif
        // /usr/local/nginx/
        ngx_str_set(&cycle->prefix, NGX_PREFIX);

#endif
    }
    // 如果-c指定了配置文件，则设置为对应文件
    if (ngx_conf_file) {
        cycle->conf_file.len = ngx_strlen(ngx_conf_file);
        cycle->conf_file.data = ngx_conf_file;

    } else {
        // 否则设置为相对与prefix的路径conf/nginx.conf
        ngx_str_set(&cycle->conf_file, NGX_CONF_PATH);
    }
    // 根据prefix、和conf_file二者确认实际配置文件路径
    if (ngx_conf_full_name(cycle, &cycle->conf_file, 0) != NGX_OK) {
        return NGX_ERROR;
    }
    // 根据conf_file设置conf_prefix
    for (p = cycle->conf_file.data + cycle->conf_file.len - 1;
         p > cycle->conf_file.data;
         p--)
    {
        if (ngx_path_separator(*p)) {
            cycle->conf_prefix.len = p - cycle->conf_file.data + 1;
            cycle->conf_prefix.data = cycle->conf_file.data;
            break;
        }
    }
    // 如果使用-g指定了额外参数，使用cycle->conf_param存储
    if (ngx_conf_params) {
        cycle->conf_param.len = ngx_strlen(ngx_conf_params);
        cycle->conf_param.data = ngx_conf_params;
    }

    if (ngx_test_config) {
        cycle->log->log_level = NGX_LOG_INFO;
    }

    return NGX_OK;
}
```

### 初始化系统相关全局变量

```c
if (ngx_os_init(log) != NGX_OK) {
    return 1;
}
```

待详细查看。目前看包括如下数据：

```
ngx_pagesize = getpagesize(); // 内存分页大小
ngx_cacheline_size = NGX_CPU_CACHE_LINE; // 缓存行的大小
ngx_ncpu = sysconf(_SC_NPROCESSORS_ONLN); // cpu数据

getrlimit(RLIMIT_NOFILE, &rlmt);
ngx_max_sockets = (ngx_int_t) rlmt.rlim_cur; // 最大sockets链接数量
```

### 初始化差错校验

```c
if (ngx_crc32_table_init() != NGX_OK) {
    return 1;
}
```

nginx使用CRC：循环冗余检测（Cycle Redundancy Check)来进行差错校验。

### 初始化slab共享内存

```
ngx_slab_sizes_init();
```

具体详见slab共享内存。

### 监听环境变量中的端口

```
if (ngx_add_inherited_sockets(&init_cycle) != NGX_OK) {
    return 1;
}
```

对于平滑升级来说，需要保证用户无感知，因此要将原本开发监听的套接字存放于环境变量，而后，由新启动的进程读取，重新进行监听。

首选从环境变量NGINX中读取到套接字：

```c
static ngx_int_t
ngx_add_inherited_sockets(ngx_cycle_t *cycle)
{
    u_char           *p, *v, *inherited;
    ngx_int_t         s;
    ngx_listening_t  *ls;
    // 首选从环境变量NGINX中读取到套接字：
    inherited = (u_char *) getenv(NGINX_VAR);

    if (inherited == NULL) {
        return NGX_OK;
    }

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                  "using inherited sockets from \"%s\"", inherited);

    if (ngx_array_init(&cycle->listening, cycle->pool, 10,
                       sizeof(ngx_listening_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    for (p = inherited, v = p; *p; p++) {
        if (*p == ':' || *p == ';') {
            s = ngx_atoi(v, p - v);
            if (s == NGX_ERROR) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                              "invalid socket number \"%s\" in " NGINX_VAR
                              " environment variable, ignoring the rest"
                              " of the variable", v);
                break;
            }

            v = p + 1;
            // 将环境变量里的套接字添加到cyyle的listening中
            ls = ngx_array_push(&cycle->listening);
            if (ls == NULL) {
                return NGX_ERROR;
            }

            ngx_memzero(ls, sizeof(ngx_listening_t));

            ls->fd = (ngx_socket_t) s;
        }
    }

    if (v != p) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                      "invalid socket number \"%s\" in " NGINX_VAR
                      " environment variable, ignoring", v);
    }

    ngx_inherited = 1;

    return ngx_set_inherited_sockets(cycle);
}
```

获取套接字对应的信息ngx_set_inherited_sockets：

```c
getsockname(ls[i].fd, ls[i].sockaddr, &ls[i].socklen); // 获取套接字的sockaddr
getsockopt; // 获取套接字上设置的额外信息

ngx_int_t
ngx_set_inherited_sockets(ngx_cycle_t *cycle)
{
    size_t                     len;
    ngx_uint_t                 i;
    ngx_listening_t           *ls;
    socklen_t                  olen;
#if (NGX_HAVE_DEFERRED_ACCEPT || NGX_HAVE_TCP_FASTOPEN)
    ngx_err_t                  err;
#endif
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
    struct accept_filter_arg   af;
#endif
#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)
    int                        timeout;
#endif
#if (NGX_HAVE_REUSEPORT)
    int                        reuseport;
#endif
    // 当前listening中的套接字均为从环境变量中继承而来，遍历每一个套接字，获取其对应的ngx_listening_t结构中字段信息
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        // 分配套接字地址
        ls[i].sockaddr = ngx_palloc(cycle->pool, sizeof(ngx_sockaddr_t));
        if (ls[i].sockaddr == NULL) {
            return NGX_ERROR;
        }
        // 使用getsockname函数获取套接字地址，如果获取失败，则ignore置1，表示忽略该套接字。
        ls[i].socklen = sizeof(ngx_sockaddr_t);
        if (getsockname(ls[i].fd, ls[i].sockaddr, &ls[i].socklen) == -1) {
            ngx_log_error(NGX_LOG_CRIT, cycle->log, ngx_socket_errno,
                          "getsockname() of the inherited "
                          "socket #%d failed", ls[i].fd);
            ls[i].ignore = 1;
            continue;
        }

        if (ls[i].socklen > (socklen_t) sizeof(ngx_sockaddr_t)) {
            ls[i].socklen = sizeof(ngx_sockaddr_t);
        }
        // 根据套接字族类型，设置addr_text_max_len大小，即用于存储addr_text字段的空间大小
        switch (ls[i].sockaddr->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            ls[i].addr_text_max_len = NGX_INET6_ADDRSTRLEN;
            len = NGX_INET6_ADDRSTRLEN + sizeof("[]:65535") - 1;
            break;
#endif

#if (NGX_HAVE_UNIX_DOMAIN)
        case AF_UNIX:
            ls[i].addr_text_max_len = NGX_UNIX_ADDRSTRLEN;
            len = NGX_UNIX_ADDRSTRLEN;
            break;
#endif

        case AF_INET:
            ls[i].addr_text_max_len = NGX_INET_ADDRSTRLEN;
            len = NGX_INET_ADDRSTRLEN + sizeof(":65535") - 1;
            break;

        default:
            ngx_log_error(NGX_LOG_CRIT, cycle->log, ngx_socket_errno,
                          "the inherited socket #%d has "
                          "an unsupported protocol family", ls[i].fd);
            ls[i].ignore = 1;
            continue;
        }

        ls[i].addr_text.data = ngx_pnalloc(cycle->pool, len);
        if (ls[i].addr_text.data == NULL) {
            return NGX_ERROR;
        }
        // 使用地址回填addr_text内容，IP:PORT
        len = ngx_sock_ntop(ls[i].sockaddr, ls[i].socklen,
                            ls[i].addr_text.data, len, 1);
        if (len == 0) {
            return NGX_ERROR;
        }

        ls[i].addr_text.len = len;
        // 设置默认的backlog
        ls[i].backlog = NGX_LISTEN_BACKLOG;

        olen = sizeof(int);
        // 使用getsockopt获取套接字类型吗，如果失败，则ignore置1
        if (getsockopt(ls[i].fd, SOL_SOCKET, SO_TYPE, (void *) &ls[i].type,
                       &olen)
            == -1)
        {
            ngx_log_error(NGX_LOG_CRIT, cycle->log, ngx_socket_errno,
                          "getsockopt(SO_TYPE) %V failed", &ls[i].addr_text);
            ls[i].ignore = 1;
            continue;
        }

        olen = sizeof(int);
        // 使用getsockopt获取套接字rcvbuf，如果没有设置，则默认值-1，后续内容一样，都是通过getsockopt获取套接字的其他特性
        if (getsockopt(ls[i].fd, SOL_SOCKET, SO_RCVBUF, (void *) &ls[i].rcvbuf,
                       &olen)
            == -1)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                          "getsockopt(SO_RCVBUF) %V failed, ignored",
                          &ls[i].addr_text);

            ls[i].rcvbuf = -1;
        }

        olen = sizeof(int);

        if (getsockopt(ls[i].fd, SOL_SOCKET, SO_SNDBUF, (void *) &ls[i].sndbuf,
                       &olen)
            == -1)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                          "getsockopt(SO_SNDBUF) %V failed, ignored",
                          &ls[i].addr_text);

            ls[i].sndbuf = -1;
        }

#if 0
        /* SO_SETFIB is currently a set only option */

#if (NGX_HAVE_SETFIB)

        olen = sizeof(int);

        if (getsockopt(ls[i].fd, SOL_SOCKET, SO_SETFIB,
                       (void *) &ls[i].setfib, &olen)
            == -1)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                          "getsockopt(SO_SETFIB) %V failed, ignored",
                          &ls[i].addr_text);

            ls[i].setfib = -1;
        }

#endif
#endif

#if (NGX_HAVE_REUSEPORT)

        reuseport = 0;
        olen = sizeof(int);

#ifdef SO_REUSEPORT_LB

        if (getsockopt(ls[i].fd, SOL_SOCKET, SO_REUSEPORT_LB,
                       (void *) &reuseport, &olen)
            == -1)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                          "getsockopt(SO_REUSEPORT_LB) %V failed, ignored",
                          &ls[i].addr_text);

        } else {
            ls[i].reuseport = reuseport ? 1 : 0;
        }

#else

        if (getsockopt(ls[i].fd, SOL_SOCKET, SO_REUSEPORT,
                       (void *) &reuseport, &olen)
            == -1)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                          "getsockopt(SO_REUSEPORT) %V failed, ignored",
                          &ls[i].addr_text);

        } else {
            ls[i].reuseport = reuseport ? 1 : 0;
        }
#endif

#endif

        if (ls[i].type != SOCK_STREAM) {
            continue;
        }

#if (NGX_HAVE_TCP_FASTOPEN)

        olen = sizeof(int);

        if (getsockopt(ls[i].fd, IPPROTO_TCP, TCP_FASTOPEN,
                       (void *) &ls[i].fastopen, &olen)
            == -1)
        {
            err = ngx_socket_errno;

            if (err != NGX_EOPNOTSUPP && err != NGX_ENOPROTOOPT
                && err != NGX_EINVAL)
            {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, err,
                              "getsockopt(TCP_FASTOPEN) %V failed, ignored",
                              &ls[i].addr_text);
            }

            ls[i].fastopen = -1;
        }

#endif

#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)

        ngx_memzero(&af, sizeof(struct accept_filter_arg));
        olen = sizeof(struct accept_filter_arg);

        if (getsockopt(ls[i].fd, SOL_SOCKET, SO_ACCEPTFILTER, &af, &olen)
            == -1)
        {
            err = ngx_socket_errno;

            if (err == NGX_EINVAL) {
                continue;
            }

            ngx_log_error(NGX_LOG_NOTICE, cycle->log, err,
                          "getsockopt(SO_ACCEPTFILTER) for %V failed, ignored",
                          &ls[i].addr_text);
            continue;
        }

        if (olen < sizeof(struct accept_filter_arg) || af.af_name[0] == '\0') {
            continue;
        }

        ls[i].accept_filter = ngx_palloc(cycle->pool, 16);
        if (ls[i].accept_filter == NULL) {
            return NGX_ERROR;
        }

        (void) ngx_cpystrn((u_char *) ls[i].accept_filter,
                           (u_char *) af.af_name, 16);
#endif

#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)

        timeout = 0;
        olen = sizeof(int);

        if (getsockopt(ls[i].fd, IPPROTO_TCP, TCP_DEFER_ACCEPT, &timeout, &olen)
            == -1)
        {
            err = ngx_socket_errno;

            if (err == NGX_EOPNOTSUPP) {
                continue;
            }

            ngx_log_error(NGX_LOG_NOTICE, cycle->log, err,
                          "getsockopt(TCP_DEFER_ACCEPT) for %V failed, ignored",
                          &ls[i].addr_text);
            continue;
        }

        if (olen < sizeof(int) || timeout == 0) {
            continue;
        }

        ls[i].deferred_accept = 1;
#endif
    }

    return NGX_OK;
}
```

### 预初始化模块

```
if (ngx_preinit_modules() != NGX_OK) {
    return 1;
}
    
ngx_int_t
ngx_preinit_modules(void)
{
    ngx_uint_t  i;

    for (i = 0; ngx_modules[i]; i++) {
        // 设置每个模块在所有模块中的index
        ngx_modules[i]->index = i;
        ngx_modules[i]->name = ngx_module_names[i];
    }

    ngx_modules_n = i;
    ngx_max_module = ngx_modules_n + NGX_MAX_DYNAMIC_MODULES;

    return NGX_OK;
}
```

### 初始化cycle

```
cycle = ngx_init_cycle(&init_cycle);
if (cycle == NULL) {
   if (ngx_test_config) {
      ngx_log_stderr(0, "configuration file %s test failed",
                           init_cycle.conf_file.data);
   }

   return 1;
}

```

```
/* 这里参数ngx_cycle_t被视为old cycle，一方面在不终止服务重新加载配置时会执行该操作，此时会传递一个old的cycle，另一方面，在一个新启动的服务来说，根据之前的操作，已经加载了相应的cycle参数，这里会继承上述获取到的部分参数，并进行升级合并 */
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    void                *rv;
    char               **senv;
    ngx_uint_t           i, n;
    ngx_log_t           *log;
    ngx_time_t          *tp;
    ngx_conf_t           conf;
    ngx_pool_t          *pool;
    ngx_cycle_t         *cycle, **old;
    ngx_shm_zone_t      *shm_zone, *oshm_zone;
    ngx_list_part_t     *part, *opart;
    ngx_open_file_t     *file;
    ngx_listening_t     *ls, *nls;
    ngx_core_conf_t     *ccf, *old_ccf;
    ngx_core_module_t   *module;
    char                 hostname[NGX_MAXHOSTNAMELEN];
    // 读取环境变量的TZ，获取时区
    ngx_timezone_update();

    /* force localtime update with a new timezone */

    tp = ngx_timeofday();
    tp->sec = 0;

    ngx_time_update();


    log = old_cycle->log;

    pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    if (pool == NULL) {
        return NULL;
    }
    pool->log = log;
    // 创建一个新的ngx_cycle_t
    cycle = ngx_pcalloc(pool, sizeof(ngx_cycle_t));
    if (cycle == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->pool = pool;
    cycle->log = log;
    // 存储老的cycle
    cycle->old_cycle = old_cycle;
    // 继承老的cycle中conf_prefix、prefix、conf_file、conf_param信息
    cycle->conf_prefix.len = old_cycle->conf_prefix.len;
    cycle->conf_prefix.data = ngx_pstrdup(pool, &old_cycle->conf_prefix);
    if (cycle->conf_prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->prefix.len = old_cycle->prefix.len;
    cycle->prefix.data = ngx_pstrdup(pool, &old_cycle->prefix);
    if (cycle->prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->conf_file.len = old_cycle->conf_file.len;
    cycle->conf_file.data = ngx_pnalloc(pool, old_cycle->conf_file.len + 1);
    if (cycle->conf_file.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }
    ngx_cpystrn(cycle->conf_file.data, old_cycle->conf_file.data,
                old_cycle->conf_file.len + 1);

    cycle->conf_param.len = old_cycle->conf_param.len;
    cycle->conf_param.data = ngx_pstrdup(pool, &old_cycle->conf_param);
    if (cycle->conf_param.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    // 根据旧的path（管理路径列表）分配相应的新的cycle中paths的内存
    n = old_cycle->paths.nelts ? old_cycle->paths.nelts : 10;

    if (ngx_array_init(&cycle->paths, pool, n, sizeof(ngx_path_t *))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }

    ngx_memzero(cycle->paths.elts, n * sizeof(ngx_path_t *));

    // 初始化config_dump数组
    if (ngx_array_init(&cycle->config_dump, pool, 1, sizeof(ngx_conf_dump_t))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }
    // 初始化config_dump_rbtree红黑树，具体参看红黑树部分解析
    ngx_rbtree_init(&cycle->config_dump_rbtree, &cycle->config_dump_sentinel,
                    ngx_str_rbtree_insert_value);
    // 根据旧的open_files（打开文件列表）分配相应的新的cycle中open_files的内存
    if (old_cycle->open_files.part.nelts) {
        n = old_cycle->open_files.part.nelts;
        for (part = old_cycle->open_files.part.next; part; part = part->next) {
            n += part->nelts;
        }

    } else {
        n = 20;
    }

    if (ngx_list_init(&cycle->open_files, pool, n, sizeof(ngx_open_file_t))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }


    // 根据旧的shared_memory（共享内存列表）分配相应的新的cycle中shared_memory的内存
    if (old_cycle->shared_memory.part.nelts) {
        n = old_cycle->shared_memory.part.nelts;
        for (part = old_cycle->shared_memory.part.next; part; part = part->next)
        {
            n += part->nelts;
        }

    } else {
        n = 1;
    }

    if (ngx_list_init(&cycle->shared_memory, pool, n, sizeof(ngx_shm_zone_t))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }

    // 根据旧的listening（监听地址列表）分配相应的新的cycle中listening的内存
    n = old_cycle->listening.nelts ? old_cycle->listening.nelts : 10;

    if (ngx_array_init(&cycle->listening, pool, n, sizeof(ngx_listening_t))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }

    ngx_memzero(cycle->listening.elts, n * sizeof(ngx_listening_t));

    // 初始化用于事件模块的reusable_connections_queue，其值为双向队列。具体参考事件模块的处理
    ngx_queue_init(&cycle->reusable_connections_queue);

    // 分配配置文件解析内存
    cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));
    if (cycle->conf_ctx == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    // 根据gethostname获取机器的hostname
    if (gethostname(hostname, NGX_MAXHOSTNAMELEN) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "gethostname() failed");
        ngx_destroy_pool(pool);
        return NULL;
    }

    /* on Linux gethostname() silently truncates name that does not fit */

    hostname[NGX_MAXHOSTNAMELEN - 1] = '\0';
    cycle->hostname.len = ngx_strlen(hostname);

    cycle->hostname.data = ngx_pnalloc(pool, cycle->hostname.len);
    if (cycle->hostname.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    ngx_strlow(cycle->hostname.data, (u_char *) hostname, cycle->hostname.len);

    // 复制全局变量ngx_modules（模块列表）到cycle的modules（深拷贝）
    if (ngx_cycle_modules(cycle) != NGX_OK) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    /* 执行每个核心模块的create_conf方法。由于核心模块管理者其所属下的各个模块。如ngx_http_module模块管理所有http模块。创建一个结构体，用于管理其下所有模块解析后的结果。具体可以参考配置解析*/
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->create_conf) {
            rv = module->create_conf(cycle);
            if (rv == NULL) {
                ngx_destroy_pool(pool);
                return NULL;
            }
            cycle->conf_ctx[cycle->modules[i]->index] = rv;
        }
    }


    senv = environ;

    // 配置解析，解析包括-g传递的参数，和配置文件
    ngx_memzero(&conf, sizeof(ngx_conf_t));
    /* STUB: init array ? */
    conf.args = ngx_array_create(pool, 10, sizeof(ngx_str_t));
    if (conf.args == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    conf.temp_pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    if (conf.temp_pool == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }


    conf.ctx = cycle->conf_ctx;
    conf.cycle = cycle;
    conf.pool = pool;
    conf.log = log;
    conf.module_type = NGX_CORE_MODULE;
    conf.cmd_type = NGX_MAIN_CONF;

#if 0
    log->log_level = NGX_LOG_DEBUG_ALL;
#endif
    // 解析-g传递的参数
    if (ngx_conf_param(&conf) != NGX_CONF_OK) {
        environ = senv;
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }
    // 解析配置文件
    if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
        environ = senv;
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }
    
    // 如果是测试配置文件正确性，正常打印error级别下错误
    if (ngx_test_config && !ngx_quiet_mode) {
        ngx_log_stderr(0, "the configuration file %s syntax is ok",
                       cycle->conf_file.data);
    }

    // 执行每个核心模块的init方法。用于将create_conf时创建的管理配置项的类中，未设置的成员，设置为默认值
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->init_conf) {
            if (module->init_conf(cycle,
                                  cycle->conf_ctx[cycle->modules[i]->index])
                == NGX_CONF_ERROR)
            {
                environ = senv;
                ngx_destroy_cycle_pools(&conf);
                return NULL;
            }
        }
    }
 
    // 如果是要给已存在服务发送信号，则结束处理。
    if (ngx_process == NGX_PROCESS_SIGNALLER) {
        return cycle;
    }
    // 获取核心模块ngx_core_module解析配置后生成的ngx_core_conf_t结构体
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 如果是测试配置文件
    if (ngx_test_config) {
        // 创建pid文件，如果是以单进程模式运行，即非master-worker方式，就不创建了
        if (ngx_create_pidfile(&ccf->pid, log) != NGX_OK) {
            goto failed;
        }

    } else if (!ngx_is_init_cycle(old_cycle)) {
    // 如果不是新开始的一个服务（即已存在的服务调用init函数时），通过查看old_cycle是否存在ctx_conf来判断

        /*
         * we do not create the pid file in the first ngx_init_cycle() call
         * because we need to write the demonized process pid
         */

        old_ccf = (ngx_core_conf_t *) ngx_get_conf(old_cycle->conf_ctx,
                                                   ngx_core_module);
        // 如果新的pid目录和当前的不一致，则创建一个最新的，并删除当前的
        if (ccf->pid.len != old_ccf->pid.len
            || ngx_strcmp(ccf->pid.data, old_ccf->pid.data) != 0)
        {
            /* new pid file name */

            if (ngx_create_pidfile(&ccf->pid, log) != NGX_OK) {
                goto failed;
            }

            ngx_delete_pidfile(old_cycle);
        }
    }

    /* 测试锁文件，即为了解决惊群现象，多个子进程见使用锁来实现负载均衡。如果系统不支持原子的锁机制，则使用文件锁的形式，这时需要创建一个文件。*/
    if (ngx_test_lockfile(cycle->lock_file.data, log) != NGX_OK) {
        goto failed;
    }

    
    // 创建管理路径，具体下面介绍
    if (ngx_create_paths(cycle, ccf->user) != NGX_OK) {
        goto failed;
    }

    // 打开日志文件，具体下面介绍
    if (ngx_log_open_default(cycle) != NGX_OK) {
        goto failed;
    }

    /* open the new files */
    // 对于需要打开的文件，执行创建
    part = &cycle->open_files.part;
    file = part->elts;

    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            file = part->elts;
            i = 0;
        }

        if (file[i].name.len == 0) {
            continue;
        }

        file[i].fd = ngx_open_file(file[i].name.data,
                                   NGX_FILE_APPEND,
                                   NGX_FILE_CREATE_OR_OPEN,
                                   NGX_FILE_DEFAULT_ACCESS);

        ngx_log_debug3(NGX_LOG_DEBUG_CORE, log, 0,
                       "log: %p %d \"%s\"",
                       &file[i], file[i].fd, file[i].name.data);

        if (file[i].fd == NGX_INVALID_FILE) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                          ngx_open_file_n " \"%s\" failed",
                          file[i].name.data);
            goto failed;
        }

#if !(NGX_WIN32)
        // 设置文件为执行时关闭，即子进程执行exec后不继承该文件描述符，适用于平滑升级
        if (fcntl(file[i].fd, F_SETFD, FD_CLOEXEC) == -1) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                          "fcntl(FD_CLOEXEC) \"%s\" failed",
                          file[i].name.data);
            goto failed;
        }
#endif
    }

    cycle->log = &cycle->new_log;
    pool->log = &cycle->new_log;


    /* create shared memory */
    // 创见共享内存，具体在共享内存介绍
    part = &cycle->shared_memory.part;
    shm_zone = part->elts;

    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            shm_zone = part->elts;
            i = 0;
        }

        if (shm_zone[i].shm.size == 0) {
            ngx_log_error(NGX_LOG_EMERG, log, 0,
                          "zero size shared memory zone \"%V\"",
                          &shm_zone[i].shm.name);
            goto failed;
        }

        shm_zone[i].shm.log = cycle->log;

        opart = &old_cycle->shared_memory.part;
        oshm_zone = opart->elts;

        for (n = 0; /* void */ ; n++) {

            if (n >= opart->nelts) {
                if (opart->next == NULL) {
                    break;
                }
                opart = opart->next;
                oshm_zone = opart->elts;
                n = 0;
            }

            if (shm_zone[i].shm.name.len != oshm_zone[n].shm.name.len) {
                continue;
            }

            if (ngx_strncmp(shm_zone[i].shm.name.data,
                            oshm_zone[n].shm.name.data,
                            shm_zone[i].shm.name.len)
                != 0)
            {
                continue;
            }

            if (shm_zone[i].tag == oshm_zone[n].tag
                && shm_zone[i].shm.size == oshm_zone[n].shm.size
                && !shm_zone[i].noreuse)
            {
                shm_zone[i].shm.addr = oshm_zone[n].shm.addr;
#if (NGX_WIN32)
                shm_zone[i].shm.handle = oshm_zone[n].shm.handle;
#endif

                if (shm_zone[i].init(&shm_zone[i], oshm_zone[n].data)
                    != NGX_OK)
                {
                    goto failed;
                }

                goto shm_zone_found;
            }

            break;
        }

        if (ngx_shm_alloc(&shm_zone[i].shm) != NGX_OK) {
            goto failed;
        }

        if (ngx_init_zone_pool(cycle, &shm_zone[i]) != NGX_OK) {
            goto failed;
        }

        if (shm_zone[i].init(&shm_zone[i], NULL) != NGX_OK) {
            goto failed;
        }

    shm_zone_found:

        continue;
    }


    /* handle the listening sockets */
    // 处理监听套接字。
    
    if (old_cycle->listening.nelts) {
        ls = old_cycle->listening.elts;
        // 遍历每一个旧版cycle监听的套接字，设置remain为0，即默认关闭所有旧套接字。
        for (i = 0; i < old_cycle->listening.nelts; i++) {
            ls[i].remain = 0;
        }

        nls = cycle->listening.elts;
        /* 遍历每个新cycle的监听套接字，和旧套接字进行比对，如果新监听的套接字中存在与旧套接字一样的，则不关闭对应的旧套接字。*/
        for (n = 0; n < cycle->listening.nelts; n++) {

            for (i = 0; i < old_cycle->listening.nelts; i++) {
                // 如果旧套接字设置了ignore，表示该套接字发生错误，不需要进行比对。
                if (ls[i].ignore) {
                    continue;
                }
                // 如果旧套接字的remain已经为1，表示已经有一个新套接字和当前套接字地址一致，不能关闭当前套接字，因此不需要再比对
                if (ls[i].remain) {
                    continue;
                }
                // 类型不一致，不用比对
                if (ls[i].type != nls[n].type) {
                    continue;
                }
                // 对比新旧套接字地址是否一致
                if (ngx_cmp_sockaddr(nls[n].sockaddr, nls[n].socklen,
                                     ls[i].sockaddr, ls[i].socklen, 1)
                    == NGX_OK)
                {
                    // 新套接字设置为旧套接字的描述符
                    nls[n].fd = ls[i].fd;
                    // previous设置为对应的旧套接字
                    nls[n].previous = &ls[i];
                    // 记录需要继续维护旧套接字（不能close）
                    ls[i].remain = 1;
                    // 如果新的套接字的backlog和旧版的不一致，则将listen置1，告知后续需要再次执行listen函数，来变更backlog
                    if (ls[i].backlog != nls[n].backlog) {
                        nls[n].listen = 1;
                    }

#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)

                    /*
                     * FreeBSD, except the most recent versions,
                     * could not remove accept filter
                     */
                    nls[n].deferred_accept = ls[i].deferred_accept;

                    if (ls[i].accept_filter && nls[n].accept_filter) {
                        if (ngx_strcmp(ls[i].accept_filter,
                                       nls[n].accept_filter)
                            != 0)
                        {
                            nls[n].delete_deferred = 1;
                            nls[n].add_deferred = 1;
                        }

                    } else if (ls[i].accept_filter) {
                        nls[n].delete_deferred = 1;

                    } else if (nls[n].accept_filter) {
                        nls[n].add_deferred = 1;
                    }
#endif

#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)

                    if (ls[i].deferred_accept && !nls[n].deferred_accept) {
                        nls[n].delete_deferred = 1;

                    } else if (ls[i].deferred_accept != nls[n].deferred_accept)
                    {
                        nls[n].add_deferred = 1;
                    }
#endif

#if (NGX_HAVE_REUSEPORT)
                    // 如果新的套接字设置了复用端口，但老的未设置，则将add_reuseport置1，告知后续需要设置套接字复用端口。
                    if (nls[n].reuseport && !ls[i].reuseport) {
                        nls[n].add_reuseport = 1;
                    }
#endif

                    break;
                }
            }
            // 如果套接字不为-1，表示当前套接字有效，后续不需要关闭套接字
            if (nls[n].fd == (ngx_socket_t) -1) {
                nls[n].open = 1;
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
                if (nls[n].accept_filter) {
                    nls[n].add_deferred = 1;
                }
#endif
#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)
                if (nls[n].deferred_accept) {
                    nls[n].add_deferred = 1;
                }
#endif
            }
        }

    } else {
        // 如果不存在旧的监听套接字，则设置每个待初始化的新套接字有效
        ls = cycle->listening.elts;
        for (i = 0; i < cycle->listening.nelts; i++) {
            ls[i].open = 1;
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
            if (ls[i].accept_filter) {
                ls[i].add_deferred = 1;
            }
#endif
#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)
            if (ls[i].deferred_accept) {
                ls[i].add_deferred = 1;
            }
#endif
        }
    }

    // 处理监听套接字，打开并绑定,详见配置项解析中，监听端口的管理章节
    if (ngx_open_listening_sockets(cycle) != NGX_OK) {
        goto failed;
    }
    // 设置套接字属性
    if (!ngx_test_config) {
        ngx_configure_listening_sockets(cycle);
    }


    /* commit the new cycle configuration */
    // 设置日志的错误输出到标注错误输出上
    if (!ngx_use_stderr) {
        (void) ngx_log_redirect_stderr(cycle);
    }

    pool->log = cycle->log;
    // 执行每个module的init_module函数
    if (ngx_init_modules(cycle) != NGX_OK) {
        /* fatal */
        exit(1);
    }


    /* close and delete stuff that lefts from an old cycle */

    /* free the unnecessary shared memory */
    // 释放不再使用的共享存储
    opart = &old_cycle->shared_memory.part;
    oshm_zone = opart->elts;

    for (i = 0; /* void */ ; i++) {

        if (i >= opart->nelts) {
            if (opart->next == NULL) {
                goto old_shm_zone_done;
            }
            opart = opart->next;
            oshm_zone = opart->elts;
            i = 0;
        }

        part = &cycle->shared_memory.part;
        shm_zone = part->elts;

        for (n = 0; /* void */ ; n++) {

            if (n >= part->nelts) {
                if (part->next == NULL) {
                    break;
                }
                part = part->next;
                shm_zone = part->elts;
                n = 0;
            }

            if (oshm_zone[i].shm.name.len != shm_zone[n].shm.name.len) {
                continue;
            }

            if (ngx_strncmp(oshm_zone[i].shm.name.data,
                            shm_zone[n].shm.name.data,
                            oshm_zone[i].shm.name.len)
                != 0)
            {
                continue;
            }

            if (oshm_zone[i].tag == shm_zone[n].tag
                && oshm_zone[i].shm.size == shm_zone[n].shm.size
                && !oshm_zone[i].noreuse)
            {
                goto live_shm_zone;
            }

            break;
        }

        ngx_shm_free(&oshm_zone[i].shm);

    live_shm_zone:

        continue;
    }

old_shm_zone_done:


    /* close the unnecessary listening sockets */
    // 关闭不再监听的旧的监听套接字
    ls = old_cycle->listening.elts;
    for (i = 0; i < old_cycle->listening.nelts; i++) {
        // 对于需要维护的，或者是-1的，不进行处理
        if (ls[i].remain || ls[i].fd == (ngx_socket_t) -1) {
            continue;
        }

        if (ngx_close_socket(ls[i].fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                          ngx_close_socket_n " listening socket on %V failed",
                          &ls[i].addr_text);
        }

#if (NGX_HAVE_UNIX_DOMAIN)
        // 对应unix域的套接字，删除文件
        if (ls[i].sockaddr->sa_family == AF_UNIX) {
            u_char  *name;

            name = ls[i].addr_text.data + sizeof("unix:") - 1;

            ngx_log_error(NGX_LOG_WARN, cycle->log, 0,
                          "deleting socket %s", name);

            if (ngx_delete_file(name) == NGX_FILE_ERROR) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                              ngx_delete_file_n " %s failed", name);
            }
        }

#endif
    }


    /* close the unnecessary open files */
    // 关闭不再需要的文件
    part = &old_cycle->open_files.part;
    file = part->elts;

    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            file = part->elts;
            i = 0;
        }

        if (file[i].fd == NGX_INVALID_FILE || file[i].fd == ngx_stderr) {
            continue;
        }

        if (ngx_close_file(file[i].fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                          ngx_close_file_n " \"%s\" failed",
                          file[i].name.data);
        }
    }

    ngx_destroy_pool(conf.temp_pool);

    if (ngx_process == NGX_PROCESS_MASTER || ngx_is_init_cycle(old_cycle)) {

        ngx_destroy_pool(old_cycle->pool);
        cycle->old_cycle = NULL;

        return cycle;
    }

    // 申请临时的内存池
    if (ngx_temp_pool == NULL) {
        ngx_temp_pool = ngx_create_pool(128, cycle->log);
        if (ngx_temp_pool == NULL) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                          "could not create ngx_temp_pool");
            exit(1);
        }

        n = 10;

        if (ngx_array_init(&ngx_old_cycles, ngx_temp_pool, n,
                           sizeof(ngx_cycle_t *))
            != NGX_OK)
        {
            exit(1);
        }
        
        ngx_memzero(ngx_old_cycles.elts, n * sizeof(ngx_cycle_t *));
        // 设置定时清理事件
        ngx_cleaner_event.handler = ngx_clean_old_cycles;
        ngx_cleaner_event.log = cycle->log;
        ngx_cleaner_event.data = &dumb;
        dumb.fd = (ngx_socket_t) -1;
    }

    ngx_temp_pool->log = cycle->log;
    // 将旧的cycle添加到全局变量ngx_old_cycles中
    old = ngx_array_push(&ngx_old_cycles);
    if (old == NULL) {
        exit(1);
    }
    *old = old_cycle;
    // 设置定时清理事件的时间
    if (!ngx_cleaner_event.timer_set) {
        ngx_add_timer(&ngx_cleaner_event, 30000);
        ngx_cleaner_event.timer_set = 1;
    }
    // 返回新创建的cycle
    return cycle;


failed:

    if (!ngx_is_init_cycle(old_cycle)) {
        old_ccf = (ngx_core_conf_t *) ngx_get_conf(old_cycle->conf_ctx,
                                                   ngx_core_module);
        if (old_ccf->environment) {
            environ = old_ccf->environment;
        }
    }

    /* rollback the new cycle configuration */

    part = &cycle->open_files.part;
    file = part->elts;

    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            file = part->elts;
            i = 0;
        }

        if (file[i].fd == NGX_INVALID_FILE || file[i].fd == ngx_stderr) {
            continue;
        }

        if (ngx_close_file(file[i].fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                          ngx_close_file_n " \"%s\" failed",
                          file[i].name.data);
        }
    }

    /* free the newly created shared memory */

    part = &cycle->shared_memory.part;
    shm_zone = part->elts;

    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            shm_zone = part->elts;
            i = 0;
        }

        if (shm_zone[i].shm.addr == NULL) {
            continue;
        }

        opart = &old_cycle->shared_memory.part;
        oshm_zone = opart->elts;

        for (n = 0; /* void */ ; n++) {

            if (n >= opart->nelts) {
                if (opart->next == NULL) {
                    break;
                }
                opart = opart->next;
                oshm_zone = opart->elts;
                n = 0;
            }

            if (shm_zone[i].shm.name.len != oshm_zone[n].shm.name.len) {
                continue;
            }

            if (ngx_strncmp(shm_zone[i].shm.name.data,
                            oshm_zone[n].shm.name.data,
                            shm_zone[i].shm.name.len)
                != 0)
            {
                continue;
            }

            if (shm_zone[i].tag == oshm_zone[n].tag
                && shm_zone[i].shm.size == oshm_zone[n].shm.size
                && !shm_zone[i].noreuse)
            {
                goto old_shm_zone_found;
            }

            break;
        }

        ngx_shm_free(&shm_zone[i].shm);

    old_shm_zone_found:

        continue;
    }

    if (ngx_test_config) {
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }

    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        if (ls[i].fd == (ngx_socket_t) -1 || !ls[i].open) {
            continue;
        }

        if (ngx_close_socket(ls[i].fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                          ngx_close_socket_n " %V failed",
                          &ls[i].addr_text);
        }
    }

    ngx_destroy_cycle_pools(&conf);

    return NULL;
}
```

#### 创建管理目录

ngx_create_paths。对于需要管理目录的配置，生成相应的目录，例如：http请求包体零时存放路径。

```
client_body_temp_path dir-path [level1 [level2 [level3]]]
```

#### 创建管理文件

ngx_http_log_set_log

```
    { ngx_string("access_log"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_HTTP_LMT_CONF|NGX_CONF_1MORE,
      ngx_http_log_set_log,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },
```

#### 打开日志文件

```
static ngx_command_t  ngx_errlog_commands[] = {

    { ngx_string("error_log"),
      NGX_MAIN_CONF|NGX_CONF_1MORE,
      ngx_error_log,
      0,
      0,
      NULL },

      ngx_null_command
};
```

#### 打开并设置监听地址

在看这部分之前，应该先查看配置项解析章节，至少需要查看其中的管理监听端口号部分。

##### ngx_open_listening_sockets打开监听地址

```c
ngx_int_t
ngx_open_listening_sockets(ngx_cycle_t *cycle)
{
    int               reuseaddr;
    ngx_uint_t        i, tries, failed;
    ngx_err_t         err;
    ngx_log_t        *log;
    ngx_socket_t      s;
    ngx_listening_t  *ls;

    reuseaddr = 1;
#if (NGX_SUPPRESS_WARN)
    failed = 0;
#endif

    log = cycle->log;

    /* TODO: configurable try number */
    // 尝试重复执行5次
    for (tries = 5; tries; tries--) {
        failed = 0;

        /* for each listening socket */

        ls = cycle->listening.elts;
        // 遍历每一个监听的listening
        for (i = 0; i < cycle->listening.nelts; i++) {
            // 如果是ignore，则忽略
            if (ls[i].ignore) {
                continue;
            }

#if (NGX_HAVE_REUSEPORT)
            // 如果是add_reuseport则表示是一个旧的监听地址，需要增加reuseport属性
            if (ls[i].add_reuseport) {

                /*
                 * to allow transition from a socket without SO_REUSEPORT
                 * to multiple sockets with SO_REUSEPORT, we have to set
                 * SO_REUSEPORT on the old socket before opening new ones
                 */

                int  reuseport = 1;

#ifdef SO_REUSEPORT_LB

                if (setsockopt(ls[i].fd, SOL_SOCKET, SO_REUSEPORT_LB,
                               (const void *) &reuseport, sizeof(int))
                    == -1)
                {
                    ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                                  "setsockopt(SO_REUSEPORT_LB) %V failed, "
                                  "ignored",
                                  &ls[i].addr_text);
                }

#else
                // 设置地址的reuseport属性
                if (setsockopt(ls[i].fd, SOL_SOCKET, SO_REUSEPORT,
                               (const void *) &reuseport, sizeof(int))
                    == -1)
                {
                    ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                                  "setsockopt(SO_REUSEPORT) %V failed, ignored",
                                  &ls[i].addr_text);
                }
#endif
                // 恢复为0，避免下次重复执行（由于整个大循环可能会执行多次）
                ls[i].add_reuseport = 0;
            }
#endif
            // 如果套接字不为-1，则说明已经是已经打开的套接字（大循环可能执行多次）
            if (ls[i].fd != (ngx_socket_t) -1) {
                continue;
            }
            // 如果是继承而来的，则跳过
            if (ls[i].inherited) {

                /* TODO: close on exit */
                /* TODO: nonblocking */
                /* TODO: deferred accept */

                continue;
            }
            // 创建套接字
            s = ngx_socket(ls[i].sockaddr->sa_family, ls[i].type, 0);

            if (s == (ngx_socket_t) -1) {
                ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                              ngx_socket_n " %V failed", &ls[i].addr_text);
                return NGX_ERROR;
            }
            // 设置套接字复用地址属性
            if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR,
                           (const void *) &reuseaddr, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                              "setsockopt(SO_REUSEADDR) %V failed",
                              &ls[i].addr_text);

                if (ngx_close_socket(s) == -1) {
                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                  ngx_close_socket_n " %V failed",
                                  &ls[i].addr_text);
                }

                return NGX_ERROR;
            }

#if (NGX_HAVE_REUSEPORT)
            // 如果用户设置了套接字的reuseport属性，则设置套接字的reuseport属性
            if (ls[i].reuseport && !ngx_test_config) {
                int  reuseport;

                reuseport = 1;

#ifdef SO_REUSEPORT_LB

                if (setsockopt(s, SOL_SOCKET, SO_REUSEPORT_LB,
                               (const void *) &reuseport, sizeof(int))
                    == -1)
                {
                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                  "setsockopt(SO_REUSEPORT_LB) %V failed",
                                  &ls[i].addr_text);

                    if (ngx_close_socket(s) == -1) {
                        ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                      ngx_close_socket_n " %V failed",
                                      &ls[i].addr_text);
                    }

                    return NGX_ERROR;
                }

#else

                if (setsockopt(s, SOL_SOCKET, SO_REUSEPORT,
                               (const void *) &reuseport, sizeof(int))
                    == -1)
                {
                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                  "setsockopt(SO_REUSEPORT) %V failed",
                                  &ls[i].addr_text);

                    if (ngx_close_socket(s) == -1) {
                        ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                      ngx_close_socket_n " %V failed",
                                      &ls[i].addr_text);
                    }

                    return NGX_ERROR;
                }
#endif
            }
#endif

#if (NGX_HAVE_INET6 && defined IPV6_V6ONLY)
            // 设置套接字属性
            if (ls[i].sockaddr->sa_family == AF_INET6) {
                int  ipv6only;

                ipv6only = ls[i].ipv6only;

                if (setsockopt(s, IPPROTO_IPV6, IPV6_V6ONLY,
                               (const void *) &ipv6only, sizeof(int))
                    == -1)
                {
                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                  "setsockopt(IPV6_V6ONLY) %V failed, ignored",
                                  &ls[i].addr_text);
                }
            }
#endif
            /* TODO: close on exit */
            // 如果选择的事件模块不是NGX_USE_IOCP_EVENT，则设置套接字为非阻塞的ioctl(s, FIONBIO, &nb)
            if (!(ngx_event_flags & NGX_USE_IOCP_EVENT)) {
                if (ngx_nonblocking(s) == -1) {
                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                  ngx_nonblocking_n " %V failed",
                                  &ls[i].addr_text);

                    if (ngx_close_socket(s) == -1) {
                        ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                      ngx_close_socket_n " %V failed",
                                      &ls[i].addr_text);
                    }

                    return NGX_ERROR;
                }
            }

            ngx_log_debug2(NGX_LOG_DEBUG_CORE, log, 0,
                           "bind() %V #%d ", &ls[i].addr_text, s);
            // 绑定套接字及其地址
            if (bind(s, ls[i].sockaddr, ls[i].socklen) == -1) {
                err = ngx_socket_errno;

                if (err != NGX_EADDRINUSE || !ngx_test_config) {
                    ngx_log_error(NGX_LOG_EMERG, log, err,
                                  "bind() to %V failed", &ls[i].addr_text);
                }

                if (ngx_close_socket(s) == -1) {
                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                  ngx_close_socket_n " %V failed",
                                  &ls[i].addr_text);
                }

                if (err != NGX_EADDRINUSE) {
                    return NGX_ERROR;
                }

                if (!ngx_test_config) {
                    failed = 1;
                }

                continue;
            }

#if (NGX_HAVE_UNIX_DOMAIN)
            // 如果是unix域套接字，则变更对应的文件属性
            if (ls[i].sockaddr->sa_family == AF_UNIX) {
                mode_t   mode;
                u_char  *name;

                name = ls[i].addr_text.data + sizeof("unix:") - 1;
                mode = (S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH);

                if (chmod((char *) name, mode) == -1) {
                    ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                                  "chmod() \"%s\" failed", name);
                }

                if (ngx_test_config) {
                    if (ngx_delete_file(name) == NGX_FILE_ERROR) {
                        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                                      ngx_delete_file_n " %s failed", name);
                    }
                }
            }
#endif

            if (ls[i].type != SOCK_STREAM) {
                ls[i].fd = s;
                continue;
            }
            // 开启监听
            if (listen(s, ls[i].backlog) == -1) {
                err = ngx_socket_errno;

                /*
                 * on OpenVZ after suspend/resume EADDRINUSE
                 * may be returned by listen() instead of bind(), see
                 * https://bugzilla.openvz.org/show_bug.cgi?id=2470
                 */

                if (err != NGX_EADDRINUSE || !ngx_test_config) {
                    ngx_log_error(NGX_LOG_EMERG, log, err,
                                  "listen() to %V, backlog %d failed",
                                  &ls[i].addr_text, ls[i].backlog);
                }

                if (ngx_close_socket(s) == -1) {
                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                                  ngx_close_socket_n " %V failed",
                                  &ls[i].addr_text);
                }

                if (err != NGX_EADDRINUSE) {
                    return NGX_ERROR;
                }

                if (!ngx_test_config) {
                    failed = 1;
                }

                continue;
            }
            // 标记已监听
            ls[i].listen = 1;
            // 设置描述符
            ls[i].fd = s;
        }
        // 如果没有错误，则跳出循环
        if (!failed) {
            break;
        }

        /* TODO: delay configurable */

        ngx_log_error(NGX_LOG_NOTICE, log, 0,
                      "try again to bind() after 500ms");
        // 如果有错，则500ms重试
        ngx_msleep(500);
    }

    if (failed) {
        ngx_log_error(NGX_LOG_EMERG, log, 0, "still could not bind()");
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

这里需要着重关注一下reuseport属性。对于监听多个地址：不同ip+同一个port，如果未开启该属性时，会失败。例如如下配置：

```nginx
http {
    server {
        listen 127.0.0.1:8884 bind;
        location /L1 {
           root /home/work/Nginx/study/;
        }
    }
    server {
        listen 8884;
        server_name chst.bcc-bdbl.baidu.com;
        location /L2 {
           root /home/work/Nginx/study/;
        }
    }
}
```

这里，对于127.0.0.1:8884这个地址，我们希望进行单独的监听（设置了bind），这时会先单独对该地址进行bind。而后，对于0.0.0.0:8884这个通配符地址来说，我们还要再执行一次bind。但由于未设置reuseport属性，监听0.0.0.0:8884将会出错。改成如下配置则可以正常监听：

```nginx
http {
    server {
        listen 127.0.0.1:8884 bind reuseport;
        location /L1 {
           root /home/work/Nginx/study/;
        }
    }
    server {
        listen 8884 reuseport;
        server_name chst.bcc-bdbl.baidu.com;
        location /L2 {
           root /home/work/Nginx/study/;
        }
    }
}
```

这里，两个地址都设置了reuseport属性，此时可以实现正常监听。由于对每个ip+port形式的地址，nginx维护一个ngx_http_listen_opt_t。因此两个listen都需要加上reuseport才行。注意如下配置也不会有问题：

```nginx
http {
    server {
        listen 127.0.0.1:8884;
        location /L1 {
           root /home/work/Nginx/study/;
        }
    }
    server {
        listen 8884;
        server_name chst.bcc-bdbl.baidu.com;
        location /L2 {
           root /home/work/Nginx/study/;
        }
    }
}
```

这时由于第一个地址并未设置bind。此时不会单独对127.0.0.1:8884创建一个套接字。而是直接在0.0.0.0:8884这个通配符地址上进行监听。对于建立的连接，通过`getsockname`函数来发现绑定到套接字上的地址，以此来区分使用哪个虚拟服务。

##### ngx_configure_listening_sockets设置套接字属性

通过setsockopt函数对套接字进行设置。

```c
void
ngx_configure_listening_sockets(ngx_cycle_t *cycle)
{
    int                        value;
    ngx_uint_t                 i;
    ngx_listening_t           *ls;

#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
    struct accept_filter_arg   af;
#endif

    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {

        ls[i].log = *ls[i].logp;

        if (ls[i].rcvbuf != -1) {
            if (setsockopt(ls[i].fd, SOL_SOCKET, SO_RCVBUF,
                           (const void *) &ls[i].rcvbuf, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(SO_RCVBUF, %d) %V failed, ignored",
                              ls[i].rcvbuf, &ls[i].addr_text);
            }
        }

        if (ls[i].sndbuf != -1) {
            if (setsockopt(ls[i].fd, SOL_SOCKET, SO_SNDBUF,
                           (const void *) &ls[i].sndbuf, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(SO_SNDBUF, %d) %V failed, ignored",
                              ls[i].sndbuf, &ls[i].addr_text);
            }
        }

        if (ls[i].keepalive) {
            value = (ls[i].keepalive == 1) ? 1 : 0;

            if (setsockopt(ls[i].fd, SOL_SOCKET, SO_KEEPALIVE,
                           (const void *) &value, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(SO_KEEPALIVE, %d) %V failed, ignored",
                              value, &ls[i].addr_text);
            }
        }

#if (NGX_HAVE_KEEPALIVE_TUNABLE)

        if (ls[i].keepidle) {
            value = ls[i].keepidle;

#if (NGX_KEEPALIVE_FACTOR)
            value *= NGX_KEEPALIVE_FACTOR;
#endif

            if (setsockopt(ls[i].fd, IPPROTO_TCP, TCP_KEEPIDLE,
                           (const void *) &value, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(TCP_KEEPIDLE, %d) %V failed, ignored",
                              value, &ls[i].addr_text);
            }
        }

        if (ls[i].keepintvl) {
            value = ls[i].keepintvl;

#if (NGX_KEEPALIVE_FACTOR)
            value *= NGX_KEEPALIVE_FACTOR;
#endif

            if (setsockopt(ls[i].fd, IPPROTO_TCP, TCP_KEEPINTVL,
                           (const void *) &value, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                             "setsockopt(TCP_KEEPINTVL, %d) %V failed, ignored",
                             value, &ls[i].addr_text);
            }
        }

        if (ls[i].keepcnt) {
            if (setsockopt(ls[i].fd, IPPROTO_TCP, TCP_KEEPCNT,
                           (const void *) &ls[i].keepcnt, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(TCP_KEEPCNT, %d) %V failed, ignored",
                              ls[i].keepcnt, &ls[i].addr_text);
            }
        }

#endif

#if (NGX_HAVE_SETFIB)
        if (ls[i].setfib != -1) {
            if (setsockopt(ls[i].fd, SOL_SOCKET, SO_SETFIB,
                           (const void *) &ls[i].setfib, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(SO_SETFIB, %d) %V failed, ignored",
                              ls[i].setfib, &ls[i].addr_text);
            }
        }
#endif

#if (NGX_HAVE_TCP_FASTOPEN)
        if (ls[i].fastopen != -1) {
            if (setsockopt(ls[i].fd, IPPROTO_TCP, TCP_FASTOPEN,
                           (const void *) &ls[i].fastopen, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(TCP_FASTOPEN, %d) %V failed, ignored",
                              ls[i].fastopen, &ls[i].addr_text);
            }
        }
#endif

#if 0
        if (1) {
            int tcp_nodelay = 1;

            if (setsockopt(ls[i].fd, IPPROTO_TCP, TCP_NODELAY,
                       (const void *) &tcp_nodelay, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(TCP_NODELAY) %V failed, ignored",
                              &ls[i].addr_text);
            }
        }
#endif

        if (ls[i].listen) {

            /* change backlog via listen() */
            // 通过listen函数来重新设置backlog大小
            if (listen(ls[i].fd, ls[i].backlog) == -1) {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "listen() to %V, backlog %d failed, ignored",
                              &ls[i].addr_text, ls[i].backlog);
            }
        }

        /*
         * setting deferred mode should be last operation on socket,
         * because code may prematurely continue cycle on failure
         */

#if (NGX_HAVE_DEFERRED_ACCEPT)

#ifdef SO_ACCEPTFILTER

        if (ls[i].delete_deferred) {
            if (setsockopt(ls[i].fd, SOL_SOCKET, SO_ACCEPTFILTER, NULL, 0)
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(SO_ACCEPTFILTER, NULL) "
                              "for %V failed, ignored",
                              &ls[i].addr_text);

                if (ls[i].accept_filter) {
                    ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                                  "could not change the accept filter "
                                  "to \"%s\" for %V, ignored",
                                  ls[i].accept_filter, &ls[i].addr_text);
                }

                continue;
            }

            ls[i].deferred_accept = 0;
        }

        if (ls[i].add_deferred) {
            ngx_memzero(&af, sizeof(struct accept_filter_arg));
            (void) ngx_cpystrn((u_char *) af.af_name,
                               (u_char *) ls[i].accept_filter, 16);

            if (setsockopt(ls[i].fd, SOL_SOCKET, SO_ACCEPTFILTER,
                           &af, sizeof(struct accept_filter_arg))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(SO_ACCEPTFILTER, \"%s\") "
                              "for %V failed, ignored",
                              ls[i].accept_filter, &ls[i].addr_text);
                continue;
            }

            ls[i].deferred_accept = 1;
        }

#endif

#ifdef TCP_DEFER_ACCEPT

        if (ls[i].add_deferred || ls[i].delete_deferred) {

            if (ls[i].add_deferred) {
                /*
                 * There is no way to find out how long a connection was
                 * in queue (and a connection may bypass deferred queue at all
                 * if syncookies were used), hence we use 1 second timeout
                 * here.
                 */
                value = 1;

            } else {
                value = 0;
            }

            if (setsockopt(ls[i].fd, IPPROTO_TCP, TCP_DEFER_ACCEPT,
                           &value, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(TCP_DEFER_ACCEPT, %d) for %V failed, "
                              "ignored",
                              value, &ls[i].addr_text);

                continue;
            }
        }

        if (ls[i].add_deferred) {
            ls[i].deferred_accept = 1;
        }

#endif

#endif /* NGX_HAVE_DEFERRED_ACCEPT */

#if (NGX_HAVE_IP_RECVDSTADDR)

        if (ls[i].wildcard
            && ls[i].type == SOCK_DGRAM
            && ls[i].sockaddr->sa_family == AF_INET)
        {
            value = 1;

            if (setsockopt(ls[i].fd, IPPROTO_IP, IP_RECVDSTADDR,
                           (const void *) &value, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(IP_RECVDSTADDR) "
                              "for %V failed, ignored",
                              &ls[i].addr_text);
            }
        }

#elif (NGX_HAVE_IP_PKTINFO)

        if (ls[i].wildcard
            && ls[i].type == SOCK_DGRAM
            && ls[i].sockaddr->sa_family == AF_INET)
        {
            value = 1;

            if (setsockopt(ls[i].fd, IPPROTO_IP, IP_PKTINFO,
                           (const void *) &value, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(IP_PKTINFO) "
                              "for %V failed, ignored",
                              &ls[i].addr_text);
            }
        }

#endif

#if (NGX_HAVE_INET6 && NGX_HAVE_IPV6_RECVPKTINFO)

        if (ls[i].wildcard
            && ls[i].type == SOCK_DGRAM
            && ls[i].sockaddr->sa_family == AF_INET6)
        {
            value = 1;

            if (setsockopt(ls[i].fd, IPPROTO_IPV6, IPV6_RECVPKTINFO,
                           (const void *) &value, sizeof(int))
                == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_socket_errno,
                              "setsockopt(IPV6_RECVPKTINFO) "
                              "for %V failed, ignored",
                              &ls[i].addr_text);
            }
        }

#endif
    }

    return;
}
```

### 测试配置的处理

```c
    if (ngx_test_config) {
        // 测试配置文件处理
        // 如果需要打印非error的日志
        if (!ngx_quiet_mode) {
            ngx_log_stderr(0, "configuration file %s test is successful",
                           cycle->conf_file.data);
        }
        // 如果需要将配置dump打印
        if (ngx_dump_config) {
            cd = cycle->config_dump.elts;

            for (i = 0; i < cycle->config_dump.nelts; i++) {

                ngx_write_stdout("# configuration file ");
                (void) ngx_write_fd(ngx_stdout, cd[i].name.data,
                                    cd[i].name.len);
                ngx_write_stdout(":" NGX_LINEFEED);

                b = cd[i].buffer;

                (void) ngx_write_fd(ngx_stdout, b->pos, b->last - b->pos);
                ngx_write_stdout(NGX_LINEFEED);
            }
        }

        return 0;
    }
```

### 发送信号处理

```c
    if (ngx_signal) {
        // 如果是向正在执行的nginx服务发送信号
        return ngx_signal_process(cycle, ngx_signal);
    }
```

```c
ngx_int_t
ngx_signal_process(ngx_cycle_t *cycle, char *sig)
{
    ssize_t           n;
    ngx_pid_t         pid;
    ngx_file_t        file;
    ngx_core_conf_t  *ccf;
    u_char            buf[NGX_INT64_LEN + 2];

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "signal process started");
    // 获取解析到的pid文件
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    ngx_memzero(&file, sizeof(ngx_file_t));

    file.name = ccf->pid;
    file.log = cycle->log;

    file.fd = ngx_open_file(file.name.data, NGX_FILE_RDONLY,
                            NGX_FILE_OPEN, NGX_FILE_DEFAULT_ACCESS);

    if (file.fd == NGX_INVALID_FILE) {
        ngx_log_error(NGX_LOG_ERR, cycle->log, ngx_errno,
                      ngx_open_file_n " \"%s\" failed", file.name.data);
        return 1;
    }
    // 读取内容
    n = ngx_read_file(&file, buf, NGX_INT64_LEN + 2, 0);

    if (ngx_close_file(file.fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      ngx_close_file_n " \"%s\" failed", file.name.data);
    }

    if (n == NGX_ERROR) {
        return 1;
    }
    // 去除末尾的\r,\n
    while (n-- && (buf[n] == CR || buf[n] == LF)) { /* void */ }
    // 获取进程id
    pid = ngx_atoi(buf, ++n);

    if (pid == (ngx_pid_t) NGX_ERROR) {
        ngx_log_error(NGX_LOG_ERR, cycle->log, 0,
                      "invalid PID number \"%*s\" in \"%s\"",
                      n, buf, file.name.data);
        return 1;
    }
    // 执行发送信号
    return ngx_os_signal_process(cycle, sig, pid);

}


typedef struct {
    int     signo;
    char   *signame;
    char   *name;
    void  (*handler)(int signo, siginfo_t *siginfo, void *ucontext);
} ngx_signal_t;


ngx_int_t
ngx_os_signal_process(ngx_cycle_t *cycle, char *name, ngx_pid_t pid)
{
    ngx_signal_t  *sig;
    // 遍历所有系统支持信号，找到发送的信号，向对应的进程发送信号。
    for (sig = signals; sig->signo != 0; sig++) {
        if (ngx_strcmp(name, sig->name) == 0) {
            if (kill(pid, sig->signo) != -1) {
                return 0;
            }

            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "kill(%P, %d) failed", pid, sig->signo);
        }
    }

    return 1;
}
```

具体对接收到信号的处理，后续介绍。

### 判断运行方式

```
    // 获取配置解析的ngx_core_module解析项
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 如果设置了master，并且在此之前未设置其他运行方式（默认NGX_PROCESS_SINGLE 0），ngx_process为全局变量，初值为0
    if (ccf->master && ngx_process == NGX_PROCESS_SINGLE) {
        // 设置运行方式为mater-worker方式运行
        ngx_process = NGX_PROCESS_MASTER;
    }
```



### 信号处理

```
    if (ngx_init_signals(cycle->log) != NGX_OK) {
        return 1;
    }
```

```
typedef struct {
    int     signo; // 信号
    char   *signame; // 信号名
    char   *name;
    void  (*handler)(int signo, siginfo_t *siginfo, void *ucontext); // 处理函数
} ngx_signal_t;

ngx_signal_t  signals[] = {
    { ngx_signal_value(NGX_RECONFIGURE_SIGNAL),
      "SIG" ngx_value(NGX_RECONFIGURE_SIGNAL),
      "reload",
      ngx_signal_handler },

    { ngx_signal_value(NGX_REOPEN_SIGNAL),
      "SIG" ngx_value(NGX_REOPEN_SIGNAL),
      "reopen",
      ngx_signal_handler },

    { ngx_signal_value(NGX_NOACCEPT_SIGNAL),
      "SIG" ngx_value(NGX_NOACCEPT_SIGNAL),
      "",
      ngx_signal_handler },

    { ngx_signal_value(NGX_TERMINATE_SIGNAL),
      "SIG" ngx_value(NGX_TERMINATE_SIGNAL),
      "stop",
      ngx_signal_handler },

    { ngx_signal_value(NGX_SHUTDOWN_SIGNAL),
      "SIG" ngx_value(NGX_SHUTDOWN_SIGNAL),
      "quit",
      ngx_signal_handler },

    { ngx_signal_value(NGX_CHANGEBIN_SIGNAL),
      "SIG" ngx_value(NGX_CHANGEBIN_SIGNAL),
      "",
      ngx_signal_handler },

    { SIGALRM, "SIGALRM", "", ngx_signal_handler },

    { SIGINT, "SIGINT", "", ngx_signal_handler },

    { SIGIO, "SIGIO", "", ngx_signal_handler },

    { SIGCHLD, "SIGCHLD", "", ngx_signal_handler },

    { SIGSYS, "SIGSYS, SIG_IGN", "", NULL },

    { SIGPIPE, "SIGPIPE, SIG_IGN", "", NULL },

    { 0, NULL, "", NULL }
};

ngx_int_t
ngx_init_signals(ngx_log_t *log)
{
    ngx_signal_t      *sig;
    struct sigaction   sa;
    // 其中signals为ngx_signal_t数组
    for (sig = signals; sig->signo != 0; sig++) {
        ngx_memzero(&sa, sizeof(struct sigaction));

        if (sig->handler) {
            sa.sa_sigaction = sig->handler;
            sa.sa_flags = SA_SIGINFO;

        } else {
            sa.sa_handler = SIG_IGN;
        }
        // 注册信号处理函数
        sigemptyset(&sa.sa_mask);
        if (sigaction(sig->signo, &sa, NULL) == -1) {
#if (NGX_VALGRIND)
            ngx_log_error(NGX_LOG_ALERT, log, ngx_errno,
                          "sigaction(%s) failed, ignored", sig->signame);
#else
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                          "sigaction(%s) failed", sig->signame);
            return NGX_ERROR;
#endif
        }
    }

    return NGX_OK;
}
```

信号处理相关内容可以查看如下文档：[信号处理](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E7%AC%AC%E5%8D%81%E7%AB%A0-%E4%BF%A1%E5%8F%B7)。其中处理函数为空的函数，即表示不对信号做任何处理，即忽略。其他信号处理函数均为ngx_signal_handler。处理如下：

```c
static void
ngx_signal_handler(int signo, siginfo_t *siginfo, void *ucontext)
{
    char            *action;
    ngx_int_t        ignore;
    ngx_err_t        err;
    ngx_signal_t    *sig;

    ignore = 0;

    err = ngx_errno;
    // 首先找到对应的信号，用于获取信号名记录log
    for (sig = signals; sig->signo != 0; sig++) {
        if (sig->signo == signo) {
            break;
        }
    }
    // 以线程安全的方式更新时间，具体参考时间章节介绍
    ngx_time_sigsafe_update();

    action = "";
    // 在每个子进程中，都会先设置进行类型ngx_process
    switch (ngx_process) {
    // master进程，或者单进程运行模式下处理
    case NGX_PROCESS_MASTER:
    case NGX_PROCESS_SINGLE:
        switch (signo) {
        // 退出信号，设置ngx_quit为1
        case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
            ngx_quit = 1;
            action = ", shutting down";
            break;

        // 终止信息。ngx_terminate设置为1
        case ngx_signal_value(NGX_TERMINATE_SIGNAL):
        case SIGINT:
            ngx_terminate = 1;
            action = ", exiting";
            break;
        // 停止接收连接信号，如果是守护进程方式运行，则将ngx_noaccept置1
        case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
            if (ngx_daemonized) {
                ngx_noaccept = 1;
                action = ", stop accepting connections";
            }
            break;
        // 重新加载配置
        case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
            ngx_reconfigure = 1;
            action = ", reconfiguring";
            break;
        // 重新打开日志文件
        case ngx_signal_value(NGX_REOPEN_SIGNAL):
            ngx_reopen = 1;
            action = ", reopening logs";
            break;
        // 变更二进制文件，即不关机重启
        case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
            /* 对应不关机升级来说，master进行会fork子进程，运行新的二进制文件。而后应该由用户向老的master进程发送终止解析，使旧版服务退出。原master退出来，新的mater进程将会被init进程接管，即此时新master进程的父进程id为init进程id：1。对于新进程来说，如果其父进程依旧为原来mater进程的子进程id时，说明原来master进程还未退出，此时会忽略该信号。对于老的master进程来说，如果ngx_new_binary大于0，表明新的二进制文件已经在运行了，此时也会忽略该信号。*/
            if (ngx_getppid() == ngx_parent || ngx_new_binary > 0) {

                /*
                 * Ignore the signal in the new binary if its parent is
                 * not changed, i.e. the old binary's process is still
                 * running.  Or ignore the signal in the old binary's
                 * process if the new binary's process is already running.
                 */

                action = ", ignoring";
                ignore = 1;
                break;
            }
            // 设置ngx_change_binary为1
            ngx_change_binary = 1;
            action = ", changing binary";
            break;

        case SIGALRM:
            // 时钟信号
            ngx_sigalrm = 1;
            break;
        // 异步io
        case SIGIO:
            ngx_sigio = 1;
            break;
        // 子进程退出或终止
        case SIGCHLD:
            ngx_reap = 1;
            break;
        }

        break;
    // worker和helper进程处理逻辑
    case NGX_PROCESS_WORKER:
    case NGX_PROCESS_HELPER:
        switch (signo) {
        // 退出debug
        case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
            if (!ngx_daemonized) {
                break;
            }
            ngx_debug_quit = 1;
            /* fall through */
        // 关闭
        case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
            ngx_quit = 1;
            action = ", shutting down";
            break;
        // 退出
        case ngx_signal_value(NGX_TERMINATE_SIGNAL):
        case SIGINT:
            ngx_terminate = 1;
            action = ", exiting";
            break;
        // 重新打开日志文件
        case ngx_signal_value(NGX_REOPEN_SIGNAL):
            ngx_reopen = 1;
            action = ", reopening logs";
            break;
        // 忽略重新读取配置、变更二进制文件，异步io
        case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
        case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
        case SIGIO:
            action = ", ignoring";
            break;
        }

        break;
    }
    
    // 记录日志，根据是否能够明确信号来源进行划分。
    if (siginfo && siginfo->si_pid) {
        ngx_log_error(NGX_LOG_NOTICE, ngx_cycle->log, 0,
                      "signal %d (%s) received from %P%s",
                      signo, sig->signame, siginfo->si_pid, action);

    } else {
        ngx_log_error(NGX_LOG_NOTICE, ngx_cycle->log, 0,
                      "signal %d (%s) received%s",
                      signo, sig->signame, action);
    }

    if (ignore) {
        ngx_log_error(NGX_LOG_CRIT, ngx_cycle->log, 0,
                      "the changing binary signal is ignored: "
                      "you should shutdown or terminate "
                      "before either old or new binary's process");
    }
    // 如果子进程终止，则执行如下处理
    if (signo == SIGCHLD) {
        ngx_process_get_status();
    }

    ngx_set_errno(err);
}
```

对应子进程退出时的处理如下：

```c
static void
ngx_process_get_status(void)
{
    int              status;
    char            *process;
    ngx_pid_t        pid;
    ngx_err_t        err;
    ngx_int_t        i;
    ngx_uint_t       one;

    one = 0;

    for ( ;; ) {
        // 不阻塞的查看其子进程中哪个进程结束
        pid = waitpid(-1, &status, WNOHANG);
        // 如果没有终止进程，则返回
        if (pid == 0) {
            return;
        }
        // 如果获取终止子进程失败
        if (pid == -1) {
            err = ngx_errno;

            if (err == NGX_EINTR) {
                continue;
            }

            if (err == NGX_ECHILD && one) {
                return;
            }

            /*
             * Solaris always calls the signal handler for each exited process
             * despite waitpid() may be already called for this process.
             *
             * When several processes exit at the same time FreeBSD may
             * erroneously call the signal handler for exited process
             * despite waitpid() may be already called for this process.
             */

            if (err == NGX_ECHILD) {
                ngx_log_error(NGX_LOG_INFO, ngx_cycle->log, err,
                              "waitpid() failed");
                return;
            }

            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, err,
                          "waitpid() failed");
            return;
        }

        // 标注找到一个退出进程
        one = 1;
        process = "unknown process";
        // 遍历保存进程信息的ngx_processes数组，找到对应的元素，设置其状态，并设置其exited为1
        for (i = 0; i < ngx_last_process; i++) {
            if (ngx_processes[i].pid == pid) {
                ngx_processes[i].status = status;
                ngx_processes[i].exited = 1;
                process = ngx_processes[i].name;
                break;
            }
        }
        // 进程异常终止，获取子进程终止的信号编号。
        if (WTERMSIG(status)) {
#ifdef WCOREDUMP
            // 可以通过WCOREDUMP获取否生成了终止进程的core文件
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0,
                          "%s %P exited on signal %d%s",
                          process, pid, WTERMSIG(status),
                          WCOREDUMP(status) ? " (core dumped)" : "");
#else
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0,
                          "%s %P exited on signal %d",
                          process, pid, WTERMSIG(status));
#endif

        } else {
            ngx_log_error(NGX_LOG_NOTICE, ngx_cycle->log, 0,
                          "%s %P exited with code %d",
                          process, pid, WEXITSTATUS(status));
        }
        // 获取子进程传递给exit或_exit参数的低八位,如果值为2，并且进程需要重启
        if (WEXITSTATUS(status) == 2 && ngx_processes[i].respawn) {
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0,
                          "%s %P exited with fatal code %d "
                          "and cannot be respawned",
                          process, pid, WEXITSTATUS(status));
            ngx_processes[i].respawn = 0;
        }
        // 释放终止进程所持有的锁
        ngx_unlock_mutexes(pid);
    }
}
```

对应进程终止相关处理，可参考如下文档：[进程控制](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E7%AC%AC%E5%85%AB%E7%AB%A0-%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6)。

释放终止进程锁的函数逻辑如下:

```c
static void
ngx_unlock_mutexes(ngx_pid_t pid)
{
    ngx_uint_t        i;
    ngx_shm_zone_t   *shm_zone;
    ngx_list_part_t  *part;
    ngx_slab_pool_t  *sp;

    /*
     * unlock the accept mutex if the abnormally exited process
     * held it
     */
    /* 如果设置了进程间的负载均衡锁，则根据该进程是否拥有该锁，进行释放操作。注意ngx_accept_mutex也是使用mmap内存映射技术来生成的，因此所有进程共享。*/
    if (ngx_accept_mutex_ptr) {
        (void) ngx_shmtx_force_unlock(&ngx_accept_mutex, pid);
    }

    /*
     * unlock shared memory mutexes if held by the abnormally exited
     * process
     */
    // 释放共享内存的锁
    part = (ngx_list_part_t *) &ngx_cycle->shared_memory.part;
    shm_zone = part->elts;

    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            shm_zone = part->elts;
            i = 0;
        }

        sp = (ngx_slab_pool_t *) shm_zone[i].shm.addr;

        if (ngx_shmtx_force_unlock(&sp->mutex, pid)) {
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0,
                          "shared memory zone \"%V\" was locked by %P",
                          &shm_zone[i].shm.name, pid);
        }
    }
}
```

### 变更运行状态为守护进程

```
    // 非继承而来，即正常启动，并且设置为守护进程运行模式（默认）
    if (!ngx_inherited && ccf->daemon) {
        // 进程变更为守护进程模式
        if (ngx_daemon(cycle->log) != NGX_OK) {
            return 1;
        }

        ngx_daemonized = 1;
    }

    if (ngx_inherited) {
        ngx_daemonized = 1;
    }
```

nginx默认为以守护进程的模式运行。关于守护进程，详细信息可以参考如下文档，其实际方法也与其大致相同。[守护进程](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%AE%88%E6%8A%A4%E8%BF%9B%E7%A8%8B)。

### 创建pid文件

```
    // 对于非继承而来的进程，会创建pid文件，继承而来的进程，已经在init cycle中创建了，详情参考上文。
    if (ngx_create_pidfile(&ccf->pid, cycle->log) != NGX_OK) {
        return 1;
    }
```

### 设置运行方式

```
    if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);

    } else {
        ngx_master_process_cycle(cycle);
    }
```

根据是单进程模式运行还是master-workers方式执行对应的方法。这里我们只看master-worker方式运行。

### 执行主体循环

根据配置，选择运行方式，分别为单进程方式运行和master-workers方式运行。

```c
    if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);

    } else {
        ngx_master_process_cycle(cycle);
    }
```

具体细节参考master进程和worker进程逻辑。

# master进程逻辑

## 整体处理函数

这里只介绍以master-worker形式运行的情况。其执行入口为如下函数：

```c
void
ngx_master_process_cycle(ngx_cycle_t *cycle)
{
    char              *title;
    u_char            *p;
    size_t             size;
    ngx_int_t          i;
    ngx_uint_t         sigio;
    sigset_t           set;
    struct itimerval   itv;
    ngx_uint_t         live;
    ngx_msec_t         delay;
    ngx_core_conf_t   *ccf;

    sigemptyset(&set);
    sigaddset(&set, SIGCHLD);
    sigaddset(&set, SIGALRM);
    sigaddset(&set, SIGIO);
    sigaddset(&set, SIGINT);
    sigaddset(&set, ngx_signal_value(NGX_RECONFIGURE_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_REOPEN_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_NOACCEPT_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_TERMINATE_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_CHANGEBIN_SIGNAL));
    
    // 暂时屏蔽上述信号，使其处于未决状态
    if (sigprocmask(SIG_BLOCK, &set, NULL) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "sigprocmask() failed");
    }
    // 设置set信号集为空
    sigemptyset(&set);


    size = sizeof(master_process);

    for (i = 0; i < ngx_argc; i++) {
        size += ngx_strlen(ngx_argv[i]) + 1;
    }

    title = ngx_pnalloc(cycle->pool, size);
    if (title == NULL) {
        /* fatal */
        exit(2);
    }
    
    p = ngx_cpymem(title, master_process, sizeof(master_process) - 1);
    for (i = 0; i < ngx_argc; i++) {
        *p++ = ' ';
        p = ngx_cpystrn(p, (u_char *) ngx_argv[i], size);
    }
    // 通过设置argv[0]的值来变更进程名，影响ps命令打印的进程名
    ngx_setproctitle(title);

    // 获取配置解析后的ngx_core_conf_t结构体
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    
    // 根据设置的workers子进程数量，生成子进程。详见子进程处理逻辑部分
    ngx_start_worker_processes(cycle, ccf->worker_processes,
                               NGX_PROCESS_RESPAWN);
    // 运行cache管理进程（复制管理）
    ngx_start_cache_manager_processes(cycle, 0);

    // 标记是否已经运行了一个新的二进制文件，如果是，则该值为新的二进制文件的pid
    ngx_new_binary = 0;
    // 收到强制关闭时延迟关闭时间
    delay = 0;
    // 在执行退出时，记录子进程关闭数量
    sigio = 0;
    // 是否还有存活的子进程
    live = 1;

    // master主进程循环
    for ( ;; ) {
        // 如果delay不为0，表示接收到强制退出的指令
        if (delay) {
            // 如果ngx_sigalrm不为0，表示之前的定时器已超时。则将delay乘以2，将sigio置为0（为了能够再次向子进程下发终止信号）
            if (ngx_sigalrm) {
                sigio = 0;
                delay *= 2;
                ngx_sigalrm = 0;
            }
            // 打印日志
            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "termination cycle: %M", delay);
            /* 使用setitime设置一个delay秒后的一个时钟信号，用来检测超时时间。（这里有个问题是，当设置了时钟信号，在时钟信号下发之前又接收到了其他信号，则会重新设置时钟信号,一般向子进程下发了退出信号后，再接收到的信号应该是SIGCHLD信号，即子进程退出信号）*/
            itv.it_interval.tv_sec = 0;
            itv.it_interval.tv_usec = 0;
            itv.it_value.tv_sec = delay / 1000;
            itv.it_value.tv_usec = (delay % 1000 ) * 1000;

            if (setitimer(ITIMER_REAL, &itv, NULL) == -1) {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                              "setitimer() failed");
            }
        }

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "sigsuspend");
        // 使用sigsuspend接收到之前被屏蔽的信号组中任意一个信号。
        sigsuspend(&set);
        // 更新缓存的时间
        ngx_time_update();

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "wake up, sigio %i", sigio);
                       
        // 接收到子进程退出信号的处理
        if (ngx_reap) {
            // 恢复标识
            ngx_reap = 0;
            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "reap children");
            // 根据退出原因进行相应的处理，具体下面介绍，返回的live为当前是否依然有子进程存活
            live = ngx_reap_children(cycle);
        }
        // 如果没有子进程存活，且收到了强制退出或者优雅的方式退出，则执行master进程退出。
        if (!live && (ngx_terminate || ngx_quit)) {
            ngx_master_process_exit(cycle);
        }
        // 强制退出信号，注意，这里处理时并未恢复ngx_terminate为0，并且只处理SIGCHLD信号
        if (ngx_terminate) {
            // 如果delay为0，表示刚接收到强制退出
            if (delay == 0) {
                // 设置延迟时间为50ms
                delay = 50;
            }
            
            /* 如果sigio不为0，认为子进程数量减少了1，并继续等待其他信号。当收到不是子进程终止的信号是，sigio会变成0，或者超过了设置的等待时间时，sigio也会变成0.这是是为了方便执行后续的校验delay时间。*/
            if (sigio) {
                sigio--;
                continue;
            }
            
            // 设置sigio为子进程总数量+1.这样正常情况下，不需要重复下发关闭信号
            sigio = ccf->worker_processes + 2 /* cache processes */;
            
            // 如果延迟关闭时间超过1000ms时，则执行强制kill命令，关闭子进程，具体逻辑下面会详细介绍
            if (delay > 1000) {
                ngx_signal_worker_processes(cycle, SIGKILL);
            } else {
                // 否则向子进程通过unix域套接字传递终止信息
                ngx_signal_worker_processes(cycle,
                                       ngx_signal_value(NGX_TERMINATE_SIGNAL));
            }
            // 不再执行其他信号的处理逻辑
            continue;
        }
        
        // 如果执行优雅的退出服务，则向子进程下发对应的SHUTDOWN信息。关闭监听端口。忽略后续处理
        if (ngx_quit) {
            ngx_signal_worker_processes(cycle,
                                        ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
            ngx_close_listening_sockets(cycle);

            continue;
        }
        
        // 收到重新读取配置信号
        if (ngx_reconfigure) {
            // 恢复表示
            ngx_reconfigure = 0;
            
            /* 已经执行了新的二进制文件的处理逻辑（由于在启动新的二进制文件前，会将旧的master进程的pid文件转移，因此正常来说，使用nginx命令是不会被旧进程接收到信号的，这应该是直接使用kill向旧进程下发的指令）。逻辑没太理解，只是又新增一批woker子进程和cache管理子进程，并设置ngx_noaccepting（表明当前依然在进行监听），忽略其余信号的处理*/
            if (ngx_new_binary) {
                ngx_start_worker_processes(cycle, ccf->worker_processes,
                                           NGX_PROCESS_RESPAWN);
                ngx_start_cache_manager_processes(cycle, 0);
                ngx_noaccepting = 0;

                continue;
            }

            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reconfiguring");
            // 重新执行初始化cycle，这里会重新读取配置。
            cycle = ngx_init_cycle(cycle);
            if (cycle == NULL) {
                cycle = (ngx_cycle_t *) ngx_cycle;
                continue;
            }

            ngx_cycle = cycle;
            // 获取新配置的核心配置信息
            ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                                   ngx_core_module);
            // 使用NGX_PROCESS_JUST_RESPAWN方式重新运行一批worker进程，用来区分旧进程，具体后续讲解
            ngx_start_worker_processes(cycle, ccf->worker_processes,
                                       NGX_PROCESS_JUST_RESPAWN);
            // 运行新的cache管理程序
            ngx_start_cache_manager_processes(cycle, 1);

            /* allow new processes to start */
            // 休眠一段时间，让子进程都启动起来
            ngx_msleep(100);
            // 标识存在子进程存活
            live = 1;
            // 关闭旧进程
            ngx_signal_worker_processes(cycle,
                                        ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }
        /* restart并非一个指令触发的操作，而是执行新的二进制文件出现意外时的止损方案。在下发了执行新的二进制文件时，会创建一个子进程来运行新的二进制文件。但是如果新的二进制没办法正常运行，且我们下发了关闭当前matser的accept时，将导致没有worker进程能够正常运行。只是，在ngx_reap_children中，如果我们收到了新启动的运行新的二进制文件的进程意外退出，并且当前没有进程在接受请求，则会将ngx_restart设为1。此时执行的操作是，在当前版本的二进制程序中再次打开worker子进程和cache管理子进程。*/
        if (ngx_restart) {
            ngx_restart = 0;
            ngx_start_worker_processes(cycle, ccf->worker_processes,
                                       NGX_PROCESS_RESPAWN);
            ngx_start_cache_manager_processes(cycle, 0);
            live = 1;
        }
        
        // 重新打开日志文件处理
        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            // 重新打开文件
            ngx_reopen_files(cycle, ccf->user);
            // 向子进程传递重新打开文件信息
            ngx_signal_worker_processes(cycle,
                                        ngx_signal_value(NGX_REOPEN_SIGNAL));
        }
        // 平滑升级，重新执行新的二进制文件，信号中获取的指令
        if (ngx_change_binary) {
            ngx_change_binary = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "changing binary");
            // 生成子进程，执行新的二进制文件
            ngx_new_binary = ngx_exec_new_binary(cycle, ngx_argv);
        }
        // 停止接收链接，信号中获取的指令
        if (ngx_noaccept) {
            ngx_noaccept = 0;
            // 标识状态，当前不接收请求
            ngx_noaccepting = 1;
            // 向子进程下发指令
            ngx_signal_worker_processes(cycle,
                                        ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }
    }
}
```

对于循环中使用的信号相关变量，参考上文中的信号处理部分。

## 启动worker子进程

### 相关数据结构

#### 进程信息ngx_process_t

ngx_process_t结构存储了进程的相关信息。其定义如下：

```c
// 进程执行的处理函数
typedef void (*ngx_spawn_proc_pt) (ngx_cycle_t *cycle, void *data);

typedef struct {
    // 进程id，为-1表示该结构未绑定一个进程
    ngx_pid_t           pid;
    // 进程状态
    int                 status;
    // 用于数据传参的两个unix域套接字
    ngx_socket_t        channel[2];
    // 进程绑定的处理函数，即fork创建进程后执行的重新
    ngx_spawn_proc_pt   proc;
    // 对应的数据信息，基本与proc的内容一致
    void               *data;
    // 进程名称。操作系统中显示的进程名称与name相同
    char               *name;
    // 生成子进程
    unsigned            respawn:1;
    // 刚生成的子进程，与respawn区分，表示新旧子进程
    unsigned            just_spawn:1;
    // 父子进程分离，用于旧版本的master进程生成新的进程运行新的二进制文件。此时该标志位来标注是运行新的二进制文件的子进程
    unsigned            detached:1;
    // 进程正在退出
    unsigned            exiting:1;
    // 进程已退出
    unsigned            exited:1;
} ngx_process_t;
```

#### 进程间传递信息ngx_channel_t

ngx_channel_t结构用于进程间传递信息，包括直接传递unix域套接字。

```c
typedef struct {
    // 需要传递的指令，接收方根据该内容来决定对应的处理
    ngx_uint_t  command;
    // 标识数据来源。进程id为pid的进程传递的指令
    ngx_pid_t   pid;
    // 对应于全局变量ngx_processes数组中的下标，在传递描述符时有用，接收方需要通过该值设置对应数组元素
    ngx_int_t   slot;
    // 传递unix域描述符时为要传递的fd，否则为-1，仅仅只传递上述信息
    ngx_fd_t    fd;
} ngx_channel_t;
```

传递unix域套接字详见；[传递文件描述符](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E4%BC%A0%E9%80%81%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6)

#### 全局变量ngx_processes

全局变量ngx_processes存储了每一个子进程当前状态。该数据会在master进程和各个子进程间进行维护（目前子进程只需要关注自己对应的一个元素）。

```c
ngx_process_t    ngx_processes[NGX_MAX_PROCESSES];
```



### 启动函数

使用ngx_start_worker_processes函数来启动子进程。其逻辑如下：

```c
// n为启动子进程数量。type为启动方式
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    ngx_int_t      i;
    // 初始化进程间传递信息的ngx_channel_t结构
    ngx_channel_t  ch;

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start worker processes");

    ngx_memzero(&ch, sizeof(ngx_channel_t));
    // 对应指令为打开通道，即在子进程中增加该描述符。
    ch.command = NGX_CMD_OPEN_CHANNEL;
    // 创建n个子进程
    for (i = 0; i < n; i++) {

        ngx_spawn_process(cycle, ngx_worker_process_cycle,
                          (void *) (intptr_t) i, "worker process", type);
        /* 对ch赋值。其中ngx_process_slot为全局变量，表示上一步ngx_spawn_process创建子进程对应的进程信息存储在全局变量ngx_processes中的下标 */
        ch.pid = ngx_processes[ngx_process_slot].pid;
        ch.slot = ngx_process_slot;
        ch.fd = ngx_processes[ngx_process_slot].channel[0];
        // 向之前已经启动的进程传递用于与该循环创建的子进程进行信息传递的域套接字
        ngx_pass_open_channel(cycle, &ch);
    }
}
```

#### ngx_spawn_process

这里ngx_spawn_process函数为创建子进程的统一处理函数。其逻辑如下：

```c
/* proc为子进程运行的函数。data为向子进程传递的额外参数信息，name为子进程名，即操作系统中显示的进程名，当respawn为负数时，表示启动的进程类型，用于控制创建何种进程 当是正数时，表示重启对应ngx_processes下标的进程*/
ngx_pid_t
ngx_spawn_process(ngx_cycle_t *cycle, ngx_spawn_proc_pt proc, void *data,
    char *name, ngx_int_t respawn)
{
    u_long     on;
    ngx_pid_t  pid;
    // 要操作的进程在ngx_processes中的下标
    ngx_int_t  s;
    // 如果respawn大于0，则要操作的进程即为respawn
    if (respawn >= 0) {
        s = respawn;

    } else {
        // 否则在ngx_processes数组中找到第一个未被使用的元素
        for (s = 0; s < ngx_last_process; s++) {
            if (ngx_processes[s].pid == -1) {
                break;
            }
        }
        // 如果超过了能够创建的最多的进程数量1024，则报错
        if (s == NGX_MAX_PROCESSES) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                          "no more than %d processes can be spawned",
                          NGX_MAX_PROCESSES);
            return NGX_INVALID_PID;
        }
    }

    // 创建的子进程不是父子进程分离新式的（父子进程分离式进程即创建的子进程运行新的二进制文件）
    if (respawn != NGX_PROCESS_DETACHED) {

        /* Solaris 9 still has no AF_LOCAL */
        // 生成用于进程间通讯的unix域套接字
        if (socketpair(AF_UNIX, SOCK_STREAM, 0, ngx_processes[s].channel) == -1)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "socketpair() failed while spawning \"%s\"", name);
            return NGX_INVALID_PID;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_CORE, cycle->log, 0,
                       "channel %d:%d",
                       ngx_processes[s].channel[0],
                       ngx_processes[s].channel[1]);
        // 变更两个套接字属性都为非阻塞式，使用ioctl方法
        if (ngx_nonblocking(ngx_processes[s].channel[0]) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_nonblocking_n " failed while spawning \"%s\"",
                          name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        if (ngx_nonblocking(ngx_processes[s].channel[1]) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_nonblocking_n " failed while spawning \"%s\"",
                          name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        /* 设置第一个unix域套接字为异步io。可参考如下内容：http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E9%9D%9E%E9%98%BB%E5%A1%9E%E5%92%8C%E5%BC%82%E6%AD%A5I-O */
        on = 1;
        if (ioctl(ngx_processes[s].channel[0], FIOASYNC, &on) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "ioctl(FIOASYNC) failed while spawning \"%s\"", name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        if (fcntl(ngx_processes[s].channel[0], F_SETOWN, ngx_pid) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "fcntl(F_SETOWN) failed while spawning \"%s\"", name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }
        // 设置套接字是执行时关闭。默认情况下，子进程中如果执行exec函数时，依然会继承父进程的套接字。的那个设置该值时，会在子进程执行exec时在子进程中关闭套接字。注意：如果只是fork创建子进程，但没有执行exec时，不会关闭。该方法主要作用在执行新的二进制文件时，避免继承原master进程的套接字。*/
        if (fcntl(ngx_processes[s].channel[0], F_SETFD, FD_CLOEXEC) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "fcntl(FD_CLOEXEC) failed while spawning \"%s\"",
                           name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        if (fcntl(ngx_processes[s].channel[1], F_SETFD, FD_CLOEXEC) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "fcntl(FD_CLOEXEC) failed while spawning \"%s\"",
                           name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }
        // 记录当前将要创建的子进程使用的unix域套接字，后续使用
        ngx_channel = ngx_processes[s].channel[1];

    } else {
        // 对应执行新的二进制文件的子进程来说，不需要与旧的master进程进行通讯，因此并未使用
        ngx_processes[s].channel[0] = -1;
        ngx_processes[s].channel[1] = -1;
    }
    // 记录当前操作子进程对应存储在ngx_processes的下标
    ngx_process_slot = s;

    // 创建子进程
    pid = fork();

    switch (pid) {
    // 创建失败的处理
    case -1:
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "fork() failed while spawning \"%s\"", name);
        ngx_close_channel(ngx_processes[s].channel, cycle->log);
        return NGX_INVALID_PID;
    // 子进程的处理。
    case 0:
        // 注意，这里ngx_parent使用的是旧master进程的pid。这时用于后续判断是否旧的master进程已退出
        ngx_parent = ngx_pid;
        // 获取当前子进程的pid
        ngx_pid = ngx_getpid();
        // 执行进程的处理函数，这里子进程一般就不会返回了。
        proc(cycle, data);
        break;

    default:
        break;
    }
    // 父进程的处理
    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start %s %P", name, pid);
    // 记录存储子进程的信息
    ngx_processes[s].pid = pid;
    // 恢复子进程退出状态（如果是重启子进程的时候，原本的exited为1，这里恢复为0）
    ngx_processes[s].exited = 0;
    // 如果是用来重启子进程，则到这里就结束了
    if (respawn >= 0) {
        return pid;
    }
    // 设置对应的ngx_processes内容
    ngx_processes[s].proc = proc;
    ngx_processes[s].data = data;
    ngx_processes[s].name = name;
    ngx_processes[s].exiting = 0;
    // 对应每一种启动的进程模式，设置对应的ngx_processes成员变量
    switch (respawn) {
    // 子进程终止时，不需要重新生成，cache管理子进程使用
    case NGX_PROCESS_NORESPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;
    // 刚启动的进程，且进程退出不需要重启，区分新建进程。cache管理进程使用
    case NGX_PROCESS_JUST_SPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;
    // 子进程意外终止时，需要重新生成标识
    case NGX_PROCESS_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;
    // 刚启动的子进程，通过RESPAWN与旧的进行进行区分，重读配置项时使用
    case NGX_PROCESS_JUST_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;
    // 父子进程分离。运行新的二进制的子进程
    case NGX_PROCESS_DETACHED:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 1;
        break;
    }
    // 变更记录当前ngx_processes使用的数量
    if (s == ngx_last_process) {
        ngx_last_process++;
    }
    // 返回创建的子进程的id
    return pid;
}
```

#### 进程间通讯

进程之间通过unix域套接字来传递信息。但在启动函数中有一个问题，即for循环中，在前面创建的子进程将无法拥有在后面创建的子进程的套接字。例如第一个进程（即在ngx_processes中下标为0的进程），将不会有第二个进程（即在ngx_processes中下标为0的进程）中的channel（两个unix域套接字，其中第一个用于其他进程向该进程发送信息，第二个用于进程本身接收信息）信息。这样将导致子进程之间无法直接进行通讯。

虽然目前nginx架构并未使用子进程之间进行通讯（都是matser与子进程进行通讯）。但为了后续的升级，nginx已经支持了子进程之间的通讯，其原理是通过unix域套接字传递文件描述符。具体原理可参考：[传递文件描述符](http://www.yinkuiwang.cn/2019/12/18/unix高级编程/#传送文件描述符)。

向之前创建的进程传递unix描述符函数逻辑如下：

```c
static void
ngx_pass_open_channel(ngx_cycle_t *cycle, ngx_channel_t *ch)
{
    ngx_int_t  i;
    // 遍历已经创建的进程
    for (i = 0; i < ngx_last_process; i++) {
         // 不需要给自己传递，不需要给未使用的ngx_processes传递，不需要给未创建channel的传递（运行新的二进制文件的子进程）
        if (i == ngx_process_slot
            || ngx_processes[i].pid == -1
            || ngx_processes[i].channel[0] == -1)
        {
            continue;
        }

        ngx_log_debug6(NGX_LOG_DEBUG_CORE, cycle->log, 0,
                      "pass channel s:%i pid:%P fd:%d to s:%i pid:%P fd:%d",
                      ch->slot, ch->pid, ch->fd,
                      i, ngx_processes[i].pid,
                      ngx_processes[i].channel[0]);

        /* TODO: NGX_AGAIN */
        // 向指定的描述符ngx_processes[i].channel[0]中传递ch信息
        ngx_write_channel(ngx_processes[i].channel[0],
                          ch, sizeof(ngx_channel_t), cycle->log);
    }
}
```

##### ngx_write_channel

其中详细介绍一下向域套接字写数据。

```c
ngx_int_t
ngx_write_channel(ngx_socket_t s, ngx_channel_t *ch, size_t size,
    ngx_log_t *log)
{
    ssize_t             n;
    ngx_err_t           err;
    struct iovec        iov[1];
    struct msghdr       msg;

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)

    union {
        struct cmsghdr  cm;
        char            space[CMSG_SPACE(sizeof(int))];
    } cmsg;
    // ch->fd为-1，表示只是简单的数据传输，并不用传递文件描述符
    if (ch->fd == -1) {
        msg.msg_control = NULL;
        msg.msg_controllen = 0;

    } else {
        // 添加文件描述符到外代数据中
        msg.msg_control = (caddr_t) &cmsg;
        msg.msg_controllen = sizeof(cmsg);

        ngx_memzero(&cmsg, sizeof(cmsg));

        cmsg.cm.cmsg_len = CMSG_LEN(sizeof(int));
        cmsg.cm.cmsg_level = SOL_SOCKET;
        cmsg.cm.cmsg_type = SCM_RIGHTS;

        /*
         * We have to use ngx_memcpy() instead of simple
         *   *(int *) CMSG_DATA(&cmsg.cm) = ch->fd;
         * because some gcc 4.4 with -O2/3/s optimization issues the warning:
         *   dereferencing type-punned pointer will break strict-aliasing rules
         *
         * Fortunately, gcc with -O1 compiles this ngx_memcpy()
         * in the same simple assignment as in the code above
         */

        ngx_memcpy(CMSG_DATA(&cmsg.cm), &ch->fd, sizeof(int));
    }

    msg.msg_flags = 0;

#else

    if (ch->fd == -1) {
        msg.msg_accrights = NULL;
        msg.msg_accrightslen = 0;

    } else {
        msg.msg_accrights = (caddr_t) &ch->fd;
        msg.msg_accrightslen = sizeof(int);
    }

#endif
    // 添加数据内容
    iov[0].iov_base = (char *) ch;
    iov[0].iov_len = size;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;
    // 向套接字发送数据
    n = sendmsg(s, &msg, 0);

    if (n == -1) {
        err = ngx_errno;
        if (err == NGX_EAGAIN) {
            return NGX_AGAIN;
        }

        ngx_log_error(NGX_LOG_ALERT, log, err, "sendmsg() failed");
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

具体发送域套接字参考：[传递文件描述符](http://www.yinkuiwang.cn/2019/12/18/unix高级编程/#传送文件描述符)。

## 启动cache管理子进程

暂时还未详细阅读，后续补充。

## 设置时钟信号

对于设置时钟信号，参考https://blog.csdn.net/lixianlin/article/details/25604779

## 子进程退出时处理

当子进程退出时，将执行ngx_reap_children函数来进行检查。具体逻辑如下：

```c
static ngx_uint_t
ngx_reap_children(ngx_cycle_t *cycle)
{
    ngx_int_t         i, n;
    // 记录是否还有子进程存活
    ngx_uint_t        live;
    ngx_channel_t     ch;
    ngx_core_conf_t  *ccf;

    ngx_memzero(&ch, sizeof(ngx_channel_t));
    // 向其余子进程传递信息指令：关闭通讯管道
    ch.command = NGX_CMD_CLOSE_CHANNEL;
    // 不需要传递文件描述符
    ch.fd = -1;

    live = 0;
    // 遍历每个ngx_processes
    for (i = 0; i < ngx_last_process; i++) {

        ngx_log_debug7(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "child: %i %P e:%d t:%d d:%d r:%d j:%d",
                       i,
                       ngx_processes[i].pid,
                       ngx_processes[i].exiting,
                       ngx_processes[i].exited,
                       ngx_processes[i].detached,
                       ngx_processes[i].respawn,
                       ngx_processes[i].just_spawn);
        // 跳过未使用的ngx_processes
        if (ngx_processes[i].pid == -1) {
            continue;
        }
        // 对于退出的进程的处理
        if (ngx_processes[i].exited) {
            // 进程不是运行新的二进制文件的处理
            if (!ngx_processes[i].detached) {
                // 关闭unix域套接字
                ngx_close_channel(ngx_processes[i].channel, cycle->log);

                ngx_processes[i].channel[0] = -1;
                ngx_processes[i].channel[1] = -1;

                ch.pid = ngx_processes[i].pid;
                ch.slot = i;
                // 向剩余存活的子进程传递关闭通讯通道的指令
                for (n = 0; n < ngx_last_process; n++) {
                    // 不需要关注退出的进程，未使用的ngx_processes和未打开channel的进程
                    if (ngx_processes[n].exited
                        || ngx_processes[n].pid == -1
                        || ngx_processes[n].channel[0] == -1)
                    {
                        continue;
                    }

                    ngx_log_debug3(NGX_LOG_DEBUG_CORE, cycle->log, 0,
                                   "pass close channel s:%i pid:%P to:%P",
                                   ch.slot, ch.pid, ngx_processes[n].pid);

                    /* TODO: NGX_AGAIN */

                    ngx_write_channel(ngx_processes[n].channel[0],
                                      &ch, sizeof(ngx_channel_t), cycle->log);
                }
            }
            // 需要在终止后重启的子进程，并且当前未收到关闭服务指令，并且进程不是正在退出时的处理
            if (ngx_processes[i].respawn
                && !ngx_processes[i].exiting
                && !ngx_terminate
                && !ngx_quit)
            {
                // 重启子进程
                if (ngx_spawn_process(cycle, ngx_processes[i].proc,
                                      ngx_processes[i].data,
                                      ngx_processes[i].name, i)
                    == NGX_INVALID_PID)
                {
                    ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                                  "could not respawn %s",
                                  ngx_processes[i].name);
                    continue;
                }

                // 向其余子进程传递用于进程间通讯的域套接字
                ch.command = NGX_CMD_OPEN_CHANNEL;
                ch.pid = ngx_processes[ngx_process_slot].pid;
                ch.slot = ngx_process_slot;
                ch.fd = ngx_processes[ngx_process_slot].channel[0];

                ngx_pass_open_channel(cycle, &ch);
                // 记录有进程存活
                live = 1;
                // 跳过后续处理
                continue;
            }
            // 如果退出的进程是执行新的二进制文件的进程
            if (ngx_processes[i].pid == ngx_new_binary) {

                ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                                       ngx_core_module);
                // 恢复pid文件，具体参考下文的执行新的二进制文件
                if (ngx_rename_file((char *) ccf->oldpid.data,
                                    (char *) ccf->pid.data)
                    == NGX_FILE_ERROR)
                {
                    ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                                  ngx_rename_file_n " %s back to %s failed "
                                  "after the new binary process \"%s\" exited",
                                  ccf->oldpid.data, ccf->pid.data, ngx_argv[0]);
                }
                // 设置运行新的二进制文件为0
                ngx_new_binary = 0;
                // 止损操作。如果当前已经设置了关闭监听，则设置ngx_restart为1，重新开启worker子进程和cache管理子进程。
                if (ngx_noaccepting) {
                    ngx_restart = 1;
                    ngx_noaccepting = 0;
                }
            }
            // 如果是最后一个进程，维护ngx_last_process数值
            if (i == ngx_last_process - 1) {
                ngx_last_process--;

            } else {
                // 否则标记对应的ngx_processes未使用
                ngx_processes[i].pid = -1;
            }

        } else if (ngx_processes[i].exiting || !ngx_processes[i].detached) {
        // 正在退出算是存活状态，并且不需要考虑运行新的二进制文件的子进程
            live = 1;
        }
    }

    return live;
}
```

## 退出master进程

如果是接收到退出信号，并且所有子进程已完成退出，则会执行master进程的退出。

```c
static void
ngx_master_process_exit(ngx_cycle_t *cycle)
{
    ngx_uint_t  i;
    // 删除pid文件
    ngx_delete_pidfile(cycle);

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exit");
    // 执行每个modules的exit_master函数
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->exit_master) {
            cycle->modules[i]->exit_master(cycle);
        }
    }
    // 关闭监听端口
    ngx_close_listening_sockets(cycle);

    /*
     * Copy ngx_cycle->log related data to the special static exit cycle,
     * log, and log file structures enough to allow a signal handler to log.
     * The handler may be called when standard ngx_cycle->log allocated from
     * ngx_cycle->pool is already destroyed.
     */


    ngx_exit_log = *ngx_log_get_file_log(ngx_cycle->log);

    ngx_exit_log_file.fd = ngx_exit_log.file->fd;
    ngx_exit_log.file = &ngx_exit_log_file;
    ngx_exit_log.next = NULL;
    ngx_exit_log.writer = NULL;

    ngx_exit_cycle.log = &ngx_exit_log;
    ngx_exit_cycle.files = ngx_cycle->files;
    ngx_exit_cycle.files_n = ngx_cycle->files_n;
    ngx_cycle = &ngx_exit_cycle;
    // 销毁内存池
    ngx_destroy_pool(cycle->pool);

    exit(0);
}
```

### 删除pid文件

会根据是否运行新的二进制文件来删除对应的pid文件。

```c
void
ngx_delete_pidfile(ngx_cycle_t *cycle)
{
    u_char           *name;
    ngx_core_conf_t  *ccf;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 如果已经运行了新的二进制文件，则删除旧的pid文件
    name = ngx_new_binary ? ccf->oldpid.data : ccf->pid.data;

    if (ngx_delete_file(name) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      ngx_delete_file_n " \"%s\" failed", name);
    }
}
```

### 关闭监听套接字

程序退出会关闭对套接字的监听。其处理逻辑如下：

```c
void
ngx_close_listening_sockets(ngx_cycle_t *cycle)
{
    ngx_uint_t         i;
    ngx_listening_t   *ls;
    ngx_connection_t  *c;
    // 如果使用的事件模块为NGX_USE_IOCP_EVENT，则直接结束
    if (ngx_event_flags & NGX_USE_IOCP_EVENT) {
        return;
    }

    ngx_accept_mutex_held = 0;
    ngx_use_accept_mutex = 0;

    ls = cycle->listening.elts;
    // 遍历每一个监听套接字
    for (i = 0; i < cycle->listening.nelts; i++) {

        c = ls[i].connection;
        // 存在连接时处理
        if (c) {
            // 存在读事件时（监听依然存活）
            if (c->read->active) {
                // 使用epoll，删除读事件
                if (ngx_event_flags & NGX_USE_EPOLL_EVENT) {

                    /*
                     * it seems that Linux-2.6.x OpenVZ sends events
                     * for closed shared listening sockets unless
                     * the events was explicitly deleted
                     */

                    ngx_del_event(c->read, NGX_READ_EVENT, 0);

                } else {
                    // 关闭读时间
                    ngx_del_event(c->read, NGX_READ_EVENT, NGX_CLOSE_EVENT);
                }
            }
            // 释放connection
            ngx_free_connection(c);

            c->fd = (ngx_socket_t) -1;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_CORE, cycle->log, 0,
                       "close listening %V #%d ", &ls[i].addr_text, ls[i].fd);
        // 关闭套接字
        if (ngx_close_socket(ls[i].fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                          ngx_close_socket_n " %V failed", &ls[i].addr_text);
        }

#if (NGX_HAVE_UNIX_DOMAIN)
        /* 删除域套接字创建的文件，只能单进程模式或者多进程模式的master进程来删除，并且当前没有新的二进制文件运行并且该域套接字不是继承而来或者不是新执行的二进制文件 */
        if (ls[i].sockaddr->sa_family == AF_UNIX
            && ngx_process <= NGX_PROCESS_MASTER
            && ngx_new_binary == 0
            && (!ls[i].inherited || ngx_getppid() != ngx_parent))
        {
            u_char *name = ls[i].addr_text.data + sizeof("unix:") - 1;

            if (ngx_delete_file(name) == NGX_FILE_ERROR) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                              ngx_delete_file_n " %s failed", name);
            }
        }

#endif

        ls[i].fd = (ngx_socket_t) -1;
    }

    cycle->listening.nelts = 0;
}
```

该函数用到了很多事件相关的处理，详细参考后面关于事件模块的介绍。

## 向子进程下发指令

master进程通过unix域套接字或者信号下发指令，函数为ngx_signal_worker_processes，其处理逻辑如下：

```c
static void
ngx_signal_worker_processes(ngx_cycle_t *cycle, int signo)
{
    ngx_int_t      i;
    ngx_err_t      err;
    ngx_channel_t  ch;
    // 初始化传递信息的ch
    ngx_memzero(&ch, sizeof(ngx_channel_t));

#if (NGX_BROKEN_SCM_RIGHTS)

    ch.command = 0;

#else
    // 根据信号类型，决定传递的信号
    switch (signo) {

    case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
        ch.command = NGX_CMD_QUIT;
        break;

    case ngx_signal_value(NGX_TERMINATE_SIGNAL):
        ch.command = NGX_CMD_TERMINATE;
        break;

    case ngx_signal_value(NGX_REOPEN_SIGNAL):
        ch.command = NGX_CMD_REOPEN;
        break;

    default:
        ch.command = 0;
    }

#endif
    // fd为-1，表示单纯传递数据
    ch.fd = -1;

    // 遍历每一个ngx_processes
    for (i = 0; i < ngx_last_process; i++) {

        ngx_log_debug7(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "child: %i %P e:%d t:%d d:%d r:%d j:%d",
                       i,
                       ngx_processes[i].pid,
                       ngx_processes[i].exiting,
                       ngx_processes[i].exited,
                       ngx_processes[i].detached,
                       ngx_processes[i].respawn,
                       ngx_processes[i].just_spawn);
        // 如果是运行新二进制重新的子进程，或者未使用的ngx_processes，则忽略
        if (ngx_processes[i].detached || ngx_processes[i].pid == -1) {
            continue;
        }
        /* 如果是刚启动的进程，则将标识just_spawn置为0，跳过处理。这里是为了处理重读配置，先关闭旧的子进程，再将新的子进程设置为旧的子进程 */
        if (ngx_processes[i].just_spawn) {
            ngx_processes[i].just_spawn = 0;
            continue;
        }
        /* 如果进程正在退出，并且收到的信号是NGX_SHUTDOWN_SIGNAL，则跳过处理。对应优雅的关闭进程，可能会下发多次NGX_SHUTDOWN_SIGNAL信号。*/
        if (ngx_processes[i].exiting
            && signo == ngx_signal_value(NGX_SHUTDOWN_SIGNAL))
        {
            continue;
        }
        // 如果设置了下发信息，则通过unix域套接字下发指令
        if (ch.command) {
            if (ngx_write_channel(ngx_processes[i].channel[0],
                                  &ch, sizeof(ngx_channel_t), cycle->log)
                == NGX_OK)
            {
                // 使用unix域套接字下发的指令，除了NGX_REOPEN_SIGNAL外，其余均是关闭子进程的，因此这里设置子进程正在关闭
                if (signo != ngx_signal_value(NGX_REOPEN_SIGNAL)) {
                    ngx_processes[i].exiting = 1;
                }

                continue;
            }
        }

        ngx_log_debug2(NGX_LOG_DEBUG_CORE, cycle->log, 0,
                       "kill (%P, %d)", ngx_processes[i].pid, signo);
        // 如果不是通过unix域套接字传递的信号，则使用kill下子进程下发信息
        if (kill(ngx_processes[i].pid, signo) == -1) {
            err = ngx_errno;
            ngx_log_error(NGX_LOG_ALERT, cycle->log, err,
                          "kill(%P, %d) failed", ngx_processes[i].pid, signo);
            // 如果是向不存在的进程下发信号，则设置对应的ngx_processes已经退出
            if (err == NGX_ESRCH) {
                ngx_processes[i].exited = 1;
                ngx_processes[i].exiting = 0;
                ngx_reap = 1;
            }

            continue;
        }

        if (signo != ngx_signal_value(NGX_REOPEN_SIGNAL)) {
            ngx_processes[i].exiting = 1;
        }
    }
}
```

对应kill函数，可参考文档：[kill和raise](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%87%BD%E6%95%B0kill%E5%92%8Craise).

## 执行新的二进制文件

执行新的二进制文件函数ngx_exec_new_binary。其处理逻辑如下：

```c
ngx_pid_t
ngx_exec_new_binary(ngx_cycle_t *cycle, char *const *argv)
{
    char             **env, *var;
    u_char            *p;
    ngx_uint_t         i, n;
    ngx_pid_t          pid;
    ngx_exec_ctx_t     ctx;
    ngx_core_conf_t   *ccf;
    ngx_listening_t   *ls;
    // ctx为执行新的二进制的传参
    ngx_memzero(&ctx, sizeof(ngx_exec_ctx_t));
    // 执行新的二进制文件的path
    ctx.path = argv[0];
    // 新的子进程的名字
    ctx.name = "new binary process";
    // 命令行参数，与旧版的请求参数一致，这里传参使用的是ngx_argv，即之前存储的启动时参数
    ctx.argv = argv;
    // n表示要额外申请的数组长度（除了目前已经在使用的长度外）
    n = 2;
    // 获取运行新的二进制文件时的环境变量表
    env = ngx_set_environment(cycle, &n);
    if (env == NULL) {
        return NGX_INVALID_PID;
    }
    // 将监听端口写入要执行的二进制文件的环境变量中，这里进行分配空间
    var = ngx_alloc(sizeof(NGINX_VAR)
                    + cycle->listening.nelts * (NGX_INT32_LEN + 1) + 2,
                    cycle->log);
    if (var == NULL) {
        ngx_free(env);
        return NGX_INVALID_PID;
    }

    p = ngx_cpymem(var, NGINX_VAR "=", sizeof(NGINX_VAR));
    // 遍历每一个监听端口，将端口写入环境变量
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        p = ngx_sprintf(p, "%ud;", ls[i].fd);
    }
    // 设置字符串末尾
    *p = '\0';
    // 添加环境变量到环境变量表，这里的n已经在ngx_set_environment进行了变更，具体参考ngx_set_environment处理
    env[n++] = var;

#if (NGX_SETPROCTITLE_USES_ENV)

    /* allocate the spare 300 bytes for the new binary process title */
    // 分配空间为新执行的二进制文件的title，这里不是很明白。监听端口变量和改变量是n为2的原因
    env[n++] = "SPARE=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX";

#endif
    // 设置环境的变量的末尾
    env[n] = NULL;
// config中增加--with-debug，增加运行时debug信息，打印环境变量
#if (NGX_DEBUG)
    {
    char  **e;
    for (e = env; *e; e++) {
        ngx_log_debug1(NGX_LOG_DEBUG_CORE, cycle->log, 0, "env: %s", *e);
    }
    }
#endif
    // 添加环境变量
    ctx.envp = (char *const *) env;
    // 通过ngx_core_module模块获取pid文件
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 变更pid文件，在pid文件后面增加.oldbin后缀，来保证新二进制文件的正常启动
    if (ngx_rename_file(ccf->pid.data, ccf->oldpid.data) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      ngx_rename_file_n " %s to %s failed "
                      "before executing new binary process \"%s\"",
                      ccf->pid.data, ccf->oldpid.data, argv[0]);

        ngx_free(env);
        ngx_free(var);

        return NGX_INVALID_PID;
    }
    // 创建新进程，运行新的二进制文件
    pid = ngx_execute(cycle, &ctx);
    // 运行失败，则恢复pid文件
    if (pid == NGX_INVALID_PID) {
        if (ngx_rename_file(ccf->oldpid.data, ccf->pid.data)
            == NGX_FILE_ERROR)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_rename_file_n " %s back to %s failed after "
                          "an attempt to execute new binary process \"%s\"",
                          ccf->oldpid.data, ccf->pid.data, argv[0]);
        }
    }
    // 释放环境变量
    ngx_free(env);
    ngx_free(var);
    // 返回pid
    return pid;
}
```

### 运行二进制文件

在ngx_execute函数中生成子进程并在子进程执行新的二进制文件。其逻辑如下：

```c
ngx_pid_t
ngx_execute(ngx_cycle_t *cycle, ngx_exec_ctx_t *ctx)
{
    // 这里运行的ngx_spawn_process为NGX_PROCESS_DETACHED，即父子进程分离
    return ngx_spawn_process(cycle, ngx_execute_proc, ctx, ctx->name,
                             NGX_PROCESS_DETACHED);
}
```

子进程执行的函数为：

```c
static void
ngx_execute_proc(ngx_cycle_t *cycle, void *data)
{
    ngx_exec_ctx_t  *ctx = data;
    // 通过data获取执行文件的path，运行的argv和环境变量
    if (execve(ctx->path, ctx->argv, ctx->envp) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "execve() failed while executing %s \"%s\"",
                      ctx->name, ctx->path);
    }

    exit(1);
}
```

具体execve执行函数可参考文档:[exec函数](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%87%BD%E6%95%B0exec)。注意，这里第一个参数是path，即运行文件的路径，并不会如filename一样，在环境变量的PATH中进行查找。因此，如果运行的nginx是通过环境变量找到的时，例如将nginx可执行文件移动到/urs/bin目录下，在终端直接运行nginx生成的程序，如果要让其升级，一定会失败，即使第三个参数中存在PATH环境变量，这时由于第一个参数是path，并不会在环境变量中查找。因此运行nginx命令，一定要是完整的可执行文件路径。至于如何在第三个参数中增加PATH环境变量，下文详细介绍。

### 设置环境变量

默认情况下，如果运行的exec系列函数没有环境变量这一参数时，新生成的子进程是直接继承父进程的环境变量表的。具体可参考如下文档：[环境表](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E7%8E%AF%E5%A2%83%E8%A1%A8)，[环境变量](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F),[exec函数](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%87%BD%E6%95%B0exec)。

但由于在平滑升级时，我们要向子进程传递正在监听的文件描述符，而传输的方式是通过环境变量进行传递。注意，对于监听的文件描述符，之前并未设置为EXCECLOSE即执行时关闭，因此fork后的子进程运行exec时依然继承父进程的套接字，这时通过getsockname和getsockopt即可获取套接字上对应监听的属性，就可以在子进程中继续监听了。

这样就实现了监听描述符之间的传递，但是这也带来了一下额外的问题，由于要通过环境变量来传递套接字，这将导致子进程无法天然的基础父进程中使用的环境变量，因此我们需要将当前nginx使用的环境变量一起传递给子进程。这一步操作在ngx_set_environment完成，其处理逻辑如下：

```c
/* 参数last用来区分使用创建，（当前）只有在创建子进程运行新的二进制文件时才会传递last为整数，其余均为NULL。last的大小表示除了当前进程需要使用的环境变量意外，需要额外申请的环境变量数组大小, 用于在返回后在环境变量中增加额外信息，如需要传递的套接字 */
char **
ngx_set_environment(ngx_cycle_t *cycle, ngx_uint_t *last)
{
    char                **p, **env;
    ngx_str_t            *var;
    ngx_uint_t            i, n;
    ngx_core_conf_t      *ccf;
    ngx_pool_cleanup_t   *cln;
    // 获取ngx_core_module模块对应的配置
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 如果只是获取当前进程使用的环境变量，并且此前已经设置了environment变量，则直接返回即可
    if (last == NULL && ccf->environment) {
        return ccf->environment;
    }
    // 获取配置中使用到的环境变量
    var = ccf->env.elts;
    /* 遍历每一个配置文件中需要使用的，或者设置的环境变量，查找是否存在时区变量，即TZ，如果不存在，则在ccf中增加时区变量，这时要保证新旧进程使用同一个时间 */
    for (i = 0; i < ccf->env.nelts; i++) {
        if (ngx_strcmp(var[i].data, "TZ") == 0
            || ngx_strncmp(var[i].data, "TZ=", 3) == 0)
        {
            goto tz_found;
        }
    }

    var = ngx_array_push(&ccf->env);
    if (var == NULL) {
        return NULL;
    }

    var->len = 2;
    var->data = (u_char *) "TZ";

    var = ccf->env.elts;

tz_found:
    // 变量每一个配置中使用到的环境变量
    n = 0;

    for (i = 0; i < ccf->env.nelts; i++) {
        // 如果配置中设置了环境变量的值，则直接使用配置文件中设置的即可，并记录使用的环境变量+1，跳过后续处理
        if (var[i].data[var[i].len] == '=') {
            n++;
            continue;
        }
        /* 如果配置文件中未设置要使用的环境变量的值，则从环境变量表中查找，是否有对应的环境变量值，如果存在，则表示要使用，并且需要传递给子进程，这里ngx_os_environ=environ */
        for (p = ngx_os_environ; *p; p++) {

            if (ngx_strncmp(*p, var[i].data, var[i].len) == 0
                && (*p)[var[i].len] == '=')
            {
                n++;
                break;
            }
        }
    }
    // 根据last创建对应的存储环境变量的字符串数组,+1为表示环境变量最后一个元素为"\0",表示终止
    if (last) {
        env = ngx_alloc((*last + n + 1) * sizeof(char *), cycle->log);
        if (env == NULL) {
            return NULL;
        }
        // 设置last到下一个该设置变量的数组下标，用于函数返回后的处理
        *last = n;

    } else {
        /* 如果是单纯的获取当前使用到的环境变量，则有可能后续程序不会自动处理创建的env，因此在pool中注册销毁函数，保证在程序退出时，内存能够正常释放，具体可参考pool内存池设计 */
        cln = ngx_pool_cleanup_add(cycle->pool, 0);
        if (cln == NULL) {
            return NULL;
        }

        env = ngx_alloc((n + 1) * sizeof(char *), cycle->log);
        if (env == NULL) {
            return NULL;
        }
        // 注册的销毁函数。先判断对应的data（即env）,是否和environ一致，如果不是，则销毁env（因为environ程序本身会消除）
        cln->handler = ngx_cleanup_environment;
        cln->data = env;
    }

    n = 0;
    // 遍历每个配置文件中使用到的环境变量和当前进程的环境表，来设置env
    for (i = 0; i < ccf->env.nelts; i++) {

        if (var[i].data[var[i].len] == '=') {
            env[n++] = (char *) var[i].data;
            continue;
        }

        for (p = ngx_os_environ; *p; p++) {

            if (ngx_strncmp(*p, var[i].data, var[i].len) == 0
                && (*p)[var[i].len] == '=')
            {
                env[n++] = *p;
                break;
            }
        }
    }
    // 标识当前的末尾
    env[n] = NULL;
    /* 如果仅仅是获取目前进程中使用的环境变量，则设置ccf->environment，避免后续重复计算。设置environ，避免后续的多余计算（只保留了当前需要使用的部分，这样后续计算会少很多）*/
    if (last == NULL) {
        ccf->environment = env;
        environ = env;
    }

    return env;
}
```

### 配置中环境变量的解析

上述的程序中大量使用的ngx_core_module模块的ccf->env数据，这里有必要介绍一下环境变量配置的解析处理逻辑

环境变量的配置语法如下：

```
env name
env name=value

eg：
env = PATH
env = PATH=/usr/bin
```

对于只有name的情况，表示我们要使用name对应的系统环境变量，对应name和value组的情况，表示我们要设置的对应name的环境变量在运行时的值。env只是变更了环境变量，更改了运行时的环境变量，如果希望在处理时，将环境变量作为一个值使用，例如作为server_name使用，则还需要额外的模块进行处理（如perl和lua模块），这里不做详细介绍。

ngx_core_module模块如下：

```c
ngx_module_t  ngx_core_module = {
    NGX_MODULE_V1,
    &ngx_core_module_ctx,                  /* module context */
    ngx_core_commands,                     /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};

static ngx_core_module_t  ngx_core_module_ctx = {
    ngx_string("core"),
    ngx_core_module_create_conf,
    ngx_core_module_init_conf
};

static ngx_command_t  ngx_core_commands[] = {
   ...
    { ngx_string("env"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_TAKE1,
      ngx_set_env,
      0,
      0,
      NULL },
  ...
}
```

其中设置环境变量的函数如下：

```c
static char *
ngx_set_env(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_core_conf_t  *ccf = conf;

    ngx_str_t   *value, *var;
    ngx_uint_t   i;
    // 向存储配置中使用的环境的变量的env中添加元素
    var = ngx_array_push(&ccf->env);
    if (var == NULL) {
        return NGX_CONF_ERROR;
    }
    // 获取解析配置的参数
    value = cf->args->elts;
    // value内容
    *var = value[1];
    // 遍历值，找到等号的位置。如果存在，则记录等号的下标为长度，否则就是实际长度值
    for (i = 0; i < value[1].len; i++) {

        if (value[1].data[i] == '=') {

            var->len = i;

            return NGX_CONF_OK;
        }
    }

    return NGX_CONF_OK;
}
```

这里有个问题是，记录的环境变量值并非一定是完整的配置文件中设置的值。这是由于配置文件中env有两个作用，一个是设置环境变量，另一个是使用环境变量。当我们只是写明了环境变量的key，如

```
env PATH
```

表示，我们要使用系统的PATH环境变量，其值为系统定义的值。

当我们写明的是环境变量的key和值时，表明我们要使用的环境变量，并且设置其值。如：

```
env PATH=/usr/bin
```

表明执行的二进制文件的PATH环境变量值为`/usr/bin`。

如何区分二者呢，nginx就通过查看设置的值是否存在`=`进行区分。同时，为了方便后续使用，将len设置为等号的位置用于区分是使用环境变量还是设置环境变量。直接判断data[len]是否等于`=`即可。

# worker进程逻辑

`ngx_master_process_cycle`函数会创建指定数量的的worker进程，每个进程执行的处理函数为ngx_worker_process_cycle，其执行逻辑如下：

```c
// data为对应worker进程的id。从0开始连续整数
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    ngx_int_t worker = (intptr_t) data;
    // 标识进程为worker进程，并记录id
    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;
    // 执行worker进程的初始化
    ngx_worker_process_init(cycle, worker);
    // 设置进程的title。ps时显示的进程名
    ngx_setproctitle("worker process");
    // 执行worker进程主循环
    for ( ;; ) {
        // 已经收到父进程优雅退出信号
        if (ngx_exiting) {
            // 判断是否所有事件均为可忽略事件，如果是则终止进程
            if (ngx_event_no_timers_left() == NGX_OK) {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
                ngx_worker_process_exit(cycle);
            }
        }
        // 记录日志
        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");
        // 执行事件函数
        ngx_process_events_and_timers(cycle);
        // 如果收到立即退出指令，则执行退出
        if (ngx_terminate) {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
            ngx_worker_process_exit(cycle);
        }
        // 如果收到优雅退出指令，相应处理
        if (ngx_quit) {
            ngx_quit = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                          "gracefully shutting down");
            // 设置进程名
            ngx_setproctitle("worker process is shutting down");
            // 如果ngx_exiting不为1，则执行如下操作，不再接受请求
            if (!ngx_exiting) {
                ngx_exiting = 1;
                ngx_set_shutdown_timer(cycle);
                ngx_close_listening_sockets(cycle);
                ngx_close_idle_connections(cycle);
            }
        }
        // 如果收到重新打开日志文件的指令，执行相应操作
        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            ngx_reopen_files(cycle, -1);
        }
    }
}
```



## work进程初始化

执行worker循环之前，会先对worker进行初始化操作。其逻辑如下：

```c
static void
ngx_worker_process_init(ngx_cycle_t *cycle, ngx_int_t worker)
{
    sigset_t          set;
    ngx_int_t         n;
    ngx_time_t       *tp;
    ngx_uint_t        i;
    ngx_cpuset_t     *cpu_affinity;
    struct rlimit     rlmt;
    ngx_core_conf_t  *ccf;
    ngx_listening_t  *ls;
    // 设置环境变量，参考master中的处理
    if (ngx_set_environment(cycle, NULL) == NULL) {
        /* fatal */
        exit(2);
    }
    // 获取核心模块
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    /*
    如果配置了进程优先级，则执行setpriority设置，具体可参考：http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E8%BF%9B%E7%A8%8B%E8%B0%83%E5%BA%A6
    */
    if (worker >= 0 && ccf->priority != 0) {
        if (setpriority(PRIO_PROCESS, 0, ccf->priority) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "setpriority(%d) failed", ccf->priority);
        }
    }
    /*
    如果配置了最大打开描述符数量，则执行setrlimit设置,具体参考：
http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%87%BD%E6%95%B0setjmp%E5%92%8Clongjmp
    */
    if (ccf->rlimit_nofile != NGX_CONF_UNSET) {
        rlmt.rlim_cur = (rlim_t) ccf->rlimit_nofile;
        rlmt.rlim_max = (rlim_t) ccf->rlimit_nofile;

        if (setrlimit(RLIMIT_NOFILE, &rlmt) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "setrlimit(RLIMIT_NOFILE, %i) failed",
                          ccf->rlimit_nofile);
        }
    }
    // 如果配置了最大允许的core文件程度，则设置该值
    if (ccf->rlimit_core != NGX_CONF_UNSET) {
        rlmt.rlim_cur = (rlim_t) ccf->rlimit_core;
        rlmt.rlim_max = (rlim_t) ccf->rlimit_core;

        if (setrlimit(RLIMIT_CORE, &rlmt) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "setrlimit(RLIMIT_CORE, %O) failed",
                          ccf->rlimit_core);
        }
    }
    
    // 获取进程的有效用户ID失败
    if (geteuid() == 0) {
         // 设置进程的组ID
        if (setgid(ccf->group) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "setgid(%d) failed", ccf->group);
            /* fatal */
            exit(2);
        }
        // 设置附属组ID
        if (initgroups(ccf->username, ccf->group) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "initgroups(%s, %d) failed",
                          ccf->username, ccf->group);
        }

#if (NGX_HAVE_PR_SET_KEEPCAPS && NGX_HAVE_CAPABILITIES)
        if (ccf->transparent && ccf->user) {
            if (prctl(PR_SET_KEEPCAPS, 1, 0, 0, 0) == -1) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                              "prctl(PR_SET_KEEPCAPS, 1) failed");
                /* fatal */
                exit(2);
            }
        }
#endif

        if (setuid(ccf->user) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "setuid(%d) failed", ccf->user);
            /* fatal */
            exit(2);
        }

#if (NGX_HAVE_CAPABILITIES)
        if (ccf->transparent && ccf->user) {
            struct __user_cap_data_struct    data;
            struct __user_cap_header_struct  header;

            ngx_memzero(&header, sizeof(struct __user_cap_header_struct));
            ngx_memzero(&data, sizeof(struct __user_cap_data_struct));

            header.version = _LINUX_CAPABILITY_VERSION_1;
            data.effective = CAP_TO_MASK(CAP_NET_RAW);
            data.permitted = data.effective;

            if (syscall(SYS_capset, &header, &data) == -1) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                              "capset() failed");
                /* fatal */
                exit(2);
            }
        }
#endif
    }
    // 绑定进程到指定的CPU上
    if (worker >= 0) {
        cpu_affinity = ngx_get_cpu_affinity(worker);

        if (cpu_affinity) {
            ngx_setaffinity(cpu_affinity, cycle->log);
        }
    }

#if (NGX_HAVE_PR_SET_DUMPABLE)

    /* allow coredump after setuid() in Linux 2.4.x */
    // 设置进程可以进行核心转存（即生成core文件）在收到应该执行核心转存的信号时。正常情况所有进程都是可以的，但是在重新设置了uid和gid之后，该位被清除，这里进行重新设置，具体可参考prctl的man手册
    if (prctl(PR_SET_DUMPABLE, 1, 0, 0, 0) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "prctl(PR_SET_DUMPABLE) failed");
    }

#endif
    // 如果设置了工作目录（配置中working_directory），则变更工作目录
    if (ccf->working_directory.len) {
        if (chdir((char *) ccf->working_directory.data) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "chdir(\"%s\") failed", ccf->working_directory.data);
            /* fatal */
            exit(2);
        }
    }

    // 设置接收所有信号（屏蔽信号集为空）
    sigemptyset(&set);

    if (sigprocmask(SIG_SETMASK, &set, NULL) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "sigprocmask() failed");
    }
    // 获取缓存中时间
    tp = ngx_timeofday();
    srandom(((unsigned) ngx_pid << 16) ^ tp->sec ^ tp->msec);

    /*
     * disable deleting previous events for the listening sockets because
     * in the worker processes there are no events at all at this point
     */
     // 将每一个监听连接的老版本设置为null（继承而来的，重新执行ngx_init_cycle会将之前的套接字设置为新的套接字中的previous）
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        ls[i].previous = NULL;
    }
    // 执行每个模块的init_process函数。ngx_core_event_modules的init_module在这里执行，初始化事件
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_process) {
            if (cycle->modules[i]->init_process(cycle) == NGX_ERROR) {
                /* fatal */
                exit(2);
            }
        }
    }
    
    /* 关闭其他进程中与master进行通讯的unix域套接字（仅保留当前自己进程的，通过ngx_process_slot来判断是否为自己使用的数组，改字段从master进程继承而来）*/
    for (n = 0; n < ngx_last_process; n++) {

        if (ngx_processes[n].pid == -1) {
            continue;
        }

        if (n == ngx_process_slot) {
            continue;
        }

        if (ngx_processes[n].channel[1] == -1) {
            continue;
        }

        if (close(ngx_processes[n].channel[1]) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "close() channel failed");
        }
    }
    // 关闭master的发送端套接字
    if (close(ngx_processes[ngx_process_slot].channel[0]) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "close() channel failed");
    }

#if 0
    ngx_last_process = 0;
#endif
    
    // 添加监听与master通讯管道事件
    if (ngx_add_channel_event(cycle, ngx_channel, NGX_READ_EVENT,
                              ngx_channel_handler)
        == NGX_ERROR)
    {
        /* fatal */
        exit(2);
    }
}
```

### 绑定进程到指定CPU

进程绑定 CPU 的好处：在多核 CPU 结构中，每个核心有各自的L1、L2缓存，而L3缓存是共用的。如果一个进程在核心间来回切换，各个核心的缓存命中率就会受到影响。相反如果进程不管如何调度，都始终可以在一个核心上执行，那么其数据的L1、L2 缓存的命中率可以显著提高。

所以，将进程与 CPU 进行绑定可以提高 CPU 缓存的命中率，从而提高性能。而进程与 CPU 绑定被称为：`CPU 亲和性`。

linux使用`sched_setaffinity`系统调用实现：

```c
int sched_setaffinity(pid_t pid, size_t cpusetsize, const cpu_set_t *mask);
```

pid为要设置的进程id,如果是0，则是调用进程本身的进程id。

cpusetsize为mask参数的大小。

mask参数是一个位图，每个位对应一个CPU，当某个位置1时，指示进程绑定到对应CPU上运行，一个进程可以绑定多个CPU。

如下函数检查和设置mask：

```c
typedef struct
{
  __cpu_mask __bits[__CPU_SETSIZE / __NCPUBITS];
} cpu_set_t;

// 初始化一个空的cpu_set_t
CPU_ZERO(cpu_set_t cpusetp);
// 设置cpu的位置1
CPU_SET(int cpu, cpu_set_t cpusetp)
// 判断是否对应cpu位置1了
CPU_ISSET(int cpu, cpu_set_t cpusetp)
```

下面看一下nginx中设置进程绑定到cpu上。

#### 配置解析

在ngx_core_module模块中进行解析。

通过配置`worker_cpu_affinity`来设置绑定关系。该值也可以是auto，其解析逻辑如下：

```c
{ ngx_string("worker_cpu_affinity"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_1MORE,
      ngx_set_cpu_affinity,
      0,
      0,
      NULL },

static char *
ngx_set_cpu_affinity(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
#if (NGX_HAVE_CPU_AFFINITY)
    ngx_core_conf_t  *ccf = conf;

    u_char            ch, *p;
    ngx_str_t        *value;
    ngx_uint_t        i, n;
    ngx_cpuset_t     *mask;
    // 重复设置
    if (ccf->cpu_affinity) {
        return "is duplicate";
    }
    // 分配mask数组，数量是参数数量减一（减去worker_cpu_affinity）
    mask = ngx_palloc(cf->pool, (cf->args->nelts - 1) * sizeof(ngx_cpuset_t));
    if (mask == NULL) {
        return NGX_CONF_ERROR;
    }
    // 设置的cpu绑定数量
    ccf->cpu_affinity_n = cf->args->nelts - 1;
    // 每个cpu绑定的位图
    ccf->cpu_affinity = mask;

    value = cf->args->elts;
    // 如果参数是auto
    if (ngx_strcmp(value[1].data, "auto") == 0) {
        // 参数大于三个，则错误
        if (cf->args->nelts > 3) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid number of arguments in "
                               "\"worker_cpu_affinity\" directive");
            return NGX_CONF_ERROR;
        }
        // 设置auto标识
        ccf->cpu_affinity_auto = 1;
        // 设置mask0，为所有位均置1
        CPU_ZERO(&mask[0]);
        for (i = 0; i < (ngx_uint_t) ngx_min(ngx_ncpu, CPU_SETSIZE); i++) {
            CPU_SET(i, &mask[0]);
        }
        // n=2,通过下面的循环处理
        n = 2;

    } else {
        n = 1;
    }
    // 处理绑定参数
    for ( /* void */ ; n < cf->args->nelts; n++) {
        // 如果参数长度大于位图字段长度，则是错误
        if (value[n].len > CPU_SETSIZE) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                         "\"worker_cpu_affinity\" supports up to %d CPUs only",
                         CPU_SETSIZE);
            return NGX_CONF_ERROR;
        }
        // 设置位图，反向遍历，进行绑定
        i = 0;
        CPU_ZERO(&mask[n - 1]);

        for (p = value[n].data + value[n].len - 1;
             p >= value[n].data;
             p--)
        {
            ch = *p;

            if (ch == ' ') {
                continue;
            }

            i++;

            if (ch == '0') {
                continue;
            }

            if (ch == '1') {
                CPU_SET(i - 1, &mask[n - 1]);
                continue;
            }

            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                          "invalid character \"%c\" in \"worker_cpu_affinity\"",
                          ch);
            return NGX_CONF_ERROR;
        }
    }

#else

    ngx_conf_log_error(NGX_LOG_WARN, cf, 0,
                       "\"worker_cpu_affinity\" is not supported "
                       "on this platform, ignored");
#endif

    return NGX_CONF_OK;
}
```

#### 获取进程绑定的cpu

```c
ngx_cpuset_t *
ngx_get_cpu_affinity(ngx_uint_t n)
{
#if (NGX_HAVE_CPU_AFFINITY)
    ngx_uint_t        i, j;
    ngx_cpuset_t     *mask;
    ngx_core_conf_t  *ccf;

    static ngx_cpuset_t  result;

    ccf = (ngx_core_conf_t *) ngx_get_conf(ngx_cycle->conf_ctx,
                                           ngx_core_module);
    // 如果未设置cpu绑定，则不执行
    if (ccf->cpu_affinity == NULL) {
        return NULL;
    }
    // 如果是自动绑定
    if (ccf->cpu_affinity_auto) {
        // 获取被全部置为的mask
        mask = &ccf->cpu_affinity[ccf->cpu_affinity_n - 1];
        // 遍历找到n对应的cpu设置标志位
        for (i = 0, j = n; /* void */ ; i++) {

            if (CPU_ISSET(i % CPU_SETSIZE, mask) && j-- == 0) {
                break;
            }

            if (i == CPU_SETSIZE && j == n) {
                /* empty mask */
                return NULL;
            }

            /* void */
        }

        CPU_ZERO(&result);
        CPU_SET(i % CPU_SETSIZE, &result);

        return &result;
    }
    // 如果设置的绑定关系小于当前worker的变换，则直接返回即可
    if (ccf->cpu_affinity_n > n) {
        return &ccf->cpu_affinity[n];
    }
    // 否则取最后一个设置的绑定关系
    return &ccf->cpu_affinity[ccf->cpu_affinity_n - 1];

#else

    return NULL;

#endif
}
```

#### 设置绑定

```c
void
ngx_setaffinity(ngx_cpuset_t *cpu_affinity, ngx_log_t *log)
{
    ngx_uint_t  i;
    // 打印日志
    for (i = 0; i < CPU_SETSIZE; i++) {
        if (CPU_ISSET(i, cpu_affinity)) {
            ngx_log_error(NGX_LOG_NOTICE, log, 0,
                          "sched_setaffinity(): using cpu #%ui", i);
        }
    }
    // 执行绑定
    if (sched_setaffinity(0, sizeof(cpu_set_t), cpu_affinity) == -1) {
        ngx_log_error(NGX_LOG_ALERT, log, ngx_errno,
                      "sched_setaffinity() failed");
    }
}
```

### unix域套接字通信事件

在执行初始化的最后，会将与其他进程交互的套接字事件添加进入事件监控中。其逻辑如下：

```c
ngx_int_t
ngx_add_channel_event(ngx_cycle_t *cycle, ngx_fd_t fd, ngx_int_t event,
    ngx_event_handler_pt handler)
{
    ngx_event_t       *ev, *rev, *wev;
    ngx_connection_t  *c;
    // 获取一个空闲连接
    c = ngx_get_connection(fd, cycle->log);

    if (c == NULL) {
        return NGX_ERROR;
    }

    c->pool = cycle->pool;

    rev = c->read;
    wev = c->write;

    rev->log = cycle->log;
    wev->log = cycle->log;
    // 设置事件为unix域套接字
    rev->channel = 1;
    wev->channel = 1;
    // 这里事件始终为读事件
    ev = (event == NGX_READ_EVENT) ? rev : wev;
    // 设置事件处理函数
    ev->handler = handler;
    // epoll事件驱动，将连接添加进入事件驱动中，其中ngx_add_conn就是epoll模块的ngx_epoll_add_connection
    if (ngx_add_conn && (ngx_event_flags & NGX_USE_EPOLL_EVENT) == 0) {
        if (ngx_add_conn(c) == NGX_ERROR) {
            ngx_free_connection(c);
            return NGX_ERROR;
        }

    } else {
        if (ngx_add_event(ev, event, 0) == NGX_ERROR) {
            ngx_free_connection(c);
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}
```

#### ngx_channel_handler可读事件触发时处理

```c
static void
ngx_channel_handler(ngx_event_t *ev)
{
    ngx_int_t          n;
    ngx_channel_t      ch;
    ngx_connection_t  *c;
    // 如果事件已经超时，则跳过处理
    if (ev->timedout) {
        ev->timedout = 0;
        return;
    }
    // 获取事件对应连接，这里是已经恢复末尾为0的指针
    c = ev->data;

    ngx_log_debug0(NGX_LOG_DEBUG_CORE, ev->log, 0, "channel handler");

    for ( ;; ) {
        // 从unix域套接字中读取内容，查看master处理对应函数
        n = ngx_read_channel(c->fd, &ch, sizeof(ngx_channel_t), ev->log);

        ngx_log_debug1(NGX_LOG_DEBUG_CORE, ev->log, 0, "channel: %i", n);
        // 如果发送错误
        if (n == NGX_ERROR) {
            // 使用epoll，则从事件驱动中删除
            if (ngx_event_flags & NGX_USE_EPOLL_EVENT) {
                ngx_del_conn(c, 0);
            }
            /* 关闭连接，并释放。这样将导致无法与master进程通过套接字通讯。这时如果master进程要关闭该worker进程，只能使用kill指令 */
            ngx_close_connection(c);
            return;
        }
        // 非epoll处理
        if (ngx_event_flags & NGX_USE_EVENTPORT_EVENT) {
            if (ngx_add_event(ev, NGX_READ_EVENT, 0) == NGX_ERROR) {
                return;
            }
        }
        // 如果是NGX_AGAIN表示未读取完成，下一轮继续读取
        if (n == NGX_AGAIN) {
            return;
        }

        ngx_log_debug1(NGX_LOG_DEBUG_CORE, ev->log, 0,
                       "channel command: %ui", ch.command);
        // 完成数据获取处理，判断传输指令
        switch (ch.command) {
        // 优雅退出指令，quit置1
        case NGX_CMD_QUIT:
            ngx_quit = 1;
            break;
        // 强制退出指令
        case NGX_CMD_TERMINATE:
            ngx_terminate = 1;
            break;
        // 重新打开打开的文件
        case NGX_CMD_REOPEN:
            ngx_reopen = 1;
            break;
        // 打开其他进程的unix域套接字，传递文件描述符使用。详情查看master中进程间通讯
        case NGX_CMD_OPEN_CHANNEL:

            ngx_log_debug3(NGX_LOG_DEBUG_CORE, ev->log, 0,
                           "get channel s:%i pid:%P fd:%d",
                           ch.slot, ch.pid, ch.fd);

            ngx_processes[ch.slot].pid = ch.pid;
            ngx_processes[ch.slot].channel[0] = ch.fd;
            break;
        // 关闭套接字
        case NGX_CMD_CLOSE_CHANNEL:

            ngx_log_debug4(NGX_LOG_DEBUG_CORE, ev->log, 0,
                           "close channel s:%i pid:%P our:%P fd:%d",
                           ch.slot, ch.pid, ngx_processes[ch.slot].pid,
                           ngx_processes[ch.slot].channel[0]);

            if (close(ngx_processes[ch.slot].channel[0]) == -1) {
                ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                              "close() channel failed");
            }

            ngx_processes[ch.slot].channel[0] = -1;
            break;
        }
    }
}
```

#### 读取unix域数据

```c
ngx_int_t
ngx_read_channel(ngx_socket_t s, ngx_channel_t *ch, size_t size, ngx_log_t *log)
{
    ssize_t             n;
    ngx_err_t           err;
    struct iovec        iov[1];
    struct msghdr       msg;

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)
    union {
        struct cmsghdr  cm;
        char            space[CMSG_SPACE(sizeof(int))];
    } cmsg;
#else
    int                 fd;
#endif

    iov[0].iov_base = (char *) ch;
    iov[0].iov_len = size;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)
    msg.msg_control = (caddr_t) &cmsg;
    msg.msg_controllen = sizeof(cmsg);
#else
    msg.msg_accrights = (caddr_t) &fd;
    msg.msg_accrightslen = sizeof(int);
#endif

    n = recvmsg(s, &msg, 0);

    if (n == -1) {
        err = ngx_errno;
        if (err == NGX_EAGAIN) {
            return NGX_AGAIN;
        }

        ngx_log_error(NGX_LOG_ALERT, log, err, "recvmsg() failed");
        return NGX_ERROR;
    }

    if (n == 0) {
        ngx_log_debug0(NGX_LOG_DEBUG_CORE, log, 0, "recvmsg() returned zero");
        return NGX_ERROR;
    }

    if ((size_t) n < sizeof(ngx_channel_t)) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "recvmsg() returned not enough data: %z", n);
        return NGX_ERROR;
    }

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)

    if (ch->command == NGX_CMD_OPEN_CHANNEL) {

        if (cmsg.cm.cmsg_len < (socklen_t) CMSG_LEN(sizeof(int))) {
            ngx_log_error(NGX_LOG_ALERT, log, 0,
                          "recvmsg() returned too small ancillary data");
            return NGX_ERROR;
        }

        if (cmsg.cm.cmsg_level != SOL_SOCKET || cmsg.cm.cmsg_type != SCM_RIGHTS)
        {
            ngx_log_error(NGX_LOG_ALERT, log, 0,
                          "recvmsg() returned invalid ancillary data "
                          "level %d or type %d",
                          cmsg.cm.cmsg_level, cmsg.cm.cmsg_type);
            return NGX_ERROR;
        }

        /* ch->fd = *(int *) CMSG_DATA(&cmsg.cm); */

        ngx_memcpy(&ch->fd, CMSG_DATA(&cmsg.cm), sizeof(int));
    }

    if (msg.msg_flags & (MSG_TRUNC|MSG_CTRUNC)) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "recvmsg() truncated data");
    }

#else

    if (ch->command == NGX_CMD_OPEN_CHANNEL) {
        if (msg.msg_accrightslen != sizeof(int)) {
            ngx_log_error(NGX_LOG_ALERT, log, 0,
                          "recvmsg() returned no ancillary data");
            return NGX_ERROR;
        }

        ch->fd = fd;
    }

#endif

    return n;
}
```

## ngx_worker_process_exit进程退出

```c
static void
ngx_worker_process_exit(ngx_cycle_t *cycle)
{
    ngx_uint_t         i;
    ngx_connection_t  *c;
    // 执行每个模块的exit_process方法
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->exit_process) {
            cycle->modules[i]->exit_process(cycle);
        }
    }
    // 如果是优雅的关闭连接
    if (ngx_exiting) {
        c = cycle->connections;
        for (i = 0; i < cycle->connection_n; i++) {
            if (c[i].fd != -1
                && c[i].read
                && !c[i].read->accept
                && !c[i].read->channel
                && !c[i].read->resolver)
            {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                              "*%uA open socket #%d left in connection %ui",
                              c[i].number, c[i].fd, i);
                ngx_debug_quit = 1;
            }
        }

        if (ngx_debug_quit) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, 0, "aborting");
            ngx_debug_point();
        }
    }

    /*
     * Copy ngx_cycle->log related data to the special static exit cycle,
     * log, and log file structures enough to allow a signal handler to log.
     * The handler may be called when standard ngx_cycle->log allocated from
     * ngx_cycle->pool is already destroyed.
     */

    ngx_exit_log = *ngx_log_get_file_log(ngx_cycle->log);

    ngx_exit_log_file.fd = ngx_exit_log.file->fd;
    ngx_exit_log.file = &ngx_exit_log_file;
    ngx_exit_log.next = NULL;
    ngx_exit_log.writer = NULL;

    ngx_exit_cycle.log = &ngx_exit_log;
    ngx_exit_cycle.files = ngx_cycle->files;
    ngx_exit_cycle.files_n = ngx_cycle->files_n;
    ngx_cycle = &ngx_exit_cycle;

    ngx_destroy_pool(cycle->pool);

    ngx_log_error(NGX_LOG_NOTICE, ngx_cycle->log, 0, "exit");

    exit(0);
}
```

## ngx_set_shutdown_timer设置关机时间

当配置的show_down字段时，会在优雅的关机时增加一个超时时间，用于加快关机，其逻辑如下：

```c
void
ngx_set_shutdown_timer(ngx_cycle_t *cycle)
{
    ngx_core_conf_t  *ccf;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    if (ccf->shutdown_timeout) {
        ngx_shutdown_event.handler = ngx_shutdown_timer_handler;
        ngx_shutdown_event.data = cycle;
        ngx_shutdown_event.log = cycle->log;
        ngx_shutdown_event.cancelable = 1;
        // 将事件添加到时间驱动中
        ngx_add_timer(&ngx_shutdown_event, ccf->shutdown_timeout);
    }
}

```

其中handler处理函数如下：

```c
static void
ngx_shutdown_timer_handler(ngx_event_t *ev)
{
    ngx_uint_t         i;
    ngx_cycle_t       *cycle;
    ngx_connection_t  *c;

    cycle = ev->data;

    c = cycle->connections;

    for (i = 0; i < cycle->connection_n; i++) {
        // 获取每一个普通的，非accept的和用于进程间通讯的连接
        if (c[i].fd == (ngx_socket_t) -1
            || c[i].read == NULL
            || c[i].read->accept
            || c[i].read->channel
            || c[i].read->resolver)
        {
            continue;
        }

        ngx_log_debug1(NGX_LOG_DEBUG_CORE, ev->log, 0,
                       "*%uA shutdown timeout", c[i].number);
        // 将连接关闭
        c[i].close = 1;
        c[i].error = 1;
        // 执行对应的读事件的处理函数，以加快关机
        c[i].read->handler(c[i].read);
    }
}
```

## ngx_close_listening_sockets关闭正在监听套接字

```c
void
ngx_close_listening_sockets(ngx_cycle_t *cycle)
{
    ngx_uint_t         i;
    ngx_listening_t   *ls;
    ngx_connection_t  *c;

    if (ngx_event_flags & NGX_USE_IOCP_EVENT) {
        return;
    }
    // 设置负载均衡锁相关全局变量
    ngx_accept_mutex_held = 0;
    ngx_use_accept_mutex = 0;
    
    // 遍历每个连接
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {

        c = ls[i].connection;
        
        if (c) {
            // 读事件为活跃状态
            if (c->read->active) {
                // 从事件驱动中删除
                if (ngx_event_flags & NGX_USE_EPOLL_EVENT) {

                    /*
                     * it seems that Linux-2.6.x OpenVZ sends events
                     * for closed shared listening sockets unless
                     * the events was explicitly deleted
                     */
          
                    ngx_del_event(c->read, NGX_READ_EVENT, 0);

                } else {
                    ngx_del_event(c->read, NGX_READ_EVENT, NGX_CLOSE_EVENT);
                }
            }
            // 释放连接
            ngx_free_connection(c);
            // 设置fd为-1
            c->fd = (ngx_socket_t) -1;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_CORE, cycle->log, 0,
                       "close listening %V #%d ", &ls[i].addr_text, ls[i].fd);
        // 关闭套接字
        if (ngx_close_socket(ls[i].fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                          ngx_close_socket_n " %V failed", &ls[i].addr_text);
        }

#if (NGX_HAVE_UNIX_DOMAIN)
        /* 如果是unix域套接字，并且是master进程或单进程模式运行，并且当前没有新的二进制文件被执行，并且不是继承而来的套接字或者是新运行的二进制程序,则删除对应的文件，这里复杂的判断是避免误删 */
        if (ls[i].sockaddr->sa_family == AF_UNIX
            && ngx_process <= NGX_PROCESS_MASTER
            && ngx_new_binary == 0
            && (!ls[i].inherited || ngx_getppid() != ngx_parent))
        {
            u_char *name = ls[i].addr_text.data + sizeof("unix:") - 1;

            if (ngx_delete_file(name) == NGX_FILE_ERROR) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                              ngx_delete_file_n " %s failed", name);
            }
        }

#endif

        ls[i].fd = (ngx_socket_t) -1;
    }

    cycle->listening.nelts = 0;
}
```

## ngx_close_idle_connections关闭空闲连接

```c
void
ngx_close_idle_connections(ngx_cycle_t *cycle)
{
    ngx_uint_t         i;
    ngx_connection_t  *c;

    c = cycle->connections;

    for (i = 0; i < cycle->connection_n; i++) {

        /* THREAD: lock */
        // 连接正在使用，并且是空闲的，则关闭连接，执行对应的读事件
        if (c[i].fd != (ngx_socket_t) -1 && c[i].idle) {
            c[i].close = 1;
            c[i].read->handler(c[i].read);
        }
    }
}
```

## ngx_reopen_files重新打开日志文件

```c
void
ngx_reopen_files(ngx_cycle_t *cycle, ngx_uid_t user)
{
    ngx_fd_t          fd;
    ngx_uint_t        i;
    ngx_list_part_t  *part;
    ngx_open_file_t  *file;
    // 打开的文件都会在open_files中存储
    part = &cycle->open_files.part;
    file = part->elts;
    // 变量所有文件
    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            file = part->elts;
            i = 0;
        }

        if (file[i].name.len == 0) {
            continue;
        }
        // 如果含义flush刷新操作，则执行对应函数
        if (file[i].flush) {
            file[i].flush(&file[i], cycle->log);
        }
        // 重新打开文件
        fd = ngx_open_file(file[i].name.data, NGX_FILE_APPEND,
                           NGX_FILE_CREATE_OR_OPEN, NGX_FILE_DEFAULT_ACCESS);

        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "reopen file \"%s\", old:%d new:%d",
                       file[i].name.data, file[i].fd, fd);

        if (fd == NGX_INVALID_FILE) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          ngx_open_file_n " \"%s\" failed", file[i].name.data);
            continue;
        }

#if !(NGX_WIN32)
        // 设置对应的文件属性
        if (user != (ngx_uid_t) NGX_CONF_UNSET_UINT) {
            ngx_file_info_t  fi;

            if (ngx_file_info(file[i].name.data, &fi) == NGX_FILE_ERROR) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                              ngx_file_info_n " \"%s\" failed",
                              file[i].name.data);

                if (ngx_close_file(fd) == NGX_FILE_ERROR) {
                    ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                                  ngx_close_file_n " \"%s\" failed",
                                  file[i].name.data);
                }

                continue;
            }

            if (fi.st_uid != user) {
                if (chown((const char *) file[i].name.data, user, -1) == -1) {
                    ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                                  "chown(\"%s\", %d) failed",
                                  file[i].name.data, user);

                    if (ngx_close_file(fd) == NGX_FILE_ERROR) {
                        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                                      ngx_close_file_n " \"%s\" failed",
                                      file[i].name.data);
                    }

                    continue;
                }
            }

            if ((fi.st_mode & (S_IRUSR|S_IWUSR)) != (S_IRUSR|S_IWUSR)) {

                fi.st_mode |= (S_IRUSR|S_IWUSR);

                if (chmod((const char *) file[i].name.data, fi.st_mode) == -1) {
                    ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                                  "chmod() \"%s\" failed", file[i].name.data);

                    if (ngx_close_file(fd) == NGX_FILE_ERROR) {
                        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                                      ngx_close_file_n " \"%s\" failed",
                                      file[i].name.data);
                    }

                    continue;
                }
            }
        }
        // 设置执行时关闭
        if (fcntl(fd, F_SETFD, FD_CLOEXEC) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "fcntl(FD_CLOEXEC) \"%s\" failed",
                          file[i].name.data);

            if (ngx_close_file(fd) == NGX_FILE_ERROR) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                              ngx_close_file_n " \"%s\" failed",
                              file[i].name.data);
            }

            continue;
        }
#endif
        // 关闭原文件描述符
        if (ngx_close_file(file[i].fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          ngx_close_file_n " \"%s\" failed",
                          file[i].name.data);
        }
        // 赋值新的文件描述符
        file[i].fd = fd;
    }
    // 重定向标准输出
    (void) ngx_log_redirect_stderr(cycle);
}
```

在使用open打开文件时，使用`O_APPEND`，保证多进程输入不会发送混乱。

在执行日志回滚时，应该先将旧文件移动到新的位置，在向master进程发送reopen信号。这时处理逻辑是，会先将缓冲区的内容写到旧的文件中。然后重新打开文件时发现文件不存在，新建文件，之后再关闭旧的文件描述符，之后使用新的文件描述符。

## ngx_process_events_and_timers事件处理

```c
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    ngx_uint_t  flags;
    ngx_msec_t  timer, delta;
    // 如果设置了时钟触发，则设置timer为未定义
    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;
        flags = 0;

    } else {
        // 设置更新时间，并设置epoll_wait超时时间为最近一个事件将要触发的时间，具体查看事件章节定时器事件
        timer = ngx_event_find_timer();
        flags = NGX_UPDATE_TIME;

#if (NGX_WIN32)

        /* handle signals from master in case of network inactivity */

        if (timer == NGX_TIMER_INFINITE || timer > 500) {
            timer = 500;
        }

#endif
    }
    // 如果使用负载均衡锁
    if (ngx_use_accept_mutex) {
        /* 如果ngx_accept_disabled大于0，则表明当前进程较为繁忙，不再接收连接，并将ngx_accept_disabled减一，直到到非正数才获取连接
        初始化ngx_accept_disabled为0，每个连接事件发生时，处理函数中会根据当前连接数量重新对该值赋值。具体可以查看http处理
        */
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;

        } else {
            // 尝试获取锁，并将监听连接的套接字添加到事件驱动中
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }
            // 如果获取到了锁，则为了加快锁的释放，让其他进程能够获取到锁，所有事件触发函数放到post队列中，延后执行
            if (ngx_accept_mutex_held) {
                flags |= NGX_POST_EVENTS;

            } else {
                // 否则，如果时间是未定义的，则设置下一次epoll_wait的超时事件为获取负载均衡锁的最小间隔时间，保证下次尝试获取负载均衡锁与本次的时间间隔满足要求。
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }
    
    // 如果ngx_posted_next_events队列不为空，则将ngx_posted_next_events队列数据添加到ngx_posted_events队列中
    if (!ngx_queue_empty(&ngx_posted_next_events)) {
        ngx_event_move_posted_next(cycle);
        timer = 0;
    }
    // 记录当前缓存事件
    delta = ngx_current_msec;
    /*
    #define ngx_process_events   ngx_event_actions.process_events
    执行事件模块处理，对应epoll模块的ngx_epoll_process_events方法
    */
    (void) ngx_process_events(cycle, timer, flags);
    // 因为上一步可能会更新时间，判断时间差值
    delta = ngx_current_msec - delta;

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "timer delta: %M", delta);
    // 处理ngx_posted_accept_events队列事件，其中执行待连接请求
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);
    // 如果持有负载均衡锁，则释放，这里并未将ngx_accept_mutex_held置0，而是在下次尝试获取锁时置0
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }
    // 如果时间发生变更，则执行事件红黑树中已经触发的所有事件，详情查看事件章节的定时器事件
    if (delta) {
        ngx_event_expire_timers();
    }
    // 执行ngx_posted_events队列中事件
    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

负载均衡主要通过原子变量`ngx_use_accept_mutex`和ngx_accept_disabled控制。ngx_accept_disabled是一个整数，初始化为0，每次新连接建立时会根据当前连接数量对该值赋值：`ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n`即所有连接数的百分之一减去当前空闲连接。当负载过高时，空闲连接将会减少，当八分之七的连接已经使用时，该值将变成正值，这时循环中将不再获取负载均衡锁，即不再建立新的连接，而是将该值减一，直到到0才继续接收连接。

### ngx_trylock_accept_mutex获取负载均衡锁

该方法会尝试获取负载均衡锁，并将监听套接字对应事件添加到事件驱动中。其逻辑如下：

```c
ngx_int_t
ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
{   
    // 成功获取到负载均衡锁。具体逻辑查看锁机制
    if (ngx_shmtx_trylock(&ngx_accept_mutex)) {

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "accept mutex locked");
        // ngx_accept_events非epoll模块使用的变量，不用考虑，如果当前已经持有负载均衡锁，则直接返回成功
        if (ngx_accept_mutex_held && ngx_accept_events == 0) {
            return NGX_OK;
        }
        // 如果之前没有获取到负载均衡锁，则将监听事件加入到epoll中，如果加入失败则释放锁
        if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
            ngx_shmtx_unlock(&ngx_accept_mutex);
            return NGX_ERROR;
        }

        ngx_accept_events = 0;
        // 设置持有负载均衡锁
        ngx_accept_mutex_held = 1;

        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "accept mutex lock failed: %ui", ngx_accept_mutex_held);
    // 如果没有获取到负载均衡锁，但是记录已经获取到，则将不应该监听的套接字事件从epoll中删除，并表示未持有
    if (ngx_accept_mutex_held) {
        if (ngx_disable_accept_events(cycle, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }

        ngx_accept_mutex_held = 0;
    }

    return NGX_OK;
}
```

注意在每次循环中，如果获取到负载均衡锁了，只会是否锁，并不会将监听事件从epoll中去除。只会在下一次循环中，未获取到负载均衡锁时才从中删除。这应该是处于效率考量的，如果其他进程负载都交高是，某一个负载较低的进程则很可能在多次循环中都能够获取到负载均衡锁，这时采用上述方法就不用每次都执行添加事件和删除事件了。

### ngx_enable_accept_events添加监听套接字对应事件到事件驱动模块

注意不是所有监听套接字都需要使用该方法进行添加。对于不使用负载均衡锁来说，不用该方法。对应设置了端口可复用的套接字来说，会为每一个进程拷贝一份监听套接字，每个进程单独进行监听，操作系统提供负载均衡操作（具体查看事件模块的ngx_event_core_module模块和ngx_events_module模块的介绍）。

其处理逻辑如下：

```c
ngx_int_t
ngx_enable_accept_events(ngx_cycle_t *cycle)
{
    ngx_uint_t         i;
    ngx_listening_t   *ls;
    ngx_connection_t  *c;

    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {

        c = ls[i].connection;
        /*
        对于支持端口复用的监听来说，如果不是为当前进程生成的ls，则对应的connection为空
        对于为当前进程创建的ls结构来说，其读事件已经被加入到事件驱动中，其active为true
        因此这里只是加入之前未被加入需要监听的套接字对应事件
        */
        if (c == NULL || c->read->active) {
            continue;
        }

        if (ngx_add_event(c->read, NGX_READ_EVENT, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}
```

### ngx_disable_accept_events从事件驱动中删除监听套接字对应事件

与添加一样，也是只能删除该删除的，对应端口复用的不能删除：

```c
// all如果为1，则是删除所有时间
static ngx_int_t
ngx_disable_accept_events(ngx_cycle_t *cycle, ngx_uint_t all)
{
    ngx_uint_t         i;
    ngx_listening_t   *ls;
    ngx_connection_t  *c;

    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {

        c = ls[i].connection;

        if (c == NULL || !c->read->active) {
            continue;
        }

#if (NGX_HAVE_REUSEPORT)

        /*
         * do not disable accept on worker's own sockets
         * when disabling accept events due to accept mutex
         */
         // 如果是支持端口复用的，并且不是要删除所有事件，则跳过
        if (ls[i].reuseport && !all) {
            continue;
        }

#endif

        if (ngx_del_event(c->read, NGX_READ_EVENT, NGX_DISABLE_EVENT)
            == NGX_ERROR)
        {
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}
```

### ngx_event_process_posted执行post队列中时间

```c
void
ngx_event_process_posted(ngx_cycle_t *cycle, ngx_queue_t *posted)
{
    ngx_queue_t  *q;
    ngx_event_t  *ev;
    // 变量每个posted队列
    while (!ngx_queue_empty(posted)) {
        // 获取第一个元素
        q = ngx_queue_head(posted);
        // 获取对应的事件
        ev = ngx_queue_data(q, ngx_event_t, queue);

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                      "posted event %p", ev);
        // 从队列中删除当前事件
        ngx_delete_posted_event(ev);
        // 执行事件对应的处理函数
        ev->handler(ev);
    }
}
```

# HTTP请求处理流程

在worker进程启动时会调用事件核心模块，监听网络请求，在接收到请求和首先会执行`ngx_event_accept`监听事件回调方法。下面就按照该方法逐一讲解http处理流程。

## 主要结构

### ngx_http_request_t请求结构

```
struct ngx_http_request_s {
    uint32_t                          signature;         /* "HTTP" */
    // 请求对应的连接
    ngx_connection_t                 *connection;
 
    // 指向存放所有http模块的上下文结构体的指针数组
    void                            **ctx;
    // 执行请求对应的存放main级别配置结构体的指针数组
    void                            **main_conf;
    // 执行请求对应存放srv级别配置结构体的指针数组
    void                            **srv_conf;
    // 执行请求对应存放loc_conf级别配置结构体的指针数组
    void                            **loc_conf;

    /* 在接收完http头部，第一次在业务上处理http请求时，http框架提供的方法是ngx_http_process_request方法。如果该方法无法一次完成请求处理，将会把控制权交到epoll事件驱动中，该请求再次被回调时，将通过ngx_http_request_handler方法来处理，该方法对可读事件的处理就是调用read_event_handler处理请求。 */
    ngx_http_event_handler_pt         read_event_handler;
    /* 与read_event_handler回调方法类似，如果ngx_http_request_handler方法判断为写事件，则调用该函数 */
    ngx_http_event_handler_pt         write_event_handler;

#if (NGX_HTTP_CACHE)
    ngx_http_cache_t                 *cache;
#endif
    
    // upstream机制对应结构
    ngx_http_upstream_t              *upstream;
    ngx_array_t                      *upstream_states;
                                         /* of ngx_http_upstream_state_t */

    ngx_pool_t                       *pool;
    // 接收http请求的内容缓冲区，主要用来接收http头部
    ngx_buf_t                        *header_in;

    // 存储解析http请求的内容
    ngx_http_headers_in_t             headers_in;
    // 发送http相应信息放到该结构中，由架构将该结构序列号http响应包发送给用户
    ngx_http_headers_out_t            headers_out;

    // 接收http请求包体的数据结构
    ngx_http_request_body_t          *request_body;

    // 延迟关闭连接的事件
    time_t                            lingering_time;
    // 请求初始化的事件戳，秒
    time_t                            start_sec;
    // 相对start_sec的毫秒偏移
    ngx_msec_t                        start_msec;

    // 下面9个均为http请求行解析出的信息
    // 请求类型
    ngx_uint_t                        method;
    // http版本
    ngx_uint_t                        http_version;
    // 请求行信息
    ngx_str_t                         request_line;
    // 请求的uri,uri长度不包含args参数部分
    ngx_str_t                         uri;
    // 请求参数
    ngx_str_t                         args;
    // 请求的文件扩展名，如访问"GET /a.txt HTTP/1.1"时，exten为{len:3,data:"txt"}
    ngx_str_t                         exten;
    // 未解码的url原始请求，当uri为"/a b"时，unparsed_uri是"/a%20b"，空格字符做完编码后是%20
    ngx_str_t                         unparsed_uri;

    // 请求方式get，put等
    ngx_str_t                         method_name;
    // http协议版本，http_protocol指向http协议版本字符串起始地址，len成员为协议版本字符串长度
    ngx_str_t                         http_protocol;
    
    ngx_str_t                         schema;

    // 需要方法给客户端的http响应。out中保存着header_out中序列化后的表示http头部的TCP流。
    ngx_chain_t                      *out;
    // 当前请求有可能是用户发送的请求，也可能是派生出的子请求，main标识一系列相关的派生子请求的原始请求，一般通过判断main和当前请求地址是否一致来判断请求是否为用户发送的原始请求。
    ngx_http_request_t               *main;
    // 请求的父请求，父请求不一定是原始请求
    ngx_http_request_t               *parent;
    // 与subrequest相关的结构
    ngx_http_postponed_request_t     *postponed;
    ngx_http_post_subrequest_t       *post_subrequest;
    // 所有子请求通过posted_requests单链表链接起来，通过调用ngx_http_run_posted_request方法来遍历单链表执行子请求。
    ngx_http_posted_request_t        *posted_requests;

    /* 全局的ngx_http_phase_engine_t结构体定义了一个ngx_http_phase_handler_t回调方法组成的数组，phase_handler与数组配合使用，表示请求下次应该执行以phase_handler为下标的回调方法 */
    ngx_int_t                         phase_handler;
    // 表示NGX_HTTP_CONFENT_PHASE阶段提供给HTTP模块处理请求的一种方式，content_handler指向HTTP模块实现的请求处理方法
    ngx_http_handler_pt               content_handler;
    /* NGX_HTTP_ACCESS_PHASE阶段判断用户是否具有访问权限时，通过access_code来传递http模块的handler回调方法返回值。0表示具有权限，否则不具有权限*/
    ngx_uint_t                        access_code;
    
    // 请求对应的变量
    ngx_http_variable_value_t        *variables;

#if (NGX_PCRE)
    ngx_uint_t                        ncaptures;
    int                              *captures;
    u_char                           *captures_data;
#endif
    // 请求速率限制
    size_t                            limit_rate;
    size_t                            limit_rate_after;

    /* used to learn the Apache compatible response length without a header */
    size_t                            header_size;

    // 请求长度
    off_t                             request_length;

    ngx_uint_t                        err_status;
    // 对应连接信息
    ngx_http_connection_t            *http_connection;
    ngx_http_v2_stream_t             *stream;

    ngx_http_log_handler_pt           log_handler;

    // 请求中打开的资源需要在结束时释放，需要把资源释放的方法定义到该成员中
    ngx_http_cleanup_t               *cleanup;

    /* 引用计数。使用subrequest时，依附在该请求上的子请求数目会返回到count上，每增加一个子请求，count计数就会加一，对应任意一个子请求派生的新的子请求，对应的main请求的count都要加一。避免count计数未清零前销毁请求。*/
    unsigned                          count:16;
    unsigned                          subrequests:8;
    // 阻塞标志未，由aio使用
    unsigned                          blocked:8;
    
    // 为1时表示当前请求正在使用异步IO请求
    unsigned                          aio:1;

    unsigned                          http_state:4;

    // 标识uri中存在特殊字符
    /* URI with "/." and on Win32 with "//" */
    unsigned                          complex_uri:1;

    /* URI with "%" */
    unsigned                          quoted_uri:1;

    /* URI with "+" */
    unsigned                          plus_in_uri:1;

    /* URI with " " */
    unsigned                          space_in_uri:1;
    // 是否为无效的请求头
    unsigned                          invalid_header:1;
    
    unsigned                          add_uri_to_alias:1;
    unsigned                          valid_location:1;
    unsigned                          valid_unparsed_uri:1;
    // 为1表示uri发送过uri重写，在NGX_HTTP_POST_REWRITE_PHASE阶段会根据该值来决定是否重新执行rewrite阶段
    unsigned                          uri_changed:1;
    // 该值为uri重写次数，初始化为11，每次发送重写，在NGX_HTTP_POST_REWRITE_PHASE阶段将该值减一，如果减到0，依据需要进行uri重写，则认为配置的rewrite出现循环，报错
    unsigned                          uri_changes:4;
    // 表示请求体存储位置
    // 将主体读取到单个内存缓冲区。 
    unsigned                          request_body_in_single_buf:1;
    // 始终将文本读取到文件中，即使放入内存缓冲区也是如此
    unsigned                          request_body_in_file_only:1;
    //  创建后不要立即取消链接文件。具有此标志的文件可以移动到另一个目录。
    unsigned                          request_body_in_persistent_file:1;
    // 请求完成时取消链接文件。当文件应该移动到另一个目录但由于某种原因未被移动时，这可能很有用。
    unsigned                          request_body_in_clean_file:1;
    // 通过用0660替换默认的0600访问掩码来启用对该文件的组访问。
    unsigned                          request_body_file_group_access:1;
    // - 记录文件错误的严重级别。
    unsigned                          request_body_file_log_level:3;
    // - 无缓冲地读取请求主体。
    unsigned                          request_body_no_buffering:1;

    unsigned                          subrequest_in_memory:1;
    unsigned                          waited:1;

#if (NGX_HTTP_CACHE)
    unsigned                          cached:1;
#endif

#if (NGX_HTTP_GZIP)
    unsigned                          gzip_tested:1;
    unsigned                          gzip_ok:1;
    unsigned                          gzip_vary:1;
#endif

#if (NGX_PCRE)
    unsigned                          realloc_captures:1;
#endif

    unsigned                          proxy:1;
    unsigned                          bypass_cache:1;
    unsigned                          no_cache:1;

    /*
     * instead of using the request context data in
     * ngx_http_limit_conn_module and ngx_http_limit_req_module
     * we use the bit fields in the request structure
     */
    unsigned                          limit_conn_status:2;
    unsigned                          limit_req_status:3;

    unsigned                          limit_rate_set:1;
    unsigned                          limit_rate_after_set:1;

#if 0
    unsigned                          cacheable:1;
#endif

    unsigned                          pipeline:1;
    unsigned                          chunked:1;
    unsigned                          header_only:1;
    unsigned                          expect_trailers:1;
    // 请求是否为keepalive请求
    unsigned                          keepalive:1;
    // 延迟关闭标志。例如，在接收完HTTP头部时如果发现包体存在，该标志会设为1，如果放弃包体，则设为0
    unsigned                          lingering_close:1;
    // 放弃包体标识
    unsigned                          discard_body:1;
    unsigned                          reading_body:1;
    // 为1时表示请求的状态是在做内部跳转
    unsigned                          internal:1;
    
    unsigned                          error_page:1;
    // 记录过滤阶段是否出错
    unsigned                          filter_finalize:1;
    unsigned                          post_action:1;
    unsigned                          request_complete:1;
    unsigned                          request_output:1;
    // 为1时表示响应的header已发送
    unsigned                          header_sent:1;
    unsigned                          expect_tested:1;
    unsigned                          root_tested:1;
    unsigned                          done:1;
    unsigned                          logged:1;
    // 缓冲中是否有待发送内容的标志位
    unsigned                          buffered:4;

    unsigned                          main_filter_need_in_memory:1;
    unsigned                          filter_need_in_memory:1;
    unsigned                          filter_need_temporary:1;
    unsigned                          preserve_body:1;
    unsigned                          allow_ranges:1;
    unsigned                          subrequest_ranges:1;
    unsigned                          single_range:1;
    unsigned                          disable_not_modified:1;
    unsigned                          stat_reading:1;
    unsigned                          stat_writing:1;
    unsigned                          stat_processing:1;

    unsigned                          background:1;
    unsigned                          health_check:1;

    /* used to parse HTTP headers */

    ngx_uint_t                        state;

    ngx_uint_t                        header_hash;
    ngx_uint_t                        lowcase_index;
    u_char                            lowcase_header[NGX_HTTP_LC_HEADER_LEN];

    u_char                           *header_name_start;
    u_char                           *header_name_end;
    u_char                           *header_start;
    u_char                           *header_end;

    /*
     * a memory that can be reused after parsing a request line
     * via ngx_http_ephemeral_t
     */

    u_char                           *uri_start;
    u_char                           *uri_end;
    u_char                           *uri_ext;
    u_char                           *args_start;
    u_char                           *request_start;
    u_char                           *request_end;
    u_char                           *method_end;
    u_char                           *schema_start;
    u_char                           *schema_end;
    u_char                           *host_start;
    u_char                           *host_end;
    u_char                           *port_start;
    u_char                           *port_end;

    unsigned                          http_minor:16;
    unsigned                          http_major:16;
};
```





## 建立连接

```c
void
ngx_event_accept(ngx_event_t *ev)
{
    socklen_t          socklen;
    ngx_err_t          err;
    ngx_log_t         *log;
    ngx_uint_t         level;
    ngx_socket_t       s;
    ngx_event_t       *rev, *wev;
    ngx_sockaddr_t     sa;
    ngx_listening_t   *ls;
    ngx_connection_t  *c, *lc;
    ngx_event_conf_t  *ecf;
#if (NGX_HAVE_ACCEPT4)
    static ngx_uint_t  use_accept4 = 1;
#endif

    // 如果超时，则将所有监听事件条件到epoll中
    if (ev->timedout) {
        if (ngx_enable_accept_events((ngx_cycle_t *) ngx_cycle) != NGX_OK) {
            return;
        }

        ev->timedout = 0;
    }
    // 获取事件模块相关配置
    ecf = ngx_event_get_conf(ngx_cycle->conf_ctx, ngx_event_core_module);
    // 获取是否一次尽可能处理所有连接，对应事件配置的multi_accept
    if (!(ngx_event_flags & NGX_USE_KQUEUE_EVENT)) {
        ev->available = ecf->multi_accept;
    }
    // 获取事件对应的connnect结构
    lc = ev->data;
    // 获取listen结构
    ls = lc->listening;
    ev->ready = 0;

    ngx_log_debug2(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                   "accept on %V, ready: %d", &ls->addr_text, ev->available);
    // 遍历该监听地址上的连接
    do {
        socklen = sizeof(ngx_sockaddr_t);
// accept4相对于accept增加了一个参数，可以设置返回的套接字属性，这里如果使用accept4，则设置返回的套接字为非阻塞
#if (NGX_HAVE_ACCEPT4)
        if (use_accept4) {
            s = accept4(lc->fd, &sa.sockaddr, &socklen, SOCK_NONBLOCK);
        } else {
            s = accept(lc->fd, &sa.sockaddr, &socklen);
        }
#else
        s = accept(lc->fd, &sa.sockaddr, &socklen);
#endif
        // 如果未成功获取到套接字
        if (s == (ngx_socket_t) -1) {
            // 获取错误码
            err = ngx_socket_errno;
            // 如果是重试，则说明accept还未准备好，直接返回
            if (err == NGX_EAGAIN) {
                ngx_log_debug0(NGX_LOG_DEBUG_EVENT, ev->log, err,
                               "accept() not ready");
                return;
            }
            // 设置日志等级
            level = NGX_LOG_ALERT;

            if (err == NGX_ECONNABORTED) {
                level = NGX_LOG_ERR;

            } else if (err == NGX_EMFILE || err == NGX_ENFILE) {
                level = NGX_LOG_CRIT;
            }

#if (NGX_HAVE_ACCEPT4)
            ngx_log_error(level, ev->log, err,
                          use_accept4 ? "accept4() failed" : "accept() failed");
            // 如果不支持accept4
            if (use_accept4 && err == NGX_ENOSYS) {
                use_accept4 = 0;
                ngx_inherited_nonblocking = 0;
                continue;
            }
#else
            ngx_log_error(level, ev->log, err, "accept() failed");
#endif
            // 软件导致的abort
            if (err == NGX_ECONNABORTED) {
                if (ngx_event_flags & NGX_USE_KQUEUE_EVENT) {
                    ev->available--;
                }

                if (ev->available) {
                    continue;
                }
            }
            // 如果进程文件描述符超过最大限制
            if (err == NGX_EMFILE || err == NGX_ENFILE) {
                // 删除epoll中所有监听的套接字
                if (ngx_disable_accept_events((ngx_cycle_t *) ngx_cycle, 1)
                    != NGX_OK)
                {
                    return;
                }
                // 使用负载均衡锁
                if (ngx_use_accept_mutex) {
                    // 如果获取锁了，则释放
                    if (ngx_accept_mutex_held) {
                        ngx_shmtx_unlock(&ngx_accept_mutex);
                        ngx_accept_mutex_held = 0;
                    }
                    // 设置ngx_accept_disabled为1，表示暂时不再接收连接
                    ngx_accept_disabled = 1;

                } else {
                    // 如果未使用负载均衡锁，则将事件添加到定时器事件中，这是程序开始时，如果超时，则将监听连接套接字添加到epoll中
                    ngx_add_timer(ev, ecf->accept_mutex_delay);
                }
            }
            // 未获得正确套接字直接返回
            return;
        }

#if (NGX_STAT_STUB)
        (void) ngx_atomic_fetch_add(ngx_stat_accepted, 1);
#endif
        // 根据连接池使用数量来决定ngx_accept_disabled大小，提供负载均衡，详见woker进程处理
        ngx_accept_disabled = ngx_cycle->connection_n / 8
                              - ngx_cycle->free_connection_n;
        // 从空闲连接中获取一个连接
        c = ngx_get_connection(s, ev->log);
        // 如果没有空闲连接池了，则关闭该连接
        if (c == NULL) {
            if (ngx_close_socket(s) == -1) {
                ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_socket_errno,
                              ngx_close_socket_n " failed");
            }

            return;
        }
        // 设置连接类型 
        c->type = SOCK_STREAM;

#if (NGX_STAT_STUB)
        (void) ngx_atomic_fetch_add(ngx_stat_active, 1);
#endif
        // 根据connection_pool_size配置的大小，设置为连接创建初始大小的内存池
        c->pool = ngx_create_pool(ls->pool_size, ev->log);
        if (c->pool == NULL) {
            ngx_close_accepted_connection(c);
            return;
        }
        // 存储客户端连接信息
        if (socklen > (socklen_t) sizeof(ngx_sockaddr_t)) {
            socklen = sizeof(ngx_sockaddr_t);
        }

        c->sockaddr = ngx_palloc(c->pool, socklen);
        if (c->sockaddr == NULL) {
            ngx_close_accepted_connection(c);
            return;
        }

        ngx_memcpy(c->sockaddr, &sa, socklen);

        log = ngx_palloc(c->pool, sizeof(ngx_log_t));
        if (log == NULL) {
            ngx_close_accepted_connection(c);
            return;
        }

        /* set a blocking mode for iocp and non-blocking mode for others */
        // 如果未使用accept4，则手动设置套接字为非阻塞
        if (ngx_inherited_nonblocking) {
            if (ngx_event_flags & NGX_USE_IOCP_EVENT) {
                if (ngx_blocking(s) == -1) {
                    ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_socket_errno,
                                  ngx_blocking_n " failed");
                    ngx_close_accepted_connection(c);
                    return;
                }
            }

        } else {
            if (!(ngx_event_flags & NGX_USE_IOCP_EVENT)) {
                if (ngx_nonblocking(s) == -1) {
                    ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_socket_errno,
                                  ngx_nonblocking_n " failed");
                    ngx_close_accepted_connection(c);
                    return;
                }
            }
        }

        *log = ls->log;
        // 设置连接对应读写的方法
        c->recv = ngx_recv;
        c->send = ngx_send;
        c->recv_chain = ngx_recv_chain;
        c->send_chain = ngx_send_chain;

        c->log = log;
        c->pool->log = log;

        c->socklen = socklen;
        c->listening = ls;
        // 设置本地监听连接的地址
        c->local_sockaddr = ls->sockaddr;
        c->local_socklen = ls->socklen;

#if (NGX_HAVE_UNIX_DOMAIN)
        if (c->sockaddr->sa_family == AF_UNIX) {
            c->tcp_nopush = NGX_TCP_NOPUSH_DISABLED;
            c->tcp_nodelay = NGX_TCP_NODELAY_DISABLED;
#if (NGX_SOLARIS)
            /* Solaris's sendfilev() supports AF_NCA, AF_INET, and AF_INET6 */
            c->sendfile = 0;
#endif
        }
#endif
        // 设置写事件ready
        rev = c->read;
        wev = c->write;

        wev->ready = 1;

        if (ngx_event_flags & NGX_USE_IOCP_EVENT) {
            rev->ready = 1;
        }
        // 如果事件设置了deferred_accept，则表明请求已经到达（listen中设置属性，只有在连接到达才从accept返回，建立连接，完成三次握手不建立连接），则设置读事件已经ready
        if (ev->deferred_accept) {
            rev->ready = 1;
#if (NGX_HAVE_KQUEUE || NGX_HAVE_EPOLLRDHUP)
            rev->available = 1;
#endif
        }

        rev->log = log;
        wev->log = log;

        /*
         * TODO: MT: - ngx_atomic_fetch_add()
         *             or protection by critical section or light mutex
         *
         * TODO: MP: - allocated in a shared memory
         *           - ngx_atomic_fetch_add()
         *             or protection by critical section or light mutex
         */
        // 连接计数增加
        c->number = ngx_atomic_fetch_add(ngx_connection_counter, 1);

#if (NGX_STAT_STUB)
        (void) ngx_atomic_fetch_add(ngx_stat_handled, 1);
#endif

        // 存储本地连接地址，将网络地址，转换为字符串形式地址
        if (ls->addr_ntop) {
            c->addr_text.data = ngx_pnalloc(c->pool, ls->addr_text_max_len);
            if (c->addr_text.data == NULL) {
                ngx_close_accepted_connection(c);
                return;
            }

            c->addr_text.len = ngx_sock_ntop(c->sockaddr, c->socklen,
                                             c->addr_text.data,
                                             ls->addr_text_max_len, 0);
            if (c->addr_text.len == 0) {
                ngx_close_accepted_connection(c);
                return;
            }
        }

#if (NGX_DEBUG)
        {
        ngx_str_t  addr;
        u_char     text[NGX_SOCKADDR_STRLEN];

        ngx_debug_accepted_connection(ecf, c);

        if (log->log_level & NGX_LOG_DEBUG_EVENT) {
            addr.data = text;
            addr.len = ngx_sock_ntop(c->sockaddr, c->socklen, text,
                                     NGX_SOCKADDR_STRLEN, 1);

            ngx_log_debug3(NGX_LOG_DEBUG_EVENT, log, 0,
                           "*%uA accept: %V fd:%d", c->number, &addr, s);
        }

        }
#endif
        // epoll不会处理
        if (ngx_add_conn && (ngx_event_flags & NGX_USE_EPOLL_EVENT) == 0) {
            if (ngx_add_conn(c) == NGX_ERROR) {
                ngx_close_accepted_connection(c);
                return;
            }
        }

        log->data = NULL;
        log->handler = NULL;
        // 执行监听套接字的处理函数
        ls->handler(c);

        if (ngx_event_flags & NGX_USE_KQUEUE_EVENT) {
            ev->available--;
        }
    // 如果是尽可能多的建立连接，则继续循环处理
    } while (ev->available);
}
```

### 数据交互

其中读取和写入函数如下：

```c
#define ngx_recv             ngx_io.recv
#define ngx_recv_chain       ngx_io.recv_chain
#define ngx_udp_recv         ngx_io.udp_recv
#define ngx_send             ngx_io.send
#define ngx_send_chain       ngx_io.send_chain
```

其中`ngx_io`初始化在`ngx_epoll_init`方法中：

```c
ngx_io = ngx_os_io;
```

其中`ngx_os_io`定义如

```c
typedef struct {
    ngx_recv_pt        recv;
    ngx_recv_chain_pt  recv_chain;
    ngx_recv_pt        udp_recv;
    ngx_send_pt        send;
    ngx_send_pt        udp_send;
    ngx_send_chain_pt  udp_send_chain;
    ngx_send_chain_pt  send_chain;
    ngx_uint_t         flags;
} ngx_os_io_t;

ngx_os_io_t ngx_os_io = {
    ngx_unix_recv,
    ngx_readv_chain,
    ngx_udp_unix_recv,
    ngx_unix_send,
    ngx_udp_unix_send,
    ngx_udp_unix_sendmsg_chain,
    ngx_writev_chain,
    0
};
```



#### 接收数据ngx_unix_recv

```c
// buf为接收数据的起始地址，size为最大长度
ssize_t
ngx_unix_recv(ngx_connection_t *c, u_char *buf, size_t size)
{
    ssize_t       n;
    ngx_err_t     err;
    ngx_event_t  *rev;

    rev = c->read;
// 忽略
#if (NGX_HAVE_KQUEUE)
...

#endif

#if (NGX_HAVE_EPOLLRDHUP)
    /* 事件的available为0表示还未准备好，为1表示为设置了deffered建立连接的事件，为-1表示从epoll中触发的事件，pending_eof表示事件是否终止 */
    if (ngx_event_flags & NGX_USE_EPOLL_EVENT) {
        ngx_log_debug2(NGX_LOG_DEBUG_EVENT, c->log, 0,
                       "recv: eof:%d, avail:%d",
                       rev->pending_eof, rev->available);

        if (rev->available == 0 && !rev->pending_eof) {
            rev->ready = 0;
            return NGX_AGAIN;
        }
    }

#endif

    do {
        // 从套接字中接收数据
        n = recv(c->fd, buf, size, 0);

        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, c->log, 0,
                       "recv: fd:%d %z of %uz", c->fd, n, size);
        // 未接收到数据
        if (n == 0) {
            rev->ready = 0;
            rev->eof = 1;

#if (NGX_HAVE_KQUEUE)

            /*
             * on FreeBSD recv() may return 0 on closed socket
             * even if kqueue reported about available data
             */

            if (ngx_event_flags & NGX_USE_KQUEUE_EVENT) {
                rev->available = 0;
            }

#endif

            return 0;
        }
        // 接收到数据
        if (n > 0) {

#if (NGX_HAVE_KQUEUE)

            ...

#endif

#if (NGX_HAVE_FIONREAD)
            
            if (rev->available >= 0) {
                rev->available -= n;

                /*
                 * negative rev->available means some additional bytes
                 * were received between kernel notification and recv(),
                 * and therefore ev->ready can be safely reset even for
                 * edge-triggered event methods
                 */
                // 负 rev->available 意味着在内核通知和 recv() 之间收到了一些额外的字节，因此即使对于边缘触发的事件方法，ev->ready 也可以安全地重置
                if (rev->available < 0) {
                    rev->available = 0;
                    rev->ready = 0;
                }

                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, c->log, 0,
                               "recv: avail:%d", rev->available);
            // 如果返回的长度和设置的最大长度一致
            } else if ((size_t) n == size) {
                /*
                *#define ngx_socket_nread(s, n)  ioctl(s, FIONREAD, n)
                *获取缓冲区中字节长度，写到available中
                */
                if (ngx_socket_nread(c->fd, &rev->available) == -1) {
                    n = ngx_connection_error(c, ngx_socket_errno,
                                             ngx_socket_nread_n " failed");
                    break;
                }

                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, c->log, 0,
                               "recv: avail:%d", rev->available);
            }

#endif

#if (NGX_HAVE_EPOLLRDHUP)
            // 使用epoll，读取字节小于最大字节长度，且客户端未关闭连接，返回n
            if ((ngx_event_flags & NGX_USE_EPOLL_EVENT)
                && ngx_use_epoll_rdhup)
            {
                if ((size_t) n < size) {
                    if (!rev->pending_eof) {
                        rev->ready = 0;
                    }

                    rev->available = 0;
                }

                return n;
            }

#endif

            if ((size_t) n < size
                && !(ngx_event_flags & NGX_USE_GREEDY_EVENT))
            {
                rev->ready = 0;
            }

            return n;
        }
        // 否则设置错误码
        err = ngx_socket_errno;

        if (err == NGX_EAGAIN || err == NGX_EINTR) {
            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, c->log, err,
                           "recv() not ready");
            n = NGX_AGAIN;

        } else {
            n = ngx_connection_error(c, err, "recv() failed");
            break;
        }
    // 如果是中断的系统调用，则重试
    } while (err == NGX_EINTR);

    rev->ready = 0;

    if (n == NGX_ERROR) {
        rev->error = 1;
    }
    // 返回读取字节数量
    return n;
}
```



## 连接处理

### 数据结构

这里大部分数据结构都已经在解析配置中介绍过了，可以查看对应部分。这里还有一个结构为`ngx_http_connection_t`结构。

#### ngx_http_connection_t

定义如下：

```
typedef struct {
    // 监听地址对应的服务信息。接收到请求后，就能知道客户端请求的地址，可以获取到对应的该结构
    ngx_http_addr_conf_t             *addr_conf;
    // 用户请求的服务对应的http配置，在接收到用户请求后，根据用户请求头来判断是在addr_conf中的哪个服务
    ngx_http_conf_ctx_t              *conf_ctx;

#if (NGX_HTTP_SSL || NGX_COMPAT)
    ngx_str_t                        *ssl_servername;
#if (NGX_PCRE)
    ngx_http_regex_t                 *ssl_servername_regex;
#endif
#endif
    // 使用buf，配合free在请求行或者请求体过大时，按照large_client_header_buffers分配的空间，后续有详细介绍
    ngx_chain_t                      *busy;
    ngx_int_t                         nbusy;
    // 空闲buf
    ngx_chain_t                      *free;
    // 协议标志位
    unsigned                          ssl:1;
    unsigned                          proxy_protocol:1;
} ngx_http_connection_t;
```

### 函数方法ngx_http_init_connection

在建立连接后调用listen结构的handler方法进行处理。handler方法定义在`ngx_http_add_listening`方法中，具体参考配置解析中的监听端口部分。定义的函数为`ngx_http_init_connection`。具体方法逻辑如下：

```c
void
ngx_http_init_connection(ngx_connection_t *c)
{
    ngx_uint_t              i;
    ngx_event_t            *rev;
    struct sockaddr_in     *sin;
    ngx_http_port_t        *port;
    ngx_http_in_addr_t     *addr;
    ngx_http_log_ctx_t     *ctx;
    ngx_http_connection_t  *hc;
#if (NGX_HAVE_INET6)
    struct sockaddr_in6    *sin6;
    ngx_http_in6_addr_t    *addr6;
#endif
    // 构建ngx_http_connection_t
    hc = ngx_pcalloc(c->pool, sizeof(ngx_http_connection_t));
    if (hc == NULL) {
        ngx_http_close_connection(c);
        return;
    }
    
    c->data = hc;

    /* find the server configuration for the address:port */
    // 根据客户端请求的ip+port获取对应的ngx_http_in_addr_t
    port = c->listening->servers;
    // 由于对于存在通配符形式的监听来说，一个port端口可能监听多个ip，如果port新式的地址存在多个，则需要查看请求的ip来确定ip+port形式的地址
    if (port->naddrs > 1) {

        /*
         * there are several addresses on this port and one of them
         * is an "*:port" wildcard so getsockname() in ngx_http_server_addr()
         * is required to determine a server address
         */
        // 获取用户实际请求地址
        if (ngx_connection_local_sockaddr(c, NULL, 0) != NGX_OK) {
            ngx_http_close_connection(c);
            return;
        }
        // 根据地址对hc->addr_conf赋值，即确定实际的请求ip+port
        switch (c->local_sockaddr->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            sin6 = (struct sockaddr_in6 *) c->local_sockaddr;

            addr6 = port->addrs;

            /* the last address is "*" */

            for (i = 0; i < port->naddrs - 1; i++) {
                if (ngx_memcmp(&addr6[i].addr6, &sin6->sin6_addr, 16) == 0) {
                    break;
                }
            }

            hc->addr_conf = &addr6[i].conf;

            break;
#endif

        default: /* AF_INET */
            sin = (struct sockaddr_in *) c->local_sockaddr;

            addr = port->addrs;

            /* the last address is "*" */

            for (i = 0; i < port->naddrs - 1; i++) {
                if (addr[i].addr == sin->sin_addr.s_addr) {
                    break;
                }
            }

            hc->addr_conf = &addr[i].conf;

            break;
        }

    } else {
        // 如果port监听只有一个IP地址，则addr_conf则为对应的ip+port的地址
        switch (c->local_sockaddr->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            addr6 = port->addrs;
            hc->addr_conf = &addr6[0].conf;
            break;
#endif

        default: /* AF_INET */
            addr = port->addrs;
            hc->addr_conf = &addr[0].conf;
            break;
        }
    }

    /* the default server configuration for the address:port */
    // 由于一个ip+port的地址下可能存在多个服务，这里先使用ip+port的默认服务对conf_ctx赋值，conf_ctx对应服务下的http配置
    hc->conf_ctx = hc->addr_conf->default_server->ctx;

    ctx = ngx_palloc(c->pool, sizeof(ngx_http_log_ctx_t));
    if (ctx == NULL) {
        ngx_http_close_connection(c);
        return;
    }

    ctx->connection = c;
    ctx->request = NULL;
    ctx->current_request = NULL;

    c->log->connection = c->number;
    c->log->handler = ngx_http_log_error;
    c->log->data = ctx;
    c->log->action = "waiting for request";

    c->log_error = NGX_ERROR_INFO;
    // 设置对应的读事件，等待请求
    rev = c->read;
    rev->handler = ngx_http_wait_request_handler;
    // 写事件为无处理
    c->write->handler = ngx_http_empty_handler;

#if (NGX_HTTP_V2)
    if (hc->addr_conf->http2) {
        rev->handler = ngx_http_v2_init;
    }
#endif

#if (NGX_HTTP_SSL)
    {
    ngx_http_ssl_srv_conf_t  *sscf;

    sscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_ssl_module);

    if (sscf->enable || hc->addr_conf->ssl) {
        hc->ssl = 1;
        c->log->action = "SSL handshaking";
        rev->handler = ngx_http_ssl_handshake;
    }
    }
#endif

    // 如果IP+port的地址开启了proxy_protocol协议支持
    if (hc->addr_conf->proxy_protocol) {
        hc->proxy_protocol = 1;
        c->log->action = "reading PROXY protocol";
    }
    // 如果读事件已经准备完成，则直接执行处理函数，对于设置了deferred的地址来说，会直到用户请求下发才建立连接，此时读事件是已经ready的
    if (rev->ready) {
        /* the deferred accept(), iocp */
        // 如果使用负载均衡锁，则应该加速锁的释放，将事件添加到post队列中
        if (ngx_use_accept_mutex) {
            ngx_post_event(rev, &ngx_posted_events);
            return;
        }

        rev->handler(rev);
        return;
    }
    // 将事件添加到事件红黑树中，设置超时事件为设置的post_accept_timeout时间，即最大的连接后到请求下发之间的超时时间
    ngx_add_timer(rev, c->listening->post_accept_timeout);
    // 设置连接为可复用的
    ngx_reusable_connection(c, 1);
    // 将事件添加进入epoll事件驱动中
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        ngx_http_close_connection(c);
        return;
    }
}
```

#### 获取请求地址

由于对应通配符监听来说，一个监听端口上可能监听了多个ip+port的地址，这时需要获取用户实际请求的ip地址，来获得对应的配置，函数方法为`ngx_connection_local_sockaddr`：

```c
ngx_int_t
ngx_connection_local_sockaddr(ngx_connection_t *c, ngx_str_t *s,
    ngx_uint_t port)
{
    socklen_t             len;
    ngx_uint_t            addr;
    ngx_sockaddr_t        sa;
    struct sockaddr_in   *sin;
#if (NGX_HAVE_INET6)
    ngx_uint_t            i;
    struct sockaddr_in6  *sin6;
#endif

    addr = 0;
    // 存在套接字地址
    if (c->local_socklen) {
        switch (c->local_sockaddr->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            sin6 = (struct sockaddr_in6 *) c->local_sockaddr;
            // 判断是否为通配符地址，如果不为通配符地址，则addr不为0
            for (i = 0; addr == 0 && i < 16; i++) {
                addr |= sin6->sin6_addr.s6_addr[i];
            }

            break;
#endif

#if (NGX_HAVE_UNIX_DOMAIN)
        case AF_UNIX:
            addr = 1;
            break;
#endif

        default: /* AF_INET */
            sin = (struct sockaddr_in *) c->local_sockaddr;
            // 如果为非通配符地址，则addr不为0
            addr = sin->sin_addr.s_addr;
            break;
        }
    }

    // 如果为通配符地址，则使用getsockname获取用户实际请求的套接字地址
    if (addr == 0) {

        len = sizeof(ngx_sockaddr_t);
        // 获取用户实际请求的套接字地址
        if (getsockname(c->fd, &sa.sockaddr, &len) == -1) {
            ngx_connection_error(c, ngx_socket_errno, "getsockname() failed");
            return NGX_ERROR;
        }

        c->local_sockaddr = ngx_palloc(c->pool, len);
        if (c->local_sockaddr == NULL) {
            return NGX_ERROR;
        }

        ngx_memcpy(c->local_sockaddr, &sa, len);

        c->local_socklen = len;
    }

    if (s == NULL) {
        return NGX_OK;
    }
    // 如果需要获取点分十进制形式地址长读，则执行计算。
    s->len = ngx_sock_ntop(c->local_sockaddr, c->local_socklen,
                           s->data, s->len, port);

    return NGX_OK;
}
```

关于套接字地址相关信息，可以查看如下文章：[套接字地址](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%AF%BB%E5%9D%80)。

## ngx_http_wait_request_handler获取请求

### 相关结构

#### ngx_proxy_protocol_s

`ngx_proxy_protocol_s`结构用于存储`proxy_protocol`协议的相关信息。proxy protocol是HAProxy的作者Willy Tarreau于2010年开发和设计的一个Internet协议，通过为tcp添加一个很小的头信息，来方便的传递客户端信息（协议栈、源IP、目的IP、源端口、目的端口等)，在网络情况复杂又需要获取用户真实IP时非常有用。其本质是在三次握手结束后由代理在连接中插入了一个携带了原始连接四元组信息的数据包。

proxy protocol的接收端必须在接收到完整有效的 proxy protocol 头部后才能开始处理连接数据。因此对于服务器的同一个监听端口，不存在兼容带proxy protocol包的连接和不带proxy protocol包的连接。如果服务器接收到的第一个数据包不符合proxy protocol的格式，那么服务器会直接终止连接。

传输数据格式为：

```http
PROXY TCP4 202.112.144.236 10.210.12.10 5678 80\r\n
PROXY TCP6 2001:da8:205::100 2400:89c0:2110:1::21 6324 80\r\n
PROXY UKNOWN\r\n
```

存储相应数据的结构为：

```c
struct ngx_proxy_protocol_s {
    ngx_str_t           src_addr;
    ngx_str_t           dst_addr;
    in_port_t           src_port;
    in_port_t           dst_port;
};
```



### 处理方法

指向完成对连接的处理后，将会等待用户下发请求，对应事件处理函数为：

```c
static void
ngx_http_wait_request_handler(ngx_event_t *rev)
{
    u_char                    *p;
    size_t                     size;
    ssize_t                    n;
    ngx_buf_t                 *b;
    ngx_connection_t          *c;
    ngx_http_connection_t     *hc;
    ngx_http_core_srv_conf_t  *cscf;

    c = rev->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "http wait request handler");
    // 事件超时，表示事件超时，释放事件及连接
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        ngx_http_close_connection(c);
        return;
    }

    if (c->close) {
        ngx_http_close_connection(c);
        return;
    }
    // 获取ngx_http_connection_t结构
    hc = c->data;
    // 获取对应的http核心配置
    cscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_core_module);
    // 客户端请求头缓存默认大小
    size = cscf->client_header_buffer_size;
    // 分配buf
    b = c->buffer;

    if (b == NULL) {
        b = ngx_create_temp_buf(c->pool, size);
        if (b == NULL) {
            ngx_http_close_connection(c);
            return;
        }

        c->buffer = b;

    } else if (b->start == NULL) {

        b->start = ngx_palloc(c->pool, size);
        if (b->start == NULL) {
            ngx_http_close_connection(c);
            return;
        }

        b->pos = b->start;
        b->last = b->start;
        b->end = b->last + size;
    }
    // 接收用户请求，详见建立连接的介绍
    n = c->recv(c, b->last, size);
    // 如果返回为再次请求，则说明客户还未传输请求
    if (n == NGX_AGAIN) {
        // 如果事件未被添加进入事件红黑树，则将其添加进红黑树，这里post_accept_timeout对应client_header_timeout配置
        if (!rev->timer_set) {
            ngx_add_timer(rev, c->listening->post_accept_timeout);
            ngx_reusable_connection(c, 1);
        }
        // 将事件添加到epoll事件驱动中
        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            ngx_http_close_connection(c);
            return;
        }

        /*
         * We are trying to not hold c->buffer's memory for an idle connection.
         */
        // 释放buf空间，减少内存使用，实际用户请求下发后再分配空间
        if (ngx_pfree(c->pool, b->start) == NGX_OK) {
            b->start = NULL;
        }

        return;
    }
    // 如果接收信息错误，则关闭连接
    if (n == NGX_ERROR) {
        ngx_http_close_connection(c);
        return;
    }
    // 未收到数据，关闭连接
    if (n == 0) {
        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                      "client closed connection");
        ngx_http_close_connection(c);
        return;
    }
    // 计算获取到的数据长度
    b->last += n;
    // 如果连接支持proxy_protocol，则解析proxy_protocol协议传输信息
    if (hc->proxy_protocol) {
        hc->proxy_protocol = 0;

        p = ngx_proxy_protocol_read(c, b->pos, b->last);

        if (p == NULL) {
            ngx_http_close_connection(c);
            return;
        }

        b->pos = p;
        // 如果只有proxy_protocol协议信息而没有请求，则将读事件添加进入post事件中
        if (b->pos == b->last) {
            c->log->action = "waiting for request";
            b->pos = b->start;
            b->last = b->start;
            ngx_post_event(rev, &ngx_posted_events);
            return;
        }
    }

    c->log->action = "reading client request line";
    // 设置连接为不可复用
    ngx_reusable_connection(c, 0);
    // 创建请求结构
    c->data = ngx_http_create_request(c);
    if (c->data == NULL) {
        ngx_http_close_connection(c);
        return;
    }
    // 设置读事件为处理请求行，执行处理请求行
    rev->handler = ngx_http_process_request_line;
    ngx_http_process_request_line(rev);
}
```

#### 创建请求结构

```c
ngx_http_request_t *
ngx_http_create_request(ngx_connection_t *c)
{
    ngx_http_request_t        *r;
    ngx_http_log_ctx_t        *ctx;
    ngx_http_core_loc_conf_t  *clcf;
    // 创建请求
    r = ngx_http_alloc_request(c);
    if (r == NULL) {
        return NULL;
    }
    // 连接上请求数量加一
    c->requests++;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 设置日志信息
    ngx_set_connection_log(c, clcf->error_log);

    ctx = c->log->data;
    ctx->request = r;
    ctx->current_request = r;

#if (NGX_STAT_STUB)
    (void) ngx_atomic_fetch_add(ngx_stat_reading, 1);
    r->stat_reading = 1;
    (void) ngx_atomic_fetch_add(ngx_stat_requests, 1);
#endif

    return r;
}
```

```c
static ngx_http_request_t *
ngx_http_alloc_request(ngx_connection_t *c)
{
    ngx_pool_t                 *pool;
    ngx_time_t                 *tp;
    ngx_http_request_t         *r;
    ngx_http_connection_t      *hc;
    ngx_http_core_srv_conf_t   *cscf;
    ngx_http_core_main_conf_t  *cmcf;

    hc = c->data;
    // 获取http的核心模块解析结构
    cscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_core_module);
    // 为请求分配缓存池，对应request_pool_size配置
    pool = ngx_create_pool(cscf->request_pool_size, c->log);
    if (pool == NULL) {
        return NULL;
    }
    // 分配缓存池结构内存
    r = ngx_pcalloc(pool, sizeof(ngx_http_request_t));
    if (r == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    r->pool = pool;
    // 设置http_connectio对应hc
    r->http_connection = hc;
    r->signature = NGX_HTTP_MODULE;
    r->connection = c;
    // 初始化配置项，这里只是临时初始化，使用默认的服务，后续接收请求后可能会进行变更
    r->main_conf = hc->conf_ctx->main_conf;
    r->srv_conf = hc->conf_ctx->srv_conf;
    r->loc_conf = hc->conf_ctx->loc_conf;
    // 设置对应的读事假处理方法
    r->read_event_handler = ngx_http_block_reading;
    // 设置请求信息
    r->header_in = hc->busy ? hc->busy->buf : c->buffer;
    // 初始化响应结构
    if (ngx_list_init(&r->headers_out.headers, r->pool, 20,
                      sizeof(ngx_table_elt_t))
        != NGX_OK)
    {
        ngx_destroy_pool(r->pool);
        return NULL;
    }
    // 初始化响应结构
    if (ngx_list_init(&r->headers_out.trailers, r->pool, 4,
                      sizeof(ngx_table_elt_t))
        != NGX_OK)
    {
        ngx_destroy_pool(r->pool);
        return NULL;
    }

    r->ctx = ngx_pcalloc(r->pool, sizeof(void *) * ngx_http_max_module);
    if (r->ctx == NULL) {
        ngx_destroy_pool(r->pool);
        return NULL;
    }

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    // 分配变量存储地址
    r->variables = ngx_pcalloc(r->pool, cmcf->variables.nelts
                                        * sizeof(ngx_http_variable_value_t));
    if (r->variables == NULL) {
        ngx_destroy_pool(r->pool);
        return NULL;
    }

#if (NGX_HTTP_SSL)
    if (c->ssl) {
        r->main_filter_need_in_memory = 1;
    }
#endif
    // 对额外信息进行初始化
    r->main = r;
    r->count = 1;

    tp = ngx_timeofday();
    r->start_sec = tp->sec;
    r->start_msec = tp->msec;

    r->method = NGX_HTTP_UNKNOWN;
    r->http_version = NGX_HTTP_VERSION_10;

    r->headers_in.content_length_n = -1;
    r->headers_in.keep_alive_n = -1;
    r->headers_out.content_length_n = -1;
    r->headers_out.last_modified_time = -1;

    r->uri_changes = NGX_HTTP_MAX_URI_CHANGES + 1;
    r->subrequests = NGX_HTTP_MAX_SUBREQUESTS + 1;

    r->http_state = NGX_HTTP_READING_REQUEST_STATE;

    r->log_handler = ngx_http_log_error_handler;

    return r;
}
```

## 处理请求行

### 相关结构

#### ngx_http_headers_in_t

`ngx_http_headers_in_t`结构存储解析后的请求头。定义如下：

```c
typedef struct {
    // 存储header的list，所有解析过的HTTP头部都在headers链表中
    ngx_list_t                        headers;
    
    // 每个成员均为RFC2616规范中定义的http头部，实际指向headers链表中的成员，当为NULL时，表示还未解析到对应的头部
    ngx_table_elt_t                  *host;
    ngx_table_elt_t                  *connection;
    ngx_table_elt_t                  *if_modified_since;
    ngx_table_elt_t                  *if_unmodified_since;
    ngx_table_elt_t                  *if_match;
    ngx_table_elt_t                  *if_none_match;
    ngx_table_elt_t                  *user_agent;
    ngx_table_elt_t                  *referer;
    ngx_table_elt_t                  *content_length;
    ngx_table_elt_t                  *content_range;
    ngx_table_elt_t                  *content_type;

    ngx_table_elt_t                  *range;
    ngx_table_elt_t                  *if_range;

    ngx_table_elt_t                  *transfer_encoding;
    ngx_table_elt_t                  *te;
    ngx_table_elt_t                  *expect;
    ngx_table_elt_t                  *upgrade;

#if (NGX_HTTP_GZIP || NGX_HTTP_HEADERS)
    ngx_table_elt_t                  *accept_encoding;
    ngx_table_elt_t                  *via;
#endif

    ngx_table_elt_t                  *authorization;

    ngx_table_elt_t                  *keep_alive;

#if (NGX_HTTP_X_FORWARDED_FOR)
    ngx_array_t                       x_forwarded_for;
#endif

#if (NGX_HTTP_REALIP)
    ngx_table_elt_t                  *x_real_ip;
#endif

#if (NGX_HTTP_HEADERS)
    ngx_table_elt_t                  *accept;
    ngx_table_elt_t                  *accept_language;
#endif

#if (NGX_HTTP_DAV)
    ngx_table_elt_t                  *depth;
    ngx_table_elt_t                  *destination;
    ngx_table_elt_t                  *overwrite;
    ngx_table_elt_t                  *date;
#endif

    ngx_str_t                         user;
    ngx_str_t                         passwd;

    ngx_array_t                       cookies;
    
    // 请求行中如果存在host，则存储在server中
    ngx_str_t                         server;
    // 根据content_length计算出的HTTP包体大小
    off_t                             content_length_n;
    time_t                            keep_alive_n;
    
    // 连接类型
    unsigned                          connection_type:2;
    unsigned                          chunked:1;
    // 浏览器类型
    unsigned                          msie:1;
    unsigned                          msie6:1;
    unsigned                          opera:1;
    unsigned                          gecko:1;
    unsigned                          chrome:1;
    unsigned                          safari:1;
    unsigned                          konqueror:1;
} ngx_http_headers_in_t;
```



### 执行方法

```c
static void
ngx_http_process_request_line(ngx_event_t *rev)
{
    ssize_t              n;
    ngx_int_t            rc, rv;
    ngx_str_t            host;
    ngx_connection_t    *c;
    ngx_http_request_t  *r;

    c = rev->data;
    r = c->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0,
                   "http process request line");
    // 如果为已超时事件，则关闭请求，直接返回
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        c->timedout = 1;
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    rc = NGX_AGAIN;
    // 由于是数据流，因此要循环读取
    for ( ;; ) {
        // 如果是需要再次读取
        if (rc == NGX_AGAIN) {
            // 执行读取请求头
            n = ngx_http_read_request_header(r);
            // 如果返回是需要重复读取（还未下发请求），或者出现错误，则跳出循环
            if (n == NGX_AGAIN || n == NGX_ERROR) {
                break;
            }
        }
        // 对当前已经读取到的内容执行解析
        rc = ngx_http_parse_request_line(r, r->header_in);
        // 完成解析的处理，即请求已经完全下发，并且解析正常
        if (rc == NGX_OK) {
            // 设置请求头部相关信息
            /* the request line has been parsed successfully */

            r->request_line.len = r->request_end - r->request_start;
            r->request_line.data = r->request_start;
            r->request_length = r->header_in->pos - r->request_start;

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                           "http request line: \"%V\"", &r->request_line);

            r->method_name.len = r->method_end - r->request_start + 1;
            r->method_name.data = r->request_line.data;

            if (r->http_protocol.data) {
                r->http_protocol.len = r->request_end - r->http_protocol.data;
            }
            // 解析uri
            if (ngx_http_process_request_uri(r) != NGX_OK) {
                break;
            }

            if (r->schema_end) {
                r->schema.len = r->schema_end - r->schema_start;
                r->schema.data = r->schema_start;
            }
            // 解析到了请求服务
            if (r->host_end) {

                host.len = r->host_end - r->host_start;
                host.data = r->host_start;
                // 规范请求的host名
                rc = ngx_http_validate_host(&host, r->pool, 0);
                // host名存在问题
                if (rc == NGX_DECLINED) {
                    ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                  "client sent invalid host in request line");
                    ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
                    break;
                }

                if (rc == NGX_ERROR) {
                    ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                    break;
                }
                // 根据请求的host获取对应的服务，有可能不是使用host获取到的，而是使用的默认server，详细信息查看ngx_http_set_virtual_server函数 
                if (ngx_http_set_virtual_server(r, &host) == NGX_ERROR) {
                    break;
                }
                // 如果获取正常获取到server块
                r->headers_in.server = host;
            }
            // 如果http版本小于10，并且正确获取到了请求的服务，则执行对请求的处理
            if (r->http_version < NGX_HTTP_VERSION_10) {

                if (r->headers_in.server.len == 0
                    && ngx_http_set_virtual_server(r, &r->headers_in.server)
                       == NGX_ERROR)
                {
                    break;
                }

                ngx_http_process_request(r);
                break;
            }

            // 初始化请求头结构
            if (ngx_list_init(&r->headers_in.headers, r->pool, 20,
                              sizeof(ngx_table_elt_t))
                != NGX_OK)
            {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }

            c->log->action = "reading client request headers";
            // 解析请求头，设置读事件为处理请求头
            rev->handler = ngx_http_process_request_headers;
            ngx_http_process_request_headers(rev);

            break;
        }
        // 如果返回不为again和ok，则说名存在错误
        if (rc != NGX_AGAIN) {

            /* there was error while a request line parsing */

            ngx_log_error(NGX_LOG_INFO, c->log, 0,
                          ngx_http_client_errors[rc - NGX_HTTP_CLIENT_ERROR]);
            // http版本错误。执行终止请求
            if (rc == NGX_HTTP_PARSE_INVALID_VERSION) {
                ngx_http_finalize_request(r, NGX_HTTP_VERSION_NOT_SUPPORTED);

            } else {
                ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
            }

            break;
        }

        /* NGX_AGAIN: a request line parsing is still incomplete */
        // 如果存储请求的buf已经用完，说明请求的长度过长，需要再分配空间
        if (r->header_in->pos == r->header_in->end) {
            // 再分配存储请求行的空间
            rv = ngx_http_alloc_large_header_buffer(r, 1);
            // 错误，则关闭请求
            if (rv == NGX_ERROR) {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }

            if (rv == NGX_DECLINED) {
                r->request_line.len = r->header_in->end - r->request_start;
                r->request_line.data = r->request_start;

                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                              "client sent too long URI");
                ngx_http_finalize_request(r, NGX_HTTP_REQUEST_URI_TOO_LARGE);
                break;
            }
        }
    }
    // 执行子请求的方法
    ngx_http_run_posted_requests(c);
}
```



#### 关闭请求ngx_http_close_request

```c
static void
ngx_http_close_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_connection_t  *c;
    // 获取用户下发的请求
    r = r->main;
    // 获取请求对应连接
    c = r->connection;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http request count:%d blk:%d", r->count, r->blocked);
    
    if (r->count == 0) {
        ngx_log_error(NGX_LOG_ALERT, c->log, 0, "http request count is zero");
    }

    // 将count减小1
    r->count--;
    // 如果还有count或者阻塞，则直接返回
    if (r->count || r->blocked) {
        return;
    }

#if (NGX_HTTP_V2)
    if (r->stream) {
        ngx_http_v2_close_stream(r->stream, rc);
        return;
    }
#endif
    // 释放请求
    ngx_http_free_request(r, rc);
    ngx_http_close_connection(c);
}
```

释放关闭请求的逻辑如下：

```c
void
ngx_http_free_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_log_t                 *log;
    ngx_pool_t                *pool;
    struct linger              linger;
    ngx_http_cleanup_t        *cln;
    ngx_http_log_ctx_t        *ctx;
    ngx_http_core_loc_conf_t  *clcf;

    log = r->connection->log;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, log, 0, "http close request");

    if (r->pool == NULL) {
        ngx_log_error(NGX_LOG_ALERT, log, 0, "http request already closed");
        return;
    }

    cln = r->cleanup;
    r->cleanup = NULL;
    // 执行所有注册到cleanup上的销毁函数
    while (cln) {
        if (cln->handler) {
            cln->handler(cln->data);
        }

        cln = cln->next;
    }

#if (NGX_STAT_STUB)

    if (r->stat_reading) {
        (void) ngx_atomic_fetch_add(ngx_stat_reading, -1);
    }

    if (r->stat_writing) {
        (void) ngx_atomic_fetch_add(ngx_stat_writing, -1);
    }

#endif

    if (rc > 0 && (r->headers_out.status == 0 || r->connection->sent == 0)) {
        r->headers_out.status = rc;
    }

    if (!r->logged) {
        log->action = "logging request";

        ngx_http_log_request(r);
    }

    log->action = "closing request";
    // 如果连接超时
    if (r->connection->timedout) {
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
        // 设置了reset_timedout_connection
        if (clcf->reset_timedout_connection) {
            linger.l_onoff = 1;
            linger.l_linger = 0;
            // 如果还有为发送报文而套接字关闭时，延迟关闭
            if (setsockopt(r->connection->fd, SOL_SOCKET, SO_LINGER,
                           (const void *) &linger, sizeof(struct linger)) == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, log, ngx_socket_errno,
                              "setsockopt(SO_LINGER) failed");
            }
        }
    }

    /* the various request strings were allocated from r->pool */
    ctx = log->data;
    ctx->request = NULL;

    r->request_line.len = 0;

    r->connection->destroyed = 1;

    /*
     * Setting r->pool to NULL will increase probability to catch double close
     * of request since the request object is allocated from its own pool.
     */

    pool = r->pool;
    r->pool = NULL;

    ngx_destroy_pool(pool);
}
```

#### 读取请求行ngx_http_read_request_header

```c
static ssize_t
ngx_http_read_request_header(ngx_http_request_t *r)
{
    ssize_t                    n;
    ngx_event_t               *rev;
    ngx_connection_t          *c;
    ngx_http_core_srv_conf_t  *cscf;

    c = r->connection;
    rev = c->read;
    // 如果header_in中还有未处理完成的字符，则直接返回
    n = r->header_in->last - r->header_in->pos;

    if (n > 0) {
        return n;
    }
    // 如果读取是ready的，则调用事件的读取方法读取
    if (rev->ready) {
        n = c->recv(c, r->header_in->last,
                    r->header_in->end - r->header_in->last);
    } else {
        // 否则，表示还未准备好，需要下次轮询
        n = NGX_AGAIN;
    }
    // 如果还未准备好
    if (n == NGX_AGAIN) {
        // 如果还未在事件红黑树中，则添加进入事件红黑树中，并设置超时时间，对应client_header_timeout配置
        if (!rev->timer_set) {
            cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
            ngx_add_timer(rev, cscf->client_header_timeout);
        }
        // 将事件添加到epoll中
        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return NGX_ERROR;
        }

        return NGX_AGAIN;
    }

    if (n == 0) {
        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                      "client prematurely closed connection");
    }
    // 读取出错
    if (n == 0 || n == NGX_ERROR) {
        c->error = 1;
        c->log->action = "reading client request headers";
        // 结束请求
        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        return NGX_ERROR;
    }
    // 设置读取到的位置
    r->header_in->last += n;

    return n;
}
```

#### 解析请求ngx_http_parse_request_line

解析http请求头需要参考http请求协议查看，可以参考[http协议](http://www.yinkuiwang.cn/2019/12/18/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C-%E8%87%AA%E9%A1%B6%E5%90%91%E4%B8%8B%E6%96%B9%E6%B3%95/#Web%E5%BA%94%E7%94%A8%E4%B8%8EHTTP%E5%8D%8F%E8%AE%AE)。

其中nginx支持了scheam url，对于schema url来说，传递的url字段为：

```url
schema://host:port/uri
// explame
http://www.baidu.com:8000/test/index?params=test
```

这里`shema`为Schema协议名称，`host`为地址，`port`为端口号，后面的为`uri`。

```c
ngx_int_t
ngx_http_parse_request_line(ngx_http_request_t *r, ngx_buf_t *b)
{
    u_char  c, ch, *p, *m;
    // 记录解析状态的状态码
    enum {
        sw_start = 0,
        sw_method,
        sw_spaces_before_uri,
        sw_schema,
        sw_schema_slash,
        sw_schema_slash_slash,
        sw_host_start,
        sw_host,
        sw_host_end,
        sw_host_ip_literal,
        sw_port,
        sw_host_http_09,
        sw_after_slash_in_uri,
        sw_check_uri,
        sw_check_uri_http_09,
        sw_uri,
        sw_http_09,
        sw_http_H,
        sw_http_HT,
        sw_http_HTT,
        sw_http_HTTP,
        sw_first_major_digit,
        sw_major_digit,
        sw_first_minor_digit,
        sw_minor_digit,
        sw_spaces_after_digit,
        sw_almost_done
    } state;

    state = r->state;
    // 遍历recv到的数据
    for (p = b->pos; p < b->last; p++) {
        ch = *p;

        switch (state) {

        /* HTTP methods: GET, HEAD, POST */
        // 开始
        case sw_start:
            // 设置请求开始
            r->request_start = p;
            
            if (ch == CR || ch == LF) {
                break;
            }
            // 限制允许出现的字节
            if ((ch < 'A' || ch > 'Z') && ch != '_' && ch != '-') {
                return NGX_HTTP_PARSE_INVALID_METHOD;
            }
            // 转移到获取方法字段
            state = sw_method;
            break;

        case sw_method:
            // 直到获取到方法字段的结尾才进行处理
            if (ch == ' ') {
                // 获取结束位置
                r->method_end = p - 1;
                // 获取开始位置
                m = r->request_start;
                // 根据字节匹配是何种方法
                switch (p - m) {

                case 3:
                    if (ngx_str3_cmp(m, 'G', 'E', 'T', ' ')) {
                        r->method = NGX_HTTP_GET;
                        break;
                    }

                    if (ngx_str3_cmp(m, 'P', 'U', 'T', ' ')) {
                        r->method = NGX_HTTP_PUT;
                        break;
                    }

                    break;

                case 4:
                    if (m[1] == 'O') {

                        if (ngx_str3Ocmp(m, 'P', 'O', 'S', 'T')) {
                            r->method = NGX_HTTP_POST;
                            break;
                        }

                        if (ngx_str3Ocmp(m, 'C', 'O', 'P', 'Y')) {
                            r->method = NGX_HTTP_COPY;
                            break;
                        }

                        if (ngx_str3Ocmp(m, 'M', 'O', 'V', 'E')) {
                            r->method = NGX_HTTP_MOVE;
                            break;
                        }

                        if (ngx_str3Ocmp(m, 'L', 'O', 'C', 'K')) {
                            r->method = NGX_HTTP_LOCK;
                            break;
                        }

                    } else {

                        if (ngx_str4cmp(m, 'H', 'E', 'A', 'D')) {
                            r->method = NGX_HTTP_HEAD;
                            break;
                        }
                    }

                    break;

                case 5:
                    if (ngx_str5cmp(m, 'M', 'K', 'C', 'O', 'L')) {
                        r->method = NGX_HTTP_MKCOL;
                        break;
                    }

                    if (ngx_str5cmp(m, 'P', 'A', 'T', 'C', 'H')) {
                        r->method = NGX_HTTP_PATCH;
                        break;
                    }

                    if (ngx_str5cmp(m, 'T', 'R', 'A', 'C', 'E')) {
                        r->method = NGX_HTTP_TRACE;
                        break;
                    }

                    break;

                case 6:
                    if (ngx_str6cmp(m, 'D', 'E', 'L', 'E', 'T', 'E')) {
                        r->method = NGX_HTTP_DELETE;
                        break;
                    }

                    if (ngx_str6cmp(m, 'U', 'N', 'L', 'O', 'C', 'K')) {
                        r->method = NGX_HTTP_UNLOCK;
                        break;
                    }

                    break;

                case 7:
                    if (ngx_str7_cmp(m, 'O', 'P', 'T', 'I', 'O', 'N', 'S', ' '))
                    {
                        r->method = NGX_HTTP_OPTIONS;
                    }

                    break;

                case 8:
                    if (ngx_str8cmp(m, 'P', 'R', 'O', 'P', 'F', 'I', 'N', 'D'))
                    {
                        r->method = NGX_HTTP_PROPFIND;
                    }

                    break;

                case 9:
                    if (ngx_str9cmp(m,
                            'P', 'R', 'O', 'P', 'P', 'A', 'T', 'C', 'H'))
                    {
                        r->method = NGX_HTTP_PROPPATCH;
                    }

                    break;
                }
                // 状态变更为uri前的空格
                state = sw_spaces_before_uri;
                break;
            }
            // 允许方法名中存在的字
            if ((ch < 'A' || ch > 'Z') && ch != '_' && ch != '-') {
                return NGX_HTTP_PARSE_INVALID_METHOD;
            }

            break;

        /* space* before URI */
        // 解析到uri前的空格
        case sw_spaces_before_uri:
            // 如果字节是 / 则表示发现了uri，记录开始，并转换状态为找到了uri开始的斜线
            if (ch == '/') {
                r->uri_start = p;
                state = sw_after_slash_in_uri;
                break;
            }
            
            // 如果不是斜线，则说明是schame，记录scheam开始的位置，并转换状态为scheam
            c = (u_char) (ch | 0x20);
            if (c >= 'a' && c <= 'z') {
                r->schema_start = p;
                state = sw_schema;
                break;
            }
            // 判断是否存在不允许的字节
            switch (ch) {
            case ' ':
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;
        // 解析schema
        case sw_schema:
            // 限制schema允许出现的字符
            c = (u_char) (ch | 0x20);
            if (c >= 'a' && c <= 'z') {
                break;
            }

            if ((ch >= '0' && ch <= '9') || ch == '+' || ch == '-' || ch == '.')
            {
                break;
            }

            switch (ch) {
            // 找到:为schema的结束标识，转好到状态为需要找到shema后的斜线
            case ':':
                r->schema_end = p;
                state = sw_schema_slash;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        case sw_schema_slash:
            // 如果需要斜线，则必须出现斜线，否则错误
            switch (ch) {
            case '/':
                // 转移状态为需要第二个斜线
                state = sw_schema_slash_slash;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        case sw_schema_slash_slash:
            // 找到第二个斜线，状态转换为主机开始
            switch (ch) {
            case '/':
                state = sw_host_start;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        case sw_host_start:
            // 主机名的开始位置
            r->host_start = p;
            // 如果以[开始，则转移到sw_host_ip_literal状态，获取主机IP
            if (ch == '[') {
                state = sw_host_ip_literal;
                break;
            }
            // 设置状态为获取主机，这里没有break，直接进入下一个
            state = sw_host;

            /* fall through */

        case sw_host:
            // 检查合法的字符，如果是合法字符，则直接跳出switch，否则紧接着检查是否为结束
            c = (u_char) (ch | 0x20);
            if (c >= 'a' && c <= 'z') {
                break;
            }

            if ((ch >= '0' && ch <= '9') || ch == '.' || ch == '-') {
                break;
            }
            // 这里也没有break

            /* fall through */
        // 如果上面一步未break，则正常应该是host终止了，下面进行检查是否正常
        case sw_host_end:
            // 记录终止
            r->host_end = p;
       
            switch (ch) {
            // 如果是:,则是正常终止了，后面为端口号
            case ':':
                state = sw_port;
                break;
            // 如果是/，也是正常终止了，后面为uri
            case '/':
                r->uri_start = p;
                state = sw_after_slash_in_uri;
                break;
            // 如果是空格，则说明uri为空，使用/作为uri，转换为检查http版本
            case ' ':
                /*
                 * use single "/" from request line to preserve pointers,
                 * if request line will be copied to large client buffer
                 */
                r->uri_start = r->schema_end + 1;
                r->uri_end = r->schema_end + 2;
                state = sw_host_http_09;
                break;
            // 只允许上述三种字符
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;
        // 如果是ip形式host
        case sw_host_ip_literal:
            // 允许出现的字符，如果是允许出现的字符，则直接跳过
            if (ch >= '0' && ch <= '9') {
                break;
            }

            c = (u_char) (ch | 0x20);
            if (c >= 'a' && c <= 'z') {
                break;
            }

            switch (ch) {
            case ':':
                break;
            // 如果是]则表示主机结尾了，否则，对于break的字符，均为允许的字符
            case ']':
                state = sw_host_end;
                break;
            case '-':
            case '.':
            case '_':
            case '~':
                /* unreserved */
                break;
            case '!':
            case '$':
            case '&':
            case '\'':
            case '(':
            case ')':
            case '*':
            case '+':
            case ',':
            case ';':
            case '=':
                /* sub-delims */
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;
        // 解析端口
        case sw_port:
            // 合法的端口字符
            if (ch >= '0' && ch <= '9') {
                break;
            }
            // 不是合法的字符，则验证是否是端口后结束了
            switch (ch) {
            // 端口好结束，记录，开始解析uri
            case '/':
                r->port_end = p;
                r->uri_start = p;
                state = sw_after_slash_in_uri;
                break;
            // 空格表示uri为空
            case ' ':
                r->port_end = p;
                /*
                 * use single "/" from request line to preserve pointers,
                 * if request line will be copied to large client buffer
                 */
                r->uri_start = r->schema_end + 1;
                r->uri_end = r->schema_end + 2;
                state = sw_host_http_09;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        /* space+ after "http://host[:port] " */
        // 解析http版本
        case sw_host_http_09:
            switch (ch) {
            case ' ':
                break;
            case CR:
                r->http_minor = 9;
                state = sw_almost_done;
                break;
            case LF:
                r->http_minor = 9;
                goto done;
            // 找到定一个H，转换状态
            case 'H':
                r->http_protocol.data = p;
                state = sw_http_H;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;


        /* check "/.", "//", "%", and "\" (Win32) in URI */
        case sw_after_slash_in_uri:
            // 合法字符
            if (usual[ch >> 5] & (1U << (ch & 0x1f))) {
                state = sw_check_uri;
                break;
            }

            switch (ch) {
            // uri结束，uri为/
            case ' ':
                r->uri_end = p;
                state = sw_check_uri_http_09;
                break;
            case CR:
                r->uri_end = p;
                r->http_minor = 9;
                state = sw_almost_done;
                break;
            case LF:
                r->uri_end = p;
                r->http_minor = 9;
                goto done;
            // 如果为/.，记录为复杂uri，转移状态待解析uri
            case '.':
                r->complex_uri = 1;
                state = sw_uri;
                break;
            // 如果为/%，记录为引用uri，转移状态待解析uri  
            case '%':
                r->quoted_uri = 1;
                state = sw_uri;
                break;
            // 如果为//，记录为引用uri，转移状态待解析uri  
            case '/':
                r->complex_uri = 1;
                state = sw_uri;
                break;
#if (NGX_WIN32)
            case '\\':
                r->complex_uri = 1;
                state = sw_uri;
                break;
#endif
            // 如果是/?则后面是参数
            case '?':
                r->args_start = p + 1;
                state = sw_uri;
                break;
            // 如果是/#，是复杂uri
            case '#':
                r->complex_uri = 1;
                state = sw_uri;
                break;
            // 是/+，表示为对应uri
            case '+':
                r->plus_in_uri = 1;
                break;
            case '\0':
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            default:
                // 否则转移到检查uri
                state = sw_check_uri;
                break;
            }
            break;

        /* check "/", "%" and "\" (Win32) in URI */
        case sw_check_uri:
            // 合法字符
            if (usual[ch >> 5] & (1U << (ch & 0x1f))) {
                break;
            }
            // 与上面一致，进行处理，对于特殊字符
            switch (ch) {
            case '/':
#if (NGX_WIN32)
                if (r->uri_ext == p) {
                    r->complex_uri = 1;
                    state = sw_uri;
                    break;
                }
#endif          
                // 记录扩展uri的起始位置置空
                r->uri_ext = NULL;
                state = sw_after_slash_in_uri;
                break;
            // 记录扩展uri的起始位置
            case '.':
                r->uri_ext = p + 1;
                break;
            // 解析uri结束
            case ' ':
                r->uri_end = p;
                state = sw_check_uri_http_09;
                break;
            // http9，头结束
            case CR:
                r->uri_end = p;
                r->http_minor = 9;
                state = sw_almost_done;
                break;
            case LF:
                r->uri_end = p;
                r->http_minor = 9;
                goto done;
#if (NGX_WIN32)
            case '\\':
                r->complex_uri = 1;
                state = sw_after_slash_in_uri;
                break;
#endif
            case '%':
                r->quoted_uri = 1;
                state = sw_uri;
                break;
            case '?':
                r->args_start = p + 1;
                state = sw_uri;
                break;
            case '#':
                r->complex_uri = 1;
                state = sw_uri;
                break;
            case '+':
                r->plus_in_uri = 1;
                break;
            case '\0':
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        /* space+ after URI */
        case sw_check_uri_http_09:
            switch (ch) {
            case ' ':
                break;
            // http9没有后面的版本号
            case CR:
                r->http_minor = 9;
                state = sw_almost_done;
                break;
            case LF:
                r->http_minor = 9;
                goto done;
            case 'H':
                r->http_protocol.data = p;
                state = sw_http_H;
                break;
            default:
                r->space_in_uri = 1;
                state = sw_check_uri;
                p--;
                break;
            }
            break;


        /* URI */
        // 解析uri
        case sw_uri:
            // 合法字符且无特殊含义的字符
            if (usual[ch >> 5] & (1U << (ch & 0x1f))) {
                break;
            }

            switch (ch) {
            // 结束，且http版本大于09
            case ' ':
                r->uri_end = p;
                state = sw_http_09;
                break;
            // 结束
            case CR:
                r->uri_end = p;
                r->http_minor = 9;
                state = sw_almost_done;
                break;
            // 完成解析
            case LF:
                r->uri_end = p;
                r->http_minor = 9;
                goto done;
            case '#':
                r->complex_uri = 1;
                break;
            case '\0':
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        /* space+ after URI */
        case sw_http_09:
            switch (ch) {
            case ' ':
                break;
            case CR:
                r->http_minor = 9;
                state = sw_almost_done;
                break;
            case LF:
                r->http_minor = 9;
                goto done;
            case 'H':
                r->http_protocol.data = p;
                state = sw_http_H;
                break;
            default:
                r->space_in_uri = 1;
                state = sw_uri;
                p--;
                break;
            }
            break;
        // 找了http版本的H了
        case sw_http_H:
            switch (ch) {
            case 'T':
                state = sw_http_HT;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;
        // 找了http版本的HT了
        case sw_http_HT:
            switch (ch) {
            case 'T':
                state = sw_http_HTT;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;
        // 找了http版本的HTT了
        case sw_http_HTT:
            switch (ch) {
            case 'P':
                state = sw_http_HTTP;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;
        // 找了http版本的HTTP了
        case sw_http_HTTP:
            switch (ch) {
            case '/':
                state = sw_first_major_digit;
                break;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        /* first digit of major HTTP version */
        case sw_first_major_digit:
            if (ch < '1' || ch > '9') {
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            // http主版本
            r->http_major = ch - '0';
            // 只支持0.*和1.*的http
            if (r->http_major > 1) {
                return NGX_HTTP_PARSE_INVALID_VERSION;
            }
            // 等待点
            state = sw_major_digit;
            break;

        /* major HTTP version or dot */
        case sw_major_digit:
            if (ch == '.') {
                state = sw_first_minor_digit;
                break;
            }

            if (ch < '0' || ch > '9') {
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
              
            r->http_major = r->http_major * 10 + (ch - '0');

            if (r->http_major > 1) {
                return NGX_HTTP_PARSE_INVALID_VERSION;
            }

            break;

        /* first digit of minor HTTP version */
        case sw_first_minor_digit:
            if (ch < '0' || ch > '9') {
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            // http副版本号
            r->http_minor = ch - '0';
            state = sw_minor_digit;
            break;

        /* minor HTTP version or end of request line */
        case sw_minor_digit:
            if (ch == CR) {
                state = sw_almost_done;
                break;
            }

            if (ch == LF) {
                goto done;
            }

            if (ch == ' ') {
                state = sw_spaces_after_digit;
                break;
            }

            if (ch < '0' || ch > '9') {
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }

            if (r->http_minor > 99) {
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }

            r->http_minor = r->http_minor * 10 + (ch - '0');
            break;

        case sw_spaces_after_digit:
            switch (ch) {
            case ' ':
                break;
            case CR:
                state = sw_almost_done;
                break;
            case LF:
                goto done;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
            break;

        /* end of request line */
        case sw_almost_done:
            r->request_end = p - 1;
            switch (ch) {
            case LF:
                goto done;
            default:
                return NGX_HTTP_PARSE_INVALID_REQUEST;
            }
        }
    }

    b->pos = p;
    r->state = state;

    return NGX_AGAIN;
  
// 完成请求行解析
done:

    b->pos = p + 1;

    if (r->request_end == NULL) {
        r->request_end = p;
    }
    // 计算版本号
    r->http_version = r->http_major * 1000 + r->http_minor;
    r->state = sw_start;
    // 版本如果为9，则只允许get请求
    if (r->http_version == 9 && r->method != NGX_HTTP_GET) {
        return NGX_HTTP_PARSE_INVALID_09_METHOD;
    }

    return NGX_OK;
}
```



#### 解析uri ngx_http_process_request_uri

解析uri主要是拆分参数和前面的uri，并对复杂uri进行处理：

```
ngx_int_t
ngx_http_process_request_uri(ngx_http_request_t *r)
{
    ngx_http_core_srv_conf_t  *cscf;
    // 是否有参数
    if (r->args_start) {
        r->uri.len = r->args_start - 1 - r->uri_start;
    } else {
        r->uri.len = r->uri_end - r->uri_start;
    }
    // 如果为复杂uri，则执行相应操作
    if (r->complex_uri || r->quoted_uri) {

        r->uri.data = ngx_pnalloc(r->pool, r->uri.len + 1);
        if (r->uri.data == NULL) {
            ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return NGX_ERROR;
        }

        cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
        // cscf->merge_slashes对应配置merge_slashes，默认为on，表示开启或者关闭将请求URI中相邻两个或更多斜线合并成一个的功能。
        if (ngx_http_parse_complex_uri(r, cscf->merge_slashes) != NGX_OK) {
            r->uri.len = 0;

            ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                          "client sent invalid request");
            ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
            return NGX_ERROR;
        }

    } else {
        r->uri.data = r->uri_start;
    }

    r->unparsed_uri.len = r->uri_end - r->uri_start;
    r->unparsed_uri.data = r->uri_start;

    r->valid_unparsed_uri = r->space_in_uri ? 0 : 1;

    if (r->uri_ext) {
        if (r->args_start) {
            r->exten.len = r->args_start - 1 - r->uri_ext;
        } else {
            r->exten.len = r->uri_end - r->uri_ext;
        }

        r->exten.data = r->uri_ext;
    }

    if (r->args_start && r->uri_end > r->args_start) {
        r->args.len = r->uri_end - r->args_start;
        r->args.data = r->args_start;
    }

#if (NGX_WIN32)
    ...
#endif

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http uri: \"%V\"", &r->uri);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http args: \"%V\"", &r->args);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http exten: \"%V\"", &r->exten);

    return NGX_OK;
}
```



#### 获取请求的服务 ngx_http_set_virtual_server

通过host获取对应的服务：

```c
static ngx_int_t
ngx_http_set_virtual_server(ngx_http_request_t *r, ngx_str_t *host)
{
    ngx_int_t                  rc;
    ngx_http_connection_t     *hc;
    ngx_http_core_loc_conf_t  *clcf;
    ngx_http_core_srv_conf_t  *cscf;

#if (NGX_SUPPRESS_WARN)
    cscf = NULL;
#endif

    hc = r->http_connection;

#if (NGX_HTTP_SSL && defined SSL_CTRL_SET_TLSEXT_HOSTNAME)

    ...忽略

#endif
    // 查找对应的服务
    rc = ngx_http_find_virtual_server(r->connection,
                                      hc->addr_conf->virtual_names,
                                      host, r, &cscf);

    if (rc == NGX_ERROR) {
        ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_ERROR;
    }

#if (NGX_HTTP_SSL && defined SSL_CTRL_SET_TLSEXT_HOSTNAME)

    ...

#endif

    if (rc == NGX_DECLINED) {
        return NGX_OK;
    }
    // 设置对应的srv和loc层级的配置，使用server层级的解析配置进行赋值，在确定location快后，使用location层级重新赋值
    r->srv_conf = cscf->ctx->srv_conf;
    r->loc_conf = cscf->ctx->loc_conf;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 设置对应服务的log
    ngx_set_connection_log(r->connection, clcf->error_log);

    return NGX_OK;
}
```

根据hostname查找服务的方式如下，参考解析配置相应部分：

```c
static ngx_int_t
ngx_http_find_virtual_server(ngx_connection_t *c,
    ngx_http_virtual_names_t *virtual_names, ngx_str_t *host,
    ngx_http_request_t *r, ngx_http_core_srv_conf_t **cscfp)
{
    ngx_http_core_srv_conf_t  *cscf;

    if (virtual_names == NULL) {
        return NGX_DECLINED;
    }
    // 在hash的联合结构中查找，参考常用结构中hash部分的介绍。这里name只包含前置通配符，后置通配符和标志名
    cscf = ngx_hash_find_combined(&virtual_names->names,
                                  ngx_hash_key(host->data, host->len),
                                  host->data, host->len);
    // 如果查找到，则设置并返回
    if (cscf) {
        *cscfp = cscf;
        return NGX_OK;
    }

#if (NGX_PCRE)
    // 如果配置支持正则表达式，则遍历正则表达式查找对应服务
    if (host->len && virtual_names->nregex) {
        ngx_int_t                n;
        ngx_uint_t               i;
        ngx_http_server_name_t  *sn;

        sn = virtual_names->regex;

#if (NGX_HTTP_SSL && defined SSL_CTRL_SET_TLSEXT_HOSTNAME)

        ...

#endif /* NGX_HTTP_SSL && defined SSL_CTRL_SET_TLSEXT_HOSTNAME */

        for (i = 0; i < virtual_names->nregex; i++) {

            n = ngx_http_regex_exec(r, sn[i].regex, host);
            // 不匹配，则继续查找
            if (n == NGX_DECLINED) {
                continue;
            }
            // 查找到则返回
            if (n == NGX_OK) {
                *cscfp = sn[i].server;
                return NGX_OK;
            }
            // 否则返回error
            return NGX_ERROR;
        }
    }

#endif /* NGX_PCRE */
    // 返回未找到
    return NGX_DECLINED;
}
```





#### 执行子请求方法ngx_http_run_posted_requests

```c
void
ngx_http_run_posted_requests(ngx_connection_t *c)
{
    ngx_http_request_t         *r;
    ngx_http_posted_request_t  *pr;
    // 遍历每个主请求的子请求，执行对应的write_event_handler处理方法
    for ( ;; ) {

        if (c->destroyed) {
            return;
        }

        r = c->data;
        pr = r->main->posted_requests;

        if (pr == NULL) {
            return;
        }
        
        r->main->posted_requests = pr->next;

        r = pr->request;

        ngx_http_set_log_request(c->log, r);

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                       "http posted request: \"%V?%V\"", &r->uri, &r->args);

        r->write_event_handler(r);
    }
}
```



#### 分配大请求内存ngx_http_alloc_large_header_buffer

当原本申请的存储请求头的空间不够时，将调用该函数获取更大的空间。

```c
static ngx_int_t
ngx_http_alloc_large_header_buffer(ngx_http_request_t *r,
    ngx_uint_t request_line)
{
    u_char                    *old, *new;
    ngx_buf_t                 *b;
    ngx_chain_t               *cl;
    ngx_http_connection_t     *hc;
    ngx_http_core_srv_conf_t  *cscf;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http alloc large header buffer");
    // 如果是请求行，并且state为0，即完成了解析，则直接返回，并设置buf全部可复用
    if (request_line && r->state == 0) {

        /* the client fills up the buffer with "\r\n" */

        r->header_in->pos = r->header_in->start;
        r->header_in->last = r->header_in->start;

        return NGX_OK;
    }
    // 获取对应解析的内容，是请求行或者请求头
    old = request_line ? r->request_start : r->header_name_start;

    cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
    // 如果没有完成解析，并且对应的buf大小超过了设置的large_client_header_buffers的size，则直接返回
    if (r->state != 0
        && (size_t) (r->header_in->pos - old)
                                     >= cscf->large_client_header_buffers.size)
    {
        return NGX_DECLINED;
    }

    hc = r->http_connection;
    // 如果free还有未使用的buf
    if (hc->free) {
        cl = hc->free;
        // 将free向后移动一个
        hc->free = cl->next;
        // 设置buf为新的还未使用的buf
        b = cl->buf;

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http large header free: %p %uz",
                       b->pos, b->end - b->last);
    // 如果创建的buf数量小于配置的限制
    } else if (hc->nbusy < cscf->large_client_header_buffers.num) {
        // 创建一个buf
        b = ngx_create_temp_buf(r->connection->pool,
                                cscf->large_client_header_buffers.size);
        if (b == NULL) {
            return NGX_ERROR;
        }
        // 创建一个ngx_chain_t
        cl = ngx_alloc_chain_link(r->connection->pool);
        if (cl == NULL) {
            return NGX_ERROR;
        }
        // 设置对应的buf为刚才申请的buf
        cl->buf = b;

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http large header alloc: %p %uz",
                       b->pos, b->end - b->last);
    // 如果超过限制，直接返回
    } else {
        return NGX_DECLINED;
    }
    // 设置ngx_chain_t的next
    cl->next = hc->busy;
    hc->busy = cl;
    // 设置使用buf数量
    hc->nbusy++;
    // 如果state为0（这里只能是请求头，如果是请求行已经返回），说明一行请求头已经解析完成，不需要拷贝残缺的信息到新的空间上，因此直接返回即可
    if (r->state == 0) {
        /*
         * r->state == 0 means that a header line was parsed successfully
         * and we do not need to copy incomplete header line and
         * to relocate the parser header pointers
         */

        r->header_in = b;

        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http large header copy: %uz", r->header_in->pos - old);
    // 如果之前的残缺的请求长度大于新申请的buf的长度，则说明配置存在错误，或者请求过长，直接返回错误
    if (r->header_in->pos - old > b->end - b->start) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
                      "too large header to copy");
        return NGX_ERROR;
    }

    new = b->start;
    // 将残缺的请求拷贝到新的buf中
    ngx_memcpy(new, old, r->header_in->pos - old);
    // 设置pos和last的位置
    b->pos = new + (r->header_in->pos - old);
    b->last = new + (r->header_in->pos - old);
    // 如果是请求行，则需要把已经解析出的部分，使用新的地址对其赋值
    if (request_line) {
        r->request_start = new;

        if (r->request_end) {
            r->request_end = new + (r->request_end - old);
        }

        r->method_end = new + (r->method_end - old);

        r->uri_start = new + (r->uri_start - old);
        r->uri_end = new + (r->uri_end - old);

        if (r->schema_start) {
            r->schema_start = new + (r->schema_start - old);
            r->schema_end = new + (r->schema_end - old);
        }

        if (r->host_start) {
            r->host_start = new + (r->host_start - old);
            if (r->host_end) {
                r->host_end = new + (r->host_end - old);
            }
        }

        if (r->port_start) {
            r->port_start = new + (r->port_start - old);
            r->port_end = new + (r->port_end - old);
        }

        if (r->uri_ext) {
            r->uri_ext = new + (r->uri_ext - old);
        }

        if (r->args_start) {
            r->args_start = new + (r->args_start - old);
        }

        if (r->http_protocol.data) {
            r->http_protocol.data = new + (r->http_protocol.data - old);
        }

    } else {
        r->header_name_start = new;
        r->header_name_end = new + (r->header_name_end - old);
        r->header_start = new + (r->header_start - old);
        r->header_end = new + (r->header_end - old);
    }
    // 设置新的存储接收信息的buf
    r->header_in = b;

    return NGX_OK;
}
```



## 处理请求头ngx_http_process_request_headers

```c
static void
ngx_http_process_request_headers(ngx_event_t *rev)
{
    u_char                     *p;
    size_t                      len;
    ssize_t                     n;
    ngx_int_t                   rc, rv;
    ngx_table_elt_t            *h;
    ngx_connection_t           *c;
    ngx_http_header_t          *hh;
    ngx_http_request_t         *r;
    ngx_http_core_srv_conf_t   *cscf;
    ngx_http_core_main_conf_t  *cmcf;

    c = rev->data;
    r = c->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0,
                   "http process request header line");
    // 如果为超时事件，则关闭请求
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        c->timedout = 1;
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }
    // 获取核心配置
    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    rc = NGX_AGAIN;

    for ( ;; ) {
        // 接收并解析用户请求
        if (rc == NGX_AGAIN) {
            // 如果buf已经使用完成，且还未完成解析，则需要再分配大空间的buf
            if (r->header_in->pos == r->header_in->end) {

                rv = ngx_http_alloc_large_header_buffer(r, 0);

                if (rv == NGX_ERROR) {
                    ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                    break;
                }
                // 如果空间超限，报错
                if (rv == NGX_DECLINED) {
                    p = r->header_name_start;

                    r->lingering_close = 1;

                    if (p == NULL) {
                        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                      "client sent too large request");
                        ngx_http_finalize_request(r,
                                            NGX_HTTP_REQUEST_HEADER_TOO_LARGE);
                        break;
                    }

                    len = r->header_in->end - p;

                    if (len > NGX_MAX_ERROR_STR - 300) {
                        len = NGX_MAX_ERROR_STR - 300;
                    }

                    ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                "client sent too long header line: \"%*s...\"",
                                len, r->header_name_start);

                    ngx_http_finalize_request(r,
                                            NGX_HTTP_REQUEST_HEADER_TOO_LARGE);
                    break;
                }
            }
            // 读取请求头
            n = ngx_http_read_request_header(r);

            if (n == NGX_AGAIN || n == NGX_ERROR) {
                break;
            }
        }

        /* the host header could change the server configuration context */
        cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
        // 解析请求头，其中underscores_in_headers对应配置allow_underscores是否允许key中存在下划线，这里每次只解析一行
        rc = ngx_http_parse_header_line(r, r->header_in,
                                        cscf->underscores_in_headers);
        // 解析正确
        if (rc == NGX_OK) {
            // 计算请求长度
            r->request_length += r->header_in->pos - r->header_name_start;
            // 如果是无效的头部或者可忽略的头部，则记录
            if (r->invalid_header && cscf->ignore_invalid_headers) {

                /* there was error while a header line parsing */

                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                              "client sent invalid header line: \"%*s\"",
                              r->header_end - r->header_name_start,
                              r->header_name_start);
                continue;
            }

            /* a header line has been parsed successfully */
            // 将解析的头添加到headers_in的headers中
            h = ngx_list_push(&r->headers_in.headers);
            if (h == NULL) {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }
            // 设置hash值，为在解析请求头时计算的值
            h->hash = r->header_hash;
            // 设置key和value
            h->key.len = r->header_name_end - r->header_name_start;
            h->key.data = r->header_name_start;
            h->key.data[h->key.len] = '\0';

            h->value.len = r->header_end - r->header_start;
            h->value.data = r->header_start;
            h->value.data[h->value.len] = '\0';
            // 设置lowcase_key
            h->lowcase_key = ngx_pnalloc(r->pool, h->key.len);
            if (h->lowcase_key == NULL) {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }
            // 如果长度和lowcase_index中使用的长度一样，则直接使用lowcase_header存储的小写字符赋值，否则调用对应函数处理
            if (h->key.len == r->lowcase_index) {
                ngx_memcpy(h->lowcase_key, r->lowcase_header, h->key.len);

            } else {
                ngx_strlow(h->lowcase_key, h->key.data, h->key.len);
            }
            // 从headers_in_hash中查找对应的头。具体查看配置继续的http头初始化和hash结构
            hh = ngx_hash_find(&cmcf->headers_in_hash, h->hash,
                               h->lowcase_key, h->key.len);
            // 如果找到了对应的头，并有对应的处理方法，则执行对应的处理方法，这一步会设置cookie，host，Keep-Alive，X-Forwarded-For，Connection
            if (hh && hh->handler(r, h, hh->offset) != NGX_OK) {
                break;
            }

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "http header: \"%V: %V\"",
                           &h->key, &h->value);

            continue;
        }
        // 如果完成了请求头的解析
        if (rc == NGX_HTTP_PARSE_HEADER_DONE) {

            /* a whole header has been parsed successfully */

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "http header done");
            // 计算长度
            r->request_length += r->header_in->pos - r->header_name_start;
            // 记录状态
            r->http_state = NGX_HTTP_PROCESS_REQUEST_STATE;
            // 处理请求头
            rc = ngx_http_process_request_header(r);

            if (rc != NGX_OK) {
                break;
            }
            // 执行请求
            ngx_http_process_request(r);

            break;
        }
        // again表示一行还未接收完成
        if (rc == NGX_AGAIN) {

            /* a header line parsing is still not complete */

            continue;
        }

        /* rc == NGX_HTTP_PARSE_INVALID_HEADER */

        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                      "client sent invalid header line");
        // 否则执行终止请求
        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        break;
    }
    // 执行子请求的方法
    ngx_http_run_posted_requests(c);
}
```

### 解析请求头ngx_http_parse_header_line

```c
ngx_int_t
ngx_http_parse_header_line(ngx_http_request_t *r, ngx_buf_t *b,
    ngx_uint_t allow_underscores)
{
    u_char      c, ch, *p;
    ngx_uint_t  hash, i;
    // 状态变量
    enum {
        sw_start = 0,
        sw_name,
        sw_space_before_value,
        sw_value,
        sw_space_after_value,
        sw_ignore_line,
        sw_almost_done,
        sw_header_almost_done
    } state;

    /* the last '\0' is not needed because string is zero terminated */
    // 大小写转换及允许字符
    static u_char  lowcase[] =
        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
        "\0\0\0\0\0\0\0\0\0\0\0\0\0-\0\0" "0123456789\0\0\0\0\0\0"
        "\0abcdefghijklmnopqrstuvwxyz\0\0\0\0\0"
        "\0abcdefghijklmnopqrstuvwxyz\0\0\0\0\0"
        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0";
    // 获取状态和已计算的hash值
    state = r->state;
    hash = r->header_hash;
    i = r->lowcase_index;
    // 变量接收到的数据
    for (p = b->pos; p < b->last; p++) {
        ch = *p;

        switch (state) {

        /* first char */
        //状态为开始
        case sw_start:
            // 记录当前的变量名的开始位置
            r->header_name_start = p;
            r->invalid_header = 0;
            // 判断直接
            switch (ch) {
            // 如果是CR '\r'则说明是最后一行，转移状态到sw_header_almost_done
            case CR:
                r->header_end = p;
                state = sw_header_almost_done;
                break;
            // 如果是LF \n，则是结束,完成解析
            case LF:
                r->header_end = p;
                goto header_done;
            // 否则就是变量名
            default:
                state = sw_name;
                // 进行字符转换，并确定是否为合法字符
                c = lowcase[ch];
                // 如果为合法字符
                if (c) {
                    // 计算hash值
                    hash = ngx_hash(0, c);
                    //在存储小写字符的位置上记录
                    r->lowcase_header[0] = c;
                    // i记录name的长度
                    i = 1;
                    break;
                }
                // 判断是否允许下划线
                if (ch == '_') {
                    if (allow_underscores) {
                        hash = ngx_hash(0, ch);
                        r->lowcase_header[0] = ch;
                        i = 1;

                    } else {
                        hash = 0;
                        i = 0;
                        r->invalid_header = 1;
                    }

                    break;
                }
                // 否则就是无效的请求头
                if (ch == '\0') {
                    return NGX_HTTP_PARSE_INVALID_HEADER;
                }

                hash = 0;
                i = 0;
                r->invalid_header = 1;

                break;

            }
            break;

        /* header name */
        // 解析变量名
        case sw_name:
            // 进行字符转换，并确定是否为合法字符
            c = lowcase[ch];

            if (c) {
                hash = ngx_hash(hash, c);
                r->lowcase_header[i++] = c;
                // 这里i要和31进行取&，确保i小于32，因为lowcase_header长度为32
                i &= (NGX_HTTP_LC_HEADER_LEN - 1);
                break;
            }

            if (ch == '_') {
                if (allow_underscores) {
                    hash = ngx_hash(hash, ch);
                    r->lowcase_header[i++] = ch;
                    i &= (NGX_HTTP_LC_HEADER_LEN - 1);

                } else {
                    r->invalid_header = 1;
                }

                break;
            }
            // 如果是:，则表示变量名结束，转移状态为值前面的空间
            if (ch == ':') {
                r->header_name_end = p;
                state = sw_space_before_value;
                break;
            }
            // 如果是 \r，则说明变量值为空，记录变量值，并转移状态为将要结束行解析
            if (ch == CR) {
                r->header_name_end = p;
                r->header_start = p;
                r->header_end = p;
                state = sw_almost_done;
                break;
            }
            // 如果是 \n，则说明变量值为空，记录变量值，结束行解析
            if (ch == LF) {
                r->header_name_end = p;
                r->header_start = p;
                r->header_end = p;
                goto done;
            }
            
            /* IIS may send the duplicate "HTTP/1.1 ..." lines */
            if (ch == '/'
                && r->upstream
                && p - r->header_name_start == 4
                && ngx_strncmp(r->header_name_start, "HTTP", 4) == 0)
            {
                state = sw_ignore_line;
                break;
            }

            if (ch == '\0') {
                return NGX_HTTP_PARSE_INVALID_HEADER;
            }

            r->invalid_header = 1;

            break;

        /* space* before header value */
        // 值前面的空间
        case sw_space_before_value:
            switch (ch) {
            case ' ':
                break;
            // 如果是 \r，则说明变量值为空，记录变量值，并转移状态为将要结束行解析
            case CR:
                r->header_start = p;
                r->header_end = p;
                state = sw_almost_done;
                break;
            // 如果是 \n，则说明变量值为空，记录变量值，结束行解析
            case LF:
                r->header_start = p;
                r->header_end = p;
                goto done;
            case '\0':
                return NGX_HTTP_PARSE_INVALID_HEADER;
            // 其余情况说明有变量值，解析变量值
            default:
                r->header_start = p;
                state = sw_value;
                break;
            }
            break;

        /* header value */
        // 解析变量值
        case sw_value:
            switch (ch) {
            // 如果是空格，则是变量值后的空间，转移状态
            case ' ':
                r->header_end = p;
                state = sw_space_after_value;
                break;
            // 如果是 \r，说明变量解析完成，并转移状态为将要结束行解析
            case CR:
                r->header_end = p;
                state = sw_almost_done;
                break;
            // 如果是 \n，说明变量解析完成，记录变量值，结束行解析
            case LF:
                r->header_end = p;
                goto done;
            case '\0':
                return NGX_HTTP_PARSE_INVALID_HEADER;
            // 其余情况都是正常值的字符
            }
            break;

        /* space* before end of header line */
        // 由于在变量值末尾的空格应该被忽略，但是变量值本身可能包含空格，因此在该状态下，能够转换到变量值的状态，这表示之前遇到的空格是值的一部分
        case sw_space_after_value:
            switch (ch) {
            case ' ':
                break;
            case CR:
                state = sw_almost_done;
                break;
            case LF:
                goto done;
            case '\0':
                return NGX_HTTP_PARSE_INVALID_HEADER;
            default:
                state = sw_value;
                break;
            }
            break;

        /* ignore header line */
        case sw_ignore_line:
            switch (ch) {
            case LF:
                state = sw_start;
                break;
            default:
                break;
            }
            break;

        /* end of header line */
        // 将要完成行解析
        case sw_almost_done:
            switch (ch) {
            // 完成行解析
            case LF:
                goto done;
            case CR:
                break;
            // 解析错误
            default:
                return NGX_HTTP_PARSE_INVALID_HEADER;
            }
            break;

        /* end of header */
        // 将要完成头解析
        case sw_header_almost_done:
            switch (ch) {
            // 只允许是\n
            case LF:
                goto header_done;
            default:
                return NGX_HTTP_PARSE_INVALID_HEADER;
            }
        }
    }
    // 如果未完成一行的解析，则记录解析状态
    b->pos = p;
    r->state = state;
    r->header_hash = hash;
    r->lowcase_index = i;

    return NGX_AGAIN;
// 完成一行的解析
done:
    // 记录解析到的位置
    b->pos = p + 1;
    // 记录下一次从开始状态进行解析
    r->state = sw_start;
    // 记录hash值
    r->header_hash = hash;
    // 记录key长度，该长度与31取&,保证小于32
    r->lowcase_index = i;

    return NGX_OK;
// 完成header的整体解析
header_done:
    // 记录解析到的位置
    b->pos = p + 1;
    // 记录下一次从开始状态进行解析
    r->state = sw_start;
    // 返回完成整个http header的解析
    return NGX_HTTP_PARSE_HEADER_DONE;
}
```

### 部分请求头处理函数

这里简单介绍一下请求头的处理。

```c
typedef struct {
    ngx_str_t                         name;
    ngx_uint_t                        offset;
    ngx_http_header_handler_pt        handler;
} ngx_http_header_t;

ngx_http_header_t  ngx_http_headers_in[] = {
    { ngx_string("Host"), offsetof(ngx_http_headers_in_t, host),
                 ngx_http_process_host },

    { ngx_string("Connection"), offsetof(ngx_http_headers_in_t, connection),
                 ngx_http_process_connection },

    { ngx_string("If-Modified-Since"),
                 offsetof(ngx_http_headers_in_t, if_modified_since),
                 ngx_http_process_unique_header_line },

    { ngx_string("If-Unmodified-Since"),
                 offsetof(ngx_http_headers_in_t, if_unmodified_since),
                 ngx_http_process_unique_header_line },

    { ngx_string("If-Match"),
                 offsetof(ngx_http_headers_in_t, if_match),
                 ngx_http_process_unique_header_line },

    { ngx_string("If-None-Match"),
                 offsetof(ngx_http_headers_in_t, if_none_match),
                 ngx_http_process_unique_header_line },

    { ngx_string("User-Agent"), offsetof(ngx_http_headers_in_t, user_agent),
                 ngx_http_process_user_agent },

    { ngx_string("Referer"), offsetof(ngx_http_headers_in_t, referer),
                 ngx_http_process_header_line },

    { ngx_string("Content-Length"),
                 offsetof(ngx_http_headers_in_t, content_length),
                 ngx_http_process_unique_header_line },

    { ngx_string("Content-Range"),
                 offsetof(ngx_http_headers_in_t, content_range),
                 ngx_http_process_unique_header_line },

    { ngx_string("Content-Type"),
                 offsetof(ngx_http_headers_in_t, content_type),
                 ngx_http_process_header_line },

    { ngx_string("Range"), offsetof(ngx_http_headers_in_t, range),
                 ngx_http_process_header_line },

    { ngx_string("If-Range"),
                 offsetof(ngx_http_headers_in_t, if_range),
                 ngx_http_process_unique_header_line },

    { ngx_string("Transfer-Encoding"),
                 offsetof(ngx_http_headers_in_t, transfer_encoding),
                 ngx_http_process_unique_header_line },

    { ngx_string("TE"),
                 offsetof(ngx_http_headers_in_t, te),
                 ngx_http_process_header_line },

    { ngx_string("Expect"),
                 offsetof(ngx_http_headers_in_t, expect),
                 ngx_http_process_unique_header_line },

    { ngx_string("Upgrade"),
                 offsetof(ngx_http_headers_in_t, upgrade),
                 ngx_http_process_header_line },

#if (NGX_HTTP_GZIP || NGX_HTTP_HEADERS)
    { ngx_string("Accept-Encoding"),
                 offsetof(ngx_http_headers_in_t, accept_encoding),
                 ngx_http_process_header_line },

    { ngx_string("Via"), offsetof(ngx_http_headers_in_t, via),
                 ngx_http_process_header_line },
#endif

    { ngx_string("Authorization"),
                 offsetof(ngx_http_headers_in_t, authorization),
                 ngx_http_process_unique_header_line },

    { ngx_string("Keep-Alive"), offsetof(ngx_http_headers_in_t, keep_alive),
                 ngx_http_process_header_line },

#if (NGX_HTTP_X_FORWARDED_FOR)
    { ngx_string("X-Forwarded-For"),
                 offsetof(ngx_http_headers_in_t, x_forwarded_for),
                 ngx_http_process_multi_header_lines },
#endif

#if (NGX_HTTP_REALIP)
    { ngx_string("X-Real-IP"),
                 offsetof(ngx_http_headers_in_t, x_real_ip),
                 ngx_http_process_header_line },
#endif

#if (NGX_HTTP_HEADERS)
    { ngx_string("Accept"), offsetof(ngx_http_headers_in_t, accept),
                 ngx_http_process_header_line },

    { ngx_string("Accept-Language"),
                 offsetof(ngx_http_headers_in_t, accept_language),
                 ngx_http_process_header_line },
#endif

#if (NGX_HTTP_DAV)
    { ngx_string("Depth"), offsetof(ngx_http_headers_in_t, depth),
                 ngx_http_process_header_line },

    { ngx_string("Destination"), offsetof(ngx_http_headers_in_t, destination),
                 ngx_http_process_header_line },

    { ngx_string("Overwrite"), offsetof(ngx_http_headers_in_t, overwrite),
                 ngx_http_process_header_line },

    { ngx_string("Date"), offsetof(ngx_http_headers_in_t, date),
                 ngx_http_process_header_line },
#endif

    { ngx_string("Cookie"), offsetof(ngx_http_headers_in_t, cookies),
                 ngx_http_process_multi_header_lines },

    { ngx_null_string, 0, NULL }
};
```

具体调用解析的方式见配置解析中的介绍，这里主要介绍一些函数的处理逻辑。

#### host处理方法ngx_http_process_host

`host`处理方法主要是解析是否为合法host，并根据host确定服务。

```c
static ngx_int_t
ngx_http_process_host(ngx_http_request_t *r, ngx_table_elt_t *h,
    ngx_uint_t offset)
{
    ngx_int_t  rc;
    ngx_str_t  host;
    // 如果已经设置了host，则重复设置，报错
    if (r->headers_in.host) {
        ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                      "client sent duplicate host header: \"%V: %V\", "
                      "previous value: \"%V: %V\"",
                      &h->key, &h->value, &r->headers_in.host->key,
                      &r->headers_in.host->value);
        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        return NGX_ERROR;
    }
    // 设置请求的host
    r->headers_in.host = h;

    host = h->value;
    // 解析是否为合法host
    rc = ngx_http_validate_host(&host, r->pool, 0);

    if (rc == NGX_DECLINED) {
        ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                      "client sent invalid host header");
        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        return NGX_ERROR;
    }

    if (rc == NGX_ERROR) {
        ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_ERROR;
    }
    // 如果请求行中出现的host长度不为0，说明请求行中host合法，优先使用请求行中的host查找服务（之前已经查找过），这里就直接返回
    if (r->headers_in.server.len) {
        return NGX_OK;
    }
    // 根据host查找对应的服务，设置配置
    if (ngx_http_set_virtual_server(r, &host) == NGX_ERROR) {
        return NGX_ERROR;
    }
    
    r->headers_in.server = host;

    return NGX_OK;
}
```



#### Connection处理方法ngx_http_process_connection

`connection`处理方法主要判断请求类型，分为`close`和`keep-alive`两种，分别表示请求相应后是否关闭连接。

```
static ngx_int_t
ngx_http_process_connection(ngx_http_request_t *r, ngx_table_elt_t *h,
    ngx_uint_t offset)
{
    // 连接类型为close
    if (ngx_strcasestrn(h->value.data, "close", 5 - 1)) {
        r->headers_in.connection_type = NGX_HTTP_CONNECTION_CLOSE;
    // 连接类型为keep-alive
    } else if (ngx_strcasestrn(h->value.data, "keep-alive", 10 - 1)) {
        r->headers_in.connection_type = NGX_HTTP_CONNECTION_KEEP_ALIVE;
    }

    return NGX_OK;
}
```

#### Content-Length&Transfer-Encoding处理方法ngx_http_process_unique_header_line

对应post请求来说，需要在接收完成请求头后告知服务器请求体数据长度，该值应该是精确的。如果该值大于实际长度时，服务器会陷入等待中，不会发送响应。当该值小于实际长度时，服务器将只读取到部分数据，如果这时使用了`keep-alive`形式连接，剩下的部分会在第二次请求中被读取到，导致下一次请求也失败。

但又是，我们可能没办法确定实际要发送的内容大小，这时就不能使用`Content-Length`头了，而应该使用`Transfer-Encoding:chunked`。数据以一系列分块的形式进行发送. `Content-Length` 首部在这种情况下不被发送. 在每一个分块的开头需要添加当前分块的长度, 以十六进制的形式表示，后面紧跟着 `\r\n` , 之后是分块本身, 后面也是`\r\n`. 终止块是一个常规的分块, 不同之处在于其长度为0.

处理方法`ngx_http_process_unique_header_line`是很多请求头的处理方法，只有有唯一值：

```c
static ngx_int_t
ngx_http_process_unique_header_line(ngx_http_request_t *r, ngx_table_elt_t *h,
    ngx_uint_t offset)
{
    ngx_table_elt_t  **ph;

    ph = (ngx_table_elt_t **) ((char *) &r->headers_in + offset);
    // 未被设置时设置，否则说明请求错误
    if (*ph == NULL) {
        *ph = h;
        return NGX_OK;
    }

    ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                  "client sent duplicate header line: \"%V: %V\", "
                  "previous value: \"%V: %V\"",
                  &h->key, &h->value, &(*ph)->key, &(*ph)->value);

    ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);

    return NGX_ERROR;
}
```

#### keep-alive处理方法ngx_http_process_header_line

`keep-alive`指示长连接相关属性，例如：`Keep-Alive: 100`表示这个TCP通道可以保持100s。`keep-alive`模式下客户端获取响应结束的方法与服务器获取请求体长度方法一致。如果是静态的响应数据，可以通过判断响应头部中的Content-Length 字段，判断数据达到这个大小就知道数据传输结束了。但是返回的数据是动态变化的，服务器不能第一时间知道数据长度，这样就没有 Content-Length 关键字了。这种情况下，服务器是分块传输数据的，`Transfer-Encoding：chunk`，这时候就要根据传输的数据块chunk来判断，数据传输结束的时候，最后的一个数据块chunk的长度是0。

HTTP的Keep-Alive与TCP的Keep Alive，有些不同，两者意图不一样。前者主要是 TCP连接复用，避免建立过多的TCP连接。而TCP的Keep Alive的意图是在于保持TCP连接的存活，就是发送心跳包。隔一段时间给连接对端发送一个探测包，如果收到对方回应的 ACK，则认为连接还是存活的，在超过一定重试次数之后还是没有收到对方的回应，则丢弃该 TCP 连接。

对于`ngx_http_process_header_line`处理方法也是十分通用的头处理逻辑：

```c
static ngx_int_t
ngx_http_process_header_line(ngx_http_request_t *r, ngx_table_elt_t *h,
    ngx_uint_t offset)
{
    ngx_table_elt_t  **ph;

    ph = (ngx_table_elt_t **) ((char *) &r->headers_in + offset);
    // 使用第一个获取到的请求头对应数据，如果存在重复，则丢弃后面的
    if (*ph == NULL) {
        *ph = h;
    }

    return NGX_OK;
}
```

### 处理请求头ngx_http_process_request_header

```c
ngx_int_t
ngx_http_process_request_header(ngx_http_request_t *r)
{
    // 没有查找到对应服务，则直接返回
    if (r->headers_in.server.len == 0
        && ngx_http_set_virtual_server(r, &r->headers_in.server)
           == NGX_ERROR)
    {
        return NGX_ERROR;
    }
    
    // 如果http版本大于1，且请求头中没有host，则请求错误
    if (r->headers_in.host == NULL && r->http_version > NGX_HTTP_VERSION_10) {
        ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                   "client sent HTTP/1.1 request without \"Host\" header");
        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        return NGX_ERROR;
    }
    
    // 如果请求头中存在请求体长度，则解析长度值
    if (r->headers_in.content_length) {
        r->headers_in.content_length_n =
                            ngx_atoof(r->headers_in.content_length->value.data,
                                      r->headers_in.content_length->value.len);

        if (r->headers_in.content_length_n == NGX_ERROR) {
            ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                          "client sent invalid \"Content-Length\" header");
            ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
            return NGX_ERROR;
        }
    }
    // 如果请求是trace方式，则直接返回
    if (r->method == NGX_HTTP_TRACE) {
        ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                      "client sent TRACE method");
        ngx_http_finalize_request(r, NGX_HTTP_NOT_ALLOWED);
        return NGX_ERROR;
    }
    
    // 如果采用chunked分块传输请求体，则设置请求体长度为空，设置标识位
    if (r->headers_in.transfer_encoding) {
        if (r->headers_in.transfer_encoding->value.len == 7
            && ngx_strncasecmp(r->headers_in.transfer_encoding->value.data,
                               (u_char *) "chunked", 7) == 0)
        {
            r->headers_in.content_length = NULL;
            r->headers_in.content_length_n = -1;
            r->headers_in.chunked = 1;

        } else {
            ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                          "client sent unknown \"Transfer-Encoding\": \"%V\"",
                          &r->headers_in.transfer_encoding->value);
            ngx_http_finalize_request(r, NGX_HTTP_NOT_IMPLEMENTED);
            return NGX_ERROR;
        }
    }
    // 如果请求类型为kep-alive，解析维持连接时间
    if (r->headers_in.connection_type == NGX_HTTP_CONNECTION_KEEP_ALIVE) {
        if (r->headers_in.keep_alive) {
            r->headers_in.keep_alive_n =
                            ngx_atotm(r->headers_in.keep_alive->value.data,
                                      r->headers_in.keep_alive->value.len);
        }
    }

    return NGX_OK;
}
```





## 请求处理ngx_http_process_request

接收完请求头后进行请求处理，是nginx核心逻辑部分。执行方法如下：

```c
void
ngx_http_process_request(ngx_http_request_t *r)
{
    ngx_connection_t  *c;

    c = r->connection;

#if (NGX_HTTP_SSL)

    ...

#endif
    /* 如果读事件依然在定时器红黑树中，则从其中剔除。
    在定时器红黑树中主要监控是否结束请求头超时，超时会将请求关闭，则完成请求头的处理后应该从定时器红黑树中删除.
    */
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

#if (NGX_STAT_STUB)
    ...
#endif
    // 设置读写事件处理均为ngx_http_request_handler
    c->read->handler = ngx_http_request_handler;
    c->write->handler = ngx_http_request_handler;
    // 设置请求的读事件为ngx_http_block_reading，其逻辑对于使用epoll驱动来说，仅仅只有一个日志记录
    r->read_event_handler = ngx_http_block_reading;

    ngx_http_handler(r);
}
```

由于nginx基本都是非阻塞试函数，在完全请求解析前，每次调用请求解析函数无法立即完成处理，将会立即返回，在epoll事件启动中接收到读事件时，继续调用解析请求头的处理。这是对于读事件的处理，在解析请求头期间，写事件只是一个空函数，并无任何处理逻辑。

在接收完请求后，对应的读写事件就发生了变更，这里变成了`ngx_http_request_handler`方法，该函数主要是判断epoll返回的事件类型，来执行对应的请求的读事件处理和写事件处理。这里读事件只是一个空函数，在完成请求前不应该出现可读事件，即使出现，也不会做任何处理。写事件是每次epoll返回都会触发的事件，因此在结束请求前，每次都会执行对应请求的写事件。

写事件的处理会在`ngx_http_handler`根据当前请求类型来决定，处理方法如下：

### 设置请求处理ngx_http_handler

```c
void
ngx_http_handler(ngx_http_request_t *r)
{
    ngx_http_core_main_conf_t  *cmcf;

    r->connection->log->action = NULL;
    // 如果请求不是内部请求，即是用户下发的请求
    if (!r->internal) {
        // 设置请求类型
        switch (r->headers_in.connection_type) {
        case 0:
            r->keepalive = (r->http_version > NGX_HTTP_VERSION_10);
            break;

        case NGX_HTTP_CONNECTION_CLOSE:
            r->keepalive = 0;
            break;

        case NGX_HTTP_CONNECTION_KEEP_ALIVE:
            r->keepalive = 1;
            break;
        }
        // 如果存在请求体，则设置延迟关闭
        r->lingering_close = (r->headers_in.content_length_n > 0
                              || r->headers_in.chunked);
        // 设置请求阶段为0
        r->phase_handler = 0;

    } else {
        // 否则，设置请求阶段为NGX_HTTP_SERVER_REWRITE_PHASE阶段的第一个ngx_http_phase_handler_t处理方法在handlers中的下标，具体可以看配置解析相关章节
        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
        r->phase_handler = cmcf->phase_engine.server_rewrite_index;
    }

    r->valid_location = 1;
#if (NGX_HTTP_GZIP)
    r->gzip_tested = 0;
    r->gzip_ok = 0;
    r->gzip_vary = 0;
#endif
    // 设置请求读事件为执行请求阶段循环，并立即执行一次
    r->write_event_handler = ngx_http_core_run_phases;
    ngx_http_core_run_phases(r);
}
```

### 核心处理循环ngx_http_core_run_phases

```c
void
ngx_http_core_run_phases(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_phase_handler_t   *ph;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    // 获取执行函数列表
    ph = cmcf->phase_engine.handlers;
    // 执行对应的check方法，这里并不一定是依次遍历每个执行方法，后续详细介绍
    while (ph[r->phase_handler].checker) {

        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

        if (rc == NGX_OK) {
            return;
        }
    }
}
```

再来看一下具体的读写事件的处理逻辑：

### epoll事件触发执行方法ngx_http_request_handler

```c
static void
ngx_http_request_handler(ngx_event_t *ev)
{
    ngx_connection_t    *c;
    ngx_http_request_t  *r;

    c = ev->data;
    r = c->data;

    ngx_http_set_log_request(c->log, r);

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http run request: \"%V?%V\"", &r->uri, &r->args);
    // 当连接关闭时，将主请求的计数加一，这里目前不是很明白，后续subrequest中详细查看
    if (c->close) {
        r->main->count++;
        ngx_http_terminate_request(r, 0);
        ngx_http_run_posted_requests(c);
        return;
    }

    if (ev->delayed && ev->timedout) {
        ev->delayed = 0;
        ev->timedout = 0;
    }
    // 如果事件是写事件，则执行对应请求的写事件处理
    if (ev->write) {
        r->write_event_handler(r);
    // 否则执行读事件处理
    } else {
        r->read_event_handler(r);
    }
    // 执行所有子请求的处理
    ngx_http_run_posted_requests(c);
}
```



## 核心处理阶段

这里需要参考配置解析中关于处理阶段的处理。在处理方法是，会执行没一个阶段的`checker`方法。下表了列出了每个阶段的含义及`checker`方法，下面会详细介绍。

| 阶段                            | 含义                                                         | `checker`方法                      | 其他                                                         |
| ------------------------------- | ------------------------------------------------------------ | ---------------------------------- | ------------------------------------------------------------ |
| `NGX_HTTP_POST_READ_PHASE`      | 收到完整的http头部后处理的阶段                               | `ngx_http_core_generic_phase`      | 允许用户增加处理函数                                         |
| `NGX_HTTP_SERVER_REWRITE_PHASE` | 重写uri阶段，这里是在执行uri查找对应的location之前的重写uri。比如在server层定义了一个`if`语句，根据指定的uri进行重写。 | `ngx_http_core_rewrite_phase`      | 允许用户添加处理                                             |
| `NGX_HTTP_FIND_CONFIG_PHASE`    | 依据uri查找对应location服务                                  | `ngx_http_core_find_config_phase`  | 不允许用户自定义函数                                         |
| `NGX_HTTP_REWRITE_PHASE`        | 在location中重写uri                                          | `ngx_http_core_rewrite_phase`      | 允许用户自定义处理函数                                       |
| `NGX_HTTP_POST_REWRITE_PHASE`   | 用于rewrite重新URI后，防止错误的配置导致死循环               | `ngx_http_core_post_rewrite_phase` | 仅在使用了location重写时才存在该阶段，且不允许用户添加处理方法。 |
| `NGX_HTTP_PREACCESS_PHASE`      | 处理NGX_HTTP_ACCESS_PHASE阶段决定访问权限前，http模块可以介入的阶段 | `ngx_http_core_generic_phase`      | 允许用户自定义处理方法。                                     |
| `NGX_HTTP_ACCESS_PHASE`         | 让HTTP模块判断是否允许这个请求访问nginx服务                  | `ngx_http_core_access_phase`       | 允许用户自定义处理方法。                                     |
| `NGX_HTTP_POST_ACCESS_PHASE`    | 如果http模块的handler处理函数返回不允许访问的错误码时，这里负责向用户发送拒绝服务的错误 | `ngx_http_core_post_access_phase`  | 仅使用·了权限解析时才存在该步骤，且不允许用户自定义处理函数。 |
| `NGX_HTTP_PRECONTENT_PHASE`     | 主要用于try_file                                             | `ngx_http_core_generic_phase`      | 允许用户自定义处理函数                                       |
| `NGX_HTTP_CONTENT_PHASE`        | 用于处理HTTP请求内容的阶段，是大部分http模块介入的阶段       | `ngx_http_core_content_phase`      | 允许用户自定义处理函数。用户主要介入的处理阶段。             |
| `NGX_HTTP_LOG_PHASE`            | 处理完成后记录日志的阶段                                     | `ngx_http_core_generic_phase`      | 允许用户自定义处理函数。                                     |

首先这里介绍一下多个阶段共用的`ngx_http_core_generic_phase`的check方法。

### ngx_http_core_generic_phase方法

```c
ngx_int_t
ngx_http_core_generic_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t  rc;

    /*
     * generic phase checker,
     * used by the post read and pre-access phases
     */

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "generic phase: %ui", r->phase_handler);
    // 执行ngx_http_phase_handler_t对应的处理函数
    rc = ph->handler(r);
    // 如果返回NGX_OK，则表示该阶段处理完成，设置下一次执行的为下一个阶段的函数
    if (rc == NGX_OK) {
        r->phase_handler = ph->next;
        return NGX_AGAIN;
    }

    // 如果返回NGX_DECLINED，则表示该处理方法完成，按顺序执行后续方法
    if (rc == NGX_DECLINED) {
        r->phase_handler++;
        return NGX_AGAIN;
    }
    // 如果返回NGX_AGAIN，表示该方法没有处理完成，存在阻塞，这时返回到ngx_http_core_run_phases函数时，会立即结束函数处理，等待事件触发，再处理。返回NGX_DONE表示处理请求完成。
    if (rc == NGX_AGAIN || rc == NGX_DONE) {
        return NGX_OK;
    }

    /* rc == NGX_ERROR || rc == NGX_HTTP_...  */
    // 如果出错，执行对应的处理
    ngx_http_finalize_request(r, rc);

    return NGX_OK;
}
```

对应使用该方法作为checker的阶段来说，每个处理函数的方法值含义如下：

| 返回值               | 含义                                                         |
| -------------------- | ------------------------------------------------------------ |
| `NGX_OK`             | 执行下一个阶段的第一个handler处理方法，这意味着，即使当前阶段后续还有一些HTTP模块设置了handler处理方法，也不会被执行。 |
| `NGX_DECLINED`       | 当前处理函数完成，执行下一个handler方法。                    |
| `NGX_AGAIN|NGX_DONE` | 当前处理函数未完成，存在阻塞调用，将控制权交给事件模块，在下次事件模块触发时，再次调用该方法。 |
| 其他                 | 出错，使用`ngx_http_finalize_request`结束请求。              |



## NGX_HTTP_POST_READ_PHASE阶段

完成请求解析后，首先执行的阶段，目前默认的nginx下不会使用该阶段。`checker`方法为`ngx_http_core_generic_phase`上文已详细介绍。官方模块的`ngx_http_realip_module`设置了该方法。这里不做详细介绍。

## NGX_HTTP_SERVER_REWRITE_PHASE阶段

`NGX_HTTP_SERVER_REWRITE_PHASE`阶段主要由`ngx_http_rewrite_module`模块处理。

首先来看一下该阶段的checker方法。

### ngx_http_core_rewrite_phase方法

```c
ngx_int_t
ngx_http_core_rewrite_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t  rc;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "rewrite phase: %ui", r->phase_handler);
    // 执行处理方法
    rc = ph->handler(r);
    // 如果返回执行下一个处理方法，则设置对应的phase_handler
    if (rc == NGX_DECLINED) {
        r->phase_handler++;
        return NGX_AGAIN;
    }
    
    if (rc == NGX_DONE) {
        return NGX_OK;
    }

    /* NGX_OK, NGX_AGAIN, NGX_ERROR, NGX_HTTP_...  */
    // 否则执行结束处理
    ngx_http_finalize_request(r, rc);

    return NGX_OK;
}
```

从上述代码可以看出，该阶段的处理函数方法值对应的含义为：

| 返回值         | 含义                                                         |
| -------------- | ------------------------------------------------------------ |
| `NGX_DECLINED` | 当前处理函数完成，处理下一个函数                             |
| `NGX_DONE`     | 当前处理函数未完成，存在阻塞调用，将控制权交给事件模块，在下次事件模块触发时，再次调用该方法。 |
| 其他           | 使用`ngx_http_finalize_request`结束请求                      |

这里的执行方法只有rewrite模块增加的处理函数：

```c
static ngx_int_t
ngx_http_rewrite_handler(ngx_http_request_t *r)
{
    ngx_int_t                     index;
    ngx_http_script_code_pt       code;
    ngx_http_script_engine_t     *e;
    ngx_http_core_srv_conf_t     *cscf;
    ngx_http_core_main_conf_t    *cmcf;
    ngx_http_rewrite_loc_conf_t  *rlcf;
    
    // 获取当前层级下的 main数组中ngx_http_core_module配置
    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    // 获取当前层级下的 srv数组中ngx_http_core_module配置
    cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
    // 获取location_rewrite阶段的下标
    index = cmcf->phase_engine.location_rewrite_index;
    // 如果当前处理阶段是location的重写，并且location层的loc_conf和server层的loc_conf一致。则跳过，server中没有location配置时，跳过该阶段
    if (r->phase_handler == index && r->loc_conf == cscf->ctx->loc_conf) {
        /* skipping location rewrite phase for server null location */
        return NGX_DECLINED;
    }
    // 获取重写模块生成的结构
    rlcf = ngx_http_get_module_loc_conf(r, ngx_http_rewrite_module);
    // 如果是空，则直接返回
    if (rlcf->codes == NULL) {
        return NGX_DECLINED;
    }
    // 分配一个脚本引擎结构
    e = ngx_pcalloc(r->pool, sizeof(ngx_http_script_engine_t));
    if (e == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    // 分配变量数组结构
    e->sp = ngx_pcalloc(r->pool,
                        rlcf->stack_size * sizeof(ngx_http_variable_value_t));
    if (e->sp == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    // 获取执行数组的起始位置
    e->ip = rlcf->codes->elts;
    // 设置请求
    e->request = r;
    e->quote = 1;
    e->log = rlcf->log;
    // 默认处理完成执行下一阶段
    e->status = NGX_DECLINED;
    // 依次执行codes数组中每个方法，直到null
    while (*(uintptr_t *) e->ip) {
        code = *(ngx_http_script_code_pt *) e->ip;
        code(e);
    }
    // 返回状态
    return e->status;
}
```

该部分需要配合rewrite模块的介绍查看。

需要注意的是，当前请求`r`中的`**main_conf`,`**srv_conf`,`**loc_conf`所处的层级与当前执行阶段一致。对应处于`server rewrite`阶段来说，这三个配置是`server`层级解析出的配置，因此执行的处理函数是server层级配置的`rewrite`模块相关配置方法。（在之前的处理中，通过Server name查找到的配置，赋值就是server层级，详见`ngx_http_set_virtual_server`函数）。对于`location rewrite`执行阶段来说，这三个配置是`location`层级解析出的配置，因此执行的处理函数是location层级配置的`rewrite`模块相关配置方法（在`NGX_HTTP_FIND_CONFIG_PHASE`阶段找到请求对应的location块后会重新对配置进行赋值，详见`ngx_http_update_location_config`方法）。

这里方法的值为`e->status`，因此如果配置了重定向时，该值会是重定向的返回码，这时会立即结束请求。在解析错误时，也需要设置`e->status`，以此来立即结束请求。



## NGX_HTTP_FIND_CONFIG_PHASE阶段

该阶段checker方法为`ngx_http_core_find_config_phase`：

```c
ngx_int_t
ngx_http_core_find_config_phase(ngx_http_request_t *r,
    ngx_http_phase_handler_t *ph)
{
    u_char                    *p;
    size_t                     len;
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *clcf;

    r->content_handler = NULL;
    r->uri_changed = 0;
    // 根据uri查找匹配的location
    rc = ngx_http_core_find_location(r);
    // 如果返回错误，则结束请求
    if (rc == NGX_ERROR) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_OK;
    }
    // 获取location层级创建的ngx_http_core_loc_conf_t
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    /*
    如果请求不是内部的，但是该模块是内部可访问的则报错。请求是内部情况如下：
    ngx_http_rewrite_module模块中使用rewrite指令修改的请求。
    内部请求创建的子请求等
    */
    if (!r->internal && clcf->internal) {
        ngx_http_finalize_request(r, NGX_HTTP_NOT_FOUND);
        return NGX_OK;
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "using configuration \"%s%V\"",
                   (clcf->noname ? "*" : (clcf->exact_match ? "=" : "")),
                   &clcf->name);
    // 根据获取的location，变更请求的相关参数
    ngx_http_update_location_config(r);

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http cl:%O max:%O",
                   r->headers_in.content_length_n, clcf->client_max_body_size);
    
    // 如果请求的包体不为-1，并且不是要丢弃请求包体，并且设置了客户最大的请求包体，该值小于请求包体长度
    if (r->headers_in.content_length_n != -1
        && !r->discard_body
        && clcf->client_max_body_size
        && clcf->client_max_body_size < r->headers_in.content_length_n)
    {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "client intended to send too large body: %O bytes",
                      r->headers_in.content_length_n);

        r->expect_tested = 1;
        // 接收请求头并丢弃
        (void) ngx_http_discard_request_body(r);
        // 结束请求
        ngx_http_finalize_request(r, NGX_HTTP_REQUEST_ENTITY_TOO_LARGE);
        return NGX_OK;
    }
    
    // 如果返回done，这里只有精准匹配，获取前缀匹配能够返回该值
    if (rc == NGX_DONE) {
        // 清理返回的headers_out的location
        ngx_http_clear_location(r);
        // 设置location
        r->headers_out.location = ngx_list_push(&r->headers_out.headers);
        if (r->headers_out.location == NULL) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return NGX_OK;
        }
        // 设置key为Location
        r->headers_out.location->hash = 1;
        ngx_str_set(&r->headers_out.location->key, "Location");
        // 根据是否存在参数，设置location的value
        if (r->args.len == 0) {
            r->headers_out.location->value = clcf->name;

        } else {
            len = clcf->name.len + 1 + r->args.len;
            p = ngx_pnalloc(r->pool, len);

            if (p == NULL) {
                ngx_http_clear_location(r);
                ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return NGX_OK;
            }

            r->headers_out.location->value.len = len;
            r->headers_out.location->value.data = p;

            p = ngx_cpymem(p, clcf->name.data, clcf->name.len);
            *p++ = '?';
            ngx_memcpy(p, r->args.data, r->args.len);
        }
        // 重定向结束请求
        ngx_http_finalize_request(r, NGX_HTTP_MOVED_PERMANENTLY);
        return NGX_OK;
    }
    // 向后移动，执行下一个处理函数
    r->phase_handler++;
    return NGX_AGAIN;
}
```

从上面代码可以看出该阶段，查找location时返回值对应的含义如下：

| 返回值      | 含义                                   |
| ----------- | -------------------------------------- |
| `NGX_ERROR` | 查找错误，结束请求                     |
| `NGX_DONE`  | 请求重定向，结束请求，向用户返回重定向 |
| 其他        | 完成当前处理函数，执行后面的处理函数   |

### 查找location方法ngx_http_core_find_location

该函数负责通过uri查找location。其中返回值含义如下：

```c
/*
 * NGX_OK       - exact or regex match 精准匹配，或者正则匹配
 * NGX_DONE     - auto redirect 自动重定向
 * NGX_AGAIN    - inclusive match 前缀匹配
 * NGX_ERROR    - regex error 正则匹配错误
 * NGX_DECLINED - no match 无匹配
 */
```

该部分需要节后配置解析一起查看。

执行逻辑如下：

```c
static ngx_int_t
ngx_http_core_find_location(ngx_http_request_t *r)
{
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *pclcf;
#if (NGX_PCRE)
    ngx_int_t                  n;
    ngx_uint_t                 noregex;
    ngx_http_core_loc_conf_t  *clcf, **clcfp;

    noregex = 0;
#endif
    // 获取当前层级的ngx_http_core_loc_conf_t。当前层级可能是server层级和location层级（因为location层级可能包含location，这时有可能需要递归查找）
    pclcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 先在精准匹配和前缀匹配值查找
    rc = ngx_http_core_find_static_location(r, pclcf->static_locations);
    // 如果返回值为NGX_AGAIN，表示前缀匹配，这时需要进一步查找location下的location,使用最长匹配
    if (rc == NGX_AGAIN) {

#if (NGX_PCRE)
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

        noregex = clcf->noregex;
#endif

        /* look up nested locations */
        // 查找location下的location，如果当前location下没有location，将返回NGX_DECLINED
        rc = ngx_http_core_find_location(r);
    }
    // 如果找到的精准匹配，或者自动重定向，则直接返回
    if (rc == NGX_OK || rc == NGX_DONE) {
        return rc;
    }

    /* rc == NGX_DECLINED or rc == NGX_AGAIN in nested location */

#if (NGX_PCRE)
    // 查找正则匹配数组
    if (noregex == 0 && pclcf->regex_locations) {
        // 遍历正则匹配数组
        for (clcfp = pclcf->regex_locations; *clcfp; clcfp++) {

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "test location: ~ \"%V\"", &(*clcfp)->name);
            // 执行正则匹配，这里可查看rewrite部分介绍的函数，这里会设置请求r中的captures，用于后续函数中使用正则捕获中的内容
            n = ngx_http_regex_exec(r, (*clcfp)->regex, &r->uri);
            // 如果正则匹配
            if (n == NGX_OK) {
                // 设置loc_conf
                r->loc_conf = (*clcfp)->loc_conf;

                /* look up nested locations */
                // 查找该location下的location
                rc = ngx_http_core_find_location(r);
                // 返回，如果出错，返回错误，否则返回正则匹配或精准匹配
                return (rc == NGX_ERROR) ? rc : NGX_OK;
            }
            // 如果不匹配，则跳过，继续变量
            if (n == NGX_DECLINED) {
                continue;
            }
            // 否则返回错误
            return NGX_ERROR;
        }
    }
#endif
    // 返回之前设置的结果
    return rc;
}
```

从上述代码可以看出，`location`匹配顺序为：精准匹配 > 正则匹配 > 前缀匹配

在精准匹配和前缀中查找的逻辑如下：

```c
/*
 * NGX_OK       - exact match 精准匹配
 * NGX_DONE     - auto redirect 自动重定向
 * NGX_AGAIN    - inclusive match 前缀匹配
 * NGX_DECLINED - no match // 不匹配
 */

static ngx_int_t
ngx_http_core_find_static_location(ngx_http_request_t *r,
    ngx_http_location_tree_node_t *node)
{
    u_char     *uri;
    size_t      len, n;
    ngx_int_t   rc, rv;

    len = r->uri.len;
    uri = r->uri.data;

    rv = NGX_DECLINED;

    for ( ;; ) {
        // 当node节点为空时，即没有精准匹配，或者前缀匹配时，直接返回
        if (node == NULL) {
            return rv;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "test location: \"%*s\"",
                       (size_t) node->len, node->name);
        // 长度使用uri长度和节点长度的较短者
        n = (len <= (size_t) node->len) ? len : node->len;
        // 判断两者的给定长度部分字符串差值
        rc = ngx_filename_cmp(uri, node->name, n);
        // 如果两者不一样大
        if (rc != 0) {
            // 根据大小决定继续在左子树还是右子树中查找
            node = (rc < 0) ? node->left : node->right;

            continue;
        }
        
        // 如果uri长度大于节点长度
        if (len > (size_t) node->len) {
            
            // 如果节点为前缀匹配
            if (node->inclusive) {
                // 向将请求的loc_conf指向当前节点的loc_conf，并设置返回值
                r->loc_conf = node->inclusive->loc_conf;
                rv = NGX_AGAIN;
                // 接下来从节点下的前缀列表中查找
                node = node->tree;
                // 因为前缀列表中存储的name和len为相对与当前节点的偏移部分值，因此需要将uri也进行偏移
                uri += n;
                len -= n;

                continue;
            }

            /* exact only */
            // 如果当前节点只支持精准匹配，则需要接着查找右子树
            node = node->right;

            continue;
        }
        // 如果uri长度和节点长度一致
        if (len == (size_t) node->len) {
            // 节点支持精准匹配，则设置loc_conf，并返回
            if (node->exact) {
                r->loc_conf = node->exact->loc_conf;
                return NGX_OK;
            // 设置loc_conf，并返回查找到的结果为前缀匹配
            } else {
                r->loc_conf = node->inclusive->loc_conf;
                return NGX_AGAIN;
            }
        }

        /* len < node->len */
        // 如果uri长度相对于节点长度大于1，并且节点设置了自动重定向，则设置loc_conf并返回。这里自动重定向在代理转发中会使用，后续详细介绍
        if (len + 1 == (size_t) node->len && node->auto_redirect) {

            r->loc_conf = (node->exact) ? node->exact->loc_conf:
                                          node->inclusive->loc_conf;
            rv = NGX_DONE;
        }

        node = node->left;
    }
}
```



### 更新请求ngx_http_update_location_config

该方法根据uri查找到的location配置，来更新对请求的限制。

```c
void
ngx_http_update_location_config(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t  *clcf;
    // 获取location层级的ngx_http_core_loc_conf_t
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 如果配置了limit_except，则使用limit_except_loc_conf作为loc_conf
    if (r->method & clcf->limit_except) {
        r->loc_conf = clcf->limit_except_loc_conf;
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    }
    // 如果请求是用户请求，则设置log
    if (r == r->main) {
        ngx_set_connection_log(r->connection, clcf->error_log);
    }
    
    /*
    如果设置了sendfile，并且系统支持sendfile，则设置标志位为1
    sendfule是linux支持的一个系统调用，在之前使用套接字传输文件内容时，需要使用read、write方法，首先从数据从内核态拷贝到用户态，再从用户态拷贝到内核态，这是十分耗时的，使用sendfile则避免来回拷贝，数据只在内核态传输。要使用sendfile，则配置：
    sendfile on
    */
    if ((ngx_io.flags & NGX_IO_SENDFILE) && clcf->sendfile) {
        r->connection->sendfile = 1;

    } else {
        r->connection->sendfile = 0;
    }
    
    /*
    如果设置了，用户post请求体只能存储在临时文件中
    配置为：client_body_in_file_only off|clean|on
    */
    if (clcf->client_body_in_file_only) {
        r->request_body_in_file_only = 1;
        r->request_body_in_persistent_file = 1;
        // 是否请求完成后就删除临时文件
        r->request_body_in_clean_file =
            clcf->client_body_in_file_only == NGX_HTTP_REQUEST_BODY_FILE_CLEAN;
        r->request_body_file_log_level = NGX_LOG_NOTICE;

    } else {
        r->request_body_file_log_level = NGX_LOG_WARN;
    }
    
    /*
    请求包体是否只存储在一个buf中
    */
    r->request_body_in_single_buf = clcf->client_body_in_single_buffer;

    // 如果请求设置了长连接
    if (r->keepalive) {
        // 如果nginx设置的长连接超时时间为0，即不支持长连接
        if (clcf->keepalive_timeout == 0) {
            r->keepalive = 0;
        // 如果设置长连接最大请求数量超过了设置值，则关闭长连接
        } else if (r->connection->requests >= clcf->keepalive_requests) {
            r->keepalive = 0;
        // 下面为针对通顶浏览器处理
        } else if (r->headers_in.msie6
                   && r->method == NGX_HTTP_POST
                   && (clcf->keepalive_disable
                       & NGX_HTTP_KEEPALIVE_DISABLE_MSIE6))
        {
            /*
             * MSIE may wait for some time if an response for
             * a POST request was sent over a keepalive connection
             */
            r->keepalive = 0;

        } else if (r->headers_in.safari
                   && (clcf->keepalive_disable
                       & NGX_HTTP_KEEPALIVE_DISABLE_SAFARI))
        {
            /*
             * Safari may send a POST request to a closed keepalive
             * connection and may stall for some time, see
             *     https://bugs.webkit.org/show_bug.cgi?id=5760
             */
            r->keepalive = 0;
        }
    }
    
    // 是否开启tcp_nopush
    if (!clcf->tcp_nopush) {
        /* disable TCP_NOPUSH/TCP_CORK use */
        r->connection->tcp_nopush = NGX_TCP_NOPUSH_DISABLED;
    }
    
    // 如果location配置设置了handler，则设置连接的处理函数，使用见NGX_HTTP_CONTENT_PHASE阶段
    if (clcf->handler) {
        r->content_handler = clcf->handler;
    }
}
```



## NGX_HTTP_REWRITE_PHASE阶段

该阶段是在location中设置了重写时的处理。

其使用的checker方法和handler方法与NGX_HTTP_SERVER_REWRITE_PHASE阶段一致。这里需要注意的是，这时使用的配置是location层级的配置。

## NGX_HTTP_REWRITE_PHASE阶段

`NGX_HTTP_REWRITE_PHASE`阶段用于在location阶段后判断是否需要再次进行查找。

check方法为：`ngx_http_core_post_rewrite_phase`,逻辑如下：

```c
ngx_int_t
ngx_http_core_post_rewrite_phase(ngx_http_request_t *r,
    ngx_http_phase_handler_t *ph)
{
    ngx_http_core_srv_conf_t  *cscf;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "post rewrite phase: %ui", r->phase_handler);
    // 如果经过NGX_HTTP_REWRITE_PHASE阶段，没有发送uri变更，则继续执行下一个处理方法
    if (!r->uri_changed) {
        r->phase_handler++;
        return NGX_AGAIN;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "uri changes: %d", r->uri_changes);

    /*
     * gcc before 3.3 compiles the broken code for
     *     if (r->uri_changes-- == 0)
     * if the r->uri_changes is defined as
     *     unsigned  uri_changes:4
     */
    // 否则，将uri变更次数减1，起始为11
    r->uri_changes--;
    // 如果值到0了，则认为配置存在环，报错
    if (r->uri_changes == 0) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "rewrite or internal redirection cycle "
                      "while processing \"%V\"", &r->uri);
        // 结束请求
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_OK;
    }
    
    // 执行下一阶段，这里的下一阶段是NGX_HTTP_REWRITE_PHASE阶段
    r->phase_handler = ph->next;
    // 设置loc_conf为server层级的loc_conf，重新执行查找
    cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
    r->loc_conf = cscf->ctx->loc_conf;

    return NGX_AGAIN;
}
```



## NGX_HTTP_PREACCESS_PHASE

该阶段一般针对当前请求进行限制。其处理checker为通用的`ngx_http_core_generic_phase`，方法这里不做详细介绍。其中执行的模块主要有`ngx_http_limit_conn_module`和`ngx_http_limit_req_module`模块，这里主要介绍一下`ngx_http_limit_req_module`模块的实现。

对于`ngx_http_limit_req_module`模块的具体实现和介绍，参考一些重要模块中对该模块的介绍。



## NGX_HTTP_ACCESS_PHASE阶段

该阶段进行权限的校验。

checker丰富为：`ngx_http_core_access_phase`

```c
ngx_int_t
ngx_http_core_access_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *clcf;

    if (r != r->main) {
        r->phase_handler = ph->next;
        return NGX_AGAIN;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "access phase: %ui", r->phase_handler);
    // 执行处理函数
    rc = ph->handler(r);
    // 返回NGX_DECLINED，继续执行下一个处理函数
    if (rc == NGX_DECLINED) {
        r->phase_handler++;
        return NGX_AGAIN;
    }
    // 返回NGX_AGAIN或者NGX_DONE说明当前处理未完成，存在阻塞，将控制权交还给epoll，下次事件到达时继续执行
    if (rc == NGX_AGAIN || rc == NGX_DONE) {
        return NGX_OK;
    }
    // 获取访问控制限制类型
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // satisfy对应配置satisfy，取值为any或者all（默认），分别表示所有权限校验均通过才能执行该请求和任意一个权限校验通过即可执行该请求
    if (clcf->satisfy == NGX_HTTP_SATISFY_ALL) {
        // 如果是所有权限校验均通过，并且本次的校验通过，则执行下一个处理函数
        if (rc == NGX_OK) {
            r->phase_handler++;
            return NGX_AGAIN;
        }
    // 如果是任意一个校验通过即可，则
    } else {
        // 如果当前的校验通过
        if (rc == NGX_OK) {
            // 设置access_code为0，表示通过校验
            r->access_code = 0;

            if (r->headers_out.www_authenticate) {
                r->headers_out.www_authenticate->hash = 0;
            }
            // 执行下一阶段的处理函数
            r->phase_handler = ph->next;
            return NGX_AGAIN;
        }
        // 如果本次权限校验返回权限禁止或者未授权
        if (rc == NGX_HTTP_FORBIDDEN || rc == NGX_HTTP_UNAUTHORIZED) {
            // 设置access_code为当前校验失败原因
            if (r->access_code != NGX_HTTP_UNAUTHORIZED) {
                r->access_code = rc;
            }
            // 执行下一阶段
            r->phase_handler++;
            return NGX_AGAIN;
        }
    }

    /* rc == NGX_ERROR || rc == NGX_HTTP_...  */
    // 如果出现为授权，且satisfy是all，判断是否需要进行延迟处理，当配置了auth_delay时，默认没有，直接关闭请求
    if (rc == NGX_HTTP_UNAUTHORIZED) {
        return ngx_http_core_auth_delay(r);
    }

    ngx_http_finalize_request(r, rc);
    return NGX_OK;
}
```

从中我们可以看出该阶段处理函数返回值对应的含义如下：

| 返回值                                     | 含义                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| `NGX_OK`                                   | 如果设置satisfy为all，则按顺序执行下一个处理方法，如果设置satisfy为any则执行下一阶段的处理方法。 |
| `NGX_DECLINED`                             | 按顺序执行下一阶段的处理方法                                 |
| `NGX_AGAIN/NGX_DONE`                       | 当前的处理未完成，将控制权交还epoll，下次继续执行当前处理方法。 |
| `NGX_HTTP_FORBIDDEN/NGX_HTTP_UNAUTHORIZED` | 如果设置satisfy为all，结束请求，如果设置satisfy为any则设置access_code成员，并执行下一个处理方法。 |
| 其他、`NGX_ERROR`                          | 结束请求                                                     |

该阶段的模块主要包括`ngx_http_access_module`模块，`ngx_http_auth_basic_module`模块和`ngx_http_auth_request_module`模块。其中前两个模块较为简单，最后一个模块涉及subrequest（后续会详细介绍），因此这里都不做详细介绍。这里只简单介绍一下使用。

### ngx_http_access_module

该模块通过简单的配置来限制请求来源的ip。例如：

```
location / {
    deny  192.168.1.1;
    allow 192.168.1.0/24;
    allow 10.1.1.0/16;
    allow 2001:0db8::/32;
    deny  all;
}
```

### ngx_http_auth_basic_module

ngx_http_auth_basic_module 模块允许通过使用“HTTP 基本身份验证”协议验证用户名和密码来限制对资源的访问。

### ngx_http_auth_request_module

ngx_http_auth_request_module 模块根据子请求的结果实现客户端授权。 如果子请求返回 2xx 响应码，则允许访问。 如果返回 401 或 403，则访问被拒绝并返回相应的错误代码。 子请求返回的任何其他响应代码都被视为错误。

## NGX_HTTP_POST_ACCESS_PHASE阶段

该阶段对NGX_HTTP_ACCESS_PHASE阶段的结果进行处理。不允许用户插入处理函数。checker方法如下：

```c
ngx_int_t
ngx_http_core_post_access_phase(ngx_http_request_t *r,
    ngx_http_phase_handler_t *ph)
{
    ngx_int_t  access_code;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "post access phase: %ui", r->phase_handler);
    // 获取NGX_HTTP_ACCESS_PHASE返回的权限code
    access_code = r->access_code;
    // 不为0，表示无权限
    if (access_code) {
        r->access_code = 0;

        if (access_code == NGX_HTTP_FORBIDDEN) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "access forbidden by rule");
        }
         
        if (access_code == NGX_HTTP_UNAUTHORIZED) {
            return ngx_http_core_auth_delay(r);
        }

        ngx_http_finalize_request(r, access_code);
        return NGX_OK;
    }
    // 有权限时执行后续处理。
    r->phase_handler++;
    return NGX_AGAIN;
}
```

## NGX_HTTP_PRECONTENT_PHASE阶段

该阶段以前主要是`try_files`的处理，现在在次基础上增加了`ngx_http_mirror_module`模块，其checker方法为通用的`ngx_http_core_generic_phase`方法。对应try_file的处理也相对简单，这里也不做详细介绍。

## NGX_HTTP_CONTENT_PHASE阶段

该阶段为核心http处理阶段，该阶段允许用户重新定义nginx服务的行为，是大部分模块介入的阶段。其checker方法如下：

```c
ngx_int_t
ngx_http_core_content_phase(ngx_http_request_t *r,
    ngx_http_phase_handler_t *ph)
{
    size_t     root;
    ngx_int_t  rc;
    ngx_str_t  path;
    /*
    如果请求设置了content_handler方法，则直接执行该方法，结束请求
    该方法为location层级的ngx_http_core_loc_conf_t结构的handler方法
    NGX_HTTP_FIND_CONFIG_PHASE阶段根据查找到的location层级配置对其赋值
    因此模块要想直接介入处理，可以在解析配置时设置对应location下的handler方法。
    后续详细介绍的代理转发模块就是直接介入该阶段
    */
    if (r->content_handler) {
        r->write_event_handler = ngx_http_request_empty_handler;
        ngx_http_finalize_request(r, r->content_handler(r));
        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "content phase: %ui", r->phase_handler);
    // 如果未设置handler方法，则执行默认的架构处理逻辑
    rc = ph->handler(r);
    // 如果返回不为NGX_DECLINED，则结束请求
    if (rc != NGX_DECLINED) {
        ngx_http_finalize_request(r, rc);
        return NGX_OK;
    }

    /* rc == NGX_DECLINED */
    // 返回是NGX_DECLINED是，判断后续还是否有处理函数，有的化就进入下一个方法的处理
    ph++;

    if (ph->checker) {
        r->phase_handler++;
        return NGX_AGAIN;
    }

    /* no content handler was found */
    // 结束请求
    if (r->uri.data[r->uri.len - 1] == '/') {

        if (ngx_http_map_uri_to_path(r, &path, &root, 0) != NULL) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "directory index of \"%s\" is forbidden", path.data);
        }

        ngx_http_finalize_request(r, NGX_HTTP_FORBIDDEN);
        return NGX_OK;
    }

    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no handler found");

    ngx_http_finalize_request(r, NGX_HTTP_NOT_FOUND);
    return NGX_OK;
}
```

上面展示了为何自定义模块可以直接介入该模块的处理方法。对于模块直接通过handler方式介入，后续会在代理转发中详细介绍，这里先简单看一下结构自定义的一些处理逻辑，表示在没有handler处理情况下的一些默认处理。

这里简单介绍一下`ngx_http_static_module`模块，看一下架构自定义的处理函数，并引出如何向客户端返回结果，为下一章节做一个铺垫。

### ngx_http_static_module模块

模块的作用就是读取磁盘上的静态文件，并把文件内容作为产生的输出。其实现为，根据用户的uri和nginx运行目录，查找是否有对应的文件，如果有，则向用户返回对应的文件，如果是一个目录，则交给`ngx_http_autoindex_handler`模块处理（也是该阶段的默认处理逻辑）可以列出这个目录的文件，或者是ngx_http_index_handler如果请求的路径下面有个默认的index文件，直接返回index文件的内容。

```
static ngx_http_module_t  ngx_http_static_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_static_init,                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    NULL,                                  /* create location configuration */
    NULL                                   /* merge location configuration */
};


ngx_module_t  ngx_http_static_module = {
    NGX_MODULE_V1,
    &ngx_http_static_module_ctx,           /* module context */
    NULL,                                  /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

该模块的处理相当简单，甚至没有comands配置。其postconfiguration方法也十分简单，功能只有一个，就是将`ngx_http_static_handler`方法添加到`NGX_HTTP_CONTENT_PHASE`阶段的处理函数数组中。

#### ngx_http_static_handler方法

这里仅简单介绍处理方法，详细细节可查看对应源码：

```c
static ngx_int_t
ngx_http_static_handler(ngx_http_request_t *r)
{
    u_char                    *last, *location;
    size_t                     root, len;
    ngx_str_t                  path;
    ngx_int_t                  rc;
    ngx_uint_t                 level;
    ngx_log_t                 *log;
    ngx_buf_t                 *b;
    ngx_chain_t                out;
    ngx_open_file_info_t       of;
    ngx_http_core_loc_conf_t  *clcf;
    // 如果请求不是get、head或者post，则直接返回
    if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD|NGX_HTTP_POST))) {
        return NGX_HTTP_NOT_ALLOWED;
    }
    // 如果uri是目录，则直接返回
    if (r->uri.data[r->uri.len - 1] == '/') {
        return NGX_DECLINED;
    }

    log = r->connection->log;

    /*
     * ngx_http_map_uri_to_path() allocates memory for terminating '\0'
     * so we do not need to reserve memory for '/' for possible redirect
     */
    // 使用uri拼接root，构建文件名
    last = ngx_http_map_uri_to_path(r, &path, &root, 0);
    if (last == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    path.len = last - path.data;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, log, 0,
                   "http filename: \"%s\"", path.data);

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 分配文件结构
    ngx_memzero(&of, sizeof(ngx_open_file_info_t));

    of.read_ahead = clcf->read_ahead;
    of.directio = clcf->directio;
    of.valid = clcf->open_file_cache_valid;
    of.min_uses = clcf->open_file_cache_min_uses;
    of.errors = clcf->open_file_cache_errors;
    of.events = clcf->open_file_cache_events;
    // 检查是否为软链，并查看是否允许软链，即disable_symlinks配置限制
    if (ngx_http_set_disable_symlinks(r, clcf, &path, &of) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    // 打开文件
    if (ngx_open_cached_file(clcf->open_file_cache, &path, &of, r->pool)
        != NGX_OK)
    {   
        // 错误处理
        switch (of.err) {

        case 0:
            return NGX_HTTP_INTERNAL_SERVER_ERROR;

        case NGX_ENOENT:
        case NGX_ENOTDIR:
        case NGX_ENAMETOOLONG:

            level = NGX_LOG_ERR;
            rc = NGX_HTTP_NOT_FOUND;
            break;

        case NGX_EACCES:
#if (NGX_HAVE_OPENAT)
        case NGX_EMLINK:
        case NGX_ELOOP:
#endif

            level = NGX_LOG_ERR;
            rc = NGX_HTTP_FORBIDDEN;
            break;

        default:

            level = NGX_LOG_CRIT;
            rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
            break;
        }

        if (rc != NGX_HTTP_NOT_FOUND || clcf->log_not_found) {
            ngx_log_error(level, log, of.err,
                          "%s \"%s\" failed", of.failed, path.data);
        }

        return rc;
    }

    r->root_tested = !r->error_page;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, log, 0, "http static fd: %d", of.fd);
    
    // 如果是目录返回错误
    if (of.is_dir) {

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, log, 0, "http dir");

        ngx_http_clear_location(r);

        r->headers_out.location = ngx_list_push(&r->headers_out.headers);
        if (r->headers_out.location == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        len = r->uri.len + 1;

        if (!clcf->alias && r->args.len == 0) {
            location = path.data + root;

            *last = '/';

        } else {
            if (r->args.len) {
                len += r->args.len + 1;
            }

            location = ngx_pnalloc(r->pool, len);
            if (location == NULL) {
                ngx_http_clear_location(r);
                return NGX_HTTP_INTERNAL_SERVER_ERROR;
            }

            last = ngx_copy(location, r->uri.data, r->uri.len);

            *last = '/';

            if (r->args.len) {
                *++last = '?';
                ngx_memcpy(++last, r->args.data, r->args.len);
            }
        }

        r->headers_out.location->hash = 1;
        ngx_str_set(&r->headers_out.location->key, "Location");
        r->headers_out.location->value.len = len;
        r->headers_out.location->value.data = location;

        return NGX_HTTP_MOVED_PERMANENTLY;
    }

#if !(NGX_WIN32) /* the not regular files are probably Unix specific */
    // 不是文件，报错
    if (!of.is_file) {
        ngx_log_error(NGX_LOG_CRIT, log, 0,
                      "\"%s\" is not a regular file", path.data);

        return NGX_HTTP_NOT_FOUND;
    }

#endif
    // post请求报错
    if (r->method == NGX_HTTP_POST) {
        return NGX_HTTP_NOT_ALLOWED;
    }
    // 丢弃请求体
    rc = ngx_http_discard_request_body(r);

    if (rc != NGX_OK) {
        return rc;
    }

    log->action = "sending response to client";
    // 设置响应头
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = of.size;
    r->headers_out.last_modified_time = of.mtime;
    
    
    if (ngx_http_set_etag(r) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    if (ngx_http_set_content_type(r) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    if (r != r->main && of.size == 0) {
        return ngx_http_send_header(r);
    }

    r->allow_ranges = 1;

    /* we need to allocate all before the header would be sent */
    // 创建存储响应体的buf结构
    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    b->file = ngx_pcalloc(r->pool, sizeof(ngx_file_t));
    if (b->file == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    // 发送响应头
    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }
    // 设置发送响应体的buf
    b->file_pos = 0;
    b->file_last = of.size;

    b->in_file = b->file_last ? 1: 0;
    b->last_buf = (r == r->main) ? 1: 0;
    b->last_in_chain = 1;

    b->file->fd = of.fd;
    b->file->name = path;
    b->file->log = log;
    b->file->directio = of.is_directio;

    out.buf = b;
    out.next = NULL;
    // 发送响应体
    return ngx_http_output_filter(r, &out);
}
```



## NGX_HTTP_LOG_PHASE阶段

该阶段赋值写日志，这里不做详细介绍。需要注意的是，无论请求在合阶段结束，正常都会在结束时执行该阶段的处理函数。

## 发送响应

### 相关结构

#### ngx_http_headers_out_t响应头

`ngx_http_headers_out_t`结构存储了响应头信息，其定义如下：

```c
typedef struct {
    // 待发送的http头部链表，与headers_in的headers类似
    ngx_list_t                        headers;
    ngx_list_t                        trailers;
    
    // 响应状态码
    ngx_uint_t                        status;
    // 响应行， 如 "HTTP/1.1 201 CREATED"
    ngx_str_t                         status_line;
    
    // 下面成员均为RFC1616规范中对应的HTTP响应头部，设置后ngx_http_header_filter_module过滤模块可以把他们加到待发送的网络包中
    ngx_table_elt_t                  *server;
    ngx_table_elt_t                  *date;
    ngx_table_elt_t                  *content_length;
    ngx_table_elt_t                  *content_encoding;
    ngx_table_elt_t                  *location;
    ngx_table_elt_t                  *refresh;
    ngx_table_elt_t                  *last_modified;
    ngx_table_elt_t                  *content_range;
    ngx_table_elt_t                  *accept_ranges;
    ngx_table_elt_t                  *www_authenticate;
    ngx_table_elt_t                  *expires;
    ngx_table_elt_t                  *etag;

    ngx_str_t                        *override_charset;
    // 可以调用ngx_http_set_content_type来设置content_type头部，该方法根据URI中的扩展名和mime.type来设置content_type
    size_t                            content_type_len;
    ngx_str_t                         content_type;
    ngx_str_t                         charset;
    u_char                           *content_type_lowcase;
    ngx_uint_t                        content_type_hash;

    ngx_array_t                       cache_control;
    ngx_array_t                       link;
    // 这里设置了响应长度后，不用再次到ngx_table_elt_t *content_length中设置响应长度
    off_t                             content_length_n;
    off_t                             content_offset;
    time_t                            date_time;
    time_t                            last_modified_time;
} ngx_http_headers_out_t;
```



### 发送响应头ngx_http_send_header

函数执行逻辑为：

```c
ngx_int_t
ngx_http_send_header(ngx_http_request_t *r)
{
    // 如果是post_action则直接返回，可查看post_action配置详细了解使用
    if (r->post_action) {
        return NGX_OK;
    }
    
    // 如果请求头已经发送，则报错返回
    if (r->header_sent) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
                      "header already sent");
        return NGX_ERROR;
    }
    // 如果请求存在错误，则设置请求头
    if (r->err_status) {
        r->headers_out.status = r->err_status;
        r->headers_out.status_line.len = 0;
    }
    
    // 执行发送响应头的过滤模块
    return ngx_http_top_header_filter(r);
}
```



### 发送响应体ngx_http_output_filter

该函数用来发送响应体。

```c
ngx_int_t
ngx_http_output_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_int_t          rc;
    ngx_connection_t  *c;

    c = r->connection;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http output filter \"%V?%V\"", &r->uri, &r->args);
    // 执行发送响应体过滤方法
    rc = ngx_http_top_body_filter(r, in);

    if (rc == NGX_ERROR) {
        /* NGX_ERROR may be returned by any filter */
        c->error = 1;
    }

    return rc;
}
```



### 发送响应过滤模块

对于发送响应来说，分为发送响应头和响应体，这两个发送响应模块均支持用户自定义处理逻辑，对于定义的处理逻辑来说，使用`ngx_http_top_header_filter`和`ngx_http_next_header_filter`来串联响应头发送链表，使用`ngx_http_top_body_filter`和`ngx_http_next_body_filter`串联请求体发送响应链表。

过滤模块的处理函数定义为：

```
typedef char *(*ngx_conf_post_handler_pt) (ngx_conf_t *cf,
    void *data, void *conf);
```

任意一个模块想要介入请求头或请求体的处理，只需要踏进如下函数即可（以ngx_http_gzip_filter_module模块举例）：

```c
typedef ngx_int_t (*ngx_http_output_header_filter_pt)(ngx_http_request_t *r);
/*
注意，这里的chain是要传输的数据，对应chain为空，表示传输未完成但没有新增加的数据需要传输，这一点很关键，
由于写响应体的过滤函数可能会被执行多次，该值会被大部分阶段用于判断是否需要介入处理
后续会详细介绍
*/
typedef ngx_int_t (*ngx_http_output_body_filter_pt)
    (ngx_http_request_t *r, ngx_chain_t *chain);

ngx_http_output_header_filter_pt  ngx_http_top_header_filter;
ngx_http_output_body_filter_pt    ngx_http_top_body_filter;

static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;
static ngx_http_output_body_filter_pt    ngx_http_next_body_filter;

/*
该函数为模块ctx的postconfiguration函数
其中ngx_http_gzip_header_filter，和ngx_http_gzip_body_filter均为该模块定义的处理函数
*/
static ngx_int_t
ngx_http_gzip_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_gzip_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_gzip_body_filter;

    return NGX_OK;
}

static ngx_int_t
ngx_http_gzip_header_filter(ngx_http_request_t *r)
{
    /*
    ...
    该模块定义的处理逻辑
    */
    // 执行ngx_http_next_header_filter模块方法
    return ngx_http_next_header_filter(r);
}


static ngx_int_t
ngx_http_gzip_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    /*
    ...
    该模块自定义处理方法
    */
        // 执行ngx_http_next_body_filter方法
        rc = ngx_http_next_body_filter(r, ctx->out);

        
    /*
    模块自定义处理方法
    */
}
```

从上述代码可以看出nginx如果将各个模块的响应头filter函数和响应体filter串联起来。

其中`ngx_http_top_header_filter`和`ngx_http_top_body_filter`模块存储了当前设置的响应头和响应体处理函数，`ngx_http_next_header_filter`和`ngx_http_next_body_filter`是作用域为文件作用域的static函数值。当该模块希望在处理响应头和响应体之前先按照该模块进行处理，则向让`ngx_http_next_header_filter`和`ngx_http_next_body_filter`存储之前旧的处理方法，再设置`ngx_http_top_header_filter`和`ngx_http_top_body_filter`为本模块的新的处理方法。在执行本模块的逻辑之后，执行`ngx_http_next_header_filter`和`ngx_http_next_body_filter`方法，即原来的`ngx_http_top_header_filter`和`ngx_http_top_body_filter`方法。这样就成功将本模块的处理方法加到了处理序列的头部。

通过这种方式，将所有希望对响应头和响应体执行处理的方法进行串联。因此这时处理的顺序就十分重要，每个模块的添加阶段都是在ctx的postconfiguration方法中，并且每个模块总是将本模块的处理方法添加到序列的头部，因此处理顺序与每个模块执行postconfiguration方法的顺序相反：

```
    &ngx_http_write_filter_module,
    &ngx_http_header_filter_module,
    &ngx_http_chunked_filter_module,
    &ngx_http_range_header_filter_module,
    &ngx_http_gzip_filter_module,
    &ngx_http_postpone_filter_module,
    &ngx_http_ssi_filter_module,
    &ngx_http_charset_filter_module,
    &ngx_http_userid_filter_module,
    &ngx_http_headers_filter_module,
    &ngx_http_copy_filter_module,
    &ngx_http_range_body_filter_module,
    &ngx_http_not_modified_filter_module,
```

上述为官方定义的响应filter处理模块及其顺序，因此实际执行的顺序与上述顺序相反。

这里比较典型的，常用的模块是`ngx_http_headers_filter_module`模块和`ngx_http_gzip_filter_module`模块。

`ngx_http_headers_filter_module`模块支持通过配置`add_header`参数。来增加响应头，例如支持跨域配置：

```
location / {  
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

    if ($request_method = 'OPTIONS') {
        return 204;
    }
} 
```

`ngx_http_gzip_filter_module`模块用来对响应进行gzip压缩。要使用gzip压缩时，首先会在header方法中增加响应头：

```
Content-Encoding:gzip
```

在下发响应体时，会按照设置的`gzip`进行压缩。

其中`ngx_http_write_filter_module`和`ngx_http_header_filter_module`分别定义了最后的响应体处理函数和响应头处理函数，后续会专门进行详细讲解。其中定义的发送响应头和响应体函数分别为：`ngx_http_write_filter`和`ngx_http_header_filter`，其`ngx_http_write_filter`为实际传输数据的处理方法，`ngx_http_header_filter`方法在完成请求头的拼接序列化后，也会调用`ngx_http_write_filter`方法。

这里有一个是否重要的需要注意的点，即nginx是全异步多阶段处理模式，因此在发送响应时也是不允许阻塞的。在发送响应头时，是不能存在阻塞的函数的，即下发的过程中，每个处理函数应该都不能是阻塞的，直到最终执行数据发送阶段即`ngx_http_write_filter`函数阶段。对于只有响应只有响应头来说，如果在该阶段存在阻塞，则会设置请求的写事件为`ngx_http_write`方法，调用事件驱动来进行后续的数据发送。

对于存在响应体的响应来说，如果发送响应头时执行的`ngx_http_write_filter`方法是阻塞的，则会将未传输的数据添加到请求的`out`结构中，后续发送响应体时，会向out中再添加数据，并将写事件添加到事件模块中，通过事件发送响应。

需要着重注意的是，发送请求响应的filter链表会执行多次，因此需要每个函数自己来实现避免重复执行。

发送请求响应filter被执行多次的原因主要有如下原因：

1. 仅需要发送响应头，但响应头未一次下发完全，此时会将请求的写事件设置为`ngx_http_writer`方法，该方法被执行时将调用`ngx_http_output_filter`方法，执行发送响应头请求链路，但这时其实是没有新数据增加的，只需要吧out链表中未传输的数据发送出去即可，所以方法中的参数in为null。所有这时避免重复执行的方式有两种：1）通过查看是否响应只有请求头，即header_only是否为1。2）通过查看in参数是否为空，为空表示没有新要增加的传输数据需要处理。
2. 在需要发送响应体时，响应体在第一次调用`ngx_http_output_filter`时已经处理完成了，要全部在参数in包含的链表中了，但是并没有一次完成传输的发送。这时会将请求的写事件设置为`ngx_http_writer`方法，该方法被执行时将调用`ngx_http_output_filter`方法，执行发送响应头请求链路，但这时其实是没有新数据增加的，只需要吧out链表中未传输的数据发送出去即可，所以方法中的参数in为null。这时避免重复执行的方式为：判断in是否为空，因为在`ngx_http_writer`方法中执行的`ngx_http_output_filter`方法，in参数均为空。
3. 在需要发送响应体时，响应头并不是通过一次调用`ngx_http_output_filter`就完成了所有数据的处理，需要调用多次`ngx_http_output_filter`方法，逐步完成响应的下发。每一次执行`ngx_http_output_filter`后并不会将`ngx_http_writer`设置请求的写事件处理，而是将模块本身的处理函数设置为写事件的处理方法，该方法会重复调用`ngx_http_output_filter`方法来下发需要处理的数据，直到下发的数据完成，再调用`ngx_http_finalize_request`方法将`ngx_http_writer`方法设置为写事件的处理方法（这时就可以通过in来避免重复处理了）。在重复调用`ngx_http_output_filter`方法增加需要传输的数据的时候处理逻辑需要区分模块来判断。对于只应该执行一次处理的模块，可以通过in中最后一个buf中的last_buf来进行判断，该值表示是需要发送的最后一部分数据，那在次之前就可以不进行任何处理，直到最后一部分数据的到来，比如`ngx_http_headers_filter_module`模块的`ngx_http_trailers_filter`响应体过滤函数，其处理逻辑即是这样，该模块的方法是在发送响应的最后增加用户指定的内容（`add_trailer`配置），当日这是应该只能执行一次的方法，这时通过判断last_buf来决定是否需要进行增加响应返回的数据。对于响应整体都需要做操作时，则有更独特的处理逻辑，以`ngx_http_gzip_filter_module`模块来说，该模块的响应头过滤函数工作是将响应体进行gzip压缩，这时需要对响应体整体进行压缩的，但是不应该在每次有新数据到来时就进行压缩，而是在数据的量级达到指定的大小时再进行压缩，这样该函数并不会直接对`ngx_http_output_filter`下发的数据立即处理，而是先缓存到该阶段，等积累到指定的大小时，再进行压缩，向之后的请求体过滤函数进行下发。（这时响应方式往往以chunked方式传输）



### 响应头发送ngx_http_header_filter

在响应头通过响应头fiter链表构建完成后，最终会通过`ngx_http_header_filter`方法向用户下发响应(`ngx_http_header_filter_module`模块负责)，其处理逻辑如下：

```c
static ngx_str_t ngx_http_status_lines[] = {

    ngx_string("200 OK"),
    ngx_string("201 Created"),
    ngx_string("202 Accepted"),
    ngx_null_string,  /* "203 Non-Authoritative Information" */
    ngx_string("204 No Content"),
    ngx_null_string,  /* "205 Reset Content" */
    ngx_string("206 Partial Content")
    /*
    ...
    */
}

static ngx_int_t
ngx_http_header_filter(ngx_http_request_t *r)
{
    u_char                    *p;
    size_t                     len;
    ngx_str_t                  host, *status_line;
    ngx_buf_t                 *b;
    ngx_uint_t                 status, i, port;
    ngx_chain_t                out;
    ngx_list_part_t           *part;
    ngx_table_elt_t           *header;
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;
    ngx_http_core_srv_conf_t  *cscf;
    u_char                     addr[NGX_SOCKADDR_STRLEN];
    
    // 如果响应已经处理过了，直接跳过
    if (r->header_sent) {
        return NGX_OK;
    }
    // 标注响应头已经处理（这里发送并不是已经完成了发送）
    r->header_sent = 1;
    // 主请求才能处理请求
    if (r != r->main) {
        return NGX_OK;
    }
    // 请求http版本校验
    if (r->http_version < NGX_HTTP_VERSION_10) {
        return NGX_OK;
    }
    // 如果是head请求，则设置只有响应头
    if (r->method == NGX_HTTP_HEAD) {
        r->header_only = 1;
    }
    
    // 设置响应头last_modified_time
    if (r->headers_out.last_modified_time != -1) {
        if (r->headers_out.status != NGX_HTTP_OK // 200
            && r->headers_out.status != NGX_HTTP_PARTIAL_CONTENT // 206
            && r->headers_out.status != NGX_HTTP_NOT_MODIFIED) // 304
        {
            r->headers_out.last_modified_time = -1;
            r->headers_out.last_modified = NULL;
        }
    }
    
    // 计算请求头长度
    len = sizeof("HTTP/1.x ") - 1 + sizeof(CRLF) - 1
          /* the end of the header */
          + sizeof(CRLF) - 1;

    /* status line */
    // 如果之前的fiter处理已经设置了状态行，则直接计算，否则返回头设置请求行
    if (r->headers_out.status_line.len) {
        len += r->headers_out.status_line.len;
        status_line = &r->headers_out.status_line;
#if (NGX_SUPPRESS_WARN)
        status = 0;
#endif

    } else {
        // 获取响应状态
        status = r->headers_out.status;

        if (status >= NGX_HTTP_OK // 200
            && status < NGX_HTTP_LAST_2XX) // 207 请求已成功处理，返回了多个状态的XML消息
        {
            /* 2XX */

            if (status == NGX_HTTP_NO_CONTENT) { // 204 没有响应体
                r->header_only = 1;
                ngx_str_null(&r->headers_out.content_type);
                r->headers_out.last_modified_time = -1;
                r->headers_out.last_modified = NULL;
                r->headers_out.content_length = NULL;
                r->headers_out.content_length_n = -1;
            }

            status -= NGX_HTTP_OK;
            status_line = &ngx_http_status_lines[status]; // 从ngx_http_status_lines数组中获取响应行
            len += ngx_http_status_lines[status].len;
        
        // 响应为3XX
        } else if (status >= NGX_HTTP_MOVED_PERMANENTLY // 301
                   && status < NGX_HTTP_LAST_3XX) // 309
        {
            /* 3XX */

            if (status == NGX_HTTP_NOT_MODIFIED) { // 304 未改变，设置只需返回响应头
                r->header_only = 1;
            }
            // 获取响应行
            status = status - NGX_HTTP_MOVED_PERMANENTLY + NGX_HTTP_OFF_3XX;
            status_line = &ngx_http_status_lines[status];
            len += ngx_http_status_lines[status].len;

        // 4xx响应处理
        } else if (status >= NGX_HTTP_BAD_REQUEST // 400
                   && status < NGX_HTTP_LAST_4XX) // 430
        {
            /* 4XX */
            status = status - NGX_HTTP_BAD_REQUEST
                            + NGX_HTTP_OFF_4XX;

            status_line = &ngx_http_status_lines[status];
            len += ngx_http_status_lines[status].len;

        // 5xx响应
        } else if (status >= NGX_HTTP_INTERNAL_SERVER_ERROR
                   && status < NGX_HTTP_LAST_5XX)
        {
            /* 5XX */
            status = status - NGX_HTTP_INTERNAL_SERVER_ERROR
                            + NGX_HTTP_OFF_5XX;

            status_line = &ngx_http_status_lines[status];
            len += ngx_http_status_lines[status].len;

        } else {
            len += NGX_INT_T_LEN + 1 /* SP */;
            status_line = NULL;
        }

        if (status_line && status_line->len == 0) {
            status = r->headers_out.status;
            len += NGX_INT_T_LEN + 1 /* SP */;
            status_line = NULL;
        }
    }
    
    // 获取http core模块location配置ngx_http_core_loc_conf_t
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
 
 
    // 设置返回的server响应头，并计算长度
    if (r->headers_out.server == NULL) {
        if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_ON) {
            len += sizeof(ngx_http_server_full_string) - 1;

        } else if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_BUILD) {
            len += sizeof(ngx_http_server_build_string) - 1;

        } else {
            len += sizeof(ngx_http_server_string) - 1;
        }
    }
    
    // 设置响应data头长度
    if (r->headers_out.date == NULL) {
        len += sizeof("Date: Mon, 28 Sep 1970 06:00:00 GMT" CRLF) - 1;
    }
    
    // 获取响应的content_type头长度
    if (r->headers_out.content_type.len) {
        len += sizeof("Content-Type: ") - 1
               + r->headers_out.content_type.len + 2;

        if (r->headers_out.content_type_len == r->headers_out.content_type.len
            && r->headers_out.charset.len)
        {
            len += sizeof("; charset=") - 1 + r->headers_out.charset.len;
        }
    }
    
    // 获取响应头content_length长度
    if (r->headers_out.content_length == NULL
        && r->headers_out.content_length_n >= 0)
    {
        len += sizeof("Content-Length: ") - 1 + NGX_OFF_T_LEN + 2;
    }
    
    // 获取响应Last-Modified长度
    if (r->headers_out.last_modified == NULL
        && r->headers_out.last_modified_time != -1)
    {
        len += sizeof("Last-Modified: Mon, 28 Sep 1970 06:00:00 GMT" CRLF) - 1;
    }
    
    // 获取请求的连接
    c = r->connection;
    
    // 如果响应为重定向，并且重定向只设置了uri，并没有设置请求的host
    if (r->headers_out.location
        && r->headers_out.location->value.len
        && r->headers_out.location->value.data[0] == '/'
        && clcf->absolute_redirect)  // nginx发起的重定向是相对的,absolute_redirect配置
    {
        r->headers_out.location->hash = 0;
        
        // 指令指定的首要虚拟主机名用于发起的绝对重定向，server_name_in_redirect配置，使用server层级的host
        if (clcf->server_name_in_redirect) { 
            cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
            host = cscf->server_name;

        // 如果请求头中包含server，使用该hoast
        } else if (r->headers_in.server.len) {
            host = r->headers_in.server;

        } else {
            // 使用连接地址，通过getsockname获取host
            host.len = NGX_SOCKADDR_STRLEN;
            host.data = addr;

            if (ngx_connection_local_sockaddr(c, &host, 0) != NGX_OK) {
                return NGX_ERROR;
            }
        }
        // 获取端口号
        port = ngx_inet_get_port(c->local_sockaddr);
        
        // 添加Location部分请求头
        len += sizeof("Location: https://") - 1
               + host.len
               + r->headers_out.location->value.len + 2;
 
        // nginx发起绝对重定向（absolute_redirect on）时指定端口,port_in_redirect配置，如果重定向需要请求头，则添加对应长度
        if (clcf->port_in_redirect) {

#if (NGX_HTTP_SSL)
            if (c->ssl)
                port = (port == 443) ? 0 : port;
            else
#endif
                port = (port == 80) ? 0 : port;

        } else {
            port = 0;
        }

        if (port) {
            len += sizeof(":65535") - 1;
        }

    } else {
        // 设置host为空
        ngx_str_null(&host);
        port = 0;
    }
    
    // 如果响应为chunked模式发送，则设置相应的请求头
    if (r->chunked) {
        len += sizeof("Transfer-Encoding: chunked" CRLF) - 1;
    }
    
    // 如果响应为101，服务切换，则设置响应请求头
    if (r->headers_out.status == NGX_HTTP_SWITCHING_PROTOCOLS) {
        len += sizeof("Connection: upgrade" CRLF) - 1;
    // 如果需要keepalive，则设置对应请求头，请求是否需要保持keepalive，由之前的阶段决定
    } else if (r->keepalive) {
        len += sizeof("Connection: keep-alive" CRLF) - 1;

        /*
         * MSIE and Opera ignore the "Keep-Alive: timeout=<N>" header.
         * MSIE keeps the connection alive for about 60-65 seconds.
         * Opera keeps the connection alive very long.
         * Mozilla keeps the connection alive for N plus about 1-10 seconds.
         * Konqueror keeps the connection alive for about N seconds.
         */
        // 如果需要添加keepalive_header响应头
        if (clcf->keepalive_header) {
            len += sizeof("Keep-Alive: timeout=") - 1 + NGX_TIME_T_LEN + 2;
        }

    } else {
        // 否则关闭连接
        len += sizeof("Connection: close" CRLF) - 1;
    }

#if (NGX_HTTP_GZIP)
    // 如果设置了gzip_vary，增加响应头
    if (r->gzip_vary) {
        if (clcf->gzip_vary) {
            len += sizeof("Vary: Accept-Encoding" CRLF) - 1;

        } else {
            r->gzip_vary = 0;
        }
    }
#endif
    // 遍历没有header，获取每个响应头的长度
    part = &r->headers_out.headers.part;
    header = part->elts;

    for (i = 0; /* void */; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }

            part = part->next;
            header = part->elts;
            i = 0;
        }

        if (header[i].hash == 0) {
            continue;
        }

        len += header[i].key.len + sizeof(": ") - 1 + header[i].value.len
               + sizeof(CRLF) - 1;
    }

    // 按照之前生成的len，分配一个buf
    b = ngx_create_temp_buf(r->pool, len);
    if (b == NULL) {
        return NGX_ERROR;
    }

    // 与计算长度时类似，对buf赋值
    /* "HTTP/1.x " */
    b->last = ngx_cpymem(b->last, "HTTP/1.1 ", sizeof("HTTP/1.x ") - 1);

    /* status line */
    if (status_line) {
        b->last = ngx_copy(b->last, status_line->data, status_line->len);

    } else {
        b->last = ngx_sprintf(b->last, "%03ui ", status);
    }
    *b->last++ = CR; *b->last++ = LF;

    if (r->headers_out.server == NULL) {
        if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_ON) {
            p = ngx_http_server_full_string;
            len = sizeof(ngx_http_server_full_string) - 1;

        } else if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_BUILD) {
            p = ngx_http_server_build_string;
            len = sizeof(ngx_http_server_build_string) - 1;

        } else {
            p = ngx_http_server_string;
            len = sizeof(ngx_http_server_string) - 1;
        }

        b->last = ngx_cpymem(b->last, p, len);
    }

    if (r->headers_out.date == NULL) {
        b->last = ngx_cpymem(b->last, "Date: ", sizeof("Date: ") - 1);
        b->last = ngx_cpymem(b->last, ngx_cached_http_time.data,
                             ngx_cached_http_time.len);

        *b->last++ = CR; *b->last++ = LF;
    }

    if (r->headers_out.content_type.len) {
        b->last = ngx_cpymem(b->last, "Content-Type: ",
                             sizeof("Content-Type: ") - 1);
        p = b->last;
        b->last = ngx_copy(b->last, r->headers_out.content_type.data,
                           r->headers_out.content_type.len);

        if (r->headers_out.content_type_len == r->headers_out.content_type.len
            && r->headers_out.charset.len)
        {
            b->last = ngx_cpymem(b->last, "; charset=",
                                 sizeof("; charset=") - 1);
            b->last = ngx_copy(b->last, r->headers_out.charset.data,
                               r->headers_out.charset.len);

            /* update r->headers_out.content_type for possible logging */

            r->headers_out.content_type.len = b->last - p;
            r->headers_out.content_type.data = p;
        }

        *b->last++ = CR; *b->last++ = LF;
    }

    if (r->headers_out.content_length == NULL
        && r->headers_out.content_length_n >= 0)
    {
        b->last = ngx_sprintf(b->last, "Content-Length: %O" CRLF,
                              r->headers_out.content_length_n);
    }

    if (r->headers_out.last_modified == NULL
        && r->headers_out.last_modified_time != -1)
    {
        b->last = ngx_cpymem(b->last, "Last-Modified: ",
                             sizeof("Last-Modified: ") - 1);
        b->last = ngx_http_time(b->last, r->headers_out.last_modified_time);

        *b->last++ = CR; *b->last++ = LF;
    }

    if (host.data) {

        p = b->last + sizeof("Location: ") - 1;

        b->last = ngx_cpymem(b->last, "Location: http",
                             sizeof("Location: http") - 1);

#if (NGX_HTTP_SSL)
        if (c->ssl) {
            *b->last++ ='s';
        }
#endif

        *b->last++ = ':'; *b->last++ = '/'; *b->last++ = '/';
        b->last = ngx_copy(b->last, host.data, host.len);

        if (port) {
            b->last = ngx_sprintf(b->last, ":%ui", port);
        }

        b->last = ngx_copy(b->last, r->headers_out.location->value.data,
                           r->headers_out.location->value.len);

        /* update r->headers_out.location->value for possible logging */

        r->headers_out.location->value.len = b->last - p;
        r->headers_out.location->value.data = p;
        ngx_str_set(&r->headers_out.location->key, "Location");

        *b->last++ = CR; *b->last++ = LF;
    }

    if (r->chunked) {
        b->last = ngx_cpymem(b->last, "Transfer-Encoding: chunked" CRLF,
                             sizeof("Transfer-Encoding: chunked" CRLF) - 1);
    }

    if (r->headers_out.status == NGX_HTTP_SWITCHING_PROTOCOLS) {
        b->last = ngx_cpymem(b->last, "Connection: upgrade" CRLF,
                             sizeof("Connection: upgrade" CRLF) - 1);

    } else if (r->keepalive) {
        b->last = ngx_cpymem(b->last, "Connection: keep-alive" CRLF,
                             sizeof("Connection: keep-alive" CRLF) - 1);

        if (clcf->keepalive_header) {
            b->last = ngx_sprintf(b->last, "Keep-Alive: timeout=%T" CRLF,
                                  clcf->keepalive_header);
        }

    } else {
        b->last = ngx_cpymem(b->last, "Connection: close" CRLF,
                             sizeof("Connection: close" CRLF) - 1);
    }

#if (NGX_HTTP_GZIP)
    if (r->gzip_vary) {
        b->last = ngx_cpymem(b->last, "Vary: Accept-Encoding" CRLF,
                             sizeof("Vary: Accept-Encoding" CRLF) - 1);
    }
#endif

    part = &r->headers_out.headers.part;
    header = part->elts;

    for (i = 0; /* void */; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }

            part = part->next;
            header = part->elts;
            i = 0;
        }

        if (header[i].hash == 0) {
            continue;
        }

        b->last = ngx_copy(b->last, header[i].key.data, header[i].key.len);
        *b->last++ = ':'; *b->last++ = ' ';

        b->last = ngx_copy(b->last, header[i].value.data, header[i].value.len);
        *b->last++ = CR; *b->last++ = LF;
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "%*s", (size_t) (b->last - b->pos), b->pos);

    /* the end of HTTP header */
    *b->last++ = CR; *b->last++ = LF;
    // 设置响应头长度
    r->header_size = b->last - b->pos;
    // 如果只需要发送响应头，设置该buf为最后一个buf
    if (r->header_only) {
        b->last_buf = 1;
    }

    out.buf = b;
    out.next = NULL;
    // 调用ngx_http_write_filter向用户下发响应
    return ngx_http_write_filter(r, &out);
}
```

该函数主要是按照之前构建的响应头拼接返回的数据。

### 响应体发送ngx_http_write_filter

其实说是响应体发送并不完成准确，因为响应头也会使用该方法发送数据。该方法由`ngx_http_write_filter_module`模块负责，其处理逻辑为：

```c
// 获取需要传输数据字节
#define ngx_buf_size(b)                                                      \
    (ngx_buf_in_memory(b) ? (off_t) ((b)->last - (b)->pos):                  \
                            ((b)->file_last - (b)->file_pos))
// 判断是否为特殊buf，特殊buf包括：需要flush的|最后一个buf|需要sync同步方式的buf                       
#define ngx_buf_special(b)                                                   \
    (((b)->flush || (b)->last_buf || (b)->sync)                              \
     && !ngx_buf_in_memory(b) && !(b)->in_file)

ngx_int_t
ngx_http_write_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    off_t                      size, sent, nsent, limit;
    ngx_uint_t                 last, flush, sync;
    ngx_msec_t                 delay;
    ngx_chain_t               *cl, *ln, **ll, *chain;
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;

    if (c->error) {
        return NGX_ERROR;
    }

    size = 0;
    flush = 0;
    sync = 0;
    last = 0;
    ll = &r->out;

    /* find the size, the flush point and the last link of the saved chain */
    /*
    r->out表示缓存的未发送的buf数据
    */
    for (cl = r->out; cl; cl = cl->next) {
        ll = &cl->next;

        ngx_log_debug7(NGX_LOG_DEBUG_EVENT, c->log, 0,
                       "write old buf t:%d f:%d %p, pos %p, size: %z "
                       "file: %O, size: %O",
                       cl->buf->temporary, cl->buf->in_file,
                       cl->buf->start, cl->buf->pos,
                       cl->buf->last - cl->buf->pos,
                       cl->buf->file_pos,
                       cl->buf->file_last - cl->buf->file_pos);

        /*
        如果buf块数据带发送的为0，并且不是特殊buf（只有特殊buf允许在buf不为0时，还在out数组中）
        */
        if (ngx_buf_size(cl->buf) == 0 && !ngx_buf_special(cl->buf)) {
            ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                          "zero size buf in writer "
                          "t:%d r:%d f:%d %p %p-%p %p %O-%O",
                          cl->buf->temporary,
                          cl->buf->recycled,
                          cl->buf->in_file,
                          cl->buf->start,
                          cl->buf->pos,
                          cl->buf->last,
                          cl->buf->file,
                          cl->buf->file_pos,
                          cl->buf->file_last);
            // 发送信号，返回eror
            ngx_debug_point();
            return NGX_ERROR;
        }
        
        // 如果buf小于0，报错
        if (ngx_buf_size(cl->buf) < 0) {
            ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                          "negative size buf in writer "
                          "t:%d r:%d f:%d %p %p-%p %p %O-%O",
                          cl->buf->temporary,
                          cl->buf->recycled,
                          cl->buf->in_file,
                          cl->buf->start,
                          cl->buf->pos,
                          cl->buf->last,
                          cl->buf->file,
                          cl->buf->file_pos,
                          cl->buf->file_last);

            ngx_debug_point();
            return NGX_ERROR;
        }
        
        // 计算待发送数据数量
        size += ngx_buf_size(cl->buf);
        
        // 如果buf需要flush或者是可复用的（需要立即发送，不然会被复用），设置需要flush
        if (cl->buf->flush || cl->buf->recycled) {
            flush = 1;
        }
        // 如果buf是同步的，设置syn
        if (cl->buf->sync) {
            sync = 1;
        }
        // 如果是最后一块buf，设置last
        if (cl->buf->last_buf) {
            last = 1;
        }
    }

    /* add the new chain to the existent one */
    // 遍历新的带发送的buf，将其添加到out后面
    for (ln = in; ln; ln = ln->next) {
        cl = ngx_alloc_chain_link(r->pool);
        if (cl == NULL) {
            return NGX_ERROR;
        }

        cl->buf = ln->buf;
        *ll = cl;
        ll = &cl->next;

        ngx_log_debug7(NGX_LOG_DEBUG_EVENT, c->log, 0,
                       "write new buf t:%d f:%d %p, pos %p, size: %z "
                       "file: %O, size: %O",
                       cl->buf->temporary, cl->buf->in_file,
                       cl->buf->start, cl->buf->pos,
                       cl->buf->last - cl->buf->pos,
                       cl->buf->file_pos,
                       cl->buf->file_last - cl->buf->file_pos);
        // 与out一样，判断是否数据错误
        if (ngx_buf_size(cl->buf) == 0 && !ngx_buf_special(cl->buf)) {
            ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                          "zero size buf in writer "
                          "t:%d r:%d f:%d %p %p-%p %p %O-%O",
                          cl->buf->temporary,
                          cl->buf->recycled,
                          cl->buf->in_file,
                          cl->buf->start,
                          cl->buf->pos,
                          cl->buf->last,
                          cl->buf->file,
                          cl->buf->file_pos,
                          cl->buf->file_last);

            ngx_debug_point();
            return NGX_ERROR;
        }

        if (ngx_buf_size(cl->buf) < 0) {
            ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                          "negative size buf in writer "
                          "t:%d r:%d f:%d %p %p-%p %p %O-%O",
                          cl->buf->temporary,
                          cl->buf->recycled,
                          cl->buf->in_file,
                          cl->buf->start,
                          cl->buf->pos,
                          cl->buf->last,
                          cl->buf->file,
                          cl->buf->file_pos,
                          cl->buf->file_last);

            ngx_debug_point();
            return NGX_ERROR;
        }

        size += ngx_buf_size(cl->buf);

        if (cl->buf->flush || cl->buf->recycled) {
            flush = 1;
        }

        if (cl->buf->sync) {
            sync = 1;
        }

        if (cl->buf->last_buf) {
            last = 1;
        }
    }

    *ll = NULL;

    ngx_log_debug3(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http write filter: l:%ui f:%ui s:%O", last, flush, size);

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    /*
     * avoid the output if there are no last buf, no flush point,
     * there are the incoming bufs and the size of all bufs
     * is smaller than "postpone_output" directive
     */
    /*
    如果不是最后的buf，或者需要flush，并且in不为空，但总的siez小于postpone_output时，直接返回
    postpone_output可配置
    避免太小的块发送
    */
    if (!last && !flush && in && size < (off_t) clcf->postpone_output) {
        return NGX_OK;
    }
    
    // 如果写事件需要delayed，设置buffered标志位
    if (c->write->delayed) {
        c->buffered |= NGX_HTTP_WRITE_BUFFERED;
        return NGX_AGAIN;
    }

    /* 
    如果buffer总大小为0，而且当前连接之前没有由于底层发送接口的原因延迟，则检查是否有特殊标记 
     */
    if (size == 0
        && !(c->buffered & NGX_LOWLEVEL_BUFFERED)
        && !(last && c->need_last_buf))
    {
        // 如果有last，flush，syn任意一个条件满足
        if (last || flush || sync) {
            // 清空out数组
            for (cl = r->out; cl; /* void */) {
                ln = cl;
                cl = cl->next;
                ngx_free_chain(r->pool, ln);
            }

            r->out = NULL;
            c->buffered &= ~NGX_HTTP_WRITE_BUFFERED;
            // 返回ok
            return NGX_OK;
        }
        // 否则报错，返回
        ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                      "the http output chain is empty");

        ngx_debug_point();

        return NGX_ERROR;
    }
    
    // 计算限速相关内容，避免后续重复计算，增加limit_rate_set位
    if (!r->limit_rate_set) {
        r->limit_rate = ngx_http_complex_value_size(r, clcf->limit_rate, 0);
        r->limit_rate_set = 1;
    }
    
    // 如果需要限速，则获取本次最多下发的数据数量
    if (r->limit_rate) {

        if (!r->limit_rate_after_set) {
            r->limit_rate_after = ngx_http_complex_value_size(r,
                                                    clcf->limit_rate_after, 0);
            r->limit_rate_after_set = 1;
        }

        limit = (off_t) r->limit_rate * (ngx_time() - r->start_sec + 1)
                - (c->sent - r->limit_rate_after);
        // 如果允许下发数量为0，则添加写事件到定时器红黑树中，并设置等待标识
        if (limit <= 0) {
            c->write->delayed = 1;
            delay = (ngx_msec_t) (- limit * 1000 / r->limit_rate + 1);
            ngx_add_timer(c->write, delay);

            c->buffered |= NGX_HTTP_WRITE_BUFFERED;

            return NGX_AGAIN;
        }
        // 如果设置了最大发送chunk，判断是否需要重新设置limit
        if (clcf->sendfile_max_chunk
            && (off_t) clcf->sendfile_max_chunk < limit)
        {
            limit = clcf->sendfile_max_chunk;
        }

    } else {
        // 否则最大发送数量为sendfile_max_chunk值
        limit = clcf->sendfile_max_chunk;
    }
    
    // 记录已经发送字节数量
    sent = c->sent;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http write filter limit %O", limit);
    // 执行send_chain下发响应，这里的函数区分平台不一致，linux下在epoll模块的epoll init中设置，为ngx_writev_chain函数
    chain = c->send_chain(c, r->out, limit);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http write filter %p", chain);
    // 如果错误，则返回
    if (chain == NGX_CHAIN_ERROR) {
        c->error = 1;
        return NGX_ERROR;
    }

    // 根据实际下发的字节数量，设置限速
    if (r->limit_rate) {

        nsent = c->sent;

        if (r->limit_rate_after) {

            sent -= r->limit_rate_after;
            if (sent < 0) {
                sent = 0;
            }

            nsent -= r->limit_rate_after;
            if (nsent < 0) {
                nsent = 0;
            }
        }

        delay = (ngx_msec_t) ((nsent - sent) * 1000 / r->limit_rate);

        if (delay > 0) {
            limit = 0;
            c->write->delayed = 1;
            ngx_add_timer(c->write, delay);
        }
    }
    
    // 如果下发的数量剩余小于2 * ngx_pagesize，避免过快再次发送，增加1ms延时
    if (limit
        && c->write->ready
        && c->sent - sent >= limit - (off_t) (2 * ngx_pagesize))
    {
        c->write->delayed = 1;
        ngx_add_timer(c->write, 1);
    }
    // 清空已经下发的out结构中的链表元素
    for (cl = r->out; cl && cl != chain; /* void */) {
        ln = cl;
        cl = cl->next;
        ngx_free_chain(r->pool, ln);
    }

    r->out = chain;
    // 如果数据还未发送完，则设置标志，返回
    if (chain) {
        c->buffered |= NGX_HTTP_WRITE_BUFFERED;
        return NGX_AGAIN;
    }
    // 如果发送完全了，则设置标志
    c->buffered &= ~NGX_HTTP_WRITE_BUFFERED;

    /* 如果由于底层发送接口导致数据未发送完全，且当前请求没有其他数据需要发送，
       此时要返回NGX_AGAIN，表示还有数据未发送 */
    if ((c->buffered & NGX_LOWLEVEL_BUFFERED) && r->postponed == NULL) {
        return NGX_AGAIN;
    }

    return NGX_OK;
}
```

上面大致分析了请求发送的过程，该函数返回值有三种：

1. `NGX_AGAIN`当前下发到该函数的数据还未发送完全，需要后续再次调用该函数下发数据。
2. `NGX_OK`当前下发到该函数的数据已经完成了发送，但需注意，这里的完成发送并非是说向用户发送的数据一定全部发送完成了，只是到该函数的数据已经发送完成了，如果当前到达该函数的数据只是要发送数据的一部分，后续部分还需要调用该函数来进行数据的下发。
3. `NGX_ERROR`发送数据出错。

#### ngx_writer_chain方法

在linux下，上述函数中执行的`send_chain`为ngx_writev_chain函数，其实现细节如下：

```c
ngx_chain_t *
ngx_writev_chain(ngx_connection_t *c, ngx_chain_t *in, off_t limit)
{
    ssize_t        n, sent;
    off_t          send, prev_send;
    ngx_chain_t   *cl;
    ngx_event_t   *wev;
    ngx_iovec_t    vec;
    struct iovec   iovs[NGX_IOVS_PREALLOCATE];

    wev = c->write;
    // 如果写事件尚未ready，则直接返回
    if (!wev->ready) {
        return in;
    }

#if (NGX_HAVE_KQUEUE)

    ...

#endif

    /* the maximum limit size is the maximum size_t value - the page size */
    // 限制一次最多下发数据数量
    if (limit == 0 || limit > (off_t) (NGX_MAX_SIZE_T_VALUE - ngx_pagesize)) {
        limit = NGX_MAX_SIZE_T_VALUE - ngx_pagesize;
    }

    send = 0;

    vec.iovs = iovs;
    vec.nalloc = NGX_IOVS_PREALLOCATE;

    for ( ;; ) {
        prev_send = send;

        /* create the iovec and coalesce the neighbouring bufs */
        // 将in链表中的数据填充到vec中，返回到第一个尚未填充的ngx_chain_t结构
        cl = ngx_output_chain_to_iovec(&vec, in, limit - send, c->log);

        if (cl == NGX_CHAIN_ERROR) {
            return NGX_CHAIN_ERROR;
        }
        
        if (cl && cl->buf->in_file) {
            ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                          "file buf in writev "
                          "t:%d r:%d f:%d %p %p-%p %p %O-%O",
                          cl->buf->temporary,
                          cl->buf->recycled,
                          cl->buf->in_file,
                          cl->buf->start,
                          cl->buf->pos,
                          cl->buf->last,
                          cl->buf->file,
                          cl->buf->file_pos,
                          cl->buf->file_last);

            ngx_debug_point();

            return NGX_CHAIN_ERROR;
        }
        // 记录需要发送的字节数量
        send += vec.size;
        // 调用writev方法下发数据
        n = ngx_writev(c, &vec);
        // 出错返回
        if (n == NGX_ERROR) {
            return NGX_CHAIN_ERROR;
        }
        
        // 记录实际发送的字节数
        sent = (n == NGX_AGAIN) ? 0 : n;
        // sent增加实际发送的字节数
        c->sent += sent;
        // 按照实际发送的字节数更新当前发送到了哪个ngx_chain_t结构，并返回
        in = ngx_chain_update_sent(in, sent);
        // 如果希望发送的数量和实际发送的数量不一致，设置写事件未ready
        if (send - prev_send != sent) {
            wev->ready = 0;
            return in;
        }
        // 如果发送数量超过了limit或者已经发送完了，则返回
        if (send >= limit || in == NULL) {
            return in;
        }
    }
}
```

关于`writev`的使用可以参考如下文档：[writev](https://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%87%BD%E6%95%B0readv%E5%92%8Cwritev)。



### ngx_http_writer方法

在上文已经介绍过，在最终下发数据时，如果没有一次下发成功，将会设置写事件的处理函数为`ngx_http_writer`方法（一般是在`ngx_http_finalize_request`中设置），来继续进行数据的下发，这里详细介绍该方法的执行逻辑：

```c
static void
ngx_http_writer(ngx_http_request_t *r)
{
    ngx_int_t                  rc;
    ngx_event_t               *wev;
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;
    wev = c->write;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, wev->log, 0,
                   "http writer handler: \"%V?%V\"", &r->uri, &r->args);

    clcf = ngx_http_get_module_loc_conf(r->main, ngx_http_core_module);
    // 如果请求已超时，则直接结束请求
    if (wev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT,
                      "client timed out");
        c->timedout = 1;

        ngx_http_finalize_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }
    
    // 如果写事件被delayed（限速）
    if (wev->delayed || r->aio) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, wev->log, 0,
                       "http writer delayed");
        // 如果不是被延迟了，设置发送超时时间,send_timeout配置
        if (!wev->delayed) {
            ngx_add_timer(wev, clcf->send_timeout);
        }
        // 设置写事件
        if (ngx_handle_write_event(wev, clcf->send_lowat) != NGX_OK) {
            ngx_http_close_request(r, 0);
        }

        return;
    }
    // 执行响应体下发fiter链表。这里就是之前说的为何需要响应头控制避免重复执行
    rc = ngx_http_output_filter(r, NULL);

    ngx_log_debug3(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http writer output filter: %i, \"%V?%V\"",
                   rc, &r->uri, &r->args);
    // 如果返回错误，则结束请求
    if (rc == NGX_ERROR) {
        ngx_http_finalize_request(r, rc);
        return;
    }
    
    // 如果buffered不为0，表示数据未完全发送
    if (r->buffered || r->postponed || (r == r->main && c->buffered)) {
        // 如果不是被delay，则条件send_timeout超时事件
        if (!wev->delayed) {
            ngx_add_timer(wev, clcf->send_timeout);
        }
        // 设置写事件
        if (ngx_handle_write_event(wev, clcf->send_lowat) != NGX_OK) {
            ngx_http_close_request(r, 0);
        }

        return;
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, wev->log, 0,
                   "http writer done: \"%V?%V\"", &r->uri, &r->args);
    // 如果数据已经发送完全，则设置写处理函数为空方法
    r->write_event_handler = ngx_http_request_empty_handler;
    // 结束请求
    ngx_http_finalize_request(r, rc);
}
```



## 请求体处理

对于post请求来说，nginx需要处理其请求体。请求体处理有两种方式，直接丢弃请求体和接收并执行相关操作。选择何种方式来处理请求体往往由模块本身决定。

接收包体时，为防止请求被其他相关操作终止，应当将请求的count加1。再完成接收时，将计数减1。对应丢弃请求体来说，加减均由架构函数完成，对应接收请求来说，加架构完成了，减需要自定义的回调函数来完成。

下面这里对两种方式进行介绍。

### 相关结构

用于处理请求体的相关结构为`ngx_http_request_body_t`.

#### ngx_http_request_body_t

其定义如下：

```c
struct ngx_http_chunked_s {
    ngx_uint_t           state;
    off_t                size;
    off_t                length;
};

typedef void (*ngx_http_client_body_handler_pt)(ngx_http_request_t *r);


typedef struct {
    // 存放http包体的临时文件
    ngx_temp_file_t                  *temp_file;
    // 接收包体的缓冲区链表，当包体全部存放缓冲区时，如果一块无法存储下，则需要使用缓冲区链表来存储
    ngx_chain_t                      *bufs;
    // 接收包体的缓冲区链表
    ngx_buf_t                        *buf;
    /*
    根据content-length头部和已接收的包体长度，计算出还需要接收的包体长度。
    chunked下该值为已接收请求长度到最大允许的长度large_client_header_buffers直接差值，用于判断是否超限
    */
    off_t                             rest;
    off_t                             received;
    ngx_chain_t                      *free;
    ngx_chain_t                      *busy;
    // 存储接收chunked请求时，接收状态
    ngx_http_chunked_t               *chunked;
    // 接收完成后的回调方法
    ngx_http_client_body_handler_pt   post_handler;
} ngx_http_request_body_t;
```



### 接收请求体ngx_http_read_client_request_body

接收客户端请求通过`ngx_http_read_client_request_body`方法实现，其逻辑如下：

```c
// 参数post_handler为接受完成请求头后处理函数
ngx_int_t
ngx_http_read_client_request_body(ngx_http_request_t *r,
    ngx_http_client_body_handler_pt post_handler)
{
    size_t                     preread;
    ssize_t                    size;
    ngx_int_t                  rc;
    ngx_buf_t                 *b;
    ngx_chain_t                out;
    ngx_http_request_body_t   *rb;
    ngx_http_core_loc_conf_t  *clcf;
    // 主请求上引用计数+1
    r->main->count++;
    /*
    请求不是主请求，或者已经接收了请求体，或者需要丢弃请求体
    设置请求体没有buf存储，直接执行回调函数
    */
    if (r != r->main || r->request_body || r->discard_body) {
        r->request_body_no_buffering = 0;
        post_handler(r);
        return NGX_OK;
    }
    
    /*
    Expect 是一个请求消息头，包含一个期望条件，表示服务器只有在满足此期望条件的情况下才能妥善地处理请求。
    规范中只规定了一个期望条件，即 Expect: 100-continue, 对此服务器可以做出如下回应：
    100 如果消息头中的期望条件可以得到满足，使得请求可以顺利进行的话，
    417 (Expectation Failed) 如果服务器不能满足期望条件的话；也可以是其他任意表示客户端错误的状态码（4xx）。
    这里进行判断，是否请求头包含该值，如果是，并且值为100-continue，则发送HTTP/1.1 100 Continue响应
    使用r->expect_tested记录是否发送过该值
    并且这次发送不会结果事件驱动的处理，nginx任务数据过少，正常情况下都应该能够立即发送完成。
    */
    if (ngx_http_test_expect(r) != NGX_OK) {
        rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
        goto done;
    }
    
    // 为请求分配一个ngx_http_request_body_t结构
    rb = ngx_pcalloc(r->pool, sizeof(ngx_http_request_body_t));
    if (rb == NULL) {
        rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
        goto done;
    }

    /*
     * set by ngx_pcalloc():
     *
     *     rb->bufs = NULL;
     *     rb->buf = NULL;
     *     rb->free = NULL;
     *     rb->busy = NULL;
     *     rb->chunked = NULL;
     */
    // 设置还需要的字节为-1
    rb->rest = -1;
    // 设置回调方法
    rb->post_handler = post_handler;

    r->request_body = rb;
    
    // 如果请求未设置请求体长度，并且非chunked的请求，则直接执行回调方法
    if (r->headers_in.content_length_n < 0 && !r->headers_in.chunked) {
        r->request_body_no_buffering = 0;
        post_handler(r);
        return NGX_OK;
    }

#if (NGX_HTTP_V2)
    if (r->stream) {
        rc = ngx_http_v2_read_request_body(r);
        goto done;
    }
#endif
    
    // 查看是否响应头中还有部分数据，如果存在部分数据，就应该是请求体的数据
    preread = r->header_in->last - r->header_in->pos;
    
    // 已经接收到部分请求体了
    if (preread) {

        /* there is the pre-read part of the request body */

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http client request body preread %uz", preread);
        // 设置buf
        out.buf = r->header_in;
        out.next = NULL;
        // 执行请求体处理方法，该方法对请求体执行过滤，和响应头和响应体一样，可以由模块定义过滤方法
        rc = ngx_http_request_body_filter(r, &out);

        if (rc != NGX_OK) {
            goto done;
        }
        
        /*
        增加请求长度，其中r->header_in->last - r->header_in->pos为解析后，请求头中剩余未解析的长度
        */
        r->request_length += preread - (r->header_in->last - r->header_in->pos);
        
        // 如果非chunked请求，并且剩余需要的字节数小于header_in中剩余数据，说明header_in结构就完成存储了请求体
        if (!r->headers_in.chunked
            && rb->rest > 0
            && rb->rest <= (off_t) (r->header_in->end - r->header_in->last))
        {
            /* the whole request body may be placed in r->header_in */
            // 参加buf，并设置相关数据信息
            b = ngx_calloc_buf(r->pool);
            if (b == NULL) {
                rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
                goto done;
            }

            b->temporary = 1;
            b->start = r->header_in->pos;
            b->pos = r->header_in->pos;
            b->last = r->header_in->last;
            b->end = r->header_in->end;

            rb->buf = b;

            r->read_event_handler = ngx_http_read_client_request_body_handler;
            r->write_event_handler = ngx_http_request_empty_handler;

            rc = ngx_http_do_read_client_request_body(r);
            goto done;
        }

    } else {
        /* set rb->rest */
        // 无数据时，仅会设置rb->rest
        if (ngx_http_request_body_filter(r, NULL) != NGX_OK) {
            rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
            goto done;
        }
    }
    
    // 如果rest为0，说明请求体已完成读取，执行回调方法
    if (rb->rest == 0) {
        /* the whole request body was pre-read */
        r->request_body_no_buffering = 0;
        post_handler(r);
        return NGX_OK;
    }
    // 出错
    if (rb->rest < 0) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
                      "negative request body rest");
        rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
        goto done;
    }
    
    /* 
    获取客户端请求体默认存储buf大小
    如果请求主体大于缓冲区，则将整个主体或仅其一部分写入临时文件。
    */
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    
    size = clcf->client_body_buffer_size;
    size += size >> 2;

    /* TODO: honor r->request_body_in_single_buf */
    // 尽可能保证请求体存在一个buf中
    if (!r->headers_in.chunked && rb->rest < size) {
        size = (ssize_t) rb->rest;

        if (r->request_body_in_single_buf) {
            size += preread;
        }

    } else {
        size = clcf->client_body_buffer_size;
    }

    rb->buf = ngx_create_temp_buf(r->pool, size);
    if (rb->buf == NULL) {
        rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
        goto done;
    }
    // 设置请求读事件处理函数，写事件处理函数
    r->read_event_handler = ngx_http_read_client_request_body_handler;
    r->write_event_handler = ngx_http_request_empty_handler;
    // 执行执行一次请求体读取解析
    rc = ngx_http_do_read_client_request_body(r);

done:
    // 如果对请求体不需要缓存，即对每部分数据读取到后都立即执行处理
    if (r->request_body_no_buffering
        && (rc == NGX_OK || rc == NGX_AGAIN))
    {   
        // 请求完成
        if (rc == NGX_OK) {
            r->request_body_no_buffering = 0;

        } else {
            // 未完成，设置reading_body，表示还有数据需要读取
            /* rc == NGX_AGAIN */
            r->reading_body = 1;
        }
        // 设置读事件为空
        r->read_event_handler = ngx_http_block_reading;
        // 执行回调方法
        post_handler(r);
    }

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        r->main->count--;
    }

    return rc;
}
```



#### ngx_http_request_body_filter

`ngx_http_request_body_filter`函数执行接收到的请求体处理。

```c
static ngx_int_t
ngx_http_request_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    // 判断请求类型，决定处理方式
    if (r->headers_in.chunked) {
        return ngx_http_request_body_chunked_filter(r, in);

    } else {
        return ngx_http_request_body_length_filter(r, in);
    }
}
```



#### ngx_http_request_body_chunked_filter

`ngx_http_request_body_chunked_filter`为chunked方式下对请求内容的处理。其逻辑如下：

```c
static ngx_int_t
ngx_http_request_body_chunked_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    size_t                     size;
    ngx_int_t                  rc;
    ngx_buf_t                 *b;
    ngx_chain_t               *cl, *out, *tl, **ll;
    ngx_http_request_body_t   *rb;
    ngx_http_core_loc_conf_t  *clcf;
    ngx_http_core_srv_conf_t  *cscf;

    rb = r->request_body;
    // 如果未设置rest
    if (rb->rest == -1) {

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http request body chunked filter");
        // 构建chunked结构，用于存储解析chunked状态
        rb->chunked = ngx_pcalloc(r->pool, sizeof(ngx_http_chunked_t));
        if (rb->chunked == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);

        r->headers_in.content_length_n = 0;
        // 设置请求体最大buf大小
        rb->rest = cscf->large_client_header_buffers.size;
    }

    out = NULL;
    ll = &out;
    // 遍历每个buf链表
    for (cl = in; cl; cl = cl->next) {

        b = NULL;

        for ( ;; ) {

            ngx_log_debug7(NGX_LOG_DEBUG_EVENT, r->connection->log, 0,
                           "http body chunked buf "
                           "t:%d f:%d %p, pos %p, size: %z file: %O, size: %O",
                           cl->buf->temporary, cl->buf->in_file,
                           cl->buf->start, cl->buf->pos,
                           cl->buf->last - cl->buf->pos,
                           cl->buf->file_pos,
                           cl->buf->file_last - cl->buf->file_pos);
            // 解析chunked数据
            rc = ngx_http_parse_chunked(r, cl->buf, rb->chunked);
            // 如果完成一个chunked的解析
            if (rc == NGX_OK) {

                /* a chunk has been parsed successfully */

                clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
                // 如果请求体过大，则报错返回
                if (clcf->client_max_body_size
                    && clcf->client_max_body_size
                       - r->headers_in.content_length_n < rb->chunked->size)
                {
                    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                                  "client intended to send too large chunked "
                                  "body: %O+%O bytes",
                                  r->headers_in.content_length_n,
                                  rb->chunked->size);

                    r->lingering_close = 1;

                    return NGX_HTTP_REQUEST_ENTITY_TOO_LARGE;
                }
                // 设置buf，将其添加到out链表中
                if (b
                    && rb->chunked->size <= 128
                    && cl->buf->last - cl->buf->pos >= rb->chunked->size)
                {
                    r->headers_in.content_length_n += rb->chunked->size;

                    if (rb->chunked->size < 8) {

                        while (rb->chunked->size) {
                            *b->last++ = *cl->buf->pos++;
                            rb->chunked->size--;
                        }

                    } else {
                        ngx_memmove(b->last, cl->buf->pos, rb->chunked->size);
                        b->last += rb->chunked->size;
                        cl->buf->pos += rb->chunked->size;
                        rb->chunked->size = 0;
                    }

                    continue;
                }

                tl = ngx_chain_get_free_buf(r->pool, &rb->free);
                if (tl == NULL) {
                    return NGX_HTTP_INTERNAL_SERVER_ERROR;
                }

                b = tl->buf;

                ngx_memzero(b, sizeof(ngx_buf_t));

                b->temporary = 1;
                b->tag = (ngx_buf_tag_t) &ngx_http_read_client_request_body;
                b->start = cl->buf->pos;
                b->pos = cl->buf->pos;
                b->last = cl->buf->last;
                b->end = cl->buf->end;
                b->flush = r->request_body_no_buffering;

                *ll = tl;
                ll = &tl->next;

                size = cl->buf->last - cl->buf->pos;

                if ((off_t) size > rb->chunked->size) {
                    cl->buf->pos += (size_t) rb->chunked->size;
                    r->headers_in.content_length_n += rb->chunked->size;
                    rb->chunked->size = 0;

                } else {
                    rb->chunked->size -= size;
                    r->headers_in.content_length_n += size;
                    cl->buf->pos = cl->buf->last;
                }

                b->last = cl->buf->pos;

                continue;
            }
            // 如果解析完成
            if (rc == NGX_DONE) {

                /* a whole response has been parsed successfully */
                // 设置rest为0，表示不需要数据了
                rb->rest = 0;
                // 创建buf，标识为最后一个buf，加到out中
                tl = ngx_chain_get_free_buf(r->pool, &rb->free);
                if (tl == NULL) {
                    return NGX_HTTP_INTERNAL_SERVER_ERROR;
                }

                b = tl->buf;

                ngx_memzero(b, sizeof(ngx_buf_t));

                b->last_buf = 1;

                *ll = tl;
                ll = &tl->next;

                break;
            }
            
            if (rc == NGX_AGAIN) {

                /* set rb->rest, amount of data we want to see next time */

                cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);

                rb->rest = ngx_max(rb->chunked->length,
                               (off_t) cscf->large_client_header_buffers.size);

                break;
            }

            /* invalid */

            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "client sent invalid chunked body");

            return NGX_HTTP_BAD_REQUEST;
        }
    }
    // 执行ngx_http_top_request_body_filter过滤
    rc = ngx_http_top_request_body_filter(r, out);
    // 更新buf链表
    ngx_chain_update_chains(r->pool, &rb->free, &rb->busy, &out,
                            (ngx_buf_tag_t) &ngx_http_read_client_request_body);

    return rc;
}
```



#### ngx_http_request_body_length_filter

该函数执行非chunked请求的请求体过滤函数。逻辑如下：

```c
static ngx_int_t
ngx_http_request_body_length_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    size_t                     size;
    ngx_int_t                  rc;
    ngx_buf_t                 *b;
    ngx_chain_t               *cl, *tl, *out, **ll;
    ngx_http_request_body_t   *rb;

    rb = r->request_body;
    // 如果rest还未设置，则设置为对应请求头中长度
    if (rb->rest == -1) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http request body content length filter");

        rb->rest = r->headers_in.content_length_n;
    }

    out = NULL;
    ll = &out;
    // 遍历每一个inbuf
    for (cl = in; cl; cl = cl->next) {
        // 如果rest为0，表示完成了请求体的接收，跳出循环
        if (rb->rest == 0) {
            break;
        }
        // 从rb的free中获取一个空闲的buf
        tl = ngx_chain_get_free_buf(r->pool, &rb->free);
        if (tl == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        b = tl->buf;
        
        ngx_memzero(b, sizeof(ngx_buf_t));
        // 设置buf熟悉
        b->temporary = 1;
        b->tag = (ngx_buf_tag_t) &ngx_http_read_client_request_body;
        // 设置buf需要处理字段为当前in中buf字段
        b->start = cl->buf->pos;
        b->pos = cl->buf->pos;
        b->last = cl->buf->last;
        b->end = cl->buf->end;
        // 判断是否需要flush，request_body_no_buffering表示是否需要将请求写到文件中1表示不需要，0表示需要
        b->flush = r->request_body_no_buffering;
        // 获取字段长度
        size = cl->buf->last - cl->buf->pos;
        // 如果长度小于需要剩余的字段，则更新rest，并设置cl全部已处理
        if ((off_t) size < rb->rest) {
            cl->buf->pos = cl->buf->last;
            rb->rest -= size;

        } else {
            // 否则，更新buf到对应结尾位置
            cl->buf->pos += (size_t) rb->rest;
            // 设置rest
            rb->rest = 0;
            // 设置buf的last为结尾位置
            b->last = cl->buf->pos;
            // 设置是最后一个buf
            b->last_buf = 1;
        }
        // 将其添加到out链表中
        *ll = tl;
        ll = &tl->next;
    }
    
    // 执行ngx_http_top_request_body_filter过滤函数
    rc = ngx_http_top_request_body_filter(r, out);
    
    // 更新链表
    ngx_chain_update_chains(r->pool, &rb->free, &rb->busy, &out,
                            (ngx_buf_tag_t) &ngx_http_read_client_request_body);

    return rc;
}
```

request_body_no_buffering标志启用读取请求主体的非缓冲模式。 在这种模式下，在调用ngx_http_read_client_request_body之后，bufs链可能仅保留正文的一部分。 要阅读下一部分，需调用ngx_http_read_unbuffered_request_body函数。 返回值NGX_AGAIN和请求标志reading_body表示有更多数据可用。 如果在调用此函数后bufs为NULL，那么此刻没有任何可读的内容。 请求回调read_event_handler将在请求主体的下一部分可用时被调用。

#### ngx_http_top_request_body_filter

`ngx_http_top_request_body_filter`方法与请求体和请求头过滤函数类似，可以通过每个模块设置next来串联成一个链表，目前该模块就只有一个函数`ngx_http_request_body_save_filter`，其逻辑如下：

```c
ngx_int_t
ngx_http_request_body_save_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_buf_t                 *b;
    ngx_chain_t               *cl;
    ngx_http_request_body_t   *rb;

    rb = r->request_body;

#if (NGX_DEBUG)

#if 0
    ...
#endif
    // 遍历，记录日志
    for (cl = in; cl; cl = cl->next) {
        ngx_log_debug7(NGX_LOG_DEBUG_EVENT, r->connection->log, 0,
                       "http body new buf t:%d f:%d %p, pos %p, size: %z "
                       "file: %O, size: %O",
                       cl->buf->temporary, cl->buf->in_file,
                       cl->buf->start, cl->buf->pos,
                       cl->buf->last - cl->buf->pos,
                       cl->buf->file_pos,
                       cl->buf->file_last - cl->buf->file_pos);
    }

#endif

    /* TODO: coalesce neighbouring buffers */
    // 将in数组追加到rb-bufs后，这里不是简单拼接，而是为每一个元素生成一个ngx_chain_t结构，不过其中的buf依然是以前的buf
    if (ngx_chain_add_copy(r->pool, &rb->bufs, in) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    // 如果不需要写入，则结束，否则执行写入临时文件逻辑，这里暂时不做介绍。
    if (r->request_body_no_buffering) {
        return NGX_OK;
    }

    if (rb->rest > 0) {

        if (rb->buf && rb->buf->last == rb->buf->end
            && ngx_http_write_request_body(r) != NGX_OK)
        {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        return NGX_OK;
    }

    /* rb->rest == 0 */

    if (rb->temp_file || r->request_body_in_file_only) {

        if (ngx_http_write_request_body(r) != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        if (rb->temp_file->file.offset != 0) {

            cl = ngx_chain_get_free_buf(r->pool, &rb->free);
            if (cl == NULL) {
                return NGX_HTTP_INTERNAL_SERVER_ERROR;
            }

            b = cl->buf;

            ngx_memzero(b, sizeof(ngx_buf_t));

            b->in_file = 1;
            b->file_last = rb->temp_file->file.offset;
            b->file = &rb->temp_file->file;

            rb->bufs = cl;
        }
    }

    return NGX_OK;
}
```



#### ngx_chain_update_chains

`该·函数更新链表，释放已经添加到rb->bufs中的链表，用于后续再次接收数据。其逻辑如下：

```c
void
ngx_chain_update_chains(ngx_pool_t *p, ngx_chain_t **free, ngx_chain_t **busy,
    ngx_chain_t **out, ngx_buf_tag_t tag)
{
    ngx_chain_t  *cl;
    // 将out加到busy队列中
    if (*out) {
        if (*busy == NULL) {
            *busy = *out;

        } else {
            for (cl = *busy; cl->next; cl = cl->next) { /* void */ }

            cl->next = *out;
        }

        *out = NULL;
    }
    // 删除busy队列中已经处理完成的元素，将其添加到free队列中,保留未处理完成的buf
    while (*busy) {
        cl = *busy;

        if (ngx_buf_size(cl->buf) != 0) {
            break;
        }

        if (cl->buf->tag != tag) {
            *busy = cl->next;
            ngx_free_chain(p, cl);
            continue;
        }

        cl->buf->pos = cl->buf->start;
        cl->buf->last = cl->buf->start;

        *busy = cl->next;
        cl->next = *free;
        *free = cl;
    }
}
```



#### ngx_http_do_read_client_request_body

该方法执行请求体解析，并且是读请求体时的事件循环，逻辑如下：

```c
static ngx_int_t
ngx_http_do_read_client_request_body(ngx_http_request_t *r)
{
    off_t                      rest;
    size_t                     size;
    ssize_t                    n;
    ngx_int_t                  rc;
    ngx_chain_t                out;
    ngx_connection_t          *c;
    ngx_http_request_body_t   *rb;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;
    rb = r->request_body;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http read client request body");
    // 循环读取请求体
    for ( ;; ) {
        for ( ;; ) {
            // 如果buf已经使用完
            if (rb->buf->last == rb->buf->end) {

                /* update chains */
                // 更新空闲链表
                rc = ngx_http_request_body_filter(r, NULL);

                if (rc != NGX_OK) {
                    return rc;
                }
                // 更新完成后，正常busy应该是空的，在request_body_no_buffering模式下不为空
                if (rb->busy != NULL) {
                    /*
                    request_body_no_buffering不缓存请求体情况下，
                    对于每次读取到的数据，应该由调用方决定如何使用，此时应该直接返回读取到的数据
                    非request_body_no_buffering下，会缓存请求体，最终使用post_handler处理
                    */
                    if (r->request_body_no_buffering) {
                        // 如果设置了读取事件，则删除
                        if (c->read->timer_set) {
                            ngx_del_timer(c->read);
                        }
                        
                        // 确保读事件在epoll中
                        if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
                            return NGX_HTTP_INTERNAL_SERVER_ERROR;
                        }

                        return NGX_AGAIN;
                    }
                    // 非request_body_no_buffering模式下，busy为空则是错误
                    return NGX_HTTP_INTERNAL_SERVER_ERROR;
                }
                // 重新使用buf空间（对应缓存请求体模式时，实际数据已经写入了文件中）
                rb->buf->pos = rb->buf->start;
                rb->buf->last = rb->buf->start;
            }

            // 获取需要读取数据大小
            size = rb->buf->end - rb->buf->last;
            rest = rb->rest - (rb->buf->last - rb->buf->pos);

            if ((off_t) size > rest) {
                size = (size_t) rest;
            }
     
            // 接收数据，并设置c->read->ready值，参考ngx_unix_recv方法
            n = c->recv(c, rb->buf->last, size);

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                           "http client request body recv %z", n);
            // 返回again
            if (n == NGX_AGAIN) {
                break;
            }

            if (n == 0) {
                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                              "client prematurely closed connection");
            }
            // 返回错误
            if (n == 0 || n == NGX_ERROR) {
                c->error = 1;
                return NGX_HTTP_BAD_REQUEST;
            }
            
            // 设置buf使用到的位置
            rb->buf->last += n;
            // 增加请求体长度
            r->request_length += n;

            /* pass buffer to request body filter chain */
            // 执行请求体过滤方法
            out.buf = rb->buf;
            out.next = NULL;

            rc = ngx_http_request_body_filter(r, &out);
            // 不是ok，直接返回
            if (rc != NGX_OK) {
                return rc;
            }
            // 读取请求体完成
            if (rb->rest == 0) {
                break;
            }

            if (rb->buf->last < rb->buf->end) {
                break;
            }
        }

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                       "http client request body rest %O", rb->rest);
        // 获取到了完整请求体
        if (rb->rest == 0) {
            break;
        }
        // 如果读取未准备好，则返回
        if (!c->read->ready) {
            // 添加读取请求体超时时间
            // 超时仅针对两次连续读取操作之间的一段时间设置，而不是针对整个请求正文的传输。
            clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
            ngx_add_timer(c->read, clcf->client_body_timeout);
            // 确保读事件在epoll中
            if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
                return NGX_HTTP_INTERNAL_SERVER_ERROR;
            }
            // 返回again，表示未完成
            return NGX_AGAIN;
        }
    }
    // 如果请求中存在部分pipelined请求，则执行对应的存储buf的拷贝，这里不详细介绍，结束请求部分会详细介绍pipelined请求
    if (ngx_http_copy_pipelined_header(r, rb->buf) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    
    // 如果依然存在读事件在红黑树中，则删除
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }
    // 非请求体无缓存模式，即请求体缓存模式下，执行post_handler方法，非缓存模式下应当有调用方决定执行逻辑
    if (!r->request_body_no_buffering) {
        r->read_event_handler = ngx_http_block_reading;
        rb->post_handler(r);
    }

    return NGX_OK;
}
```



#### ngx_http_read_client_request_body_handler

`ngx_http_read_client_request_body_handler`方法是无法一次就完整读取请求体时设置的请求读事件方法，其逻辑如下：

```c
static void
ngx_http_read_client_request_body_handler(ngx_http_request_t *r)
{
    ngx_int_t  rc;
    // 如果读事件超时，使用408返回码结束请求
    if (r->connection->read->timedout) {
        r->connection->timedout = 1;
        ngx_http_finalize_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }
    // 执行读取函数
    rc = ngx_http_do_read_client_request_body(r);
    // 如果返回错误 >= 300,则直接结束请求
    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        ngx_http_finalize_request(r, rc);
    }
}
```



### ngx_http_discard_request_body丢弃请求体

除了需要接收请求体外，可以选择直接丢弃请求体，丢弃不是意味着不接受，而是接收后不做任何处理，直接丢弃。如果不接收请求体，将导致和客户端的连接异常。`ngx_http_discard_request_body`方法用来接收请求体，其逻辑相比于接收请求体的`ngx_http_read_client_request_body`方法更简单一些：

```c
ngx_int_t
ngx_http_discard_request_body(ngx_http_request_t *r)
{
    ssize_t       size;
    ngx_int_t     rc;
    ngx_event_t  *rev;
    // 如果不是主请求，或者已经完成了请求体丢弃，或者已经设置了接收请求体，则直接返回
    if (r != r->main || r->discard_body || r->request_body) {
        return NGX_OK;
    }

#if (NGX_HTTP_V2)
    if (r->stream) {
        r->stream->skip_data = 1;
        return NGX_OK;
    }
#endif
    // 和接收请求体时一样，判断expect
    if (ngx_http_test_expect(r) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    rev = r->connection->read;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0, "http set discard body");
    // 如果设置了读事件超时，则删除
    if (rev->timer_set) {
        ngx_del_timer(rev);
    }
    // 请求体长度小于0，并且是非chunked请求，则立即返回
    if (r->headers_in.content_length_n <= 0 && !r->headers_in.chunked) {
        return NGX_OK;
    }

    size = r->header_in->last - r->header_in->pos;
    /*
    如果请求头中存在部分请求体，或者是chunked请求，执行丢弃请求体过滤函数，
    该函数主要用于判断是否结束和是否出错
    */
    if (size || r->headers_in.chunked) {
        rc = ngx_http_discard_request_body_filter(r, r->header_in);
        // 出错直接返回
        if (rc != NGX_OK) {
            return rc;
        }
        
        if (r->headers_in.content_length_n == 0) {
            return NGX_OK;
        }
    }
    // 读取请求体，直接丢弃（不做任何处理）
    rc = ngx_http_read_discarded_request_body(r);
    // 成功，直接返回，并不用延迟关闭
    if (rc == NGX_OK) {
        r->lingering_close = 0;
        return NGX_OK;
    }
    // 出错，立即返回
    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        return rc;
    }

    /* rc == NGX_AGAIN */
    // 如果未读取完全，则设置请求的读取事件，该事件重复执行ngx_http_read_discarded_request_body方法
    r->read_event_handler = ngx_http_discarded_request_body_handler;
    // 确保读事件在epoll中
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    // count自增1，避免该请求下的其余请求关闭该请求
    r->count++;
    // 设置正在执行丢弃请求体
    r->discard_body = 1;

    return NGX_OK;
}
```

这里整体执行逻辑与接收请求体类似，例如`ngx_http_read_discarded_request_body`对应接收请求体的`ngx_http_do_read_client_request_body`，`ngx_http_discarded_request_body_handler`对应`ngx_http_read_client_request_body_handler`方法，这里不做详细介绍。

## 结束请求ngx_http_finalize_request

前文很多地方使用了该方法，作为响应结束，这里详细讲解该函数执行逻辑：

```c
void
ngx_http_finalize_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_connection_t          *c;
    ngx_http_request_t        *pr;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;

    ngx_log_debug5(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http finalize request: %i, \"%V?%V\" a:%d, c:%d",
                   rc, &r->uri, &r->args, r == c->data, r->main->count);
    // 如果返回完成，则调用ngx_http_finalize_connection结束请求
    if (rc == NGX_DONE) {
        ngx_http_finalize_connection(r);
        return;
    }
    
    // 如果返回ok，但过滤链表执行出错了，则记录连接error
    if (rc == NGX_OK && r->filter_finalize) {
        c->error = 1;
    }
    
    /* 
    如果响应为NGX_DECLINED，设置content_handler为空，，然后继续执行后续的处理阶段
    这里可以在NGX_HTTP_CONTENT_PHASE阶段模块实现的函数返回后，设置继续处理后续阶段
    */
    if (rc == NGX_DECLINED) {
        r->content_handler = NULL;
        r->write_event_handler = ngx_http_core_run_phases;
        ngx_http_core_run_phases(r);
        return;
    }
    
    // 前不是主请求，并且请求存在子请求，则执行子请求的处理函数
    if (r != r->main && r->post_subrequest) {
        rc = r->post_subrequest->handler(r, r->post_subrequest->data, rc);
    }
    
    // 如果前一个返回为error，或者，返回为408（超时）|499（客户端已关闭）|出错
    if (rc == NGX_ERROR
        || rc == NGX_HTTP_REQUEST_TIME_OUT
        || rc == NGX_HTTP_CLIENT_CLOSED_REQUEST
        || c->error)
    {
        // 执行请求的post action,如果成功，直接返回
        if (ngx_http_post_action(r) == NGX_OK) {
            return;
        }
        // 结束请求
        ngx_http_terminate_request(r, rc);
        return;
    }

    // 
    if (rc >= NGX_HTTP_SPECIAL_RESPONSE // 300 
        || rc == NGX_HTTP_CREATED // 201 成功请求并创建了资源，通常为post请求。
        || rc == NGX_HTTP_NO_CONTENT) // 204 无内容
    {
        // rc为444，无响应，设置连接超时，结束请求
        if (rc == NGX_HTTP_CLOSE) {
            c->timedout = 1;
            ngx_http_terminate_request(r, rc);
            return;
        }
        
        // 如果连接为主请求，从事件红黑树中删除读写事件
        if (r == r->main) {
            if (c->read->timer_set) {
                ngx_del_timer(c->read);
            }

            if (c->write->timer_set) {
                ngx_del_timer(c->write);
            }
        }
        
        // 设置连接的读写事件为ngx_http_request_handler，仅在epoll事件驱动中触发事件时执行
        c->read->handler = ngx_http_request_handler;
        c->write->handler = ngx_http_request_handler;
        // 执行ngx_http_special_response_handler并结束请求
        ngx_http_finalize_request(r, ngx_http_special_response_handler(r, rc));
        return;
    }
    // 如果请求不是主请求
    if (r != r->main) {
        // 如果请求中还有数据未发送，参考ngx_http_writer_filter函数，或者还有子请求
        if (r->buffered || r->postponed) {
            // 设置写事件为ngx_http_writer
            if (ngx_http_set_write_handler(r) != NGX_OK) {
                ngx_http_terminate_request(r, 0);
            }

            return;
        }
        // 获取请求的父请求
        pr = r->parent;
        
        /* 
        该子请求已经处理完毕，如果它拥有发送数据的权利，则将权利移交给父请求
        */
        if (r == c->data || r->background) {
            // 记录log
            if (!r->logged) {

                clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

                if (clcf->log_subrequest) {
                    ngx_http_log_request(r);
                }

                r->logged = 1;

            } else {
                ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                              "subrequest: \"%V?%V\" logged again",
                              &r->uri, &r->args);
            }
            // 设置请求完成
            r->done = 1;

            if (r->background) {
                ngx_http_finalize_connection(r);
                return;
            }
            // 主请求数量减1
            r->main->count--;
            /* 如果该子请求不是提前完成，则从父请求的postponed链表中删除 */
            if (pr->postponed && pr->postponed->request == r) {
                pr->postponed = pr->postponed->next;
            }
            /* 将发送权利移交给父请求，父请求下次执行的时候会发送它的postponed链表中可以
               发送的数据节点，或者将发送权利移交给它的下一个子请求 */
            c->data = pr;

        } else {

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                           "http finalize non-active request: \"%V?%V\"",
                           &r->uri, &r->args);
            /* 
            到这里其实表明该子请求提前执行完成，而且它没有产生任何数据，则它下次再次获得
            执行机会时，将会执行ngx_http_request_finalzier函数，它实际上是执行
            ngx_http_finalzie_request（r,0），也就是什么都不干，直到轮到它发送数据时，
            ngx_http_finalzie_request函数会将它从父请求的postponed链表中删除 
            */
            r->write_event_handler = ngx_http_request_finalizer;

            if (r->waited) {
                r->done = 1;
            }
        }
        /* 将父请求加入posted_request队尾，获得一次运行机会 */
        if (ngx_http_post_request(pr, NULL) != NGX_OK) {
            r->main->count++;
            ngx_http_terminate_request(r, 0);
            return;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                       "http wake parent request: \"%V?%V\"",
                       &pr->uri, &pr->args);

        return;
    }
    
    /* 
    这里是处理主请求结束的逻辑，如果主请求有未发送的数据或者未处理的子请求，
    则给主请求添加写事件，并设置合适的write event hander，
    以便下次写事件来的时候继续处理 
    */
    if (r->buffered || c->buffered || r->postponed) {

        if (ngx_http_set_write_handler(r) != NGX_OK) {
            ngx_http_terminate_request(r, 0);
        }

        return;
    }

    if (r != c->data) {
        ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                      "http finalize non-active request: \"%V?%V\"",
                      &r->uri, &r->args);
        return;
    }
    // 请求完成
    r->done = 1;
    // 设置请求的读写事件均为空
    r->read_event_handler = ngx_http_block_reading;
    r->write_event_handler = ngx_http_request_empty_handler;
    // 如果没有post_action请求，则记录请求完成
    if (!r->post_action) {
        r->request_complete = 1;
    }
    // 执行post_action请求
    if (ngx_http_post_action(r) == NGX_OK) {
        return;
    }
    // 从事件红黑树中删除读写事件
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

    if (c->write->timer_set) {
        c->write->delayed = 0;
        ngx_del_timer(c->write);
    }
    // 如果客户端关闭连接，直接关闭请求
    if (c->read->eof) {
        ngx_http_close_request(r, 0);
        return;
    }
    // 否则执行结束连接
    ngx_http_finalize_connection(r);
}
```

上述内容中存在大量子请求相关信息，具体会在代理转发中详细介绍。



### ngx_http_finalize_connection

该函数为请求结束，对连接进行处理，主要包括是否需要保持连接，以及对请求体的处理：

```c
static void
ngx_http_finalize_connection(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t  *clcf;

#if (NGX_HTTP_V2)
    if (r->stream) {
        ngx_http_close_request(r, 0);
        return;
    }
#endif

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 如果主请求不为1
    if (r->main->count != 1) {
        // 如果需要丢弃请求体
        if (r->discard_body) {
            // 设置请求读事件处理函数为丢弃请求体
            r->read_event_handler = ngx_http_discarded_request_body_handler;
            // 增加到事件红黑树中，超时事件为lingering_timeout配置
            ngx_add_timer(r->connection->read, clcf->lingering_timeout);
            // 设置延迟关闭时间
            if (r->lingering_time == 0) {
                r->lingering_time = ngx_time()
                                      + (time_t) (clcf->lingering_time / 1000);
            }
        }
        // 执行关闭请求，并返回
        ngx_http_close_request(r, 0);
        return;
    }
    // 获取主请求
    r = r->main;
    /*
    如果正在读取请求体，则设置keepalive为0
    （这里设置keepalive是在请求完全结束时设置，
    如果还在读取请求头，则不应该进行该设置，因此将该值置0）
    设置需要延迟关闭
    */
    if (r->reading_body) {
        r->keepalive = 0;
        r->lingering_close = 1;
    }
    
    // 如果未收到需要关机信号，并且需要keepalive，则设置长连接
    if (!ngx_terminate
         && !ngx_exiting
         && r->keepalive
         && clcf->keepalive_timeout > 0)
    {
        ngx_http_set_keepalive(r);
        return;
    }

    // 需要延迟关闭时，设置延迟关闭，并返回
    if (clcf->lingering_close == NGX_HTTP_LINGERING_ALWAYS
        || (clcf->lingering_close == NGX_HTTP_LINGERING_ON
            && (r->lingering_close
                || r->header_in->pos < r->header_in->last
                || r->connection->read->ready)))
    {
        /*
        设置延迟关闭，先关闭套接字的写端
        设置读事件处理函数为ngx_http_lingering_close_handler
        设置写事件为空
        ngx_http_lingering_close_handler函数读取套接字，直到结束或者超过延迟关闭事件
        完成读取后调用ngx_http_close_request关闭请求
        */
        ngx_http_set_lingering_close(r);
        return;
    }
    // 否则，结束请求
    ngx_http_close_request(r, 0);
}
```



#### ngx_http_discarded_request_body_handler

该事件处理丢失请求体：

```c
void
ngx_http_discarded_request_body_handler(ngx_http_request_t *r)
{
    ngx_int_t                  rc;
    ngx_msec_t                 timer;
    ngx_event_t               *rev;
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;
    rev = c->read;
    // 如果超时，则直接结束返回
    if (rev->timedout) {
        c->timedout = 1;
        c->error = 1;
        ngx_http_finalize_request(r, NGX_ERROR);
        return;
    }
    // 如果设置了lingering_time，则比较超时时间是否达到，达到直接结束请求
    if (r->lingering_time) {
        timer = (ngx_msec_t) r->lingering_time - (ngx_msec_t) ngx_time();

        if ((ngx_msec_int_t) timer <= 0) {
            r->discard_body = 0;
            r->lingering_close = 0;
            ngx_http_finalize_request(r, NGX_ERROR);
            return;
        }

    } else {
        timer = 0;
    }
    
    // 执行读取响应体并丢弃
    rc = ngx_http_read_discarded_request_body(r);
    // 返回ok，表示完成，调用ngx_http_finalize_request结束请求
    if (rc == NGX_OK) {
        r->discard_body = 0; // 不需要再读取请求体了
        r->lingering_close = 0;
        ngx_http_finalize_request(r, NGX_DONE);
        return;
    }
    
    // 如果返回大于30x，则直接出错结束请求
    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        c->error = 1;
        ngx_http_finalize_request(r, NGX_ERROR);
        return;
    }

    /* rc == NGX_AGAIN */
    // 将读事件添加到epoll中
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        c->error = 1;
        ngx_http_finalize_request(r, NGX_ERROR);
        return;
    }
    // 如果设置了lingering_time，并且未超时，则计算新的超时时间，添加到红黑树中
    if (timer) {

        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

        timer *= 1000;

        if (timer > clcf->lingering_timeout) {
            timer = clcf->lingering_timeout;
        }

        ngx_add_timer(rev, timer);
    }
}
```



#### ngx_http_set_keepalive

如果请求是长连接，并且nginx支持长连接，且未达到长连接超时条件，则设置连接为长连接：

```c
static void
ngx_http_set_keepalive(ngx_http_request_t *r)
{
    int                        tcp_nodelay;
    ngx_buf_t                 *b, *f;
    ngx_chain_t               *cl, *ln;
    ngx_event_t               *rev, *wev;
    ngx_connection_t          *c;
    ngx_http_connection_t     *hc;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;
    rev = c->read;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "set http keepalive handler");
    // 如果正在接收响应体，并丢弃响应体
    if (r->discard_body) {
        // 设置写事件为空
        r->write_event_handler = ngx_http_request_empty_handler;
        // 设置读取响应体超时时间
        r->lingering_time = ngx_time() + (time_t) (clcf->lingering_time / 1000);
        ngx_add_timer(rev, clcf->lingering_timeout);
        return;
    }

    c->log->action = "closing request";
    // 获取请求对应的信息
    hc = r->http_connection;
    // 获取存储请求的缓冲区
    b = r->header_in;
    // 如果buf中存在未处理的数据，表示是存在另一个请求，即pipelined请求
    if (b->pos < b->last) {

        /* the pipelined request */
        /*
        如果header_in使用的buf不是连接的buffer
        （当请求头使用的buffer过大时，将分配额外的空间，这时使用的buffer就不是连接对应的buffer了）
        则使用的是http_connection中对应的buf
        */
        if (b != c->buffer) {

            /*
             * If the large header buffers were allocated while the previous
             * request processing then we do not use c->buffer for
             * the pipelined request (see ngx_http_create_request()).
             *
             * Now we would move the large header buffers to the free list.
             */
            // 回收之前未使用的buf
            for (cl = hc->busy; cl; /* void */) {
                ln = cl;
                cl = cl->next;

                if (ln->buf == b) {
                    ngx_free_chain(c->pool, ln);
                    continue;
                }

                f = ln->buf;
                f->pos = f->start;
                f->last = f->start;

                ln->next = hc->free;
                hc->free = ln;
            }
            // 创建一下新的ngx_chain_t
            cl = ngx_alloc_chain_link(c->pool);
            if (cl == NULL) {
                ngx_http_close_request(r, 0);
                return;
            }
            // 设置对应的buf
            cl->buf = b;
            cl->next = NULL;
            
            hc->busy = cl;
            hc->nbusy = 1;
        }
    }

    /* guard against recursive call from ngx_http_finalize_connection() */
    // 为防止造成ngx_http_finalize_connection()递归调用，设置keepalive为0
    r->keepalive = 0;
    // 释放请求
    ngx_http_free_request(r, 0);
    // data绑定到hc上
    c->data = hc;
    // 添加读事件处理
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        ngx_http_close_connection(c);
        return;
    }
    // 设置写事件为空方法
    wev = c->write;
    wev->handler = ngx_http_empty_handler;
    // 如果存在新的请求
    if (b->pos < b->last) {

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "pipelined request");

        c->log->action = "reading client pipelined request line";
        // 新建一个请求
        r = ngx_http_create_request(c);
        if (r == NULL) {
            ngx_http_close_connection(c);
            return;
        }
        // 设置请求为pipeline的
        r->pipeline = 1;

        c->data = r;

        c->sent = 0;
        c->destroyed = 0;
        // 如果读事件在事件红黑树中，则从中删除
        if (rev->timer_set) {
            ngx_del_timer(rev);
        }
        // 设置读事件处理函数为读取请求行，执行新的请求处理，添加到全局变量ngx_posted_events，在worker主循环中进行处理
        rev->handler = ngx_http_process_request_line;
        ngx_post_event(rev, &ngx_posted_events);
        return;
    }

    /*
     * To keep a memory footprint as small as possible for an idle keepalive
     * connection we try to free c->buffer's memory if it was allocated outside
     * the c->pool.  The large header buffers are always allocated outside the
     * c->pool and are freed too.
     */
    // 释放buffer空间，保证空闲的连接占用空间尽可能小
    b = c->buffer;

    if (ngx_pfree(c->pool, b->start) == NGX_OK) {

        /*
         * the special note for ngx_http_keepalive_handler() that
         * c->buffer's memory was freed
         */

        b->pos = NULL;

    } else {
        b->pos = b->start;
        b->last = b->start;
    }
   
    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0, "hc free: %p",
                   hc->free);
    // 释放hc空间
    if (hc->free) {
        for (cl = hc->free; cl; /* void */) {
            ln = cl;
            cl = cl->next;
            ngx_pfree(c->pool, ln->buf->start);
            ngx_free_chain(c->pool, ln);
        }

        hc->free = NULL;
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0, "hc busy: %p %i",
                   hc->busy, hc->nbusy);
    // 释放hc的busy空间
    if (hc->busy) {
        for (cl = hc->busy; cl; /* void */) {
            ln = cl;
            cl = cl->next;
            ngx_pfree(c->pool, ln->buf->start);
            ngx_free_chain(c->pool, ln);
        }

        hc->busy = NULL;
        hc->nbusy = 0;
    }

#if (NGX_HTTP_SSL)
    if (c->ssl) {
        ngx_ssl_free_buffer(c);
    }
#endif
    // 设置读事件处理函数为ngx_http_keepalive_handler
    rev->handler = ngx_http_keepalive_handler;

    if (wev->active && (ngx_event_flags & NGX_USE_LEVEL_EVENT)) {
        if (ngx_del_event(wev, NGX_WRITE_EVENT, 0) != NGX_OK) {
            ngx_http_close_connection(c);
            return;
        }
    }

    c->log->action = "keepalive";
    // 如果连接支持tcp_nopush，设置对应的连接套接字熟悉
    if (c->tcp_nopush == NGX_TCP_NOPUSH_SET) {
        if (ngx_tcp_push(c->fd) == -1) {
            ngx_connection_error(c, ngx_socket_errno, ngx_tcp_push_n " failed");
            ngx_http_close_connection(c);
            return;
        }

        c->tcp_nopush = NGX_TCP_NOPUSH_UNSET;
        tcp_nodelay = ngx_tcp_nodelay_and_tcp_nopush ? 1 : 0;

    } else {
        tcp_nodelay = 1;
    }

    if (tcp_nodelay && clcf->tcp_nodelay && ngx_tcp_nodelay(c) != NGX_OK) {
        ngx_http_close_connection(c);
        return;
    }

#if 0
    /* if ngx_http_request_t was freed then we need some other place */
    r->http_state = NGX_HTTP_KEEPALIVE_STATE;
#endif
    // 标注事件是空闲的，并且是可复用的
    c->idle = 1;
    ngx_reusable_connection(c, 1);
    // 添加对应的keepalive超时时间到红黑树中
    ngx_add_timer(rev, clcf->keepalive_timeout);
    // 如果读事件是ready，则添加到ngx_posted_events队列中，在worker循环中立即执行
    if (rev->ready) {
        ngx_post_event(rev, &ngx_posted_events);
    }
}
```



#### ngx_http_keepalive_handler处理函数

连接处于`keepalive`时，对应的读事件处理逻辑如下：

```c
static void
ngx_http_keepalive_handler(ngx_event_t *rev)
{
    size_t             size;
    ssize_t            n;
    ngx_buf_t         *b;
    ngx_connection_t  *c;

    c = rev->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "http keepalive handler");
    // 如果事件超时，或者客户端主动关闭连了，则直接关闭连接
    if (rev->timedout || c->close) {
        ngx_http_close_connection(c);
        return;
    }

#if (NGX_HAVE_KQUEUE)

    ...

#endif
    // 获取对应buf
    b = c->buffer;
    // 获取对应需要处理的数据
    size = b->end - b->start;
    // 如果pos为空，表示buf被回收了，则重新创建buf
    if (b->pos == NULL) {

        /*
         * The c->buffer's memory was freed by ngx_http_set_keepalive().
         * However, the c->buffer->start and c->buffer->end were not changed
         * to keep the buffer size.
         */

        b->pos = ngx_palloc(c->pool, size);
        if (b->pos == NULL) {
            ngx_http_close_connection(c);
            return;
        }

        b->start = b->pos;
        b->last = b->pos;
        b->end = b->pos + size;
    }

    /*
     * MSIE closes a keepalive connection with RST flag
     * so we ignore ECONNRESET here.
     */

    c->log_error = NGX_ERROR_IGNORE_ECONNRESET;
    ngx_set_socket_errno(0);
    // 接收请求
    n = c->recv(c, b->last, size);
    c->log_error = NGX_ERROR_INFO;
    // 请求返回NGX_AGAIN，则设置读事件到epoll中，并清空buf
    if (n == NGX_AGAIN) {
        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            ngx_http_close_connection(c);
            return;
        }

        /*
         * Like ngx_http_set_keepalive() we are trying to not hold
         * c->buffer's memory for a keepalive connection.
         */

        if (ngx_pfree(c->pool, b->start) == NGX_OK) {

            /*
             * the special note that c->buffer's memory was freed
             */

            b->pos = NULL;
        }

        return;
    }
    // 出错
    if (n == NGX_ERROR) {
        ngx_http_close_connection(c);
        return;
    }

    c->log->handler = NULL;

    if (n == 0) {
        ngx_log_error(NGX_LOG_INFO, c->log, ngx_socket_errno,
                      "client %V closed keepalive connection", &c->addr_text);
        ngx_http_close_connection(c);
        return;
    }
    // 成功读取到数据，设置连接为非空闲状态
    b->last += n;

    c->log->handler = ngx_http_log_error;
    c->log->action = "reading client request line";

    c->idle = 0;
    ngx_reusable_connection(c, 0);
    // 创建请求
    c->data = ngx_http_create_request(c);
    if (c->data == NULL) {
        ngx_http_close_connection(c);
        return;
    }

    c->sent = 0;
    c->destroyed = 0;
    // 将keepalive超时事件从时间红黑树中删除
    ngx_del_timer(rev);
    // 设置处理函数为处理请求行
    rev->handler = ngx_http_process_request_line;
    // 立即执行
    ngx_http_process_request_line(rev);
}
```



#### ngx_http_set_lingering_close

对应需要延迟关闭的请求，执行该方法，设置延迟关闭。延迟关闭主要是防止客户端还有下发的数据，而nginx直接关闭套接字，导致向客户端下发的数据没有成功传输完成。参考上面关于延迟关闭配置的解释。

```c
static void
ngx_http_set_lingering_close(ngx_http_request_t *r)
{
    ngx_event_t               *rev, *wev;
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    rev = c->read;
    // 设置度事件为延迟关闭处理
    rev->handler = ngx_http_lingering_close_handler;
    // 计算最晚关闭事件，根据当前时间和lingering_time配置，这里计算单位是s
    r->lingering_time = ngx_time() + (time_t) (clcf->lingering_time / 1000);
    // 将读事件添加到事件处理红黑树中，按照lingering_timeout配置
    ngx_add_timer(rev, clcf->lingering_timeout);
    // 将读事件处理添加到epoll中
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        ngx_http_close_request(r, 0);
        return;
    }
    // 设置写事件处理为空方法
    wev = c->write;
    wev->handler = ngx_http_empty_handler;

    if (wev->active && (ngx_event_flags & NGX_USE_LEVEL_EVENT)) {
        if (ngx_del_event(wev, NGX_WRITE_EVENT, 0) != NGX_OK) {
            ngx_http_close_request(r, 0);
            return;
        }
    }
    
    // 关闭套接字的写端
    if (ngx_shutdown_socket(c->fd, NGX_WRITE_SHUTDOWN) == -1) {
        ngx_connection_error(c, ngx_socket_errno,
                             ngx_shutdown_socket_n " failed");
        ngx_http_close_request(r, 0);
        return;
    }
    // 如果读事件已准备就绪则立即执行。
    if (rev->ready) {
        ngx_http_lingering_close_handler(rev);
    }
}
```



#### ngx_http_lingering_close_handler

延迟关闭时读事件的处理函数：

```c
static void
ngx_http_lingering_close_handler(ngx_event_t *rev)
{
    ssize_t                    n;
    ngx_msec_t                 timer;
    ngx_connection_t          *c;
    ngx_http_request_t        *r;
    ngx_http_core_loc_conf_t  *clcf;
    u_char                     buffer[NGX_HTTP_LINGERING_BUFFER_SIZE];

    c = rev->data;
    r = c->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http lingering close handler");
    // 如果读事件已超时，则直接关闭请求
    if (rev->timedout) {
        ngx_http_close_request(r, 0);
        return;
    }
    // 是否超过最长延迟关闭时间
    timer = (ngx_msec_t) r->lingering_time - (ngx_msec_t) ngx_time();
    // 如果超过最长延迟关闭时间，则直接关闭请求
    if ((ngx_msec_int_t) timer <= 0) {
        ngx_http_close_request(r, 0);
        return;
    }

    do {
        // 读取数据
        n = c->recv(c, buffer, NGX_HTTP_LINGERING_BUFFER_SIZE);

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0, "lingering read: %z", n);

        if (n == NGX_AGAIN) {
            break;
        }
        // 如果出错，或者读取到的数据长度为0，关闭请求
        if (n == NGX_ERROR || n == 0) {
            ngx_http_close_request(r, 0);
            return;
        }
    // 如果读事件始终处于ready状态
    } while (rev->ready);
    
    // 确保读事件在epoll中
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        ngx_http_close_request(r, 0);
        return;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 获取下一次触发延迟的时间，比较当前时间举例最大延迟时间的长度和lingering_timeout值，取较小者
    timer *= 1000;

    if (timer > clcf->lingering_timeout) {
        timer = clcf->lingering_timeout;
    }
    // 条件读事件到事件处理红黑树中
    ngx_add_timer(rev, timer);
}
```



### ngx_http_close_request

该请求关闭当前连接：

```c
static void
ngx_http_close_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_connection_t  *c;

    r = r->main;
    c = r->connection;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http request count:%d blk:%d", r->count, r->blocked);
    // 如果请求上count为0，表示已经关闭了
    if (r->count == 0) {
        ngx_log_error(NGX_LOG_ALERT, c->log, 0, "http request count is zero");
    }
    // 将请求减1
    r->count--;
    
    // 如果还存在子请求，或者还有阻塞的调用方法，则直接返回
    if (r->count || r->blocked) {
        return;
    }

#if (NGX_HTTP_V2)
    if (r->stream) {
        ngx_http_v2_close_stream(r->stream, rc);
        return;
    }
#endif
    // 释放请求
    ngx_http_free_request(r, rc);
    // 关闭连接，调用ngx_close_connect关闭连接，销毁内存池
    ngx_http_close_connection(c);
}
```



### ngx_http_free_request释放请求

```c
void
ngx_http_free_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_log_t                 *log;
    ngx_pool_t                *pool;
    struct linger              linger;
    ngx_http_cleanup_t        *cln;
    ngx_http_log_ctx_t        *ctx;
    ngx_http_core_loc_conf_t  *clcf;

    log = r->connection->log;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, log, 0, "http close request");

    if (r->pool == NULL) {
        ngx_log_error(NGX_LOG_ALERT, log, 0, "http request already closed");
        return;
    }

    cln = r->cleanup;
    r->cleanup = NULL;
    // 执行清理函数列表
    while (cln) {
        if (cln->handler) {
            cln->handler(cln->data);
        }

        cln = cln->next;
    }

#if (NGX_STAT_STUB)

    ...

#endif
    // 设置响应的返回
    if (rc > 0 && (r->headers_out.status == 0 || r->connection->sent == 0)) {
        r->headers_out.status = rc;
    }
    // 如果还未写log，则执行写log，这里写log为执行NGX_HTTP_LOG_PHASE阶段的处理函数
    if (!r->logged) {
        log->action = "logging request";

        ngx_http_log_request(r);
    }

    log->action = "closing request";
    /*
    如果是连接超时导致的关闭，并且设置了reset_timedout_connection快速关闭，则设置套接字选项
    可参考配置中快速close部分解释
    */
    if (r->connection->timedout) {
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

        if (clcf->reset_timedout_connection) {
            linger.l_onoff = 1;
            linger.l_linger = 0;

            if (setsockopt(r->connection->fd, SOL_SOCKET, SO_LINGER,
                           (const void *) &linger, sizeof(struct linger)) == -1)
            {
                ngx_log_error(NGX_LOG_ALERT, log, ngx_socket_errno,
                              "setsockopt(SO_LINGER) failed");
            }
        }
    }

    /* the various request strings were allocated from r->pool */
    ctx = log->data;
    ctx->request = NULL;

    r->request_line.len = 0;

    r->connection->destroyed = 1;

    /*
     * Setting r->pool to NULL will increase probability to catch double close
     * of request since the request object is allocated from its own pool.
     */

    pool = r->pool;
    r->pool = NULL;
    // 销毁请求的内存池
    ngx_destroy_pool(pool);
}
```



### ngx_http_terminate_request

在请求出现错误，或者客户端主动关闭时，会执行该函数立即关闭请求。

```c
struct ngx_http_posted_request_s {
    ngx_http_request_t               *request;
    ngx_http_posted_request_t        *next;
};

typedef struct {
    ngx_http_posted_request_t         terminal_posted_request;
} ngx_http_ephemeral_t;

static void
ngx_http_terminate_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_http_cleanup_t    *cln;
    ngx_http_request_t    *mr;
    ngx_http_ephemeral_t  *e;
    // 获取主请求
    mr = r->main;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http terminate request count:%d", mr->count);
    // 设置响应头
    if (rc > 0 && (mr->headers_out.status == 0 || mr->connection->sent == 0)) {
        mr->headers_out.status = rc;
    }
    
    // 执行主请求的清理函数
    cln = mr->cleanup;
    mr->cleanup = NULL;

    while (cln) {
        if (cln->handler) {
            cln->handler(cln->data);
        }

        cln = cln->next;
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http terminate cleanup count:%d blk:%d",
                   mr->count, mr->blocked);
    
    // 如果设置了写事件
    if (mr->write_event_handler) {
        // 如果存在阻塞
        if (mr->blocked) {
            /*
            设置连接发送错误，并设置写事件为ngx_http_request_finalizer
            ngx_http_request_finalizer只执行ngx_http_finalize_request方法和写日志
            */
            r->connection->error = 1;
            r->write_event_handler = ngx_http_request_finalizer;
            return;
        }
        /*
        #define ngx_http_ephemeral(r)  (void *) (&r->uri_start)
        */
        e = ngx_http_ephemeral(mr);
        // 设置posted_requests为空
        mr->posted_requests = NULL;
        /*
        设置写事件为ngx_http_terminate_handler
        ngx_http_terminate_handler方法将请求的count置1，执行ngx_http_close_request
        */
        mr->write_event_handler = ngx_http_terminate_handler;
        // 将请求自身添加到其posted_requests中，不是很理解原因
        (void) ngx_http_post_request(mr, &e->terminal_posted_request);
        return;
    }
    // 关闭请求
    ngx_http_close_request(mr, rc);
}
```



### ngx_http_set_write_handler

在结束请求时，如果还有别的阻塞数据还未传输完成，则通过该函数设置对应的写事件处理，确保数据都正常传输到客户端，执行逻辑如下：

```c
static ngx_int_t
ngx_http_set_write_handler(ngx_http_request_t *r)
{
    ngx_event_t               *wev;
    ngx_http_core_loc_conf_t  *clcf;

    r->http_state = NGX_HTTP_WRITING_REQUEST_STATE;
    // 如果还处于需要丢失请求体状态，则设置读函数为丢失请求体处理函数，否则处理函数为检查是否客户端主动关闭
    r->read_event_handler = r->discard_body ?
                                ngx_http_discarded_request_body_handler:
                                ngx_http_test_reading;
    // 设置写事件为ngx_http_writer
    r->write_event_handler = ngx_http_writer;

    wev = r->connection->write;
    
    if (wev->ready && wev->delayed) {
        return NGX_OK;
    }
    // 设置发送超时时间
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    if (!wev->delayed) {
        ngx_add_timer(wev, clcf->send_timeout);
    }
    // 确保写事件在epoll中
    if (ngx_handle_write_event(wev, clcf->send_lowat) != NGX_OK) {
        ngx_http_close_request(r, 0);
        return NGX_ERROR;
    }

    return NGX_OK;
}
```



### ngx_http_special_response_handler

对于需要返回特殊响应的请求，通过该函数做最后的响应。包括响应大于等于301，或者为201或204的响应。处理逻辑如下：

```c
ngx_int_t
ngx_http_special_response_handler(ngx_http_request_t *r, ngx_int_t error)
{
    ngx_uint_t                 i, err;
    ngx_http_err_page_t       *err_page;
    ngx_http_core_loc_conf_t  *clcf;

    ngx_log_debug3(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http special response: %i, \"%V?%V\"",
                   error, &r->uri, &r->args);

    r->err_status = error;
    // 如果响应保持长连接，判断是否允许
    if (r->keepalive) {
        switch (error) {
            case NGX_HTTP_BAD_REQUEST: // 400  异常请求
            case NGX_HTTP_REQUEST_ENTITY_TOO_LARGE: // 413 post请求体过大
            case NGX_HTTP_REQUEST_URI_TOO_LARGE: // 414 请求的uri过大
            case NGX_HTTP_TO_HTTPS: // 497 需要从http转到https
            case NGX_HTTPS_CERT_ERROR: //495 SSL证书错误
            case NGX_HTTPS_NO_CERT: // 496 无证书
            case NGX_HTTP_INTERNAL_SERVER_ERROR: // 500 服务器内部错误
            case NGX_HTTP_NOT_IMPLEMENTED: // 501 服务器无法识别的请求
                r->keepalive = 0; // 上述情况下，关闭长连接
        }
    }

    if (r->lingering_close) {
        switch (error) {
            case NGX_HTTP_BAD_REQUEST: // 400 
            case NGX_HTTP_TO_HTTPS: // 497
            case NGX_HTTPS_CERT_ERROR: // 495
            case NGX_HTTPS_NO_CERT: // 496
                r->lingering_close = 0; // 上述情况下关闭延迟关闭
        }
    }

    r->headers_out.content_type.len = 0;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    
    // 如果请求还未结果error_page处理，并且配置了error_page，并且还有rewrite的次数，执行error_page匹配
    if (!r->error_page && clcf->error_pages && r->uri_changes != 0) {
        // 如果不支持error_page进行重定向，设置已经执行过error_page
        if (clcf->recursive_error_pages == 0) {
            r->error_page = 1;
        }

        err_page = clcf->error_pages->elts;
        // 执行匹配，如果err page配置的错误码和error一致，则执行对应的处理，结束
        for (i = 0; i < clcf->error_pages->nelts; i++) {
            if (err_page[i].status == error) {
                return ngx_http_send_error_page(r, &err_page[i]);
            }
        }
    }

    r->expect_tested = 1;
    
    // 丢弃请求体，如果存在的化
    if (ngx_http_discard_request_body(r) != NGX_OK) {
        r->keepalive = 0;
    }
    /*
    启用了为微软浏览器发出刷新而不是重定向
    请求缺失为微软浏览器
    需要执行临时重定向或永久重定向
    满足以上提交时，向客户端发送刷新的响应
    */
    if (clcf->msie_refresh
        && r->headers_in.msie
        && (error == NGX_HTTP_MOVED_PERMANENTLY
            || error == NGX_HTTP_MOVED_TEMPORARILY))
    {
        return ngx_http_send_refresh(r);
    }
    
    // 执行错误码的映射
    if (error == NGX_HTTP_CREATED) {
        /* 201 */
        err = 0;

    } else if (error == NGX_HTTP_NO_CONTENT) {
        /* 204 */
        err = 0;

    } else if (error >= NGX_HTTP_MOVED_PERMANENTLY
               && error < NGX_HTTP_LAST_3XX)
    {
        /* 3XX */
        err = error - NGX_HTTP_MOVED_PERMANENTLY + NGX_HTTP_OFF_3XX;

    } else if (error >= NGX_HTTP_BAD_REQUEST
               && error < NGX_HTTP_LAST_4XX)
    {
        /* 4XX */
        err = error - NGX_HTTP_BAD_REQUEST + NGX_HTTP_OFF_4XX;

    } else if (error >= NGX_HTTP_NGINX_CODES
               && error < NGX_HTTP_LAST_5XX)
    {
        /* 49X, 5XX */
        err = error - NGX_HTTP_NGINX_CODES + NGX_HTTP_OFF_5XX;
        switch (error) {
            case NGX_HTTP_TO_HTTPS:
            case NGX_HTTPS_CERT_ERROR:
            case NGX_HTTPS_NO_CERT:
            case NGX_HTTP_REQUEST_HEADER_TOO_LARGE:
                r->err_status = NGX_HTTP_BAD_REQUEST;
        }

    } else {
        /* unknown code, zero body */
        err = 0;
    }
    // 发送特殊响应
    return ngx_http_send_special_response(r, clcf, err);
}
```



#### ngx_http_send_error_page

按照err page配置发送响应。

```c
static ngx_int_t
ngx_http_send_error_page(ngx_http_request_t *r, ngx_http_err_page_t *err_page)
{
    ngx_int_t                  overwrite;
    ngx_str_t                  uri, args;
    ngx_table_elt_t           *location;
    ngx_http_core_loc_conf_t  *clcf;

    overwrite = err_page->overwrite;

    if (overwrite && overwrite != NGX_HTTP_OK) {
        r->expect_tested = 1;
    }

    if (overwrite >= 0) {
        r->err_status = overwrite;
    }
    
    // err_page配置支持添加变量，因此需要进行变量编译
    if (ngx_http_complex_value(r, &err_page->value, &uri) != NGX_OK) {
        return NGX_ERROR;
    }
    
    // 如果生成的uri开头为/，则是执行内部重定向
    if (uri.len && uri.data[0] == '/') {

        if (err_page->value.lengths) {
            ngx_http_split_args(r, &uri, &args);

        } else {
            args = err_page->args;
        }

        if (r->method != NGX_HTTP_HEAD) {
            r->method = NGX_HTTP_GET;
            r->method_name = ngx_http_core_get_method;
        }

        return ngx_http_internal_redirect(r, &uri, &args);
    }
    
    // 如果uri开头为@,则匹配内部命名location
    if (uri.len && uri.data[0] == '@') {
        return ngx_http_named_location(r, &uri);
    }
    // 如果都不是，则应当执行重定向，并且重定向是指定的链接
    r->expect_tested = 1;
    // 丢弃请求体
    if (ngx_http_discard_request_body(r) != NGX_OK) {
        r->keepalive = 0;
    }
    // 创建location头
    location = ngx_list_push(&r->headers_out.headers);

    if (location == NULL) {
        return NGX_ERROR;
    }
    // 设置状态为临时重定向，overwrite为err page我重新的响应值
    if (overwrite != NGX_HTTP_MOVED_PERMANENTLY
        && overwrite != NGX_HTTP_MOVED_TEMPORARILY
        && overwrite != NGX_HTTP_SEE_OTHER
        && overwrite != NGX_HTTP_TEMPORARY_REDIRECT
        && overwrite != NGX_HTTP_PERMANENT_REDIRECT)
    {
        r->err_status = NGX_HTTP_MOVED_TEMPORARILY;
    }

    location->hash = 1;
    ngx_str_set(&location->key, "Location");
    location->value = uri;

    ngx_http_clear_location(r);

    r->headers_out.location = location;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 如果是微软浏览器，发送刷新响应，而不是重定向
    if (clcf->msie_refresh && r->headers_in.msie) {
        return ngx_http_send_refresh(r);
    }
    // 发送特殊响应
    return ngx_http_send_special_response(r, clcf, r->err_status
                                                   - NGX_HTTP_MOVED_PERMANENTLY
                                                   + NGX_HTTP_OFF_3XX);
}
```



#### ngx_http_internal_redirect内部重定向

该函数执行内部重定向，逻辑如下：

```c
ngx_int_t
ngx_http_internal_redirect(ngx_http_request_t *r,
    ngx_str_t *uri, ngx_str_t *args)
{
    ngx_http_core_srv_conf_t  *cscf;
    // 重定向时，将uri_changes减1
    r->uri_changes--;

    if (r->uri_changes == 0) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "rewrite or internal redirection cycle "
                      "while internally redirecting to \"%V\"", uri);

        r->main->count++;
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_DONE;
    }
    // 设置uri
    r->uri = *uri;

    if (args) {
        r->args = *args;

    } else {
        ngx_str_null(&r->args);
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "internal redirect: \"%V?%V\"", uri, &r->args);

    ngx_http_set_exten(r);

    /* clear the modules contexts */
    ngx_memzero(r->ctx, sizeof(void *) * ngx_http_max_module);
    // 将请求的loc_conf恢复成server层级的loc_conf
    cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
    r->loc_conf = cscf->ctx->loc_conf;
    // 按照新的uri，更新请求
    ngx_http_update_location_config(r);

#if (NGX_HTTP_CACHE)
    r->cache = NULL;
#endif
    // 设置请求为内部请求
    r->internal = 1;
    r->valid_unparsed_uri = 0;
    r->add_uri_to_alias = 0;
    // 将主请求count增加
    r->main->count++;
    // 执行http处理，这里会从server_rewrite_index阶段执行请求主循环
    ngx_http_handler(r);

    return NGX_DONE;
}
```



#### ngx_http_named_location内部命名location

该函数用于执行nginx内部的location重定向，即包含@符号的location配置。逻辑如下：

```c
ngx_int_t
ngx_http_named_location(ngx_http_request_t *r, ngx_str_t *name)
{
    ngx_http_core_srv_conf_t    *cscf;
    ngx_http_core_loc_conf_t   **clcfp;
    ngx_http_core_main_conf_t   *cmcf;
    
    r->main->count++;
    r->uri_changes--;

    if (r->uri_changes == 0) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "rewrite or internal redirection cycle "
                      "while redirect to named location \"%V\"", name);

        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_DONE;
    }

    if (r->uri.len == 0) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "empty URI in redirect to named location \"%V\"", name);

        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_DONE;
    }

    cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
    // 如果server层级存在named_locations，则进行匹配查找
    if (cscf->named_locations) {

        for (clcfp = cscf->named_locations; *clcfp; clcfp++) {

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "test location: \"%V\"", &(*clcfp)->name);

            if (name->len != (*clcfp)->name.len
                || ngx_strncmp(name->data, (*clcfp)->name.data, name->len) != 0)
            {
                continue;
            }

            ngx_log_debug3(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "using location: %V \"%V?%V\"",
                           name, &r->uri, &r->args);
            // 查找到后，更新请求为内部请求，设置对应的loc_conf，并更新请求
            r->internal = 1;
            r->content_handler = NULL;
            r->uri_changed = 0;
            r->loc_conf = (*clcfp)->loc_conf;

            /* clear the modules contexts */
            ngx_memzero(r->ctx, sizeof(void *) * ngx_http_max_module);

            ngx_http_update_location_config(r);

            cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
            // 设置执行阶段为location_rewrite_index
            r->phase_handler = cmcf->phase_engine.location_rewrite_index;
            // 设置请求写事件方法为执行核心请求阶段
            r->write_event_handler = ngx_http_core_run_phases;
            // 立即允许
            ngx_http_core_run_phases(r);

            return NGX_DONE;
        }
    }
    // 如果没有匹配的内部location
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                  "could not find named location \"%V\"", name);
    // 以内部错误结束请求
    ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);

    return NGX_DONE;
}
```



#### ngx_http_send_refresh

对应微软浏览器，需要将重定向设置为刷新响应。逻辑如下：

```c
static u_char ngx_http_msie_refresh_head[] =
"<html><head><meta http-equiv=\"Refresh\" content=\"0; URL=";

static u_char ngx_http_msie_refresh_tail[] =
"\"></head><body></body></html>" CRLF;

static ngx_int_t
ngx_http_send_refresh(ngx_http_request_t *r)
{
    u_char       *p, *location;
    size_t        len, size;
    uintptr_t     escape;
    ngx_int_t     rc;
    ngx_buf_t    *b;
    ngx_chain_t   out;

    len = r->headers_out.location->value.len;
    location = r->headers_out.location->value.data;

    escape = 2 * ngx_escape_uri(NULL, location, len, NGX_ESCAPE_REFRESH);
    // 获取响应体大小
    size = sizeof(ngx_http_msie_refresh_head) - 1
           + escape + len
           + sizeof(ngx_http_msie_refresh_tail) - 1;

    r->err_status = NGX_HTTP_OK;
    // 设置响应类型长度
    r->headers_out.content_type_len = sizeof("text/html") - 1;
    ngx_str_set(&r->headers_out.content_type, "text/html");
    r->headers_out.content_type_lowcase = NULL;

    r->headers_out.location->hash = 0;
    r->headers_out.location = NULL;
    // 设置响应长度
    r->headers_out.content_length_n = size;

    if (r->headers_out.content_length) {
        r->headers_out.content_length->hash = 0;
        r->headers_out.content_length = NULL;
    }
    // 清理相关的响应头
    ngx_http_clear_accept_ranges(r);
    ngx_http_clear_last_modified(r);
    ngx_http_clear_etag(r);
    // 发送响应头
    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || r->header_only) {
        return rc;
    }
    // 创建buf，存储响应体，并构建响应体
    b = ngx_create_temp_buf(r->pool, size);
    if (b == NULL) {
        return NGX_ERROR;
    }

    p = ngx_cpymem(b->pos, ngx_http_msie_refresh_head,
                   sizeof(ngx_http_msie_refresh_head) - 1);

    if (escape == 0) {
        p = ngx_cpymem(p, location, len);

    } else {
        p = (u_char *) ngx_escape_uri(p, location, len, NGX_ESCAPE_REFRESH);
    }

    b->last = ngx_cpymem(p, ngx_http_msie_refresh_tail,
                         sizeof(ngx_http_msie_refresh_tail) - 1);

    b->last_buf = (r == r->main) ? 1 : 0;
    b->last_in_chain = 1;

    out.buf = b;
    out.next = NULL;
    // 发送响应体
    return ngx_http_output_filter(r, &out);
}
```



#### ngx_http_send_special_response发送特殊响应

```c
static ngx_int_t
ngx_http_send_special_response(ngx_http_request_t *r,
    ngx_http_core_loc_conf_t *clcf, ngx_uint_t err)
{
    u_char       *tail;
    size_t        len;
    ngx_int_t     rc;
    ngx_buf_t    *b;
    ngx_uint_t    msie_padding;
    ngx_chain_t   out[3];
    /*
    server_tokens配置出错时响应体中包含的nginx是否包含版本信息
    就是我们在浏览页面，出错可能就只看到最中间有一个nginx或者nginx+版本和一写错误信息
    这可以配置是否包含版本，不同情况下返回的长度不一致
    */
    if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_ON) {
        len = sizeof(ngx_http_error_full_tail) - 1;
        tail = ngx_http_error_full_tail;

    } else if (clcf->server_tokens == NGX_HTTP_SERVER_TOKENS_BUILD) {
        len = sizeof(ngx_http_error_build_tail) - 1;
        tail = ngx_http_error_build_tail;

    } else {
        len = sizeof(ngx_http_error_tail) - 1;
        tail = ngx_http_error_tail;
    }
    
    /*
    msie_padding启用或禁用向状态大于 400 的 MSIE 客户端的响应添加注释以将响应大小增加到 512 字节。
    */
    msie_padding = 0;
    /* 
    对应特殊错误的响应，nginx都会配置默认的错误页面（如果需要的话）
    static ngx_str_t ngx_http_error_pages[] = {

    ngx_null_string,                     /* 201, 204 */

#define NGX_HTTP_LAST_2XX  202
#define NGX_HTTP_OFF_3XX   (NGX_HTTP_LAST_2XX - 201)

    /* ngx_null_string, */               /* 300 */
    ngx_string(ngx_http_error_301_page),
    ngx_string(ngx_http_error_302_page),
    ngx_string(ngx_http_error_303_page),
    ...
    }
    static char ngx_http_error_301_page[] =
"<html>" CRLF
"<head><title>301 Moved Permanently</title></head>" CRLF
"<body>" CRLF
"<center><h1>301 Moved Permanently</h1></center>" CRLF
;
    */
    if (ngx_http_error_pages[err].len) {
        // 获取响应头长度
        r->headers_out.content_length_n = ngx_http_error_pages[err].len + len;
        if (clcf->msie_padding
            && (r->headers_in.msie || r->headers_in.chrome)
            && r->http_version >= NGX_HTTP_VERSION_10
            && err >= NGX_HTTP_OFF_4XX)
        {   
            // 需要进行填充是，增加长度
            r->headers_out.content_length_n +=
                                         sizeof(ngx_http_msie_padding) - 1;
            msie_padding = 1;
        }

        r->headers_out.content_type_len = sizeof("text/html") - 1;
        ngx_str_set(&r->headers_out.content_type, "text/html");
        r->headers_out.content_type_lowcase = NULL;

    } else {
        // 如果没有响应体，则对应长度为0
        r->headers_out.content_length_n = 0;
    }
    
    // 置空content_length，避免重复发送
    if (r->headers_out.content_length) {
        r->headers_out.content_length->hash = 0;
        r->headers_out.content_length = NULL;
    }
    
    // 清空部分响应头
    ngx_http_clear_accept_ranges(r);
    ngx_http_clear_last_modified(r);
    ngx_http_clear_etag(r);
    // 发送响应头
    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || r->header_only) {
        return rc;
    }
    
    // 如果响应体为空，则下发一个空的buf，到ngx_http_output_filter方法，告知发送数据已处理完毕
    if (ngx_http_error_pages[err].len == 0) {
        return ngx_http_send_special(r, NGX_HTTP_LAST);
    }
    /*
    构建响应体
    buf0存储error_pages信息
    buf1存储nginx版本信息，即tail
    buf2存储扩展数据的mapping信息（如果有的化）
    */
    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }

    b->memory = 1;
    b->pos = ngx_http_error_pages[err].data;
    b->last = ngx_http_error_pages[err].data + ngx_http_error_pages[err].len;

    out[0].buf = b;
    out[0].next = &out[1];

    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }

    b->memory = 1;

    b->pos = tail;
    b->last = tail + len;

    out[1].buf = b;
    out[1].next = NULL;

    if (msie_padding) {
        b = ngx_calloc_buf(r->pool);
        if (b == NULL) {
            return NGX_ERROR;
        }

        b->memory = 1;
        b->pos = ngx_http_msie_padding;
        b->last = ngx_http_msie_padding + sizeof(ngx_http_msie_padding) - 1;

        out[1].next = &out[2];
        out[2].buf = b;
        out[2].next = NULL;
    }

    if (r == r->main) {
        b->last_buf = 1;
    }

    b->last_in_chain = 1;
    // 执行响应体下发
    return ngx_http_output_filter(r, &out[0]);
}
```



### ngx_http_post_action

post_action配置一般用于统计等信息，即作用是在请求完成之后执行另一个请求。例如配置如下：

```c
server {
    server_name _;

    location / {
        root /sites/default;
        post_action @add
    }
    
    location @add {
    		proxy_pass http://backend/add
    }
}
```

这时，在请求完成`/`后，会立即执行`@add`的请求。其实现逻辑为：

```c
static ngx_int_t
ngx_http_post_action(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t  *clcf;
    
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    // 如果配置了post_action
    if (clcf->post_action.data == NULL) {
        return NGX_DECLINED;
    }
    
    // 如果请求已经执行过了post_action，并且达到了uri_change最大限制次数，则直接通过
    if (r->post_action && r->uri_changes == 0) {
        return NGX_DECLINED;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "post action: \"%V\"", &clcf->post_action);
    
    r->main->count--;

    r->http_version = NGX_HTTP_VERSION_9;
    r->header_only = 1;
    // 记录post_action已经执行
    r->post_action = 1;
    // 设置请求的读事件为空函数
    r->read_event_handler = ngx_http_block_reading;
    // 按照post_action配置选择执行的location，下面两个方法前文都有介绍
    if (clcf->post_action.data[0] == '/') {
        ngx_http_internal_redirect(r, &clcf->post_action, NULL);

    } else {
        ngx_http_named_location(r, &clcf->post_action);
    }

    return NGX_OK;
}
```



# 解析配置

## 初始化核心模块

执行核心模块的ctx->create_conf方法：

```c
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->create_conf) {
            rv = module->create_conf(cycle);
            if (rv == NULL) {
                ngx_destroy_pool(pool);
                return NULL;
            }
            cycle->conf_ctx[cycle->modules[i]->index] = rv;
        }
    }
```

其中，http核心模块的ctx为：

```
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

该步骤主要是分配配置存储空间，并为核心模块关注的全局配置赋初值。需要执行的有

ngx_core_module：其返回为

```
typedef struct {
    ngx_flag_t                daemon;
    ngx_flag_t                master;

    ngx_msec_t                timer_resolution;
    ngx_msec_t                shutdown_timeout;

    ngx_int_t                 worker_processes;
    ngx_int_t                 debug_points;

    ngx_int_t                 rlimit_nofile;
    off_t                     rlimit_core;

    int                       priority;

    ngx_uint_t                cpu_affinity_auto;
    ngx_uint_t                cpu_affinity_n;
    ngx_cpuset_t             *cpu_affinity;

    char                     *username;
    ngx_uid_t                 user;
    ngx_gid_t                 group;

    ngx_str_t                 working_directory;
    ngx_str_t                 lock_file;

    ngx_str_t                 pid;
    ngx_str_t                 oldpid;

    ngx_array_t               env;
    char                    **environment;

    ngx_uint_t                transparent;  /* unsigned  transparent:1; */
} ngx_core_conf_t;
```

ngx_regex_module_ctx，其返回为：

```
typedef struct {
    ngx_flag_t  pcre_jit;
} ngx_regex_conf_t;
```



## 配置及参数解析

```c
    ngx_memzero(&conf, sizeof(ngx_conf_t));
    /* STUB: init array ? */
    conf.args = ngx_array_create(pool, 10, sizeof(ngx_str_t));
    if (conf.args == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    conf.temp_pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    if (conf.temp_pool == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }


    conf.ctx = cycle->conf_ctx;
    conf.cycle = cycle;
    conf.pool = pool;
    conf.log = log;
    conf.module_type = NGX_CORE_MODULE;
    conf.cmd_type = NGX_MAIN_CONF;

#if 0
    log->log_level = NGX_LOG_DEBUG_ALL;
#endif
    // 解析启动参数 -g
    if (ngx_conf_param(&conf) != NGX_CONF_OK) {
        environ = senv;
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }
    // 解析配置文件
    if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
        environ = senv;
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }
```

### 相关结构

#### ngx_conf_t结构

```c
ngx_conf_s {
    char                 *name;
    // 词法解析后，解析出的值
    ngx_array_t          *args;

    ngx_cycle_t          *cycle;
    ngx_pool_t           *pool;
    ngx_pool_t           *temp_pool;
    ngx_conf_file_t      *conf_file;
    ngx_log_t            *log;

    void                 *ctx;
    ngx_uint_t            module_type;
    ngx_uint_t            cmd_type;

    ngx_conf_handler_pt   handler;
    void                 *handler_conf;
};

typedef char *(*ngx_conf_handler_pt)(ngx_conf_t *cf,
    ngx_command_t *dummy, void *conf);
```

ngx_conf_param函数如下：

```c
char *
ngx_conf_param(ngx_conf_t *cf)
{
    char             *rv;
    ngx_str_t        *param;
    ngx_buf_t         b;
    ngx_conf_file_t   conf_file;

    param = &cf->cycle->conf_param;

    if (param->len == 0) {
        return NGX_CONF_OK;
    }

    ngx_memzero(&conf_file, sizeof(ngx_conf_file_t));

    ngx_memzero(&b, sizeof(ngx_buf_t));

    b.start = param->data;
    b.pos = param->data;
    b.last = param->data + param->len;
    b.end = b.last;
    b.temporary = 1;

    conf_file.file.fd = NGX_INVALID_FILE;
    conf_file.file.name.data = NULL;
    conf_file.line = 0;

    cf->conf_file = &conf_file;
    cf->conf_file->buffer = &b;

    rv = ngx_conf_parse(cf, NULL);

    cf->conf_file = NULL;

    return rv;
}
```

#### ngx_conf_file_t结构

```c
typedef struct {
    ngx_file_t            file;
    ngx_buf_t            *buffer;
    ngx_buf_t            *dump;
    ngx_uint_t            line;
} ngx_conf_file_t;
```

执行的ngx_conf_parse函数如下：

```c
char *
ngx_conf_parse(ngx_conf_t *cf, ngx_str_t *filename)
{
    char             *rv;
    ngx_fd_t          fd;
    ngx_int_t         rc;
    ngx_buf_t         buf;
    ngx_conf_file_t  *prev, conf_file;
    // 解析方式：解析文件、解析块配置（如server、location）、解析参数（-g参数）
    enum {
        parse_file = 0,
        parse_block,
        parse_param
    } type;

#if (NGX_SUPPRESS_WARN)
    fd = NGX_INVALID_FILE;
    prev = NULL;
#endif

    if (filename) {

        /* open configuration file */
        // 打开配置文件
        fd = ngx_open_file(filename->data, NGX_FILE_RDONLY, NGX_FILE_OPEN, 0);

        if (fd == NGX_INVALID_FILE) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, ngx_errno,
                               ngx_open_file_n " \"%s\" failed",
                               filename->data);
            return NGX_CONF_ERROR;
        }
        // 存储原来的conf_file
        prev = cf->conf_file;

        cf->conf_file = &conf_file;
        // 获取文件信息
        if (ngx_fd_info(fd, &cf->conf_file->file.info) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_EMERG, cf->log, ngx_errno,
                          ngx_fd_info_n " \"%s\" failed", filename->data);
        }

        cf->conf_file->buffer = &buf;

        buf.start = ngx_alloc(NGX_CONF_BUFFER, cf->log);
        if (buf.start == NULL) {
            goto failed;
        }

        buf.pos = buf.start;
        buf.last = buf.start;
        buf.end = buf.last + NGX_CONF_BUFFER;
        buf.temporary = 1;

        cf->conf_file->file.fd = fd;
        cf->conf_file->file.name.len = filename->len;
        cf->conf_file->file.name.data = filename->data;
        cf->conf_file->file.offset = 0;
        cf->conf_file->file.log = cf->log;
        cf->conf_file->line = 1;

        type = parse_file;

        if (ngx_dump_config
#if (NGX_DEBUG)
            || 1
#endif
           )
        {
            // 需要dump配置时，增加配置到config_dump_rbtree和config_dump
            if (ngx_conf_add_dump(cf, filename) != NGX_OK) {
                goto failed;
            }

        } else {
            cf->conf_file->dump = NULL;
        }

    } else if (cf->conf_file->file.fd != NGX_INVALID_FILE) {

        type = parse_block;

    } else {
        type = parse_param;
    }


    for ( ;; ) {
        // 词法解析，解析配置或参数,返回解析结构状态或错误
        rc = ngx_conf_read_token(cf);

        /*
         * ngx_conf_read_token() may return
         *
         *    NGX_ERROR             there is error
         *    NGX_OK                the token terminated by ";" was found
         *    NGX_CONF_BLOCK_START  the token terminated by "{" was found
         *    NGX_CONF_BLOCK_DONE   the "}" was found
         *    NGX_CONF_FILE_DONE    the configuration file is done
         */

        if (rc == NGX_ERROR) {
            goto done;
        }
        
        // 下面三个faild表示解析类型和词法解析结果不匹配，返回错误
        if (rc == NGX_CONF_BLOCK_DONE) {

            if (type != parse_block) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "unexpected \"}\"");
                goto failed;
            }

            goto done;
        }

        if (rc == NGX_CONF_FILE_DONE) {

            if (type == parse_block) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "unexpected end of file, expecting \"}\"");
                goto failed;
            }

            goto done;
        }

        if (rc == NGX_CONF_BLOCK_START) {

            if (type == parse_param) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "block directives are not supported "
                                   "in -g option");
                goto failed;
            }
        }

        /* rc == NGX_OK || rc == NGX_CONF_BLOCK_START */
        // 如果解析的模块自定义了解析后处理方法
        if (cf->handler) {

            /*
             * the custom handler, i.e., that is used in the http's
             * "types { ... }" directive
             */

            if (rc == NGX_CONF_BLOCK_START) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "unexpected \"{\"");
                goto failed;
            }
            // 模块自定义处理函数，例如ngx_http_core_commands中typescommends中自定义的ngx_http_core_types函数。
            rv = (*cf->handler)(cf, NULL, cf->handler_conf);
            if (rv == NGX_CONF_OK) {
                continue;
            }

            if (rv == NGX_CONF_ERROR) {
                goto failed;
            }

            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "%s", rv);

            goto failed;
        }

        // 默认处理逻辑
        rc = ngx_conf_handler(cf, rc);

        if (rc == NGX_ERROR) {
            goto failed;
        }
    }

failed:

    rc = NGX_ERROR;

done:

    if (filename) {
        if (cf->conf_file->buffer->start) {
            ngx_free(cf->conf_file->buffer->start);
        }

        if (ngx_close_file(fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_ALERT, cf->log, ngx_errno,
                          ngx_close_file_n " %s failed",
                          filename->data);
            rc = NGX_ERROR;
        }

        cf->conf_file = prev;
    }

    if (rc == NGX_ERROR) {
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
```

### 词法解析（ngx_conf_read_token）

ngx_conf_read_token函数如下：

```c
static ngx_int_t
ngx_conf_read_token(ngx_conf_t *cf)
{
    u_char      *start, ch, *src, *dst;
    off_t        file_size;
    size_t       len;
    ssize_t      n, size;
    // 记录当前状态：分别表示：查找到解析块、需要一个分割符、需要解析下一个语句、注释、变量
    ngx_uint_t   found, need_space, last_space, sharp_comment, variable;
    // 转义、单引号、双引号、起始行
    ngx_uint_t   quoted, s_quoted, d_quoted, start_line;
    ngx_str_t   *word;
    ngx_buf_t   *b, *dump;

    found = 0;
    need_space = 0;
    last_space = 1;
    sharp_comment = 0;
    variable = 0;
    quoted = 0;
    s_quoted = 0;
    d_quoted = 0;

    cf->args->nelts = 0;
    b = cf->conf_file->buffer;
    dump = cf->conf_file->dump;
    start = b->pos;
    start_line = cf->conf_file->line;

    file_size = ngx_file_size(&cf->conf_file->file.info);

    for ( ;; ) {
        // 当前内存中的数据已经解析结束
        if (b->pos >= b->last) {
            // 文件已经解析完成（如果是非文件，则都为0）
            if (cf->conf_file->file.offset >= file_size) {
                // 如果解析出了值或者状态不是要查找下一个解析块（但文件已经解析完了），则报错
                if (cf->args->nelts > 0 || !last_space) {
                    // 解析的是参数-g时的报错
                    if (cf->conf_file->file.fd == NGX_INVALID_FILE) {
                        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                           "unexpected end of parameter, "
                                           "expecting \";\"");
                        return NGX_ERROR;
                    }

                    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                  "unexpected end of file, "
                                  "expecting \";\" or \"}\"");
                    return NGX_ERROR;
                }
                // 如果没有错误，则返回配置解析完成
                return NGX_CONF_FILE_DONE;
            }
            // start为执行该函数时的b->pos
            len = b->pos - start;
            // 如果解析到长度大于NGX_CONF_BUFFER时，还未完成一个语句的解析，则是参数过长，报错
            if (len == NGX_CONF_BUFFER) {
                cf->conf_file->line = start_line;
                // 当前状态为找到一个双引号
                if (d_quoted) {
                    ch = '"';
                // 当前状态为找到一个单引号
                } else if (s_quoted) {
                    ch = '\'';
                // 否则直接报错，告知出错的起始位置
                } else {
                    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                       "too long parameter \"%*s...\" started",
                                       10, start);
                    return NGX_ERROR;
                }
                // 报错，并告知可能缺少单引号或双引号
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "too long parameter, probably "
                                   "missing terminating \"%c\" character", ch);
                return NGX_ERROR;
            }
            // 如果存在已分析语句，但未完成（未return），则将分析开始到当前这部分值，拷贝到b->start上，以进行继续分析
            if (len) {
                ngx_memmove(b->start, start, len);
            }
            
            // 计算文件中未读取的大小
            size = (ssize_t) (file_size - cf->conf_file->file.offset);
            // 如果文件剩余大小大于当前buf中还剩余的空间（b->end - (b->start + len)），则减少读取数量
            if (size > b->end - (b->start + len)) {
                size = b->end - (b->start + len);
            }
            // 从文件中读取数据
            n = ngx_read_file(&cf->conf_file->file, b->start + len, size,
                              cf->conf_file->file.offset);

            if (n == NGX_ERROR) {
                return NGX_ERROR;
            }

            if (n != size) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   ngx_read_file_n " returned "
                                   "only %z bytes instead of %z",
                                   n, size);
                return NGX_ERROR;
            }
            // 继续解析的起始位置
            b->pos = b->start + len;
            b->last = b->pos + n;
            start = b->start;

            if (dump) {
                dump->last = ngx_cpymem(dump->last, b->pos, size);
            }
        }
        // 读取当前pos的字符，并将pos后移
        ch = *b->pos++;
        
        // #define LF     (u_char) '\n'
        // 如果字符为换行
        if (ch == LF) {
            cf->conf_file->line++;
            // 如果当前解析处于注释，则删除注释
            if (sharp_comment) {
                sharp_comment = 0;
            }
        }
        // 如果当前解析处于注释内，直接跳过
        if (sharp_comment) {
            continue;
        }
        // 如果当前在专业状态，则转义结束，并结束转义状态
        if (quoted) {
            quoted = 0;
            continue;
        }
        // 如果需要一个分隔符
        if (need_space) {
        // #define LF     (u_char) '\n'
        // #define CR     (u_char) '\r'
        // 如果是如下分隔符，则需要解析下一个语句，且不再需要分隔符
            if (ch == ' ' || ch == '\t' || ch == CR || ch == LF) {
                last_space = 1;
                need_space = 0;
                continue;
            }
            // 如果解析到分号，则表示当前语句解析结束，返回ok
            if (ch == ';') {
                return NGX_OK;
            }
            // 如果解析到{则表示将要解析块配置
            if (ch == '{') {
                return NGX_CONF_BLOCK_START;
            }
            // 解析到)，则需要解析下一个语句，并且不需要
            if (ch == ')') {
                last_space = 1;
                need_space = 0;
            // 否则报错
            } else {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "unexpected \"%c\"", ch);
                return NGX_ERROR;
            }
        }
        // 如果需要解析下一个语句
        if (last_space) {
            // start为当前字符的地址
            start = b->pos - 1;
            start_line = cf->conf_file->line;
    
            if (ch == ' ' || ch == '\t' || ch == CR || ch == LF) {
                continue;
            }
            // 根据ch判断
            switch (ch) {
            // 如果是分号或{，如果解析出的语句为0，则表示配置错误。如果是{则表示后续要解析语法快。
            case ';':
            case '{':
                if (cf->args->nelts == 0) {
                    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                       "unexpected \"%c\"", ch);
                    return NGX_ERROR;
                }

                if (ch == '{') {
                    return NGX_CONF_BLOCK_START;
                }

                return NGX_OK;
            // 如果是}，则判断解析出的语句是否为0，不为0则表示，应该要进行解析，但读取到结束，表示配置错误。否则返回块结束解析完成
            case '}':
                if (cf->args->nelts != 0) {
                    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                       "unexpected \"}\"");
                    return NGX_ERROR;
                }

                return NGX_CONF_BLOCK_DONE;
            // 如果是#，则表示为注释
            case '#':
                sharp_comment = 1;
                continue;
            // 如果是转义，则记录转义状态，并进行当前语句解析
            case '\\':
                quoted = 1;
                last_space = 0;
                continue;
            // 如果是双引号，则起始位置应该不包括引号本身，并标识需要下一个双引号，并进行当前语句的解析
            case '"':
                start++;
                d_quoted = 1;
                last_space = 0;
                continue;
            // 如果是单引号，则起始位置应该不包括引号本身，并标识需要下一个单引号，并进行当前语句的解析
            case '\'':
                start++;
                s_quoted = 1;
                last_space = 0;
                continue;
            // 如果是$,则表示要解析变量，并进行当前语句的解析
            case '$':
                variable = 1;
                last_space = 0;
                continue;
            // 默认进入当前语句的解析
            default:
                last_space = 0;
            }
        } else {
        // 如果正处于解析当前语句时
            // 如果要解析变量，并且ch为{,即${}情况，则直接continue
            if (ch == '{' && variable) {
                continue;
            }
            // variable标识置为0
            variable = 0;

            if (ch == '\\') {
                quoted = 1;
                continue;
            }

            if (ch == '$') {
                variable = 1;
                continue;
            }
            
            // 如果需要双引号，并且ch是双引号，则不再需要双引号，并需要分隔符，并找到一个词
            if (d_quoted) {
                if (ch == '"') {
                    d_quoted = 0;
                    need_space = 1;
                    found = 1;
                }

            // 如果需要单引号，并且ch是单引号，则不再需要单引号，并需要分隔符，并找到一个词
            } else if (s_quoted) {
                if (ch == '\'') {
                    s_quoted = 0;
                    need_space = 1;
                    found = 1;
                }
            // 如果是下列字符，则表示需要解析下一个词，并且标识当前查找到一个词
            } else if (ch == ' ' || ch == '\t' || ch == CR || ch == LF
                       || ch == ';' || ch == '{')
            {
                last_space = 1;
                found = 1;
            }

            // 查找到一个词，增加到args中
            if (found) {
                //向args添加，并分配空间
                word = ngx_array_push(cf->args);
                if (word == NULL) {
                    return NGX_ERROR;
                }

                word->data = ngx_pnalloc(cf->pool, b->pos - 1 - start + 1);
                if (word->data == NULL) {
                    return NGX_ERROR;
                }

                for (dst = word->data, src = start, len = 0;
                     src < b->pos - 1;
                     len++)
                {
                    if (*src == '\\') {
                        switch (src[1]) {
                        case '"':
                        case '\'':
                        case '\\':
                            src++;
                            break;

                        case 't':
                            *dst++ = '\t';
                            src += 2;
                            continue;

                        case 'r':
                            *dst++ = '\r';
                            src += 2;
                            continue;

                        case 'n':
                            *dst++ = '\n';
                            src += 2;
                            continue;
                        }

                    }
                    *dst++ = *src++;
                }
                *dst = '\0';
                word->len = len;

                if (ch == ';') {
                    return NGX_OK;
                }

                if (ch == '{') {
                    return NGX_CONF_BLOCK_START;
                }

                found = 0;
            }
        }
    }
}
```

### 默认处理（ngx_conf_handler）

ngx_conf_handler函数如下：

```c
static ngx_int_t
ngx_conf_handler(ngx_conf_t *cf, ngx_int_t last)
{
    char           *rv;
    void           *conf, **confp;
    ngx_uint_t      i, found;
    ngx_str_t      *name;
    ngx_command_t  *cmd;
    
    // 获取词法解析后的第一个词
    name = cf->args->elts;

    found = 0;
    // 遍历所有modules
    for (i = 0; cf->cycle->modules[i]; i++) {
        // 取modules下的commands
        cmd = cf->cycle->modules[i]->commands;
        if (cmd == NULL) {
            continue;
        }
        // 遍历commands列表
        for ( /* void */ ; cmd->name.len; cmd++) {
            // 判断该commands是否是该配置对应的模块，通过比较长度和值
            if (name->len != cmd->name.len) {
                continue;
            }

            if (ngx_strcmp(name->data, cmd->name.data) != 0) {
                continue;
            }
            
            // 找到对于的commends的标识
            found = 1;

            // 如果该modules类型不是核心模块，或者该模块不是当前解析所关注的模块时，跳过该模块
            if (cf->cycle->modules[i]->type != NGX_CONF_MODULE
                && cf->cycle->modules[i]->type != cf->module_type)
            {
                continue;
            }

            /* is the directive's location right ? */

            // 如果commands的类型和当前解析的块不一致时，跳过。对于commands的类型详见设置配置项解析方式
            if (!(cmd->type & cf->cmd_type)) {
                continue;
            }
            // 如果commends不是需要进行块解析，并且词法解析返回结果不是ok时，错误
            if (!(cmd->type & NGX_CONF_BLOCK) && last != NGX_OK) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                  "directive \"%s\" is not terminated by \";\"",
                                  name->data);
                return NGX_ERROR;
            }
            // 如果commends是需要进行块解析，但词法解析返回结果不是NGX_CONF_BLOCK_START时，错误
            if ((cmd->type & NGX_CONF_BLOCK) && last != NGX_CONF_BLOCK_START) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "directive \"%s\" has no opening \"{\"",
                                   name->data);
                return NGX_ERROR;
            }

            /* is the directive's argument count right ? */
            // 根据commends所需配置项参数数量和词法解析出的数量，进行正确性校验
            if (!(cmd->type & NGX_CONF_ANY)) {

                if (cmd->type & NGX_CONF_FLAG) {

                    if (cf->args->nelts != 2) {
                        goto invalid;
                    }

                } else if (cmd->type & NGX_CONF_1MORE) {

                    if (cf->args->nelts < 2) {
                        goto invalid;
                    }

                } else if (cmd->type & NGX_CONF_2MORE) {

                    if (cf->args->nelts < 3) {
                        goto invalid;
                    }

                } else if (cf->args->nelts > NGX_CONF_MAX_ARGS) {

                    goto invalid;

                } else if (!(cmd->type & argument_number[cf->args->nelts - 1]))
                {
                    goto invalid;
                }
            }

            /* set up the directive's configuration context */

            conf = NULL;
            // commends需要解析的内容为全局配置（{}外）
            if (cmd->type & NGX_DIRECT_CONF) {
                conf = ((void **) cf->ctx)[cf->cycle->modules[i]->index];
            // commends需要解析的内容为NGX_MAIN_CONF，这只在核心模块进行处理，核心模块管理其下的所有模块，如ngx_http_module模块，管理所有的http模块。对于cmd->type解释详见下一节
            } else if (cmd->type & NGX_MAIN_CONF) {
                conf = &(((void **) cf->ctx)[cf->cycle->modules[i]->index]);
            // 对于非上述两种情况，此时cf->ctx与cycle的ctx_conf不是一个纬度了，而是由NGX_MAIN_CONF模块创建的配置数字，cmd->conf是改模块在其核心模块创建的配置中的偏移。如果http模块创建的ngx_http_conf_ctx_t，其中包含main_conf、srv_conf、loc_conf，此时的cmd->conf表示是这三个中的哪一个。
            } else if (cf->ctx) {
                confp = *(void **) ((char *) cf->ctx + cmd->conf);

                if (confp) {
                    conf = confp[cf->cycle->modules[i]->ctx_index];
                }
            }
            // 调用模块的set函数
            rv = cmd->set(cf, cmd, conf);

            if (rv == NGX_CONF_OK) {
                return NGX_OK;
            }

            if (rv == NGX_CONF_ERROR) {
                return NGX_ERROR;
            }

            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "\"%s\" directive %s", name->data, rv);

            return NGX_ERROR;
        }
    }

    if (found) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "\"%s\" directive is not allowed here", name->data);

        return NGX_ERROR;
    }

    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                       "unknown directive \"%s\"", name->data);

    return NGX_ERROR;

invalid:

    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                       "invalid number of arguments in \"%s\" directive",
                       name->data);

    return NGX_ERROR;
}
```

从上面代码，可以看出来，cycle中的ctx_conf中，并不是每个模块都有一个对应的`ctx_conf[modules[i]->index]`。一般都是核心模块会存在一个对应的配置。核心模块下的`ctx_conf[modules[i]->index]`配置，会分配好其管理的每个子模块的存储空间。查找子模块时，通过其所属的核心模块的配置地址，再通过模块的ctx_index查找到对应配置。（具体查找方式较为复杂，详见http模块解析）。

### 设置配置项解析方式（ngx_commend_S）

下面介绍读取配置时是如何使用ngx_commend_s的：

```c
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

#### ngx_str_t  name

配置名称，和解析出的名称做比较，确定是否是该模块关注的配置。

#### ngx_uint_t type

type决定配置项可以在哪些块出现（如http、server、location、if、upstream），以及可以携带参数类型和个数。下表列出了可以的取值。可同时取多个值，通过$ \vert $连接。

<table>
  <tr>
    <td>type类型</td>
    <td>type取值</td>
    <td>含义</td>
  </tr>
  <tr>
    <td rowspan="2">处理配置项时获取当前配置块的方式</td>
    <td>NGX_DIRECT_CONF</td>
    <td>一般由NGX_CORE_MODULE类型的核心模块使用，仅与下面的NGX_MAIN_CONF同时设置，表示模块需要解析不属于任何{}内的全局配置项。它实际上会指定set方法里的第三个参数conf的值，使之指向每个模块解析全局配置项的配置结构体</td>
  </tr>
  <tr>
    <td>NGX_ANY_CONF</td>
    <td>当前未使用</td>
  </tr>
  <tr>
    <td rowspan="12">配置项可以在哪些{}配置块中出现</td>
    <td>NGX_MAIN_CONF</td>
    <td>配置项可以出现在全局配置中，即不属于任何{}配置块</td>
  </tr>
  <tr>
    <td>NGX_EVENT_CONF</td>
    <td>配置项可以出现在events{}块内</td>
  </tr>
  <tr>
    <td>NGX_MAIL_MAIN_CONF</td>
    <td>配置项可以出现在mail{}块或imap{}内</td>
  </tr>
  <tr>
    <td>NGX_MAIL_SRV_CONF</td>
    <td>配置项可以出现在server{}块内，但server{}必须在mail{}块或imap{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_MAIN_CONF</td>
    <td>配置项可以出现在http{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_SRV_CONF</td>
    <td>配置项可以出现在server{}块内，但server{}块必须在http{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_SRV_CONF</td>
    <td>配置项可以出现在server{}块内，但server{}块必须在http{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_LOC_CONF</td>
    <td>配置项可以出现在location{}块内，但location{}块必须在http{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_UPS_CONF</td>
    <td>配置项可以出现在upstream{}块内，但upstream{}块必须在http{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_SIF_CONF</td>
    <td>配置项可以出现在server{}块内的if{}中，目前仅rewrite模块使用。if必须属于http{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_LIF_CONF</td>
    <td>配置项可以出现在loc{}块内的if{}中，目前仅rewrite模块使用。if必须属于http{}内</td>
  </tr>
  <tr>
    <td>NGX_HTTP_LMT_CONF</td>
    <td>配置项可以出现在limit_except{}内。limit_except必须属于http{}内</td>
  </tr>
  <tr>
    <td rowspan="13">配置项参数限制</td>
    <td>NGX_CONF_NOARGS</td>
    <td>配置项不携带任何参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE1</td>
    <td>配置项必须携带1个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE2</td>
    <td>配置项必须携带2个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE3</td>
    <td>配置项必须携带3个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE4</td>
    <td>配置项必须携带4个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE5</td>
    <td>配置项必须携带5个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE6</td>
    <td>配置项必须携带6个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE7</td>
    <td>配置项必须携带7个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE12</td>
    <td>配置项必须携带1~2个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE13</td>
    <td>配置项必须携带1~3个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE23</td>
    <td>配置项必须携带2~3个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE123</td>
    <td>配置项必须携带1~3个参数</td>
  </tr>
  <tr>
    <td>NGX_CONF_TAKE1234</td>
    <td>配置项必须携带1~4个参数</td>
  </tr>
  <tr>
    <td rowspan="7">限制配置项后参数出现形式</td>
    <td>NGX_CONF_ARGS_NUMBER</td>
    <td>目前未使用，无意义</td>
  </tr>
  <tr>
    <td>NGX_CONF_BLOCK</td>
    <td>配置项定义了一种新的{}块。例如http、server、location等配置，其type必须为NGX_CONF_BLOCK</td>
  </tr>
  <tr>
    <td>NGX_CONF_ANY</td>
    <td>不验证配置项携带参数个数</td>
  </tr>
  <tr>
    <td>NGX_CONF_FLAG</td>
    <td>配置项携带参数只能是1个，并且参数只能是on/off</td>
  </tr>
  <tr>
    <td>NGX_CONF_1MORE</td>
    <td>配置项携带参数必须超过1个</td>
  </tr>
  <tr>
    <td>NGX_CONF_2MORE</td>
    <td>配置项携带参数必须超过2个</td>
  </tr>
  <tr>
    <td>NGX_CONF_MULTI</td>
    <td>表示当前配置项可以出现在任意块中</td>
  </tr>
</table>

 如果HTTP模块中定义的配置项在nginx.conf配置文件中实际出现的位置和参数格式与type意义不符，那么Nginx启动报错。

每个进程都有一个唯一的ngx_cycle_t核心结构体，其成员conf_ctx维护所有模块配置结构决定配置项可以在哪些块出现决定配置项可以在哪些块出现体。其类型是void****。conf_ctx意义为首先指向一个成员皆为指针的数组，其中每个成员指针又指向另外一个成员皆为指针的数组，第二个子数组中的成员指针才会指向个模块生成的配置结构体。这是为了事件模块、http模块、mail模块而设计的。而NGX_CORE_MODULE类型的核心模块解析配置项时，配置项一定是全局的，不会从属任何{}配置块，其不需要这种双数组设计。解析标识为NGX_DIRECT_CONF类型的配置项时，会将void****转换为void**。

#### char(set)(ngx_conf_t *cf, ngx_command_t* cmd, void *conf)

`set`回调方法，使用位置在ngx_conf_handler中已展现。对于set方法，我们即可以实现一个回调方法来处理，也可以使用Nginx预设的14个解析配置项方法。预设方法如下：

| 预设方法名                  | 行为                                                         |
| --------------------------- | ------------------------------------------------------------ |
| ngx_conf_set_flag_slot      | 如果nginx.conf文件中某个配置项的参数是on或off（打开或关闭），而且在Nginx模块的代码中使用ngx_flag_t变量来保存这个配置项的参数，就可以将set回调方法设为ngx_conf_set_flag_slot。当nginx.conf文件中参数是on时，代码中的ngx_flag_t类型变量设为1，参数off时为0. |
| ngx_conf_set_str_slot       | 如果配置项后只有一个参数，同时在代码中我们希望用ngx_str_t类型变量来报错这个配置项的参数，则可以使用ngx_conf_set_str_slot方法 |
| ngx_conf_set_str_array_slot | 如果配置项会出现多次，每个配置项后面跟着1个参数，而程序中使用一个ngx_array_t动态数组来存储所有参数，且数组中的每个参数都使用ngx_str_t来存储，可以设置该方法。 |
| ngx_conf_set_keyval_slot    | 与ngx_conf_set_str_array_slot类似，也是使用ngx_array_t动态数组来存储所有参数。只是每个配置项的参数不再是1个，而必须是2个，且以配置项名 关键字 值的形式出现在nginx.conf中，同时改方法把这些配置项转化为数组，其中每个元素都存储这key/value对。 |
| ngx_conf_set_num_slot       | 配置项后必须携带1个参数，且只能是数字，存储这个参数的变量必须是整数 |
| ngx_conf_set_size_slot      | 配置项后必须携带1个参数，表示空间大小，可以是一个数字，此时表示字节数（Byte）。如果后面跟着K或k，表示kilobyte，1kb=1024B,如果后面更早m或M，就表示Megabyte，1MB=1024KB。该函数解析后，都转换成byte。 |
| ngx_conf_set_off_slot       | 配置项后必须携带1个参数，表示空间上的偏移，可以是一个数字，此时表示字节数（Byte）。如果后面跟着K或k，表示kilobyte，1kb=1024B,如果后面更早m或M，就表示Megabyte，1MB=1024KB。还可以是g或G。该函数解析后，都转换成byte。 |
| ngx_conf_set_msec_slot      | 配置项后必须携带1个参数，表示时间。无单位表示秒。m表示分钟。h表示小时。d表示天。w表示周。m表示月（30天）。y表示年。解析后转化为毫秒的单位 |
| ngx_conf_set_sec_slot       | 与ngx_conf_set_msec_slot类似，不过解析后转化为秒             |
| ngx_conf_set_bufs_slot      | 配置项后必须携带2个参数。第一个参数是数字，第二个参数表示空间大小。例如"gzip-buffers 4 8k"（通常用来表示有多少个ngx_buf_t缓冲区）。第一个不可带单位。第二个为存储大小。该配置对应Nginx最常用的多缓冲区的解决方案（如接收对端发来的TCP流） |
| ngx_conf_set_enum_slot      | 配置项后必须携带1个参数，其取值范围是我们设定好的字符串之一。 |
| ngx_conf_set_bitmask_slot   | 与ngx_conf_set_enum_slot类似。                               |
| ngx_conf_set_access_slot    | 这个方法用于设置目录或者文件的读写权限。配置项后可以携带1~3个参数，可以是如下形式：user:rw group:rw all:rw。其意义与Linux上文件或目录的权限一致，但user/group/all后面权限只可以设置rw，或者r |
| ngx_conf_set_path_slot      | 用于设置路径，配置项后面必须携带1个参数，表示一个有意义的路径。该方法会把参数转化为ngx_path_t结构 |

#### ngx_uint_t conf

conf用于指示配置项所处内存的相对偏移位置，仅在type中没有设置NGX_DIRECT_CONF和NGX_MAIN_CONF时才会生效。对于HTTP模块，conf是必须要设置的，其取值范围如下：

| conf在HTTP模块的取值      | 意义                                                         |
| ------------------------- | ------------------------------------------------------------ |
| NGX_HTTP_MAIN_CONF_OFFSET | 使用create_main_conf方法产生的结构体来存储解析出的配置项参数 |
| NGX_HTTP_SRV_CONF_OFFSET  | 使用create_srv_conf方法产生的结构体来存储解析出的配置项参数  |
| NGX_HTTP_LOC_CONF_OFFSET  | 使用create_loc_conf方法产生的结构体来存储解析出的配置项参数  |

HTTP框架可以使用预设的14种方法自动地将解析出的配置项写入HTTP模块代码定义的结构体中，但HTTP模块中可能会定义3个结 构体，分别用于存储main、srv、loc级别的配置项(对应于create_main_conf、 create_srv_conf、create_loc_conf方法创建的结构体)，而HTTP框架自动解析时需要知道应把解析出的配置项值写入哪个结构体中，这将由conf成员完成。

对conf的设置是与ngx_http_module_t实现的回调方法相关的。 如果用于存储这个配置项的数据结构是由create_main_conf回调方法完成的，那么必须把conf 设置为NGX_HTTP_MAIN_CONF_OFFSET。同样，如果这个配置项所属的数据结构是由 create_srv_conf回调方法完成的，那么必须把conf设置为NGX_HTTP_SRV_CONF_OFFSET。 可如果create_loc_conf负责生成存储这个配置项的数据结构，就得将conf设置为 NGX_HTTP_LOC_CONF_OFFSET。

目前，功能较为简单的HTTP模块都只实现了create_loc_conf回调方法，对于http{}、 server{}块内出现的同名配置项，都是并入某个location{}内create_loc_conf方法产生的结构体 中的。当我们希望同时出现在http{}、server{}、 location{}块的同名配置项，在HTTP模块的代码中保存于不同的变量中时，就需要实现 create_main_conf方法、create_srv_conf方法产生新的结构体，从而以不同的结构体独立保存 不同级别的配置项，而不是全部合并到某个location下create_loc_conf方法生成的结构体中。

#### ngx_uint_t offset

offset表示当前配置项在整个存储配置项的结构体中的偏移位置(以字节(Byte)为单位)。在使用Nginx预设的解析配置项方法时，就必须指定offset，这 样Nginx首先通过conf成员找到应该用哪个结构体来存放，然后通过offset成员找到这个结构 体中的相应成员，以便存放该配置。如果是自定义的专用配置项解析方法(只解析某一个配 置项)，则可以不设置offset的值。其设置方式主要方式为：

```c
#define offsetof(type,member)(size_t)&(((type*)0)->member)
```

#### void *post

post指针有许多用处，其使用方式是在调用set方法内调用。

如果自定义了配置项的回调方法，那么post指针的用途完全由用户来定义。如果不使用 它，那么随意设为NULL即可。如果想将一些数据结构或者方法的指针传过来，那么使用post 也可以。

如果使用Nginx预设的配置项解析方法，就需要根据这些预设方法来决定post的使用方 式。表4-4说明了post相对于14个预设方法的用途。

<table>
  <tr>
    <td>pos使用方法</td>
    <td>适用的预设配置项解析方法</td>
  </tr>
  <tr>
    <td rowspan="9">可以选择是否实现。如果设置NULL，则表示不实现，否则必须实现为指向ngx_conf_post_t结构的指针。ngx_conf_post_t中包含一个方法指针，表示在解析当前配置项完毕后，需要回调这个方法</td>
    <td>ngx_conf_set_flag_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_str_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_str_array_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_keyval_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_num_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_size_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_off_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_msec_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_sec_slot</td>
  </tr>
  <tr>
    <td>指向ngx_conf_enum_t数组，表示当前配置项的参数必须设置为ngx_conf_enum_t规定的值（类似枚举）。必须定义</td>
    <td>ngx_conf_set_enum_slot</td>
  </tr>
  <tr>
    <td>指向ngx_conf_bitmask_t数组，表示当前配置项的参数必须设置为ngx_conf_bitmask_t规定的值（类似枚举）。必须定义</td>
    <td>ngx_conf_set_enum_slot</td>
  </tr>
  <tr>
    <td rowspan="3">无任何用处</td>
    <td>ngx_conf_set_buffs_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_path_slot</td>
  </tr>
  <tr>
    <td>ngx_conf_set_access_slot</td>
  </tr>
</table>

有9个预设方法在使用post是可以设置为ngx_conf_post_t结构体来使用，其定义如下：

```c
typedef char (ngx_conf_post_handler_pt) (ngx_conf_t cf, void data, void *conf);
typedef struct {
  ngx_conf_post_handler_pt post_handler;
} ngx_conf_post_t;
```

#### 预设配置项处理方法工作原理

```c
char *
ngx_conf_set_num_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    // conf为存储参数的结构体指针。该结构体是每个模块自定义的。
    char  *p = conf;

    ngx_int_t        *np;
    ngx_str_t        *value;
    ngx_conf_post_t  *post;

    // 根据ngx_command_t中offset偏移量，可以找到结构体中成员
    np = (ngx_int_t *) (p + cmd->offset);

    if (*np != NGX_CONF_UNSET) {
        return "is duplicate";
    }
    // value将指向配置项的参数
    value = cf->args->elts;
    // 将字符串的参数转换为整数，并设置到create_loc_conf等方法生成的结构体的相关成员上
    *np = ngx_atoi(value[1].data, value[1].len);
    if (*np == NGX_ERROR) {
        return "invalid number";
    }
    // 如果ngx_command_t中post已经实现，那么还需要调用post->post_handler方法。目前Nginx官方模块未使用
    if (cmd->post) {
        post = cmd->post;
        return post->post_handler(cf, post, np);
    }

    return NGX_CONF_OK;
}
```

## http模块配置解析

[![RcHHKK.png](https://z3.ax1x.com/2021/07/02/RcHHKK.png)](https://imgtu.com/i/RcHHKK)



在默认的处理函数`ngx_conf_handler`中，会会调用核心模块的commands数组的每个元素的set方法。对于http来说，核心模块为

```c
static ngx_command_t  ngx_http_commands[] = {

    { ngx_string("http"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_http_block,
      0,
      0,
      NULL },

      ngx_null_command
};


static ngx_core_module_t  ngx_http_module_ctx = {
    ngx_string("http"),
    NULL,
    NULL
};


ngx_module_t  ngx_http_module = {
    NGX_MODULE_V1,
    &ngx_http_module_ctx,                  /* module context */
    ngx_http_commands,                     /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

### http层set函数

执行set方法即执行ngx_http_block函数：

```c
static char *
ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    char                        *rv;
    ngx_uint_t                   mi, m, s;
    ngx_conf_t                   pcf;
    ngx_http_module_t           *module;
    ngx_http_conf_ctx_t         *ctx;
    ngx_http_core_loc_conf_t    *clcf;
    ngx_http_core_srv_conf_t   **cscfp;
    ngx_http_core_main_conf_t   *cmcf;

    if (*(ngx_http_conf_ctx_t **) conf) {
        return "is duplicate";
    }

    /* the main http context */

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    *(ngx_http_conf_ctx_t **) conf = ctx;


    /* count the number of the http modules and set up their indices */
    // 获取http模块数量，并设置其ctx_index
    ngx_http_max_module = ngx_count_modules(cf->cycle, NGX_HTTP_MODULE);


    /* the http main_conf context, it is the same in the all http contexts */
    // 初始化ctx
    ctx->main_conf = ngx_pcalloc(cf->pool,
                                 sizeof(void *) * ngx_http_max_module);
    if (ctx->main_conf == NULL) {
        return NGX_CONF_ERROR;
    }


    /*
     * the http null srv_conf context, it is used to merge
     * the server{}s' srv_conf's
     */

    ctx->srv_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->srv_conf == NULL) {
        return NGX_CONF_ERROR;
    }


    /*
     * the http null loc_conf context, it is used to merge
     * the server{}s' loc_conf's
     */

    ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->loc_conf == NULL) {
        return NGX_CONF_ERROR;
    }


    /*
     * create the main_conf's, the null srv_conf's, and the null loc_conf's
     * of the all http modules
     */
    // 执行每个http模块的create_main_conf、create_srv_conf、create_loc_conf方法，其返回为每个模块在对于块下用于存储解析结果的结构体
    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;
        mi = cf->cycle->modules[m]->ctx_index;

        if (module->create_main_conf) {
            ctx->main_conf[mi] = module->create_main_conf(cf);
            if (ctx->main_conf[mi] == NULL) {
                return NGX_CONF_ERROR;
            }
        }

        if (module->create_srv_conf) {
            ctx->srv_conf[mi] = module->create_srv_conf(cf);
            if (ctx->srv_conf[mi] == NULL) {
                return NGX_CONF_ERROR;
            }
        }

        if (module->create_loc_conf) {
            ctx->loc_conf[mi] = module->create_loc_conf(cf);
            if (ctx->loc_conf[mi] == NULL) {
                return NGX_CONF_ERROR;
            }
        }
    }

    pcf = *cf;
    cf->ctx = ctx;
    // 执行每个模块的preconfiguration方法，增加变量。
    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;

        if (module->preconfiguration) {
            if (module->preconfiguration(cf) != NGX_OK) {
                return NGX_CONF_ERROR;
            }
        }
    }

    /* parse inside the http{} block */
    // 解析http块下配置
    cf->module_type = NGX_HTTP_MODULE;
    cf->cmd_type = NGX_HTTP_MAIN_CONF;
    rv = ngx_conf_parse(cf, NULL);

    if (rv != NGX_CONF_OK) {
        goto failed;
    }

    /*
     * init http{} main_conf's, merge the server{}s' srv_conf's
     * and its location{}s' loc_conf's
     */
    // 解析后合并及其他操作，下一章节介绍
    cmcf = ctx->main_conf[ngx_http_core_module.ctx_index];
    cscfp = cmcf->servers.elts;

    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;
        mi = cf->cycle->modules[m]->ctx_index;

        /* init http{} main_conf's */

        if (module->init_main_conf) {
            rv = module->init_main_conf(cf, ctx->main_conf[mi]);
            if (rv != NGX_CONF_OK) {
                goto failed;
            }
        }

        rv = ngx_http_merge_servers(cf, cmcf, module, mi);
        if (rv != NGX_CONF_OK) {
            goto failed;
        }
    }


    /* create location trees */

    for (s = 0; s < cmcf->servers.nelts; s++) {

        clcf = cscfp[s]->ctx->loc_conf[ngx_http_core_module.ctx_index];

        if (ngx_http_init_locations(cf, cscfp[s], clcf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }

        if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }


    if (ngx_http_init_phases(cf, cmcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    if (ngx_http_init_headers_in_hash(cf, cmcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }


    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;

        if (module->postconfiguration) {
            if (module->postconfiguration(cf) != NGX_OK) {
                return NGX_CONF_ERROR;
            }
        }
    }

    if (ngx_http_variables_init_vars(cf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    /*
     * http{}'s cf->ctx was needed while the configuration merging
     * and in postconfiguration process
     */

    *cf = pcf;


    if (ngx_http_init_phase_handlers(cf, cmcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }


    /* optimize the lists of ports, addresses and server names */

    if (ngx_http_optimize_servers(cf, cmcf, cmcf->ports) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;

failed:

    *cf = pcf;

    return rv;
}
```

#### ngx_http_conf_ctx_t

上面代码中ngx_http_conf_ctx_t类定义如下：

```
typedef struct {
    void        **main_conf;
    void        **srv_conf;
    void        **loc_conf;
} ngx_http_conf_ctx_t;
```

分别用来存储main、server、loc块下配置。

#### ngx_http_core_module模块生成的三个结构

在执行create_main_conf、create_srv_conf、create_loc_conf时，会生成三个结构体，其中ngx_http_core_module是http的核心模块。其生成的三个结构体如下：

##### ngx_http_core_main_conf_t

```
typedef struct {
    ngx_array_t                servers;         /* ngx_http_core_srv_conf_t */

    ngx_http_phase_engine_t    phase_engine;

    ngx_hash_t                 headers_in_hash;

    ngx_hash_t                 variables_hash;

    ngx_array_t                variables;         /* ngx_http_variable_t */
    ngx_array_t                prefix_variables;  /* ngx_http_variable_t */
    ngx_uint_t                 ncaptures;

    ngx_uint_t                 server_names_hash_max_size;
    ngx_uint_t                 server_names_hash_bucket_size;

    ngx_uint_t                 variables_hash_max_size;
    ngx_uint_t                 variables_hash_bucket_size;

    ngx_hash_keys_arrays_t    *variables_keys;

    ngx_array_t               *ports;

    ngx_http_phase_t           phases[NGX_HTTP_LOG_PHASE + 1];
} ngx_http_core_main_conf_t;
```



##### ngx_http_core_srv_conf_t

```
typedef struct {
    /* array of the ngx_http_server_name_t, "server_name" directive */
    ngx_array_t                 server_names;

    /* server ctx */
    ngx_http_conf_ctx_t        *ctx;

    u_char                     *file_name;
    ngx_uint_t                  line;

    ngx_str_t                   server_name;

    size_t                      connection_pool_size;
    size_t                      request_pool_size;
    size_t                      client_header_buffer_size;

    ngx_bufs_t                  large_client_header_buffers;

    ngx_msec_t                  client_header_timeout;

    ngx_flag_t                  ignore_invalid_headers;
    ngx_flag_t                  merge_slashes;
    ngx_flag_t                  underscores_in_headers;

    unsigned                    listen:1; // 配置中是否设置了监听
#if (NGX_PCRE)
    unsigned                    captures:1;
#endif

    ngx_http_core_loc_conf_t  **named_locations;
} ngx_http_core_srv_conf_t;
```



##### ngx_http_core_loc_conf_s

```
struct ngx_http_core_loc_conf_s {
    ngx_str_t     name;          /* location name */

#if (NGX_PCRE)
    ngx_http_regex_t  *regex;
#endif
    // nginx将if语句和limit_except也看作为ngx_http_core_loc_conf_s，这里标识其类型为非location块的
    unsigned      noname:1;   /* "if () {}" block or limit_except */ 
    unsigned      lmt_excpt:1;
    // 以’@’开头的，如location @test {}
    unsigned      named:1;
    // 类似 location = / {}，所谓准确匹配。
    unsigned      exact_match:1;
    // 没有正则，指类似location ^~ /a { ... } 的location。
    unsigned      noregex:1;

    unsigned      auto_redirect:1;
#if (NGX_HTTP_GZIP)
    unsigned      gzip_disable_msie6:2;
    unsigned      gzip_disable_degradation:2;
#endif

    ngx_http_location_tree_node_t   *static_locations;
#if (NGX_PCRE)
    ngx_http_core_loc_conf_t       **regex_locations;
#endif

    /* pointer to the modules' loc_conf */
    void        **loc_conf;

    uint32_t      limit_except;
    void        **limit_except_loc_conf;

    ngx_http_handler_pt  handler;

    /* location name length for inclusive location with inherited alias */
    size_t        alias;
    ngx_str_t     root;                    /* root, alias */
    ngx_str_t     post_action;

    ngx_array_t  *root_lengths;
    ngx_array_t  *root_values;

    ngx_array_t  *types;
    ngx_hash_t    types_hash;
    ngx_str_t     default_type;

    off_t         client_max_body_size;    /* client_max_body_size */
    off_t         directio;                /* directio */
    off_t         directio_alignment;      /* directio_alignment */

    size_t        client_body_buffer_size; /* client_body_buffer_size */
    size_t        send_lowat;              /* send_lowat */
    size_t        postpone_output;         /* postpone_output */
    size_t        sendfile_max_chunk;      /* sendfile_max_chunk */
    size_t        read_ahead;              /* read_ahead */
    size_t        subrequest_output_buffer_size;
                                           /* subrequest_output_buffer_size */

    ngx_http_complex_value_t  *limit_rate; /* limit_rate */
    ngx_http_complex_value_t  *limit_rate_after; /* limit_rate_after */

    ngx_msec_t    client_body_timeout;     /* client_body_timeout */
    ngx_msec_t    send_timeout;            /* send_timeout */
    ngx_msec_t    keepalive_timeout;       /* keepalive_timeout */
    ngx_msec_t    lingering_time;          /* lingering_time */
    ngx_msec_t    lingering_timeout;       /* lingering_timeout */
    ngx_msec_t    resolver_timeout;        /* resolver_timeout */
    ngx_msec_t    auth_delay;              /* auth_delay */

    ngx_resolver_t  *resolver;             /* resolver */

    time_t        keepalive_header;        /* keepalive_timeout */

    ngx_uint_t    keepalive_requests;      /* keepalive_requests */
    ngx_uint_t    keepalive_disable;       /* keepalive_disable */
    ngx_uint_t    satisfy;                 /* satisfy */
    ngx_uint_t    lingering_close;         /* lingering_close */
    ngx_uint_t    if_modified_since;       /* if_modified_since */
    ngx_uint_t    max_ranges;              /* max_ranges */
    ngx_uint_t    client_body_in_file_only; /* client_body_in_file_only */

    ngx_flag_t    client_body_in_single_buffer;
                                           /* client_body_in_singe_buffer */
    ngx_flag_t    internal;                /* internal */
    ngx_flag_t    sendfile;                /* sendfile */
    ngx_flag_t    aio;                     /* aio */
    ngx_flag_t    aio_write;               /* aio_write */
    ngx_flag_t    tcp_nopush;              /* tcp_nopush */
    ngx_flag_t    tcp_nodelay;             /* tcp_nodelay */
    ngx_flag_t    reset_timedout_connection; /* reset_timedout_connection */
    ngx_flag_t    absolute_redirect;       /* absolute_redirect */
    ngx_flag_t    server_name_in_redirect; /* server_name_in_redirect */
    ngx_flag_t    port_in_redirect;        /* port_in_redirect */
    ngx_flag_t    msie_padding;            /* msie_padding */
    ngx_flag_t    msie_refresh;            /* msie_refresh */
    ngx_flag_t    log_not_found;           /* log_not_found */
    ngx_flag_t    log_subrequest;          /* log_subrequest */
    ngx_flag_t    recursive_error_pages;   /* recursive_error_pages */
    ngx_uint_t    server_tokens;           /* server_tokens */
    ngx_flag_t    chunked_transfer_encoding; /* chunked_transfer_encoding */
    ngx_flag_t    etag;                    /* etag */

#if (NGX_HTTP_GZIP)
    ngx_flag_t    gzip_vary;               /* gzip_vary */

    ngx_uint_t    gzip_http_version;       /* gzip_http_version */
    ngx_uint_t    gzip_proxied;            /* gzip_proxied */

#if (NGX_PCRE)
    ngx_array_t  *gzip_disable;            /* gzip_disable */
#endif
#endif

#if (NGX_THREADS || NGX_COMPAT)
    ngx_thread_pool_t         *thread_pool;
    ngx_http_complex_value_t  *thread_pool_value;
#endif

#if (NGX_HAVE_OPENAT)
    ngx_uint_t    disable_symlinks;        /* disable_symlinks */
    ngx_http_complex_value_t  *disable_symlinks_from;
#endif

    ngx_array_t  *error_pages;             /* error_page */

    ngx_path_t   *client_body_temp_path;   /* client_body_temp_path */

    ngx_open_file_cache_t  *open_file_cache;
    time_t        open_file_cache_valid;
    ngx_uint_t    open_file_cache_min_uses;
    ngx_flag_t    open_file_cache_errors;
    ngx_flag_t    open_file_cache_events;

    ngx_log_t    *error_log;

    ngx_uint_t    types_hash_max_size;
    ngx_uint_t    types_hash_bucket_size;

    ngx_queue_t  *locations;

#if 0
    ngx_http_core_loc_conf_t  *prev_location;
#endif
};
```

在处理http块内main配置时，对每个http模块创建了三个结构体，这是为了把同时出现在http、server、location块中相同的配置进行合并而准备的。例如，有一个与server相关的配置（例如负责指定每个TCP连接池大小的connection_pool_size配置项）同时出现在http和server中，那么对其感兴趣的http模块有权决定srv结构内成员究竟是以main级别的为准还是已srv的为准。对loc级别的配置也是如此，loc级别的需要http下创建的loc和server创建的loc和loc级别本身创建的loc三者共同决定。因此main级别的配置会被创建1次（在解析http时创建），server级别的配置会被创建2次（解析http块一次和解析server块一次），loc级别配置会被创建三次（解析http块一次、解析server块一次和解析loc块一次）。具体配置在内存中结构之后会详细解释。

在创建配置后，就会继续解析配置，此时，遇到的第一个http模块就是ngx_http_core_module。从上述代码可以看出来，nginx配置项解析采用的是深度优先搜索模式，其解析的顺序是十分重要的，一定要先解析核心模块，再解析每类模块在该模块下的代理（如http模块的代理为ngx_http_core_module），之后才能再解析该类模块下其他模块。

#### http级配置解析内存结构

[![RW0I2j.png](https://z3.ax1x.com/2021/07/04/RW0I2j.png)](https://imgtu.com/i/RW0I2j)

<center/>图10-1</center>
ngx_http_core_module是第一个http模块，其ctx_index为0，因此，数组中第一个指针指向ngx_http_core_module生成的三个结构体。要由ngx_cycle_t获得main级别配置方式如下：

```
#define ngx_http_get_module_main_conf(r, module)                             \
    (r)->main_conf[module.ctx_index]
```



### server级别set函数

在http中，接着执行解析函数时，当解析到server配置时，首先会遇到ngx_http_core_module核心模块的commands中的server。如下：

```c
{ ngx_string("server"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_http_core_server,
      0,
      0,
      NULL },
```

其函数如下：

```c
static char *
ngx_http_core_server(ngx_conf_t *cf, ngx_command_t *cmd, void *dummy)
{
    char                        *rv;
    void                        *mconf;
    size_t                       len;
    u_char                      *p;
    ngx_uint_t                   i;
    ngx_conf_t                   pcf;
    ngx_http_module_t           *module;
    struct sockaddr_in          *sin;
    ngx_http_conf_ctx_t         *ctx, *http_ctx;
    ngx_http_listen_opt_t        lsopt;
    ngx_http_core_srv_conf_t    *cscf, **cscfp;
    ngx_http_core_main_conf_t   *cmcf;

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }
    // 这里的ctx为http块解析时生成的ngx_http_conf_ctx_t，具体可看http的set代码。
    http_ctx = cf->ctx;
    // 由于server块中不含main层级的配置项，因此其main_conf执行http层生成的main_conf即可
    ctx->main_conf = http_ctx->main_conf;

    /* the server{}'s srv_conf */
    // 创建server和location块的存储结构
    ctx->srv_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->srv_conf == NULL) {
        return NGX_CONF_ERROR;
    }

    /* the server{}'s loc_conf */

    ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->loc_conf == NULL) {
        return NGX_CONF_ERROR;
    }

    for (i = 0; cf->cycle->modules[i]; i++) {
        if (cf->cycle->modules[i]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[i]->ctx;

        if (module->create_srv_conf) {
            mconf = module->create_srv_conf(cf);
            if (mconf == NULL) {
                return NGX_CONF_ERROR;
            }

            ctx->srv_conf[cf->cycle->modules[i]->ctx_index] = mconf;
        }

        if (module->create_loc_conf) {
            mconf = module->create_loc_conf(cf);
            if (mconf == NULL) {
                return NGX_CONF_ERROR;
            }

            ctx->loc_conf[cf->cycle->modules[i]->ctx_index] = mconf;
        }
    }


    /* the server configuration context */
    // 使用server层ngx_http_core_module模块生成的ngx_http_core_srv_conf_t的ctx来存储server块生成的配置内存结构
    cscf = ctx->srv_conf[ngx_http_core_module.ctx_index];
    cscf->ctx = ctx;

    // 获取main层ngx_http_core_module块生成的ngx_http_core_main_conf_t
    cmcf = ctx->main_conf[ngx_http_core_module.ctx_index];
    // 将server层ngx_http_core_module模块生成的ngx_http_core_srv_conf_t添加到main层ngx_http_core_module块生成的ngx_http_core_main_conf_t的server中，使用该方式将main层的配置内存结构和server层配置内存结构串联起来。
    cscfp = ngx_array_push(&cmcf->servers);
    if (cscfp == NULL) {
        return NGX_CONF_ERROR;
    }

    *cscfp = cscf;


    /* parse inside server{} */
    // 继续解析server块。
    pcf = *cf;
    cf->ctx = ctx;
    cf->cmd_type = NGX_HTTP_SRV_CONF;

    rv = ngx_conf_parse(cf, NULL);

    *cf = pcf;
    // 如果正常解析，但并未设置监听，则使用默认监听（root 80， user 8000）。具体可以看下一部分的管理监听端口章节
    if (rv == NGX_CONF_OK && !cscf->listen) {
        ngx_memzero(&lsopt, sizeof(ngx_http_listen_opt_t));

        p = ngx_pcalloc(cf->pool, sizeof(struct sockaddr_in));
        if (p == NULL) {
            return NGX_CONF_ERROR;
        }

        lsopt.sockaddr = (struct sockaddr *) p;

        sin = (struct sockaddr_in *) p;

        sin->sin_family = AF_INET;
#if (NGX_WIN32)
        sin->sin_port = htons(80);
#else
        sin->sin_port = htons((getuid() == 0) ? 80 : 8000);
#endif
        sin->sin_addr.s_addr = INADDR_ANY;

        lsopt.socklen = sizeof(struct sockaddr_in);

        lsopt.backlog = NGX_LISTEN_BACKLOG;
        lsopt.rcvbuf = -1;
        lsopt.sndbuf = -1;
#if (NGX_HAVE_SETFIB)
        lsopt.setfib = -1;
#endif
#if (NGX_HAVE_TCP_FASTOPEN)
        lsopt.fastopen = -1;
#endif
        lsopt.wildcard = 1;

        len = NGX_INET_ADDRSTRLEN + sizeof(":65535") - 1;

        p = ngx_pnalloc(cf->pool, len);
        if (p == NULL) {
            return NGX_CONF_ERROR;
        }

        lsopt.addr_text.data = p;
        lsopt.addr_text.len = ngx_sock_ntop(lsopt.sockaddr, lsopt.socklen, p,
                                            len, 1);

        if (ngx_http_add_listen(cf, cscf, &lsopt) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }

    return rv;
}
```



#### server级配置解析内存结构

[![R5gtCn.png](https://z3.ax1x.com/2021/07/05/R5gtCn.png)](https://imgtu.com/i/R5gtCn)

<center/>图10-3</center>
解析每一个server块时都会创建一个新的 ngx_http_conf_ctx_t结构体，其中的main_conf将指向http块下main_conf指针数组，而srv_conf和 loc_conf数组则都会重新分配，它们的内容就是所有HTTP模块的create_srv_conf方法、 create_loc_conf方法创建的结构体指针。

http层的存储结构和server层存储关联方式为：将server层ngx_http_core_module模块生成的ngx_http_core_srv_conf_t添加到main层ngx_http_core_module块生成的ngx_http_core_main_conf_t的server中，其中server层ngx_http_core_module模块生成的ngx_http_core_srv_conf_t中的ctx指向server层生成的ngx_http_conf_ctx_t结构体。使用该方式将main层的配置内存结构和server层配置内存结构串联起来。

### 管理监听端口号

nginx配置，使用listen来设置监听端口：

```
listen address:port [default(deprecated in 0.8.21) | default_server | [backlog=num | rcvbuf=size | sndbuf=size | accept_filter=filter | deferred | bind | ipv6only=[on|off] ssl | so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]]];

#default 
listion 80;

#conf block
server
```

listen决定nginx服务监听端口。listen后可以加IP地址、端口号或者主机名。如

```
listen 127.0.0.1:8002;
listen 127.0.0.1; //default port 80
listen *:8000;
```

地址后可以加其他参数。

| 参数           | 含义                                                         |
| -------------- | ------------------------------------------------------------ |
| default        | 将所在server块作为整个web服务的默认server块。当一个请求无法匹配配置文件中的所有主机域名时，就会选择默认虚拟主机。如果所有server都未指定，则默认选第一个。 |
| backlog=num    | TCP中backlog大小。（默认-1，无限制）在TCP建立三次连接时，进程还未监听句柄，此时backlog队列将会放置这些连接。如果backlog已满，还有客户端企图建立连接，则会失败。 |
| rcvbuf=size    | 设置监听句柄的SO_RCVBUF参数。                                |
| sndbuf=size    | 设置监听句柄的SO_SNDBUF参数。                                |
| accept_filter  | 设置accept过滤，只对FreeBSD系统有用。                        |
| deferred       | 设置该参数时，若用户建立了TCP连接（三次握手），内核也不会对该连接调度worker进程来处理，只会在用户真正发送请求时才会分配worker进程。使用于大并发情况下。 |
| bind           | 绑定当前端口/地址对，如127.0.0.1:8000。                      |
| ssl            | 在当前监听的端口上建立的连接必须基于SSL。                    |
| so_keepalive   | 当客户端与服务器端三次握手正式建立tcp以后，默认情况下，除非客户端或服务器端关闭上层socket，否则tcp会始终保持连接，如果这个时候网络断掉，这个链接就会变成一个死链接，会占用服务器资源。解决方式为：大多数的上游应用会通过心跳机制来检测对方是否存活，不存会则由上游应用程序关闭socket释放链接。对于tcp探活来说，存在三个参数：tcp_keepalive_time（tcp建立链接后指定时间无数据传输，则会发出探活数据包）、tcp_keepalive_probes（发出探活数据包次数）、tcp_keepalive_intvl（探活数据包之间间隔时间）so_keepalive`=`on 表示开启tcp探活，并且使用系统内核的参数。so_keepalive=30m::10 表示开启tcp探活，30分钟后伍数据会发送探活包，时间间隔使用系统默认的，发送10次探活包。 |
| fastopen       | number，HTTP 处于保持连接（keepalive）状态时，允许不经过三次握手的 TCP 连接的队列的最大数 |
| sentfib        | number，为监听套接字设置关联路由表，仅在 FreeBSD 系统上有效  |
| ipv6only       | on，只接收 IPv6 连接或接收 IPv6 和 IPv4 连接                 |
| reuseport      | 默认情况下，所有的工作进程会共享一个 socket 去监听同一 IP 和端口的组合。该参数启用后，允许每个工作进程有独立的 socket 去监听同一 IP 和端口的组合，内核会对传人的连接进行负载均衡。 |
| proxy_protocol | 在指定监听端口上启用 proxy_protocol 协议支持                 |

解析时，其属于ngx_http_core_commands，具体如下：

```c
    { ngx_string("listen"),
      NGX_HTTP_SRV_CONF|NGX_CONF_1MORE,
      ngx_http_core_listen,
      NGX_HTTP_SRV_CONF_OFFSET,
      0,
      NULL },
```

#### 对应相关数据结构

##### ngx_http_listen_opt_t

```
typedef struct {
    struct sockaddr           *sockaddr; // 套接字地址
    socklen_t                  socklen; // 套接字地址长度
    ngx_str_t                  addr_text; // 配置中监听文本

    unsigned                   set:1; // 用户是否进行了相关设置
    unsigned                   default_server:1; // 是否为默认使用的server
    unsigned                   bind:1; 
    unsigned                   wildcard:1; // 是否为通配符形式监听地址
    unsigned                   ssl:1;
    unsigned                   http2:1;
#if (NGX_HAVE_INET6)
    unsigned                   ipv6only:1;
#endif
    unsigned                   deferred_accept:1;
    unsigned                   reuseport:1;
    unsigned                   so_keepalive:2;
    unsigned                   proxy_protocol:1;

    int                        backlog; // 对应配置backlog
    int                        rcvbuf; // 对应配置rcvbuf
    int                        sndbuf;// 对应配置sndbuf
#if (NGX_HAVE_SETFIB)
    int                        setfib;
#endif
#if (NGX_HAVE_TCP_FASTOPEN)
    int                        fastopen;
#endif
#if (NGX_HAVE_KEEPALIVE_TUNABLE)
    int                        tcp_keepidle; // 建立链接后无数据传输超出该时间，进行数据探活
    int                        tcp_keepintvl; // 探活包之间时间间隔
    int                        tcp_keepcnt; // 探活次数
#endif

#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
    char                      *accept_filter;
#endif
} ngx_http_listen_opt_t;
```

##### ngx_url_t

该结构是用于存储解析url后的结果信息的：

```
typedef struct {
    ngx_str_t                 url; // url
    ngx_str_t                 host; // 主机名
    ngx_str_t                 port_text; // 端口号对应文本
    ngx_str_t                 uri; // uri

    in_port_t                 port; // 端口号
    in_port_t                 default_port; // 默认端口号
    in_port_t                 last_port; // 当端口号是一个区间时，最后一个端口号
    int                       family; // 协议族

    unsigned                  listen:1; // 是否为监听
    unsigned                  uri_part:1;
    unsigned                  no_resolve:1; // 是否不解析主机（url中不一定是ip形式请求，可能是主机，如localhost）

    unsigned                  no_port:1; // 是否用户未指名端口号
    unsigned                  wildcard:1; // 是否监听地址为通配符形式

    socklen_t                 socklen; // 套接字地址长度
    ngx_sockaddr_t            sockaddr; // 套接字地址结构，其是临时变量，会将每一个sockaddr添加到addrs中

    ngx_addr_t               *addrs; // 对应的监听地址
    ngx_uint_t                naddrs; // 对应的监听地址数量

    char                     *err; // 存储解析后错误信息
} ngx_url_t;

typedef struct {
    struct sockaddr          *sockaddr;
    socklen_t                 socklen;
    ngx_str_t                 name;
} ngx_addr_t;
```

##### ngx_http_conf_port_t

该类是记录端口号对应每一个地址的结构。（由于一个机器可能存在多个ip地址）

```
typedef struct {
    ngx_int_t                  family;
    in_port_t                  port;
    ngx_array_t                addrs;     /* array of ngx_http_conf_addr_t */
} ngx_http_conf_port_t;
```

其中ngx_http_core_main_conf_t中的ports参数的成员即是ngx_http_conf_port_t。

##### ngx_http_conf_addr_t

由于一个端口，我们可以同时监听多个ip（一个机器可能存在多个ip地址）。nginx使用ngx_http_conf_addr_t结构来表示一个对应着具体地址的监听端口，因此一个ngx_http_conf_port_t可能对应多个ngx_http_conf_addr_t。具体结构如下：

```
typedef struct {
    // 监听套接字对应的各种属性
    ngx_http_listen_opt_t      opt;
    // 下面三个散列表用于加速寻找对应监听端口上的新连接，确定到达使用哪个server{}虚拟主机下的配置来进行处理，散列表的值是ngx_http_core_srv_conf_t结构体
    ngx_hash_t                 hash;
    ngx_hash_wildcard_t       *wc_head; // 通配符前缀散列表
    ngx_hash_wildcard_t       *wc_tail; // 通配符后置散列表

#if (NGX_PCRE)
    ngx_uint_t                 nregex; // regex数量
    ngx_http_server_name_t    *regex; // 正则表达式匹配的虚拟主机
#endif

    /* the default server configuration for this address:port */
    ngx_http_core_srv_conf_t  *default_server; // 监听端口下默认的虚拟主机
    ngx_array_t                servers;  /* array of ngx_http_core_srv_conf_t */
} ngx_http_conf_addr_t;
```

#### 监听端口与server{}虚拟主机间内存关系

对于一个如下的配置：

```nginx
http {
  server {
    server_name A;
    listen 127.0,0.1:8000;
    listen 80;
    location /L1 {}
  }
  server {
    server_name B;
    listen 80;
    listen 8080;
    listen 173.39.160.51:8000;
    location /L1{}
  }
}
```

[![fGIoYn.png](https://z3.ax1x.com/2021/08/10/fGIoYn.png)](https://imgtu.com/i/fGIoYn)

#### set方法

##### ngx_http_core_listen

其中对应的ngx_http_core_listen方法如下：

```
static char *
ngx_http_core_listen(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    // server层级下的ngx_http_core_srv_conf_t
    ngx_http_core_srv_conf_t *cscf = conf;

    ngx_str_t              *value, size;
    ngx_url_t               u;
    ngx_uint_t              n;
    ngx_http_listen_opt_t   lsopt;

    cscf->listen = 1;

    value = cf->args->elts;

    ngx_memzero(&u, sizeof(ngx_url_t));
    // 对listen后的字符串进行解析，标识为监听，并设置默认端口
    u.url = value[1];
    u.listen = 1;
    u.default_port = 80;
    // 解析参数，解析完成后，u的addrs就存储这ip+port的地址对
    if (ngx_parse_url(cf->pool, &u) != NGX_OK) {
        if (u.err) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "%s in \"%V\" of the \"listen\" directive",
                               u.err, &u.url);
        }

        return NGX_CONF_ERROR;
    }
    // 解析其余参数，先分配存储结构，并设置默认值
    ngx_memzero(&lsopt, sizeof(ngx_http_listen_opt_t));

    lsopt.backlog = NGX_LISTEN_BACKLOG;
    lsopt.rcvbuf = -1;
    lsopt.sndbuf = -1;
#if (NGX_HAVE_SETFIB)
    lsopt.setfib = -1;
#endif
#if (NGX_HAVE_TCP_FASTOPEN)
    lsopt.fastopen = -1;
#endif
#if (NGX_HAVE_INET6)
    lsopt.ipv6only = 1;
#endif
    // 从第二个参数开始进行解析
    for (n = 2; n < cf->args->nelts; n++) {
        // 设置了default_server，则将default_server置1
        if (ngx_strcmp(value[n].data, "default_server") == 0
            || ngx_strcmp(value[n].data, "default") == 0)
        {
            lsopt.default_server = 1;
            continue;
        }

        if (ngx_strcmp(value[n].data, "bind") == 0) {
            lsopt.set = 1;
            lsopt.bind = 1;
            continue;
        }

#if (NGX_HAVE_SETFIB)
        if (ngx_strncmp(value[n].data, "setfib=", 7) == 0) {
            lsopt.setfib = ngx_atoi(value[n].data + 7, value[n].len - 7);
            lsopt.set = 1;
            lsopt.bind = 1;

            if (lsopt.setfib == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid setfib \"%V\"", &value[n]);
                return NGX_CONF_ERROR;
            }

            continue;
        }
#endif

#if (NGX_HAVE_TCP_FASTOPEN)
        if (ngx_strncmp(value[n].data, "fastopen=", 9) == 0) {
            lsopt.fastopen = ngx_atoi(value[n].data + 9, value[n].len - 9);
            lsopt.set = 1;
            lsopt.bind = 1;

            if (lsopt.fastopen == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid fastopen \"%V\"", &value[n]);
                return NGX_CONF_ERROR;
            }

            continue;
        }
#endif

        if (ngx_strncmp(value[n].data, "backlog=", 8) == 0) {
            lsopt.backlog = ngx_atoi(value[n].data + 8, value[n].len - 8);
            lsopt.set = 1;
            lsopt.bind = 1;

            if (lsopt.backlog == NGX_ERROR || lsopt.backlog == 0) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid backlog \"%V\"", &value[n]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        if (ngx_strncmp(value[n].data, "rcvbuf=", 7) == 0) {
            size.len = value[n].len - 7;
            size.data = value[n].data + 7;

            lsopt.rcvbuf = ngx_parse_size(&size);
            lsopt.set = 1;
            lsopt.bind = 1;

            if (lsopt.rcvbuf == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid rcvbuf \"%V\"", &value[n]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        if (ngx_strncmp(value[n].data, "sndbuf=", 7) == 0) {
            size.len = value[n].len - 7;
            size.data = value[n].data + 7;

            lsopt.sndbuf = ngx_parse_size(&size);
            lsopt.set = 1;
            lsopt.bind = 1;

            if (lsopt.sndbuf == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid sndbuf \"%V\"", &value[n]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        if (ngx_strncmp(value[n].data, "accept_filter=", 14) == 0) {
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
            lsopt.accept_filter = (char *) &value[n].data[14];
            lsopt.set = 1;
            lsopt.bind = 1;
#else
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "accept filters \"%V\" are not supported "
                               "on this platform, ignored",
                               &value[n]);
#endif
            continue;
        }

        if (ngx_strcmp(value[n].data, "deferred") == 0) {
#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)
            lsopt.deferred_accept = 1;
            lsopt.set = 1;
            lsopt.bind = 1;
#else
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "the deferred accept is not supported "
                               "on this platform, ignored");
#endif
            continue;
        }

        if (ngx_strncmp(value[n].data, "ipv6only=o", 10) == 0) {
#if (NGX_HAVE_INET6 && defined IPV6_V6ONLY)
            if (ngx_strcmp(&value[n].data[10], "n") == 0) {
                lsopt.ipv6only = 1;

            } else if (ngx_strcmp(&value[n].data[10], "ff") == 0) {
                lsopt.ipv6only = 0;

            } else {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid ipv6only flags \"%s\"",
                                   &value[n].data[9]);
                return NGX_CONF_ERROR;
            }

            lsopt.set = 1;
            lsopt.bind = 1;

            continue;
#else
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "ipv6only is not supported "
                               "on this platform");
            return NGX_CONF_ERROR;
#endif
        }

        if (ngx_strcmp(value[n].data, "reuseport") == 0) {
#if (NGX_HAVE_REUSEPORT)
            lsopt.reuseport = 1;
            lsopt.set = 1;
            lsopt.bind = 1;
#else
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "reuseport is not supported "
                               "on this platform, ignored");
#endif
            continue;
        }

        if (ngx_strcmp(value[n].data, "ssl") == 0) {
#if (NGX_HTTP_SSL)
            lsopt.ssl = 1;
            continue;
#else
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "the \"ssl\" parameter requires "
                               "ngx_http_ssl_module");
            return NGX_CONF_ERROR;
#endif
        }

        if (ngx_strcmp(value[n].data, "http2") == 0) {
#if (NGX_HTTP_V2)
            lsopt.http2 = 1;
            continue;
#else
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "the \"http2\" parameter requires "
                               "ngx_http_v2_module");
            return NGX_CONF_ERROR;
#endif
        }

        if (ngx_strcmp(value[n].data, "spdy") == 0) {
            ngx_conf_log_error(NGX_LOG_WARN, cf, 0,
                               "invalid parameter \"spdy\": "
                               "ngx_http_spdy_module was superseded "
                               "by ngx_http_v2_module");
            continue;
        }

        if (ngx_strncmp(value[n].data, "so_keepalive=", 13) == 0) {

            if (ngx_strcmp(&value[n].data[13], "on") == 0) {
                lsopt.so_keepalive = 1;

            } else if (ngx_strcmp(&value[n].data[13], "off") == 0) {
                lsopt.so_keepalive = 2;

            } else {

#if (NGX_HAVE_KEEPALIVE_TUNABLE)
                u_char     *p, *end;
                ngx_str_t   s;

                end = value[n].data + value[n].len;
                s.data = value[n].data + 13;

                p = ngx_strlchr(s.data, end, ':');
                if (p == NULL) {
                    p = end;
                }

                if (p > s.data) {
                    s.len = p - s.data;

                    lsopt.tcp_keepidle = ngx_parse_time(&s, 1);
                    if (lsopt.tcp_keepidle == (time_t) NGX_ERROR) {
                        goto invalid_so_keepalive;
                    }
                }

                s.data = (p < end) ? (p + 1) : end;

                p = ngx_strlchr(s.data, end, ':');
                if (p == NULL) {
                    p = end;
                }

                if (p > s.data) {
                    s.len = p - s.data;

                    lsopt.tcp_keepintvl = ngx_parse_time(&s, 1);
                    if (lsopt.tcp_keepintvl == (time_t) NGX_ERROR) {
                        goto invalid_so_keepalive;
                    }
                }

                s.data = (p < end) ? (p + 1) : end;

                if (s.data < end) {
                    s.len = end - s.data;

                    lsopt.tcp_keepcnt = ngx_atoi(s.data, s.len);
                    if (lsopt.tcp_keepcnt == NGX_ERROR) {
                        goto invalid_so_keepalive;
                    }
                }

                if (lsopt.tcp_keepidle == 0 && lsopt.tcp_keepintvl == 0
                    && lsopt.tcp_keepcnt == 0)
                {
                    goto invalid_so_keepalive;
                }

                lsopt.so_keepalive = 1;

#else

                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "the \"so_keepalive\" parameter accepts "
                                   "only \"on\" or \"off\" on this platform");
                return NGX_CONF_ERROR;

#endif
            }

            lsopt.set = 1;
            lsopt.bind = 1;

            continue;

#if (NGX_HAVE_KEEPALIVE_TUNABLE)
        invalid_so_keepalive:

            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid so_keepalive value: \"%s\"",
                               &value[n].data[13]);
            return NGX_CONF_ERROR;
#endif
        }

        if (ngx_strcmp(value[n].data, "proxy_protocol") == 0) {
            lsopt.proxy_protocol = 1;
            continue;
        }

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid parameter \"%V\"", &value[n]);
        return NGX_CONF_ERROR;
    }
    // 对每一个解析出的地址，和配置的属性，使用ngx_http_add_listen方法添加ip+port形式的地址到main层级的ngx_http_core_main_conf_t的ports中
    for (n = 0; n < u.naddrs; n++) {
        lsopt.sockaddr = u.addrs[n].sockaddr;
        lsopt.socklen = u.addrs[n].socklen;
        lsopt.addr_text = u.addrs[n].name;
        lsopt.wildcard = ngx_inet_wildcard(lsopt.sockaddr);

        if (ngx_http_add_listen(cf, cscf, &lsopt) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }

    return NGX_CONF_OK;
}
```

这里需要着重介绍一下bind标志位。bind表示是否一个ip+port的地址进行绑定。这样的目的是，如果在一个在同样的端口上，存在一个通配符形式地址（即任意ip地址）时。对于bind的地址来说，会先开启一个地址的监听，让bind的端口走单独的监听，其他未绑定的地址走通配符监听。这样主要是因为我们可能希望在该bind的地址上设置一些额外信息，如backlog、rcvbuf、sndbuf等内容。因此在设置这些额外参数时，就会默认将该地址设置为bind。当然也可以通过配置中直接设置bind的方式指明需要单独进行监听。

##### 解析url

###### ngx_parse_url

解析参数的代码如下：

```
ngx_int_t
ngx_parse_url(ngx_pool_t *pool, ngx_url_t *u)
{
    u_char  *p;
    size_t   len;

    p = u->url.data;
    len = u->url.len;

    if (len >= 5 && ngx_strncasecmp(p, (u_char *) "unix:", 5) == 0) {
        return ngx_parse_unix_domain_url(pool, u);
    }

    if (len && p[0] == '[') {
        return ngx_parse_inet6_url(pool, u);
    }

    return ngx_parse_inet_url(pool, u);
}
```

这里使用的是ngx_parse_inet_url函数。具体逻辑如下：

```
static ngx_int_t
ngx_parse_inet_url(ngx_pool_t *pool, ngx_url_t *u)
{
    u_char              *host, *port, *last, *uri, *args, *dash;
    size_t               len;
    ngx_int_t            n;
    struct sockaddr_in  *sin;
    // 分配套接字地址
    u->socklen = sizeof(struct sockaddr_in);
    sin = (struct sockaddr_in *) &u->sockaddr;
    // 设置套接字族
    sin->sin_family = AF_INET;

    u->family = AF_INET;

    host = u->url.data;

    last = host + u->url.len;
    // 查找设置的端口号起始地址。ngx_strlchr函数找到第一个指定的字符，并返回对应的指针。这里查找到:8000这种端口号
    port = ngx_strlchr(host, last, ':');
    // 查找到uri起始地址
    uri = ngx_strlchr(host, last, '/');
    // 查找参数的起始地址
    args = ngx_strlchr(host, last, '?');
    // 如果存在参数，并且uri为空，或者参数的起始地址小于uri的起始地址，则将uri设置为参数的起始地址
    if (args) {
        if (uri == NULL || args < uri) {
            uri = args;
        }
    }
    // 存在uri
    if (uri) {
        // 如果是在解析监听，则报错返回
        if (u->listen || !u->uri_part) {
            u->err = "invalid host";
            return NGX_ERROR;
        }
        // uri长度为从末尾到uri起始地址
        u->uri.len = last - uri;
        u->uri.data = uri;
        // 将last变更为uri地址
        last = uri;
        // 如果uri起始地址小于端口号起始地址，则说明不存在端口号
        if (uri < port) {
            port = NULL;
        }
    }
    // 如果查找到:80形式的端口号
    if (port) {
        // 获取端口号起始地址
        port++;
        // 获取端口号长度
        len = last - port;
        // 如果在解析监听
        if (u->listen) {
            // 查找监听端口是否为一个范围，即:8000-8100形式
            dash = ngx_strlchr(port, last, '-');
            // 如果是范围
            if (dash) {
                dash++;
                // 解析末尾端口
                n = ngx_atoi(dash, last - dash);

                if (n < 1 || n > 65535) {
                    u->err = "invalid port";
                    return NGX_ERROR;
                }

                u->last_port = (in_port_t) n;
                // 变更长度为起始端口长度
                len = dash - port - 1;
            }
        }
        // 获取端口（如果是监听，则是起始端口）
        n = ngx_atoi(port, len);

        if (n < 1 || n > 65535) {
            u->err = "invalid port";
            return NGX_ERROR;
        }
        // 如果范围不合法，报错
        if (u->last_port && n > u->last_port) {
            u->err = "invalid port range";
            return NGX_ERROR;
        }
        // 设置端口为n
        u->port = (in_port_t) n;
        sin->sin_port = htons((in_port_t) n);
        // 设置端口对应的文本
        u->port_text.len = last - port;
        u->port_text.data = port;

        last = port - 1;

    } else {
        //未解析到:8000形式端口
        // 如果uri为空
        if (uri == NULL) {
            // 如果是在解析监听
            if (u->listen) {

                /* test value as port only */
                // 认为整个文本全是端口，即listen 80；形式，并尝试解析
                len = last - host;
                // 查看是否为范围监听
                dash = ngx_strlchr(host, last, '-');

                if (dash) {
                    dash++;

                    n = ngx_atoi(dash, last - dash);

                    if (n == NGX_ERROR) {
                        goto no_port;
                    }

                    if (n < 1 || n > 65535) {
                        u->err = "invalid port";

                    } else {
                        u->last_port = (in_port_t) n;
                    }

                    len = dash - host - 1;
                }

                n = ngx_atoi(host, len);
                // 如果解析到端口
                if (n != NGX_ERROR) {

                    if (u->err) {
                        return NGX_ERROR;
                    }

                    if (n < 1 || n > 65535) {
                        u->err = "invalid port";
                        return NGX_ERROR;
                    }

                    if (u->last_port && n > u->last_port) {
                        u->err = "invalid port range";
                        return NGX_ERROR;
                    }

                    u->port = (in_port_t) n;
                    sin->sin_port = htons((in_port_t) n);
                    sin->sin_addr.s_addr = INADDR_ANY;

                    u->port_text.len = last - host;
                    u->port_text.data = host;
                    // 由于是listen未指定ip，因此机器所有ip均被监听，因此属于通配符形式监听
                    u->wildcard = 1;
                    // 添加监听端口到addrs中
                    return ngx_inet_add_addr(pool, u, &u->sockaddr.sockaddr,
                                             u->socklen, 1);
                }
            }
        }

no_port:
        // 未查找到端口号，即只有ip的情况，设置默认端口
        u->err = NULL;
        u->no_port = 1;
        u->port = u->default_port;
        sin->sin_port = htons(u->default_port);
        u->last_port = 0;
    }
    // 只有主机名，或主机名+端口号形式的配置，才会执行如下处理
    len = last - host;
    // 如果没有主机，则报错
    if (len == 0) {
        u->err = "no host";
        return NGX_ERROR;
    }

    u->host.len = len;
    u->host.data = host;
    // 如果是解析监听，并且主机名是*，设置sin_addr.s_addr为INADDR_ANY，即监听所有ip，并设置wildcard为1
    if (u->listen && len == 1 && *host == '*') {
        sin->sin_addr.s_addr = INADDR_ANY;
        u->wildcard = 1;
        return ngx_inet_add_addr(pool, u, &u->sockaddr.sockaddr, u->socklen, 1);
    }
    // 获取主机ip，将127.0.0.1转换为int形式的地址
    sin->sin_addr.s_addr = ngx_inet_addr(host, len);
    // ip合法（如果主机名是域名，则返回就是INADDR_NONE）
    if (sin->sin_addr.s_addr != INADDR_NONE) {
        // ip是INADDR_ANY，设置wildcard
        if (sin->sin_addr.s_addr == INADDR_ANY) {
            u->wildcard = 1;
        }
        // 添加地址到addrs
        return ngx_inet_add_addr(pool, u, &u->sockaddr.sockaddr, u->socklen, 1);
    }
    // 如果不解析主机名
    if (u->no_resolve) {
        return NGX_OK;
    }
    // 使用getaddrinfo函数解析域名，获取ip，并添加地址到u的addrs中
    if (ngx_inet_resolve_host(pool, u) != NGX_OK) {
        return NGX_ERROR;
    }
    // 根据域名解析出的地址，对u的相关参数进行赋值
    u->family = u->addrs[0].sockaddr->sa_family;
    u->socklen = u->addrs[0].socklen;
    ngx_memcpy(&u->sockaddr, u->addrs[0].sockaddr, u->addrs[0].socklen);
    u->wildcard = ngx_inet_wildcard(&u->sockaddr.sockaddr);

    return NGX_OK;
}
```

###### ngx_inet_add_addr

该方法用于将ngx_url_t解析出的地址添加到其addrs参数中。具体逻辑如下：

```c
static ngx_int_t
ngx_inet_add_addr(ngx_pool_t *pool, ngx_url_t *u, struct sockaddr *sockaddr,
    socklen_t socklen, ngx_uint_t total)
{
    u_char           *p;
    size_t            len;
    ngx_uint_t        i, nports;
    ngx_addr_t       *addr;
    struct sockaddr  *sa;
    // 如果是范围监听，则会有last_port
    nports = u->last_port ? u->last_port - u->port + 1 : 1;
    // 初始化addrs
    if (u->addrs == NULL) {
        u->addrs = ngx_palloc(pool, total * nports * sizeof(ngx_addr_t));
        if (u->addrs == NULL) {
            return NGX_ERROR;
        }
    }
    // 对每一个监听端口进行处理
    for (i = 0; i < nports; i++) {
        // 初始化地址，并对地址中参数进行赋值
        sa = ngx_pcalloc(pool, socklen);
        if (sa == NULL) {
            return NGX_ERROR;
        }

        ngx_memcpy(sa, sockaddr, socklen);

        ngx_inet_set_port(sa, u->port + i);
        // 根据不同协议族对地址赋值
        switch (sa->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            len = NGX_INET6_ADDRSTRLEN + sizeof("[]:65536") - 1;
            break;
#endif

        default: /* AF_INET */
            len = NGX_INET_ADDRSTRLEN + sizeof(":65535") - 1;
        }

        p = ngx_pnalloc(pool, len);
        if (p == NULL) {
            return NGX_ERROR;
        }

        len = ngx_sock_ntop(sa, socklen, p, len, 1);
        // 添加地址到addrs中
        addr = &u->addrs[u->naddrs++];

        addr->sockaddr = sa;
        addr->socklen = socklen;

        addr->name.len = len;
        addr->name.data = p;
    }

    return NGX_OK;
}
```

###### ngx_inet_resolve_host

该方法将域名解析为ip地址。使用的方法为getaddrinfo函数。具体可以参考[地址查询](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%9C%B0%E5%9D%80%E6%9F%A5%E8%AF%A2)

代码逻辑如下：

```c
ngx_int_t
ngx_inet_resolve_host(ngx_pool_t *pool, ngx_url_t *u)
{
    u_char           *host;
    ngx_uint_t        n;
    struct addrinfo   hints, *res, *rp;

    host = ngx_alloc(u->host.len + 1, pool->log);
    if (host == NULL) {
        return NGX_ERROR;
    }

    (void) ngx_cpystrn(host, u->host.data, u->host.len + 1);
    // hint来选择符合特定条件的地址
    ngx_memzero(&hints, sizeof(struct addrinfo));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
#ifdef AI_ADDRCONFIG
    hints.ai_flags = AI_ADDRCONFIG;
#endif

    if (getaddrinfo((char *) host, NULL, &hints, &res) != 0) {
        u->err = "host not found";
        ngx_free(host);
        return NGX_ERROR;
    }

    ngx_free(host);
    // 遍历获取到的每一个地址，查找ipv4或ipv6的地址数量
    for (n = 0, rp = res; rp != NULL; rp = rp->ai_next) {

        switch (rp->ai_family) {

        case AF_INET:
        case AF_INET6:
            break;

        default:
            continue;
        }

        n++;
    }

    if (n == 0) {
        u->err = "host not found";
        goto failed;
    }

    /* MP: ngx_shared_palloc() */

    for (rp = res; rp != NULL; rp = rp->ai_next) {

        switch (rp->ai_family) {

        case AF_INET:
        case AF_INET6:
            break;

        default:
            continue;
        }
        // 将每一个ipv4或ipv6地址都添加到addrs中
        if (ngx_inet_add_addr(pool, u, rp->ai_addr, rp->ai_addrlen, n)
            != NGX_OK)
        {
            goto failed;
        }
    }

    freeaddrinfo(res);
    return NGX_OK;

failed:

    freeaddrinfo(res);
    return NGX_ERROR;
}
```

这里配置的主机可以任意配置，在设置监听时，如果解析出的ip地址不是机器表示的ip，则报错。

##### 添加监听地址到ports中

###### ngx_http_add_listen

```c
ngx_int_t
ngx_http_add_listen(ngx_conf_t *cf, ngx_http_core_srv_conf_t *cscf,
    ngx_http_listen_opt_t *lsopt)
{
    in_port_t                   p;
    ngx_uint_t                  i;
    struct sockaddr            *sa;
    ngx_http_conf_port_t       *port;
    ngx_http_core_main_conf_t  *cmcf;
    // 获取main级别下的ngx_http_core_main_conf_t
    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    // 创建ports空间
    if (cmcf->ports == NULL) {
        cmcf->ports = ngx_array_create(cf->temp_pool, 2,
                                       sizeof(ngx_http_conf_port_t));
        if (cmcf->ports == NULL) {
            return NGX_ERROR;
        }
    }

    sa = lsopt->sockaddr;
    p = ngx_inet_get_port(sa);
    // 遍历每个port，查找是当前端口已存在
    port = cmcf->ports->elts;
    for (i = 0; i < cmcf->ports->nelts; i++) {

        if (p != port[i].port || sa->sa_family != port[i].family) {
            continue;
        }

        /* a port is already in the port list */
        // 对于已存在的端口，执行如下函数来添加对应的addr
        return ngx_http_add_addresses(cf, cscf, &port[i], lsopt);
    }

    /* add a port to the port list */
   // 添加当前端口到ports中
    port = ngx_array_push(cmcf->ports);
    if (port == NULL) {
        return NGX_ERROR;
    }

    port->family = sa->sa_family;
    port->port = p;
    port->addrs.elts = NULL;
    // 添加对应的addr
    return ngx_http_add_address(cf, cscf, port, lsopt);
}
```

###### ngx_http_add_address

```c
static ngx_int_t
ngx_http_add_address(ngx_conf_t *cf, ngx_http_core_srv_conf_t *cscf,
    ngx_http_conf_port_t *port, ngx_http_listen_opt_t *lsopt)
{
    ngx_http_conf_addr_t  *addr;
    // 分配空间
    if (port->addrs.elts == NULL) {
        if (ngx_array_init(&port->addrs, cf->temp_pool, 4,
                           sizeof(ngx_http_conf_addr_t))
            != NGX_OK)
        {
            return NGX_ERROR;
        }
    }

#if (NGX_HTTP_V2 && NGX_HTTP_SSL                                              \
     && !defined TLSEXT_TYPE_application_layer_protocol_negotiation           \
     && !defined TLSEXT_TYPE_next_proto_neg)

    if (lsopt->http2 && lsopt->ssl) {
        ngx_conf_log_error(NGX_LOG_WARN, cf, 0,
                           "nginx was built with OpenSSL that lacks ALPN "
                           "and NPN support, HTTP/2 is not enabled for %V",
                           &lsopt->addr_text);
    }

#endif
    // 添加addr
    addr = ngx_array_push(&port->addrs);
    if (addr == NULL) {
        return NGX_ERROR;
    }
    // 设置对应成员
    addr->opt = *lsopt;
    addr->hash.buckets = NULL;
    addr->hash.size = 0;
    addr->wc_head = NULL;
    addr->wc_tail = NULL;
#if (NGX_PCRE)
    addr->nregex = 0;
    addr->regex = NULL;
#endif
    addr->default_server = cscf;
    addr->servers.elts = NULL;
    // 绑定对应的server模块
    return ngx_http_add_server(cf, cscf, addr);
}
```

###### gx_http_add_server

绑定ip+port形式的地址到对应的server块中。逻辑如下：

```c
static ngx_int_t
ngx_http_add_server(ngx_conf_t *cf, ngx_http_core_srv_conf_t *cscf,
    ngx_http_conf_addr_t *addr)
{
    ngx_uint_t                  i;
    ngx_http_core_srv_conf_t  **server;
    // 分配server空间
    if (addr->servers.elts == NULL) {
        if (ngx_array_init(&addr->servers, cf->temp_pool, 4,
                           sizeof(ngx_http_core_srv_conf_t *))
            != NGX_OK)
        {
            return NGX_ERROR;
        }

    } else {
        // 查找当前地址中是否已经存在对该server服务的绑定关系，如果存在，则报错
        server = addr->servers.elts;
        for (i = 0; i < addr->servers.nelts; i++) {
            if (server[i] == cscf) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "a duplicate listen %V",
                                   &addr->opt.addr_text);
                return NGX_ERROR;
            }
        }
    }
    // 在servers中添加对应的server配置块
    server = ngx_array_push(&addr->servers);
    if (server == NULL) {
        return NGX_ERROR;
    }

    *server = cscf;

    return NGX_OK;
}
```

###### ngx_http_add_addresses

当监听端口（port形式的地址）已经存在一个ngx_http_conf_port_t结构时，这时添加一个addr（ip+port形式的地址）需要使用该方法。该方法会先对比是否已存在对应的ip+port形式的地址，再进行相应操作。具体逻辑如下：

```c
static ngx_int_t
ngx_http_add_addresses(ngx_conf_t *cf, ngx_http_core_srv_conf_t *cscf,
    ngx_http_conf_port_t *port, ngx_http_listen_opt_t *lsopt)
{
    ngx_uint_t             i, default_server, proxy_protocol;
    ngx_http_conf_addr_t  *addr;
#if (NGX_HTTP_SSL)
    ngx_uint_t             ssl;
#endif
#if (NGX_HTTP_V2)
    ngx_uint_t             http2;
#endif

    /*
     * we cannot compare whole sockaddr struct's as kernel
     * may fill some fields in inherited sockaddr struct's
     */

    addr = port->addrs.elts;
    // 遍历addrs，查找是否已存在与当前ip+port对应的地址相同的地址
    for (i = 0; i < port->addrs.nelts; i++) {

        if (ngx_cmp_sockaddr(lsopt->sockaddr, lsopt->socklen,
                             addr[i].opt.sockaddr,
                             addr[i].opt.socklen, 0)
            != NGX_OK)
        {
            continue;
        }

        /* the address is already in the address list */
        // 当前地址已存在时进行如下处理

        // 在该地址中的server中增加对应的server服务
        if (ngx_http_add_server(cf, cscf, &addr[i]) != NGX_OK) {
            return NGX_ERROR;
        }

        /* preserve default_server bit during listen options overwriting */
        // 记录当前的opt是否设置了默认的服务
        default_server = addr[i].opt.default_server;

        proxy_protocol = lsopt->proxy_protocol || addr[i].opt.proxy_protocol;

#if (NGX_HTTP_SSL)
        ssl = lsopt->ssl || addr[i].opt.ssl;
#endif
#if (NGX_HTTP_V2)
        http2 = lsopt->http2 || addr[i].opt.http2;
#endif
        // 如果当前地址进行了额外参数的设置，如rcvbuf=size | sndbuf=size，不包括default_server
        if (lsopt->set) {
            // 当前的opt也设置进行了相应的设置，这时会报错，因为使用opt中相应的参数对监听进行设置，但同一个ip+port组成的地址只能进行一次设置，这时所有服务对该地址进行监听时（如果有其他服务），则使用同样的设置。
            if (addr[i].opt.set) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "duplicate listen options for %V",
                                   &addr[i].opt.addr_text);
                return NGX_ERROR;
            }
            // 设置地址的opt，对应于进行了额外设置的opt
            addr[i].opt = *lsopt;
        }

        /* check the duplicate "default" server for this address:port */
        // 如果当前解析出的listen设置为了默认的server服务
        if (lsopt->default_server) {
            // 之前已经设置了默认的server服务，则报错
            if (default_server) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "a duplicate default server for %V",
                                   &addr[i].opt.addr_text);
                return NGX_ERROR;
            }
            // 标识已经设置了默认的服务
            default_server = 1;
            // 地址中default_server执行当前的server块
            addr[i].default_server = cscf;
        }
        // 记录是否已经设置了默认的服务块
        addr[i].opt.default_server = default_server;
        addr[i].opt.proxy_protocol = proxy_protocol;
#if (NGX_HTTP_SSL)
        addr[i].opt.ssl = ssl;
#endif
#if (NGX_HTTP_V2)
        addr[i].opt.http2 = http2;
#endif

        return NGX_OK;
    }

    /* add the address to the addresses list that bound to this port */
    // 未找到相同的地址，则在port中增加一个ip+port形式的地址
    return ngx_http_add_address(cf, cscf, port, lsopt);
}
```

#### server_name处理

快速查找请求对应的server块和server_name密切相关，这里先来看一下配置中server_name相应的处理。注意一个server块支持多个server_name，且server_name支持精确名，通配符和正则表达式。在未设置时，默认为空。

##### 相关数据结构

###### ngx_http_server_name_t

ngx_http_server_name_t结构用户存储服务名，并构建服务名到server的关系

```
typedef struct {
#if (NGX_PCRE)
    ngx_http_regex_t          *regex;
#endif
    ngx_http_core_srv_conf_t  *server;   /* virtual name server conf */
    ngx_str_t                  name;
} ngx_http_server_name_t;
```



##### 处理逻辑

```server
    { ngx_string("server_name"),
      NGX_HTTP_SRV_CONF|NGX_CONF_1MORE,
      ngx_http_core_server_name,
      NGX_HTTP_SRV_CONF_OFFSET,
      0,
      NULL },
```

```
static char *
ngx_http_core_server_name(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    // 获取server层级下的ngx_http_core_srv_conf_t
    ngx_http_core_srv_conf_t *cscf = conf;

    u_char                   ch;
    ngx_str_t               *value;
    ngx_uint_t               i;
    ngx_http_server_name_t  *sn;

    value = cf->args->elts;
    // 对于每一个server_name进行处理。
    for (i = 1; i < cf->args->nelts; i++) {

        ch = value[i].data[0];
        // 检验配置语法准确性
        if ((ch == '*' && (value[i].len < 3 || value[i].data[1] != '.'))
            || (ch == '.' && value[i].len < 2))
        {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "server name \"%V\" is invalid", &value[i]);
            return NGX_CONF_ERROR;
        }

        if (ngx_strchr(value[i].data, '/')) {
            ngx_conf_log_error(NGX_LOG_WARN, cf, 0,
                               "server name \"%V\" has suspicious symbols",
                               &value[i]);
        }
        // 在ngx_http_core_srv_conf_t的server_names中增加元素
        sn = ngx_array_push(&cscf->server_names);
        if (sn == NULL) {
            return NGX_CONF_ERROR;
        }

#if (NGX_PCRE)
        sn->regex = NULL;
#endif
        sn->server = cscf;
        // 对于服务名是hostname来说，使用之前获取到的hostname
        if (ngx_strcasecmp(value[i].data, (u_char *) "$hostname") == 0) {
            sn->name = cf->cycle->hostname;

        } else {
            sn->name = value[i];
        }
        // 对应首字母为~的表达式来说，是使用正则匹配
        if (value[i].data[0] != '~') {
            ngx_strlow(sn->name.data, sn->name.data, sn->name.len);
            continue;
        }

#if (NGX_PCRE)
        {
        //构建正则匹配
        u_char               *p;
        ngx_regex_compile_t   rc;
        u_char                errstr[NGX_MAX_CONF_ERRSTR];

        if (value[i].len == 1) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "empty regex in server name \"%V\"", &value[i]);
            return NGX_CONF_ERROR;
        }

        value[i].len--;
        value[i].data++;

        ngx_memzero(&rc, sizeof(ngx_regex_compile_t));

        rc.pattern = value[i];
        rc.err.len = NGX_MAX_CONF_ERRSTR;
        rc.err.data = errstr;

        for (p = value[i].data; p < value[i].data + value[i].len; p++) {
            if (*p >= 'A' && *p <= 'Z') {
                rc.options = NGX_REGEX_CASELESS;
                break;
            }
        }

        sn->regex = ngx_http_regex_compile(cf, &rc);
        if (sn->regex == NULL) {
            return NGX_CONF_ERROR;
        }

        sn->name = value[i];
        cscf->captures = (rc.captures > 0);
        }
#else
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "using regex \"%V\" "
                           "requires PCRE library", &value[i]);

        return NGX_CONF_ERROR;
#endif
    }

    return NGX_CONF_OK;
}
```

注意，对应某个模块的serve_name未明确配置来说，其对应的ngx_http_core_srv_conf_t的server_names中将为空。

#### server块的快速查找

##### 相关数据结构

相关数据结构可以先不看，先看处理函数，当遇到对应结构时再查看。

###### ngx_http_port_t

每个监听地址实际对应的地址数量和具体每个地址信息。由于含有通配符的地址，可能只需要进行一个监听，但对应多个地址。

```
typedef struct {
    /* ngx_http_in_addr_t or ngx_http_in6_addr_t */
    void                      *addrs; // 每个地址的详细信息，按协议区分
    ngx_uint_t                 naddrs; // 地址数据
} ngx_http_port_t;
```

###### ngx_listening_t

前文已介绍过。

###### ngx_http_in_addr_t

```
typedef struct {
    in_addr_t                  addr; // ipv4地址
    ngx_http_addr_conf_t       conf; // 监听地址对应的详细信息
} ngx_http_in_addr_t;
```

###### ngx_http_addr_conf_s

```
struct ngx_http_addr_conf_s {
    /* the default server configuration for this address:port */
    ngx_http_core_srv_conf_t  *default_server; // 监听地址对应的默认server

    ngx_http_virtual_names_t  *virtual_names; // 监听地址对于的server集合

    unsigned                   ssl:1;
    unsigned                   http2:1;
    unsigned                   proxy_protocol:1;
};
```

###### ngx_http_virtual_names_t

```
typedef struct {
    ngx_hash_combined_t        names; // 存在server信息的hash结构，包含精准，前置通配符，后置通配符

    ngx_uint_t                 nregex; // 正则表达式server name数量
    ngx_http_server_name_t    *regex; // 每一个正则表达式server name
} ngx_http_virtual_names_t;
```



##### ngx_http_optimize_servers

经过上述的set方法，已经构建了从main级别的ngx_http_core_main_conf_t成员ports（ngx_http_conf_port_t，port形式的地址）数组、到addrs（ngx_http_conf_addr_t，ip+port形式的地址）、再到servers（ngx_http_core_srv_conf_t，每一个虚拟server）数组的关系。但每次查找时，使用ip+port的地址，再查找对应的server是相对复杂的，如果有大量的server块，并且监听的是同一个ip+port形式的地址。这时逐个进行server_name和请求的host匹配是过于缓慢的。为了加速查找，这里在ip+port形式的地址中，增加了hash表。已加速查找。具体hash表的构建在http层set函数ngx_http_block的ngx_http_optimize_servers方法中。其逻辑如下：

```
    /* optimize the lists of ports, addresses and server names */

    if (ngx_http_optimize_servers(cf, cmcf, cmcf->ports) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
```

```c
static ngx_int_t
ngx_http_optimize_servers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf,
    ngx_array_t *ports)
{
    ngx_uint_t             p, a;
    ngx_http_conf_port_t  *port;
    ngx_http_conf_addr_t  *addr;

    if (ports == NULL) {
        return NGX_OK;
    }
    // 遍历每一个port地址下的ip+port地址
    port = ports->elts;
    for (p = 0; p < ports->nelts; p++) {
        /* 对ip+port的地址进行排序，将通配符形式地址，即任意ip+port的排到最后（如果有，不需要考虑是否bind()ed，因为对应通配符的地址来说bind()无意义）。最开始是bind()ed的地址。中间是未bind()ed的地址。*/
        ngx_sort(port[p].addrs.elts, (size_t) port[p].addrs.nelts,
                 sizeof(ngx_http_conf_addr_t), ngx_http_cmp_conf_addrs);

        /*
         * check whether all name-based servers have the same
         * configuration as a default server for given address:port
         */
        // 遍历ip+port地址
        addr = port[p].addrs.elts;
        for (a = 0; a < port[p].addrs.nelts; a++) {
            // 如果ip+port地址下有多个server块，或者只有一个server块（会被设置为默认server块）存在正则匹配时
            if (addr[a].servers.nelts > 1
#if (NGX_PCRE)
                || addr[a].default_server->captures
#endif
               )
            {
                // 构建hash表，方便查找
                if (ngx_http_server_names(cf, cmcf, &addr[a]) != NGX_OK) {
                    return NGX_ERROR;
                }
            }
        }
        // 构建需要进行监听的结构
        if (ngx_http_init_listening(cf, &port[p]) != NGX_OK) {
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}
```



##### ngx_http_server_names

```c
static ngx_int_t
ngx_http_server_names(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf,
    ngx_http_conf_addr_t *addr)
{
    ngx_int_t                   rc;
    ngx_uint_t                  n, s;
    ngx_hash_init_t             hash;
    ngx_hash_keys_arrays_t      ha;
    ngx_http_server_name_t     *name;
    ngx_http_core_srv_conf_t  **cscfp;
#if (NGX_PCRE)
    ngx_uint_t                  regex, i;

    regex = 0;
#endif
    // 分配构建hash表的ngx_hash_keys_arrays_t结构，具体可查看前文的正则表达式
    ngx_memzero(&ha, sizeof(ngx_hash_keys_arrays_t));

    ha.temp_pool = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, cf->log);
    if (ha.temp_pool == NULL) {
        return NGX_ERROR;
    }

    ha.pool = cf->pool;

    if (ngx_hash_keys_array_init(&ha, NGX_HASH_LARGE) != NGX_OK) {
        goto failed;
    }

    cscfp = addr->servers.elts;
    // 遍历ip+port地址下每一个server块的配置
    for (s = 0; s < addr->servers.nelts; s++) {

        name = cscfp[s]->server_names.elts;
        /* 遍历每个seerver块下的server_names（ngx_http_server_name_t数组），注意，如果未显式配置server_name，则数组为空*/
        for (n = 0; n < cscfp[s]->server_names.nelts; n++) {

#if (NGX_PCRE)
            // 如果server_name为正则表达式,则跳过
            if (name[n].regex) {
                regex++;
                continue;
            }
#endif
            // 添加server_name到要构建的hash表的ngx_hash_keys_arrays_t结构中
            rc = ngx_hash_add_key(&ha, &name[n].name, name[n].server,
                                  NGX_HASH_WILDCARD_KEY);

            if (rc == NGX_ERROR) {
                return NGX_ERROR;
            }

            if (rc == NGX_DECLINED) {
                ngx_log_error(NGX_LOG_EMERG, cf->log, 0,
                              "invalid server name or wildcard \"%V\" on %V",
                              &name[n].name, &addr->opt.addr_text);
                return NGX_ERROR;
            }

            if (rc == NGX_BUSY) {
                ngx_log_error(NGX_LOG_WARN, cf->log, 0,
                              "conflicting server name \"%V\" on %V, ignored",
                              &name[n].name, &addr->opt.addr_text);
            }
        }
    }
    // 构建hash表的hash函数，即相关配置，其中server_names_hash_max_size，和server_names_hash_bucket_size可配置
    hash.key = ngx_hash_key_lc;
    hash.max_size = cmcf->server_names_hash_max_size;
    hash.bucket_size = cmcf->server_names_hash_bucket_size;
    hash.name = "server_names_hash";
    hash.pool = cf->pool;
    // 如果精确名数组不为空，则构建精确名hash表
    if (ha.keys.nelts) {
        hash.hash = &addr->hash;
        hash.temp_pool = NULL;

        if (ngx_hash_init(&hash, ha.keys.elts, ha.keys.nelts) != NGX_OK) {
            goto failed;
        }
    }
    //构建前缀通配符
    if (ha.dns_wc_head.nelts) {

        ngx_qsort(ha.dns_wc_head.elts, (size_t) ha.dns_wc_head.nelts,
                  sizeof(ngx_hash_key_t), ngx_http_cmp_dns_wildcards);

        hash.hash = NULL;
        hash.temp_pool = ha.temp_pool;

        if (ngx_hash_wildcard_init(&hash, ha.dns_wc_head.elts,
                                   ha.dns_wc_head.nelts)
            != NGX_OK)
        {
            goto failed;
        }

        addr->wc_head = (ngx_hash_wildcard_t *) hash.hash;
    }
    // 构建后置通配符
    if (ha.dns_wc_tail.nelts) {

        ngx_qsort(ha.dns_wc_tail.elts, (size_t) ha.dns_wc_tail.nelts,
                  sizeof(ngx_hash_key_t), ngx_http_cmp_dns_wildcards);

        hash.hash = NULL;
        hash.temp_pool = ha.temp_pool;

        if (ngx_hash_wildcard_init(&hash, ha.dns_wc_tail.elts,
                                   ha.dns_wc_tail.nelts)
            != NGX_OK)
        {
            goto failed;
        }

        addr->wc_tail = (ngx_hash_wildcard_t *) hash.hash;
    }

    ngx_destroy_pool(ha.temp_pool);

#if (NGX_PCRE)
    // 如果存在通配符名，则在addr的regex中添加正则表达式server name
    if (regex == 0) {
        return NGX_OK;
    }

    addr->nregex = regex;
    addr->regex = ngx_palloc(cf->pool, regex * sizeof(ngx_http_server_name_t));
    if (addr->regex == NULL) {
        return NGX_ERROR;
    }

    i = 0;

    for (s = 0; s < addr->servers.nelts; s++) {

        name = cscfp[s]->server_names.elts;

        for (n = 0; n < cscfp[s]->server_names.nelts; n++) {
            if (name[n].regex) {
                addr->regex[i++] = name[n];
            }
        }
    }

#endif

    return NGX_OK;

failed:

    ngx_destroy_pool(ha.temp_pool);

    return NGX_ERROR;
}
```

##### ngx_http_init_listening

```c
static ngx_int_t
ngx_http_init_listening(ngx_conf_t *cf, ngx_http_conf_port_t *port)
{
    ngx_uint_t                 i, last, bind_wildcard;
    ngx_listening_t           *ls;
    ngx_http_port_t           *hport;
    ngx_http_conf_addr_t      *addr;

    addr = port->addrs.elts;
    last = port->addrs.nelts;

    /*
     * If there is a binding to an "*:port" then we need to bind() to
     * the "*:port" only and ignore other implicit bindings.  The bindings
     * have been already sorted: explicit bindings are on the start, then
     * implicit bindings go, and wildcard binding is in the end.
     */
     /* 对于一个port存在通配符形式地址来说，未bind()的ip+port不需要额外进行监听，只需要监听bind（）的地址。注意这里已经是排好序的ip+port形式的地址，其中bind()的在最前面（非通配符形式），其次是未bind（）地址，最后是通配符地址 */
     // 如果最后一个是通配符形式地址
    if (addr[last - 1].opt.wildcard) {
        // 设置最后一个为bind的，保证最后所有地址均被监听
        addr[last - 1].opt.bind = 1;
        // 记录存在通配符形式地址
        bind_wildcard = 1;

    } else {
        // 记录不存在通配符形式地址
        bind_wildcard = 0;
    }

    i = 0;
    // 遍历每个地址
    while (i < last) {
        // 如果存在通配符形式地址，并且当前地址未绑定时，直接跳过，注意，最后一个地址已被强制设置为bind。
        if (bind_wildcard && !addr[i].opt.bind) {
            // 记录有多少个不需要单独进行监听的地址
            i++;
            continue;
        }
        // 对于需要进行监听的地址，将其增加到listening队列中。
        ls = ngx_http_add_listening(cf, &addr[i]);
        if (ls == NULL) {
            return NGX_ERROR;
        }
        // 由于将不需要单独监听的地址进行了合并，因此需要将每个地址中查找server块的hash表也进行统一管理。
        hport = ngx_pcalloc(cf->pool, sizeof(ngx_http_port_t));
        if (hport == NULL) {
            return NGX_ERROR;
        }

        ls->servers = hport;
        // 地址合并数量，除最后一个通配符（如果有的化）地址以外，其他需要单独监听的应该都是1
        hport->naddrs = i + 1;
        // 根据协议进行分别处理
        switch (ls->sockaddr->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            if (ngx_http_add_addrs6(cf, hport, addr) != NGX_OK) {
                return NGX_ERROR;
            }
            break;
#endif
        default: /* AF_INET */
            if (ngx_http_add_addrs(cf, hport, addr) != NGX_OK) {
                return NGX_ERROR;
            }
            break;
        }
        // 处理完成，将地址向前移。这是为了方便最后通配符形式的地址，进行统一处理
        addr++;
        /* 处理完成，将总数量减一。这是由于对于需要单独监听的地址来说，记录合并数量的i并未增加，要保障遍历的正确性，需要对总数减一。*/
        last--;
    }

    return NGX_OK;
}
```

###### ngx_http_add_listening

```c
static ngx_listening_t *
ngx_http_add_listening(ngx_conf_t *cf, ngx_http_conf_addr_t *addr)
{
    ngx_listening_t           *ls;
    ngx_http_core_loc_conf_t  *clcf;
    ngx_http_core_srv_conf_t  *cscf;
    // 将需要监听的地址增加到cycle的listening中，并对ngx_listening_t结构的参数赋默认初值
    ls = ngx_create_listening(cf, addr->opt.sockaddr, addr->opt.socklen);
    if (ls == NULL) {
        return NULL;
    }

    ls->addr_ntop = 1;
    // 设置建立链接后处理函数，进一步解析见http的处理部分
    ls->handler = ngx_http_init_connection;

    cscf = addr->default_server;
    ls->pool_size = cscf->connection_pool_size;
    ls->post_accept_timeout = cscf->client_header_timeout;

    clcf = cscf->ctx->loc_conf[ngx_http_core_module.ctx_index];
    // 根据addr的opt对ls设置参数值。
    ls->logp = clcf->error_log;
    ls->log.data = &ls->addr_text;
    ls->log.handler = ngx_accept_log_error;

#if (NGX_WIN32)
    {
    ngx_iocp_conf_t  *iocpcf = NULL;

    if (ngx_get_conf(cf->cycle->conf_ctx, ngx_events_module)) {
        iocpcf = ngx_event_get_conf(cf->cycle->conf_ctx, ngx_iocp_module);
    }
    if (iocpcf && iocpcf->acceptex_read) {
        ls->post_accept_buffer_size = cscf->client_header_buffer_size;
    }
    }
#endif

    ls->backlog = addr->opt.backlog;
    ls->rcvbuf = addr->opt.rcvbuf;
    ls->sndbuf = addr->opt.sndbuf;

    ls->keepalive = addr->opt.so_keepalive;
#if (NGX_HAVE_KEEPALIVE_TUNABLE)
    ls->keepidle = addr->opt.tcp_keepidle;
    ls->keepintvl = addr->opt.tcp_keepintvl;
    ls->keepcnt = addr->opt.tcp_keepcnt;
#endif

#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
    ls->accept_filter = addr->opt.accept_filter;
#endif

#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)
    ls->deferred_accept = addr->opt.deferred_accept;
#endif

#if (NGX_HAVE_INET6)
    ls->ipv6only = addr->opt.ipv6only;
#endif

#if (NGX_HAVE_SETFIB)
    ls->setfib = addr->opt.setfib;
#endif

#if (NGX_HAVE_TCP_FASTOPEN)
    ls->fastopen = addr->opt.fastopen;
#endif

#if (NGX_HAVE_REUSEPORT)
    ls->reuseport = addr->opt.reuseport;
#endif

    return ls;
}
```

###### ngx_http_add_addrs

ipv4和ipv6对应的处理类似，这里以ipv4举例：

```c
static ngx_int_t
ngx_http_add_addrs(ngx_conf_t *cf, ngx_http_port_t *hport,
    ngx_http_conf_addr_t *addr)
{
    ngx_uint_t                 i;
    ngx_http_in_addr_t        *addrs;
    struct sockaddr_in        *sin;
    ngx_http_virtual_names_t  *vn;
    // 根据需要进行地址合并监听的数量，来分配ngx_http_in_addr_t的数量
    hport->addrs = ngx_pcalloc(cf->pool,
                               hport->naddrs * sizeof(ngx_http_in_addr_t));
    if (hport->addrs == NULL) {
        return NGX_ERROR;
    }

    addrs = hport->addrs;
    // 遍历addr（即ip+port对应的地址结构ngx_http_conf_addr_t）对addrs赋值。
    for (i = 0; i < hport->naddrs; i++) {

        sin = (struct sockaddr_in *) addr[i].opt.sockaddr;
        addrs[i].addr = sin->sin_addr.s_addr;
        /* 设置默认server块。这里如果有两个监听同样的ip+port地址的server块，但都没有设置server_name，则第二个的服务永远不会被访问到 */
        addrs[i].conf.default_server = addr[i].default_server;
#if (NGX_HTTP_SSL)
        addrs[i].conf.ssl = addr[i].opt.ssl;
#endif
#if (NGX_HTTP_V2)
        addrs[i].conf.http2 = addr[i].opt.http2;
#endif
        addrs[i].conf.proxy_protocol = addr[i].opt.proxy_protocol;
        /* 如果ip+port地址下的server name未存在hash（三种）并且无正则表达式，则直接跳过。即此时所有server块，均无明确server name。 这时只能通过default_server进行服务服务，因此其他服务将无法访问到*/
        if (addr[i].hash.buckets == NULL
            && (addr[i].wc_head == NULL
                || addr[i].wc_head->hash.buckets == NULL)
            && (addr[i].wc_tail == NULL
                || addr[i].wc_tail->hash.buckets == NULL)
#if (NGX_PCRE)
            && addr[i].nregex == 0
#endif
            )
        {
            continue;
        }
        // 如果上述有一个存在，则会构建一个ngx_http_virtual_names_t进行存储
        vn = ngx_palloc(cf->pool, sizeof(ngx_http_virtual_names_t));
        if (vn == NULL) {
            return NGX_ERROR;
        }

        addrs[i].conf.virtual_names = vn;
        // 存储hash表
        vn->names.hash = addr[i].hash;
        vn->names.wc_head = addr[i].wc_head;
        vn->names.wc_tail = addr[i].wc_tail;
#if (NGX_PCRE)
        // 存储正则表达式。
        vn->nregex = addr[i].nregex;
        vn->regex = addr[i].regex;
#endif
    }

    return NGX_OK;
}
```





### location层set函数

在server中，接着执行解析函数时，当解析到location配置时，首先会遇到ngx_http_core_module核心模块的commands中的location。如下：

```c
{ ngx_string("location"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_BLOCK|NGX_CONF_TAKE12,
      ngx_http_core_location,
      NGX_HTTP_SRV_CONF_OFFSET,
      0,
      NULL },
```

其函数如下：

```c
static char *
ngx_http_core_location(ngx_conf_t *cf, ngx_command_t *cmd, void *dummy)
{
    char                      *rv;
    u_char                    *mod;
    size_t                     len;
    ngx_str_t                 *value, *name;
    ngx_uint_t                 i;
    ngx_conf_t                 save;
    ngx_http_module_t         *module;
    ngx_http_conf_ctx_t       *ctx, *pctx;
    ngx_http_core_loc_conf_t  *clcf, *pclcf;

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }
    // 这里的ctx为server层创建的ngx_http_conf_ctx_t。由于loc层不会有main和server层的配置，因此这两部分指向server层创建的空间即可
    pctx = cf->ctx;
    ctx->main_conf = pctx->main_conf;
    ctx->srv_conf = pctx->srv_conf;
    // 创建loc_conf存储空间
    ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->loc_conf == NULL) {
        return NGX_CONF_ERROR;
    }

    for (i = 0; cf->cycle->modules[i]; i++) {
        if (cf->cycle->modules[i]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[i]->ctx;

        if (module->create_loc_conf) {
            ctx->loc_conf[cf->cycle->modules[i]->ctx_index] =
                                                   module->create_loc_conf(cf);
            if (ctx->loc_conf[cf->cycle->modules[i]->ctx_index] == NULL) {
                return NGX_CONF_ERROR;
            }
        }
    }
    // 获取loc层ngx_http_core_module创建的ngx_http_core_loc_conf_s
    clcf = ctx->loc_conf[ngx_http_core_module.ctx_index];
    // loc层的ngx_http_core_loc_conf_s的loc_conf指向loc层创建的loc_conf
    clcf->loc_conf = ctx->loc_conf;
    // 获取词法解析后的数量。这里value只会是2、3，由于location中的commands中的NGX_CONF_TAKE12限制。正常来说是3，详见location配置。2是由于中间的参数和后面的参数中间未添加空格导致。这里分别进行处理。
    value = cf->args->elts;
    // 对解析参数进行处理，来对clcf赋值。只有~和~*时需要进行正则匹配，对于正则匹配的介绍，后续进行
    if (cf->args->nelts == 3) {

        len = value[1].len;
        mod = value[1].data;
        name = &value[2];

        if (len == 1 && mod[0] == '=') {

            clcf->name = *name;
            clcf->exact_match = 1;

        } else if (len == 2 && mod[0] == '^' && mod[1] == '~') {

            clcf->name = *name;
            clcf->noregex = 1;

        } else if (len == 1 && mod[0] == '~') {

            if (ngx_http_core_regex_location(cf, clcf, name, 0) != NGX_OK) {
                return NGX_CONF_ERROR;
            }

        } else if (len == 2 && mod[0] == '~' && mod[1] == '*') {

            if (ngx_http_core_regex_location(cf, clcf, name, 1) != NGX_OK) {
                return NGX_CONF_ERROR;
            }

        } else {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid location modifier \"%V\"", &value[1]);
            return NGX_CONF_ERROR;
        }

    } else {

        name = &value[1];

        if (name->data[0] == '=') {

            clcf->name.len = name->len - 1;
            clcf->name.data = name->data + 1;
            clcf->exact_match = 1;

        } else if (name->data[0] == '^' && name->data[1] == '~') {

            clcf->name.len = name->len - 2;
            clcf->name.data = name->data + 2;
            clcf->noregex = 1;

        } else if (name->data[0] == '~') {

            name->len--;
            name->data++;

            if (name->data[0] == '*') {

                name->len--;
                name->data++;

                if (ngx_http_core_regex_location(cf, clcf, name, 1) != NGX_OK) {
                    return NGX_CONF_ERROR;
                }

            } else {
                if (ngx_http_core_regex_location(cf, clcf, name, 0) != NGX_OK) {
                    return NGX_CONF_ERROR;
                }
            }

        } else {

            clcf->name = *name;

            if (name->data[0] == '@') {
                clcf->named = 1;
            }
        }
    }
    // 获取上一层（正常来说是server层，如果是location中嵌套location，则是当前location的上一层location）ngx_http_core_module创建的ngx_http_core_loc_conf_s
    pclcf = pctx->loc_conf[ngx_http_core_module.ctx_index];
    // 对于location块中嵌套location的处理
    if (cf->cmd_type == NGX_HTTP_LOC_CONF) {

        /* nested location */

#if 0
        clcf->prev_location = pclcf;
#endif

        if (pclcf->exact_match) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "location \"%V\" cannot be inside "
                               "the exact location \"%V\"",
                               &clcf->name, &pclcf->name);
            return NGX_CONF_ERROR;
        }

        if (pclcf->named) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "location \"%V\" cannot be inside "
                               "the named location \"%V\"",
                               &clcf->name, &pclcf->name);
            return NGX_CONF_ERROR;
        }

        if (clcf->named) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "named location \"%V\" can be "
                               "on the server level only",
                               &clcf->name);
            return NGX_CONF_ERROR;
        }

        len = pclcf->name.len;

#if (NGX_PCRE)
        if (clcf->regex == NULL
            && ngx_filename_cmp(clcf->name.data, pclcf->name.data, len) != 0)
#else
        if (ngx_filename_cmp(clcf->name.data, pclcf->name.data, len) != 0)
#endif
        {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "location \"%V\" is outside location \"%V\"",
                               &clcf->name, &pclcf->name);
            return NGX_CONF_ERROR;
        }
    }
    // 在上一层（正常来说是server层，如果是location中嵌套location，则是当前location的上一层location）ngx_http_core_module创建的ngx_http_core_loc_conf_s中的location变量里增加loc层创建的ngx_http_core_loc_conf_s，以此将server层和location层的解析关系进行连接。这里建立的是ngx_http_location_queue_t结构，其中存储了当前loc下的配置信息。
    if (ngx_http_add_location(cf, &pclcf->locations, clcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
    // 继续解析location层的配置
    save = *cf;
    cf->ctx = ctx;
    cf->cmd_type = NGX_HTTP_LOC_CONF;

    rv = ngx_conf_parse(cf, NULL);

    *cf = save;

    return rv;
}
```

#### loc级配置解析内存结构

[![RI50gJ.png](https://z3.ax1x.com/2021/07/06/RI50gJ.png)](https://imgtu.com/i/RI50gJ)

<center/>图10-5</center>
解析每一个location块时都会创建一个新的 ngx_http_conf_ctx_t结构体，其中的main_conf、srv_conf将指向http块下main_conf、srv_conf指针数组，loc_conf数组则都会重新分配，内容就是所有HTTP模块的create_loc_conf方法创建的结构体指针。

server层的存储结构和location层存储关联方式为：将location层ngx_http_core_module模块生成的ngx_http_core_loc_conf_s添加到server层ngx_http_core_module块生成的ngx_http_core_loc_conf_s的locations中，ngx_http_core_loc_conf_s的locations为一个队列，其元素为ngx_http_location_queue_t。定义如下：

```c
typedef struct {
    // queue为双向队列容器，用于将ngx_http_location_queue_t链接起来
    ngx_queue_t                      queue;
    // 如果location中的字符串可以精确匹配（包括正则表达式），exact指向对应的ngx_http_core_loc_conf_t结构体。否则为null
    ngx_http_core_loc_conf_t        *exact;
    // 如果location中的字符串无法精确匹配（包括了自定义的通配符），inclusive指向对应的ngx_http_core_loc_conf_t结构体。否则为null
    ngx_http_core_loc_conf_t        *inclusive;
    // 指向location名称
    ngx_str_t                       *name;
    u_char                          *file_name;
    ngx_uint_t                       line;
    ngx_queue_t                      list;
} ngx_http_location_queue_t;

```

使用ngx_http_location_queue_t将server层配置与location层配置相关联，server层的ngx_http_core_loc_conf_s的locations存储了其下所有location块生成的ngx_http_location_queue_t。最终解析后结构可能如下：

[![R7D2DO.png](https://z3.ax1x.com/2021/07/06/R7D2DO.png)](https://imgtu.com/i/R7D2DO)

location本身是可以进行嵌套的，嵌套后结构也是可以按照上面的方式进行扩招，即上一层的ngx_http_core_loc_conf_s中的locations存储下一层的所有location解析。例如：

```
http {
    mytest_num 1;
    server {
        server_name A;
        listen 8000;
        listen 80;
        mytest_num 2;
        location /L1 {
            mytest_num 3;
            ...
            location /L1/CL1 {
       }
    }
}
```

其解析结果为：

[![R7DzPs.png](https://z3.ax1x.com/2021/07/06/R7DzPs.png)](https://imgtu.com/i/R7DzPs)



向locations中增加ngx_http_core_loc_conf_t的代码如下：

```c
ngx_int_t
ngx_http_add_location(ngx_conf_t *cf, ngx_queue_t **locations,
    ngx_http_core_loc_conf_t *clcf)
{
    ngx_http_location_queue_t  *lq;

    if (*locations == NULL) {
        *locations = ngx_palloc(cf->temp_pool,
                                sizeof(ngx_http_location_queue_t));
        if (*locations == NULL) {
            return NGX_ERROR;
        }

        ngx_queue_init(*locations);
    }

    lq = ngx_palloc(cf->temp_pool, sizeof(ngx_http_location_queue_t));
    if (lq == NULL) {
        return NGX_ERROR;
    }

    if (clcf->exact_match
#if (NGX_PCRE)
        || clcf->regex
#endif
        || clcf->named || clcf->noname)
    {
        lq->exact = clcf;
        lq->inclusive = NULL;

    } else {
      // location ^~ 会使用包含匹配
        lq->exact = NULL;
        lq->inclusive = clcf;
    }

    lq->name = &clcf->name;
    lq->file_name = cf->conf_file->file.name.data;
    lq->line = cf->conf_file->line;

    ngx_queue_init(&lq->list);

    ngx_queue_insert_tail(*locations, &lq->queue);

    return NGX_OK;
}
```

从上述代码可以看出来，locations并非直接存储的ngx_http_location_queue_t。而是一个ngx_queue_t，即nginx自己实现的双向队列。这里存储的是ngx_http_location_queue_t结构中的queue队列，可以通过ngx_queue_t支持的操作，获取到对应的源数据。

#### 正则表达式解析





## 配置项合并

合并配置项，主要是合并main层的srv_conf与server层的srv_conf合并，以及main层的loc_conf与server层的loc_conf和location层的loc_conf三者之间的合并。即如下图：

[![Rq8iGT.png](https://z3.ax1x.com/2021/07/07/Rq8iGT.png)](https://imgtu.com/i/Rq8iGT)

在http层解析完成配置后，就会进行配置项的合并：

```c
    // 获取http级别ngx_http_core_module创建的http_ngx_core_main_conf_t
    cmcf = ctx->main_conf[ngx_http_core_module.ctx_index];
    cscfp = cmcf->servers.elts;

    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;
        mi = cf->cycle->modules[m]->ctx_index;

        /* init http{} main_conf's */
        // 执行每个模块的init_main_conf方法，解析后，对main级别的main_conf进行进一步处理
        if (module->init_main_conf) {
            rv = module->init_main_conf(cf, ctx->main_conf[mi]);
            if (rv != NGX_CONF_OK) {
                goto failed;
            }
        }
        // 合并配置项

        rv = ngx_http_merge_servers(cf, cmcf, module, mi);
        if (rv != NGX_CONF_OK) {
            goto failed;
        }
    }
```

### ngx_http_merge_servers

```c
static char *
ngx_http_merge_servers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf,
    ngx_http_module_t *module, ngx_uint_t ctx_index)
{
    char                        *rv;
    ngx_uint_t                   s;
    ngx_http_conf_ctx_t         *ctx, saved;
    ngx_http_core_loc_conf_t    *clcf;
    ngx_http_core_srv_conf_t   **cscfp;
    // 获取main级别ngx_http_core_module创建的http_ngx_core_main_conf_t的servers
    cscfp = cmcf->servers.elts;
    ctx = (ngx_http_conf_ctx_t *) cf->ctx;
    saved = *ctx;
    rv = NGX_CONF_OK;

    for (s = 0; s < cmcf->servers.nelts; s++) {

        /* merge the server{}s' srv_conf's */
        // 使ctx->srv_conf变更为server层创建的ngx_http_conf_ctx_t的srv_conf
        ctx->srv_conf = cscfp[s]->ctx->srv_conf;
        
        if (module->merge_srv_conf) {
            // 调用模块的merge_srv_conf（如果有），合并main级别和server级别的srv_conf。参数中，前者为main级别下创建的srv_conf，后者为server级别创建的srv_conf，merger后，对后者进行变更（即server层）
            rv = module->merge_srv_conf(cf, saved.srv_conf[ctx_index],
                                        cscfp[s]->ctx->srv_conf[ctx_index]);
            if (rv != NGX_CONF_OK) {
                goto failed;
            }
        }
        
        if (module->merge_loc_conf) {

            /* merge the server{}'s loc_conf */
            // 使ctx->loc_conf变更为server层创建的ngx_http_conf_ctx_t的loc_conf
            ctx->loc_conf = cscfp[s]->ctx->loc_conf;
            // 调用模块的merge_loc_conf（如果有），合并main级别和server级别的loc_conf。参数中，前者为main级别下创建的loc_conf，后者为server级别创建的loc_conf，merger后，对后者进行变更（即server层）
            rv = module->merge_loc_conf(cf, saved.loc_conf[ctx_index],
                                        cscfp[s]->ctx->loc_conf[ctx_index]);
            if (rv != NGX_CONF_OK) {
                goto failed;
            }

            /* merge the locations{}' loc_conf's */
            // 获取server级别ngx_http_core_module创建的ngx_http_conf_ctx_t的loc_conf
            clcf = cscfp[s]->ctx->loc_conf[ngx_http_core_module.ctx_index];
            // 调用ngx_http_merge_locations模块，merge server层（已经和main层合并过）的loc_conf和location层的loc_conf，参数中，前者为server下的location块列表，后者为server层的loc_conf
            rv = ngx_http_merge_locations(cf, clcf->locations,
                                          cscfp[s]->ctx->loc_conf,
                                          module, ctx_index);
            if (rv != NGX_CONF_OK) {
                goto failed;
            }
        }
    }

failed:

    *ctx = saved;

    return rv;
}
```



### ngx_http_merge_locations

```c
static char *
ngx_http_merge_locations(ngx_conf_t *cf, ngx_queue_t *locations,
    void **loc_conf, ngx_http_module_t *module, ngx_uint_t ctx_index)
{
    char                       *rv;
    ngx_queue_t                *q;
    ngx_http_conf_ctx_t        *ctx, saved;
    ngx_http_core_loc_conf_t   *clcf;
    ngx_http_location_queue_t  *lq;

    if (locations == NULL) {
        return NGX_CONF_OK;
    }
    // 获取上一层（无嵌套location时，为server层，在嵌套location时，为上一层location）的ngx_http_conf_ctx_t
    ctx = (ngx_http_conf_ctx_t *) cf->ctx;
    saved = *ctx;
    // 循环server块下，每一个location块，将每一个location块中的ctx_index对应模块与server层的ctx_index进行合并
    for (q = ngx_queue_head(locations);
         q != ngx_queue_sentinel(locations);
         q = ngx_queue_next(q))
    {
        // 由于ngx_queue_t是ngx_http_location_queue_t的第一个元素，因此可以直接转换。
        lq = (ngx_http_location_queue_t *) q;
        // 获取location块生成的ngx_http_core_loc_conf_t
        clcf = lq->exact ? lq->exact : lq->inclusive;
        // 将ctx->loc_conf变更为下一层（无嵌套location时，为空，有嵌套location时，为为一层location）的ngx_http_conf_ctx_t中的loc_conf
        ctx->loc_conf = clcf->loc_conf;
        // 调用模块的merge_loc_conf（如果有），合并server级别和location级别的loc_conf。参数中，前者为server级别下创建的loc_conf，后者为location级别创建的loc_conf，merger后，对后者进行变更（即location层）
        rv = module->merge_loc_conf(cf, loc_conf[ctx_index],
                                    clcf->loc_conf[ctx_index]);
        if (rv != NGX_CONF_OK) {
            return rv;
        }
        // 对于存在嵌套的location，循环调用ngx_http_merge_locations
        rv = ngx_http_merge_locations(cf, clcf->locations, clcf->loc_conf,
                                      module, ctx_index);
        if (rv != NGX_CONF_OK) {
            return rv;
        }
    }

    *ctx = saved;

    return NGX_CONF_OK;
}
```



## 创建location配置树

当请求到达时，需要先找到对应的server模块，再根据请求的uri找到对应的location配置块。location存储在server层级创建的ngx_http_core_loc_conf_s的locations，使用双向链表存储，这时查找效率太慢，因此为了加速查询，构建一个快速查询的树结构。其代码在http层set函数ngx_http_block。

```c
    /* create location trees */
    // cmcf为http层创建的ngx_http_core_main_conf_t
    for (s = 0; s < cmcf->servers.nelts; s++) {
        // cscfp为http层创建的ngx_http_core_main_conf_t的servers，即server层创建的ngx_http_core_srv_conf_t列表
        // clcf为server层创建的ngx_http_core_loc_conf_t
        clcf = cscfp[s]->ctx->loc_conf[ngx_http_core_module.ctx_index];
        /* 按照类型排序location，排序完后的队列：  (exact_match 或 inclusive) (排序好的，如果某个exact_match名字和inclusive location相同，exact_match排在前面)
       |  regex（未排序）| named(排序好的)  |  noname（未排序）*/
        if (ngx_http_init_locations(cf, cscfp[s], clcf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
        // 构建location的静态树
        if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }

```

locations排序和分隔ngx_http_init_locations：

```c
static ngx_int_t
ngx_http_init_locations(ngx_conf_t *cf, ngx_http_core_srv_conf_t *cscf,
    ngx_http_core_loc_conf_t *pclcf)
{
    ngx_uint_t                   n;
    ngx_queue_t                 *q, *locations, *named, tail;
    ngx_http_core_loc_conf_t    *clcf;
    ngx_http_location_queue_t   *lq;
    ngx_http_core_loc_conf_t   **clcfp;
#if (NGX_PCRE)
    ngx_uint_t                   r;
    ngx_queue_t                 *regex;
#endif
    // 获取server层下包含所有location
    locations = pclcf->locations;

    if (locations == NULL) {
        return NGX_OK;
    }
    // 使用ngx_http_cmp_locations对location进行排序，排序结果为(exact_match 或 inclusive) (排序好的，如果某个exact_match名字和inclusive location相同，exact_match排在前面)|  regex（未排序）| named(排序好的)  |  noname（未排序）
    ngx_queue_sort(locations, ngx_http_cmp_locations);

    named = NULL;
    n = 0;
#if (NGX_PCRE)
    regex = NULL;
    r = 0;
#endif
    // 获取regex类型的location数量和与named的分隔位置，获取named类型的location数量和其与noname的分隔位置
    for (q = ngx_queue_head(locations);
         q != ngx_queue_sentinel(locations);
         q = ngx_queue_next(q))
    {
        lq = (ngx_http_location_queue_t *) q;

        clcf = lq->exact ? lq->exact : lq->inclusive;
        // 递归的对location内包含的locations进行排序和切割
        if (ngx_http_init_locations(cf, NULL, clcf) != NGX_OK) {
            return NGX_ERROR;
        }

#if (NGX_PCRE)

        if (clcf->regex) {
            r++;

            if (regex == NULL) {
                regex = q;
            }

            continue;
        }

#endif

        if (clcf->named) {
            n++;

            if (named == NULL) {
                named = q;
            }

            continue;
        }
        // 对于noname类型的location，找到第一个，直接结束，if和limit_except
        if (clcf->noname) {
            break;
        }
    }
    // 如果未查找到双向队列的末尾，说明含有noname，直接将noname的数据切割掉。
    if (q != ngx_queue_sentinel(locations)) {
        ngx_queue_split(locations, q, &tail);
    }
    // 如果找到了named的location（@），则将named的部分切割出来，存储到server层创建的ngx_http_core_srv_conf_t的named_locations中
    if (named) {
        clcfp = ngx_palloc(cf->pool,
                           (n + 1) * sizeof(ngx_http_core_loc_conf_t *));
        if (clcfp == NULL) {
            return NGX_ERROR;
        }

        cscf->named_locations = clcfp;

        for (q = named;
             q != ngx_queue_sentinel(locations);
             q = ngx_queue_next(q))
        {
            lq = (ngx_http_location_queue_t *) q;

            *(clcfp++) = lq->exact;
        }

        *clcfp = NULL;

        ngx_queue_split(locations, named, &tail);
    }

#if (NGX_PCRE)
    // 如果存在正则匹配的location，则将正则匹配的location切割出来，放到server层创建的ngx_http_core_loc_conf_t的regex_locations中
    if (regex) {

        clcfp = ngx_palloc(cf->pool,
                           (r + 1) * sizeof(ngx_http_core_loc_conf_t *));
        if (clcfp == NULL) {
            return NGX_ERROR;
        }

        pclcf->regex_locations = clcfp;

        for (q = regex;
             q != ngx_queue_sentinel(locations);
             q = ngx_queue_next(q))
        {
            lq = (ngx_http_location_queue_t *) q;

            *(clcfp++) = lq->exact;
        }

        *clcfp = NULL;

        ngx_queue_split(locations, regex, &tail);
    }

#endif
    // 至此server层创建的ngx_http_core_loc_conf_t的locations中只有排序好的exact_match 或 inclusive类型的location
    return NGX_OK;
}
```

对exact_match 或 inclusive类型的location构建静态树：

```c
static ngx_int_t
ngx_http_init_static_location_trees(ngx_conf_t *cf,
    ngx_http_core_loc_conf_t *pclcf)
{
    ngx_queue_t                *q, *locations;
    ngx_http_core_loc_conf_t   *clcf;
    ngx_http_location_queue_t  *lq;

    locations = pclcf->locations;

    if (locations == NULL) {
        return NGX_OK;
    }

    if (ngx_queue_empty(locations)) {
        return NGX_OK;
    }
    // 递归的对location内包含location的情况进行处理。
    for (q = ngx_queue_head(locations);
         q != ngx_queue_sentinel(locations);
         q = ngx_queue_next(q))
    {
        lq = (ngx_http_location_queue_t *) q;

        clcf = lq->exact ? lq->exact : lq->inclusive;

        if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
            return NGX_ERROR;
        }
    }
    /* 该函数把同一层次里的名称相同的不同 location 合并在一起。
     * "名称相同，但又是不同的 location"?
     * 这主要是因为 location 有绝对匹配（对应 exact 字段）和包含匹配（也就是前缀
     * 匹配，以指定前缀开头的都匹配上，所以是包含匹配，对应 inclusive 字段）。若
     * 这两个 location 的名称相同（如都为 "/"），则可以把它们共用在一个队列节点里 */
    if (ngx_http_join_exact_locations(cf, locations) != NGX_OK) {
        return NGX_ERROR;
    }

    /* 递归每个location节点，得到当前节点的名字为其前缀的location的列表，保存在当前节点的list字段下 这里是将前缀匹配的location进行和其他的location合并，构建一个前缀匹配的list*/
    ngx_http_create_locations_list(locations, ngx_queue_head(locations));

    /* 递归建立location三叉排序树 */
    pclcf->static_locations = ngx_http_create_locations_tree(cf, locations, 0);
    if (pclcf->static_locations == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

融合exact和inclusive类型的location代码：

```c
static ngx_int_t
ngx_http_join_exact_locations(ngx_conf_t *cf, ngx_queue_t *locations)
{
    ngx_queue_t                *q, *x;
    ngx_http_location_queue_t  *lq, *lx;

    q = ngx_queue_head(locations);

    while (q != ngx_queue_last(locations)) {

        x = ngx_queue_next(q);

        lq = (ngx_http_location_queue_t *) q;
        lx = (ngx_http_location_queue_t *) x;
        // 前后两个一致，则将后面的inclusive赋值到前一个inclusive，将后一个删除
        if (lq->name->len == lx->name->len
            && ngx_filename_cmp(lq->name->data, lx->name->data, lx->name->len)
               == 0)
        {
            if ((lq->exact && lx->exact) || (lq->inclusive && lx->inclusive)) {
                ngx_log_error(NGX_LOG_EMERG, cf->log, 0,
                              "duplicate location \"%V\" in %s:%ui",
                              lx->name, lx->file_name, lx->line);

                return NGX_ERROR;
            }

            lq->inclusive = lx->inclusive;

            ngx_queue_remove(x);

            continue;
        }

        q = ngx_queue_next(q);
    }

    return NGX_OK;
}
```

构架前缀列表ngx_http_create_locations_list：

```c
static void
ngx_http_create_locations_list(ngx_queue_t *locations, ngx_queue_t *q)
{
    u_char                     *name;
    size_t                      len;
    ngx_queue_t                *x, tail;
    ngx_http_location_queue_t  *lq, *lx;
    // 如果当前的location是locations列表中最后一个，直接结束
    if (q == ngx_queue_last(locations)) {
        return;
    }

    lq = (ngx_http_location_queue_t *) q;
    // 如果当前的location不包含前缀类型的location，则接着向后处理
    if (lq->inclusive == NULL) {
        ngx_http_create_locations_list(locations, ngx_queue_next(q));
        return;
    }
    // 如果当前的location包含前缀类型的location，则构建前缀列表
    len = lq->name->len;
    name = lq->name->data;
    // 找到以当前的location的name作为前缀的所有location
    for (x = ngx_queue_next(q);
         x != ngx_queue_sentinel(locations);
         x = ngx_queue_next(x))
    {
        lx = (ngx_http_location_queue_t *) x;

        if (len > lx->name->len
            || ngx_filename_cmp(name, lx->name->data, len) != 0)
        {
            break;
        }
    }

    q = ngx_queue_next(q);
    // 如果没有以当前的location的name作为前缀的所有location，则接着处理后面的列表。
    if (q == x) {
        ngx_http_create_locations_list(locations, x);
        return;
    }

    // 以当前的location将locations切割成两部分
    ngx_queue_split(locations, q, &tail);
    // 将后半部分location列表添加到当前location对应的ngx_http_location_queue_t成员list上
    ngx_queue_add(&lq->list, &tail);
    // 如果后半部分全部是能够以当前的location的name作为前缀的location，则直接递归处理该后半部分的location即可
    if (x == ngx_queue_sentinel(locations)) {
        ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list));
        return;
    }

    // 如果后半部分只有部分是能够以当前的location的name作为前缀的location，将剩余的location重新拼接到locations列表中
    ngx_queue_split(&lq->list, x, &tail);
    ngx_queue_add(locations, &tail);

    // 递归处理能够以当前的location的name作为前缀的location列表
    ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list));
    // 接着处理之后的locations
    ngx_http_create_locations_list(locations, x);
}
```

例如，对于如下的location配置：

```
location ^~ /abc
location = /abc
location = /bdf
location = /bdf/hl
location ^~ /efg
location ^~ /efghjk
location ^~ /efghjo
location = /efghjk/npq
location = /fgk
```

则经过ngx_http_create_locations_list后，locations结构如下：

```
/abc ---> /bef ---> /bef/hl ---> /efg ---> /fgk
                                  |
                                 \|/
                                 /efghjk ---> /efghjo
                                  |
                                 \|/ 
                                 /efghjk/npq
```

构建静态树ngx_http_create_locations_tree：

```c
static ngx_http_location_tree_node_t *
ngx_http_create_locations_tree(ngx_conf_t *cf, ngx_queue_t *locations,
    size_t prefix)
{
    size_t                          len;
    ngx_queue_t                    *q, tail;
    ngx_http_location_queue_t      *lq;
    ngx_http_location_tree_node_t  *node;
    // 获取locations中间的location
    q = ngx_queue_middle(locations);

    lq = (ngx_http_location_queue_t *) q;
    len = lq->name->len - prefix;

    node = ngx_palloc(cf->pool,
                      offsetof(ngx_http_location_tree_node_t, name) + len);
    if (node == NULL) {
        return NULL;
    }

    node->left = NULL;
    node->right = NULL;
    node->tree = NULL;
    node->exact = lq->exact;
    node->inclusive = lq->inclusive;

    node->auto_redirect = (u_char) ((lq->exact && lq->exact->auto_redirect)
                           || (lq->inclusive && lq->inclusive->auto_redirect));

    node->len = (u_char) len;
    ngx_memcpy(node->name, &lq->name->data[prefix], len);
    // 从中间切割locations
    ngx_queue_split(locations, q, &tail);
    // 如果切割后locations为空，则将该location下的list（前缀列表），递归改操作。
    if (ngx_queue_empty(locations)) {
        /*
         * ngx_queue_split() insures that if left part is empty,
         * then right one is empty too
         */
        goto inclusive;
    }
    // 左子树执行构建
    node->left = ngx_http_create_locations_tree(cf, locations, prefix);
    if (node->left == NULL) {
        return NULL;
    }

    ngx_queue_remove(q);
    
    if (ngx_queue_empty(&tail)) {
        goto inclusive;
    }
    // 右子树执行构建
    node->right = ngx_http_create_locations_tree(cf, &tail, prefix);
    if (node->right == NULL) {
        return NULL;
    }

inclusive:

    if (ngx_queue_empty(&lq->list)) {
        return node;
    }

    node->tree = ngx_http_create_locations_tree(cf, &lq->list, prefix + len);
    if (node->tree == NULL) {
        return NULL;
    }

    return node;
}
```

构建出的ngx_http_location_tree_node_t结构为：

```c
struct ngx_http_location_tree_node_s {
    ngx_http_location_tree_node_t   *left; // 左子树
    ngx_http_location_tree_node_t   *right; // 右子树
    ngx_http_location_tree_node_t   *tree; // 当前location对应的list(前缀匹配列表)

    ngx_http_core_loc_conf_t        *exact; // 如果location是精准匹配，则该值执行location层级下的ngx_http_core_loc_conf_t 
    ngx_http_core_loc_conf_t        *inclusive;// 如果location是前缀匹配，则该值执行location层级下的ngx_http_core_loc_conf_t

    u_char                           auto_redirect;
    // 计算节点所处相对于上层节点偏移长度，例如该节点是，1234，上层节点是123，则这里长度为1
    u_char                           len;
    // 这里只是有一个字符，但是通过C字符串通过末尾\0来确定实际name
    u_char                           name[1];
};
```



## http头初始化

```
if (ngx_http_init_headers_in_hash(cf, cmcf) != NGX_OK) {
    return NGX_CONF_ERROR;
}
```

对于每一个http头会对应到一个结构体：

```c
typedef struct {
    ngx_str_t                         name;
    ngx_uint_t                        offset;
    ngx_http_header_handler_pt        handler;
} ngx_http_header_t;
```

其中handler为请求中存在该头时的处理函数。

```c
static ngx_int_t
ngx_http_init_headers_in_hash(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
{
    ngx_array_t         headers_in;
    ngx_hash_key_t     *hk;
    ngx_hash_init_t     hash;
    ngx_http_header_t  *header;

    if (ngx_array_init(&headers_in, cf->temp_pool, 32, sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // nginx默认包含的一批http头部，并包含对其处理，即handler函数，如Host、Connection等
    for (header = ngx_http_headers_in; header->name.len; header++) {
        hk = ngx_array_push(&headers_in);
        if (hk == NULL) {
            return NGX_ERROR;
        }

        hk->key = header->name;
        hk->key_hash = ngx_hash_key_lc(header->name.data, header->name.len);
        hk->value = header;
    }
    // http的请求头构成的hash结构，存储于main级别创建的ngx_http_core_main_conf_t的headers_in_hash
    hash.hash = &cmcf->headers_in_hash;
    // 构建hash转换的函数
    hash.key = ngx_hash_key_lc;
    // 最大允许的桶数量
    hash.max_size = 512;
    // 每个桶最大包含的字节数
    hash.bucket_size = ngx_align(64, ngx_cacheline_size);
    // hash表名，用户日志记录
    hash.name = "headers_in_hash";
    hash.pool = cf->pool;
    hash.temp_pool = NULL;
    // 构建hash表，详见散列表
    if (ngx_hash_init(&hash, headers_in.elts, headers_in.nelts) != NGX_OK) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```



## 处理阶段

main级别创建的ngx_http_core_main_conf_t的phases字段管理http处理阶段：

```c
typedef struct {
    ....
    ngx_http_phase_t           phases[NGX_HTTP_LOG_PHASE + 1];
} ngx_http_core_main_conf_t;

typedef struct {
    ngx_array_t                handlers;
} ngx_http_phase_t;

typedef ngx_int_t (*ngx_http_handler_pt)(ngx_http_request_t *r);
```

每一个阶段对应一个函数列表，每个阶段执行时，依次执行。

存在如下阶段：

```c
typedef enum {
    // 收到完整的http头部后处理的阶段
    NGX_HTTP_POST_READ_PHASE = 0,

    // 在将请求的URI与location表达式匹配前，修改请求的URI（所谓重定向）是一个独立的http阶段
    NGX_HTTP_SERVER_REWRITE_PHASE,
    
    // 根据请求的URI寻找匹配的location表达式，该阶段只能由ngx_http_core_module模块实现，不建议其他HTTP模块重新定义该阶段
    NGX_HTTP_FIND_CONFIG_PHASE,
    // 在NGX_HTTP_FIND_CONFIG_PHASE阶段寻找到匹配的location之后再修改请求的URI
    NGX_HTTP_REWRITE_PHASE,
    // 该阶段是用于rewrite重新URI后，防止错误的配置导致死循环。控制方式是检查rewrite次数，超过10次就认为是有死循环，该阶段就返回500，表示服务器内部错误
    NGX_HTTP_POST_REWRITE_PHASE,

    // 处理NGX_HTTP_ACCESS_PHASE阶段决定访问权限前，http模块可以介入的阶段
    NGX_HTTP_PREACCESS_PHASE,

    // 让HTTP模块判断是否允许这个请求访问nginx服务
    NGX_HTTP_ACCESS_PHASE,
    // 在NGX_HTTP_ACCESS_PHASE阶段，如果http模块的handler处理函数返回不允许访问的错误码时，这里负责向用户发送拒绝服务的错误
    NGX_HTTP_POST_ACCESS_PHASE,

    // 
    NGX_HTTP_PRECONTENT_PHASE,

    // 用于处理HTTP请求内容的阶段，是大部分http模块介入的阶段
    NGX_HTTP_CONTENT_PHASE,
    // 处理完成后记录日志的阶段
    NGX_HTTP_LOG_PHASE
} ngx_http_phases;
```

有些阶段是必须的，有些阶段是可选的。上述11个阶段中NGX_HTTP_FIND_CONFIG_PHASE、NGX_HTTP_POST_REWRITE_PHASE、NGX_HTTP_POST_ACCESS_PHASE三个阶段不允许http模块加入增加的ngx_http_handler_pt方法处理用户请求，其仅由http框架实现。

### 初始化处理阶段

```c
    // 初始化处理阶段
    if (ngx_http_init_phases(cf, cmcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
    ...
    // 每个模块在postconfiguration中可以在每个阶段中增加处理
    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;

        if (module->postconfiguration) {
            if (module->postconfiguration(cf) != NGX_OK) {
                return NGX_CONF_ERROR;
            }
        }
    }
    ....
    *cf = pcf;


    if (ngx_http_init_phase_handlers(cf, cmcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
```

ngx_http_init_phases方法会初始化必须的处理阶段到main级别创建的ngx_http_core_main_conf_t中的phases中。其中phases是一个存储每个阶段处理函数的临时变量。处理方法如下：

```c
static ngx_int_t
ngx_http_init_phases(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
{
    // 阶段0
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_POST_READ_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 阶段1
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_SERVER_REWRITE_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 阶段3
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 阶段5
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 阶段6
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers,
                       cf->pool, 2, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 阶段8
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_PRECONTENT_PHASE].handlers,
                       cf->pool, 2, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // 阶段9
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers,
                       cf->pool, 4, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // 阶段10
    if (ngx_array_init(&cmcf->phases[NGX_HTTP_LOG_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

在http初始化过程中，任何HTTP模块都可以在postconfiguration中增加自定义的方法到phases中某个阶段的数组中。

例如：ngx_http_index_module模块的postconfiguration方法ngx_http_index_init会添加NGX_HTTP_CONTENT_PHASE阶段处理方法：

```c
static ngx_int_t
ngx_http_index_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_index_handler;

    return NGX_OK;
}
```

默认的NGX_HTTP_CONTENT_PHASE由ngx_http_static_module模块添加。

### 请求处理阶段

phases只是临时存储请求阶段，真正请求时，使用的是phase_engine字段。先看一下其结构：

```c
typedef struct {
    ...
    ngx_http_phase_engine_t    phase_engine;

    ...
} ngx_http_core_main_conf_t;

typedef struct {
    ngx_http_phase_handler_t  *handlers; // 由ngx_http_phase_handler_t构成的数组首地址，表示一个请求可能经历的所有ngx_http_phase_handler_t处理方法
    ngx_uint_t                 server_rewrite_index; // 表示NGX_HTTP_SERVER_REWRITE_PHASE阶段的第一个ngx_http_phase_handler_t处理方法在handlers中的下标。用于执行任意http请求时，快速跳转到NGX_HTTP_SERVER_REWRITE_PHASE处理阶段
    ngx_uint_t                 location_rewrite_index;// 表示NGX_HTTP_REWRITE_PHASE阶段的第一个ngx_http_phase_handler_t处理方法在handlers中的下标。用于执行任意http请求时，快速跳转到NGX_HTTP_REWRITE_PHASE处理阶段
} ngx_http_phase_engine_t;


typedef struct ngx_http_phase_handler_s  ngx_http_phase_handler_t;

// 一个http处理阶段的checker检查方法，仅由http框架实现，以控制http的处理流程
typedef ngx_int_t (*ngx_http_phase_handler_pt)(ngx_http_request_t *r,
    ngx_http_phase_handler_t *ph);

struct ngx_http_phase_handler_s {
    /* 在处理到某一个http阶段时，框架会首先在checker方法已实现的基础上先调用checker方法来处理请求，而不会调用任何阶段的handler方法。只有在checker方法中才会调用handler方法。所有checker方法均通过ngx_http_core_module模块实现，普通http模块无法重新定义 */
    ngx_http_phase_handler_pt  checker;
    /* 除ngx_http_core_module模块以外的模块，只能通过定义handler方法才能介入某一个http处理阶段的处理 */
    ngx_http_handler_pt        handler;
    /* next使得处理阶段可以不按照handlers定义的数组顺序执行，即可以向前跳跃数个阶段，执行之前执行过的方法，也可以向后跳跃数个阶段。通常，next表示下一个处理阶段中第一个ngx_http_phase_handler_s下标，next是用来跳转处理的过程，如何使用取决于每个阶段的处理逻辑，大部分时候还都是按照顺序，依次执行，并不是都使用next进行跳转 */
    ngx_uint_t                 next;
};
```

在经过初始化处理阶段后，就会执行ngx_http_init_phase_handlers方法来确定阶段执行顺序。

```c
static ngx_int_t
ngx_http_init_phase_handlers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
{
    ngx_int_t                   j;
    ngx_uint_t                  i, n;
    // find_config_index表示NGX_HTTP_FIND_CONFIG_PHASE阶段对应的在handlers中的索引。use_rewrite, use_access表示，是否使用了NGX_HTTP_REWRITE_PHASE阶段和NGX_HTTP_ACCESS_PHASE阶段（默认使用）
    ngx_uint_t                  find_config_index, use_rewrite, use_access;
    ngx_http_handler_pt        *h;
    ngx_http_phase_handler_t   *ph;
    ngx_http_phase_handler_pt   checker;

    cmcf->phase_engine.server_rewrite_index = (ngx_uint_t) -1;
    cmcf->phase_engine.location_rewrite_index = (ngx_uint_t) -1;
    find_config_index = 0;
    // 判断是否使用NGX_HTTP_REWRITE_PHASE阶段和NGX_HTTP_ACCESS_PHASE阶段
    use_rewrite = cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers.nelts ? 1 : 0;
    use_access = cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers.nelts ? 1 : 0;

    // NGX_HTTP_FIND_CONFIG_PHASE阶段不允许用户自己添加处理函数，因此不会在phases中存在，因此要自己分配空间
    // 当使用NGX_HTTP_REWRITE_PHASE阶段和NGX_HTTP_ACCESS_PHASE阶段时，就会相应的增加NGX_HTTP_POST_REWRITE_PHASE阶段和NGX_HTTP_POST_ACCESS_PHASE阶段，这两个阶段也是不允许用户自己添加处理函数，因此不会在phases中存在，因此要自己分配空间。
    n = 1                  /* find config phase */
        + use_rewrite      /* post rewrite phase */
        + use_access;      /* post access phase */
    // 查找除了上面三个阶段以外，其他阶段总共有多少个处理函数
    for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
        n += cmcf->phases[i].handlers.nelts;
    }
    // 分配空间
    ph = ngx_pcalloc(cf->pool,
                     n * sizeof(ngx_http_phase_handler_t) + sizeof(void *));
    if (ph == NULL) {
        return NGX_ERROR;
    }

    cmcf->phase_engine.handlers = ph;
    // n表示下一个要加入到phase_engine数组中元素的下标
    n = 0;
    // 遍历每个阶段进行处理
    for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
        h = cmcf->phases[i].handlers.elts;

        switch (i) {
        // NGX_HTTP_SERVER_REWRITE_PHASE阶段，n即当前阶段的第一个ngx_http_phase_handler_s，因此phase_engine.server_rewrite_index指向该值
        case NGX_HTTP_SERVER_REWRITE_PHASE:
            if (cmcf->phase_engine.server_rewrite_index == (ngx_uint_t) -1) {
                cmcf->phase_engine.server_rewrite_index = n;
            }
            checker = ngx_http_core_rewrite_phase;

            break;
        // NGX_HTTP_FIND_CONFIG_PHASE阶段，由于该阶段不允许用户自定义处理方法，因此直接指定checker方法，将本身添加到phase_engine队列中，忽略用户在phase中增加的处理函数，并使用find_config_index记录该阶段在phase_engine队列的下标，这里未设置next值，与checker函数本身有关，具体参加下一小节
        case NGX_HTTP_FIND_CONFIG_PHASE:
            find_config_index = n;

            ph->checker = ngx_http_core_find_config_phase;
            n++;
            ph++;

            continue;
        
        // NGX_HTTP_REWRITE_PHASE阶段，处理逻辑与NGX_HTTP_SERVER_REWRITE_PHASE类似
        case NGX_HTTP_REWRITE_PHASE:
            if (cmcf->phase_engine.location_rewrite_index == (ngx_uint_t) -1) {
                cmcf->phase_engine.location_rewrite_index = n;
            }
            checker = ngx_http_core_rewrite_phase;

            break;
        // NGX_HTTP_POST_REWRITE_PHASE阶段，如果用户使用了NGX_HTTP_REWRITE_PHASE阶段，则需要增加该阶段，阶段处理结束后，执行NGX_HTTP_FIND_CONFIG_PHASE阶段，因此next使用之前存储的find_config_index值，且该阶段不允许用户自定义处理函数
        case NGX_HTTP_POST_REWRITE_PHASE:
            if (use_rewrite) {
                ph->checker = ngx_http_core_post_rewrite_phase;
                ph->next = find_config_index;
                n++;
                ph++;
            }

            continue;
        
        /* NGX_HTTP_ACCESS_PHASE阶段，这里在未将该阶段加入phase_engine时，就对n进行增加是和其下一阶段NGX_HTTP_POST_ACCESS_PHASE有关。当存在该阶段时，如果校验权限为无权限时，执行NGX_HTTP_POST_ACCESS_PHASE阶段，这时只需要依次执行phase_engine数组即可，当校验有权限时，则有可能（取决于是要全部校验通过，还是任意一个条件满足）直接跳转到NGX_HTTP_POST_ACCESS_PHASE后面的一个阶段。由于NGX_HTTP_POST_ACCESS_PHASE只会有一个元素，因此这里增加1后，执行的就是NGX_HTTP_POST_ACCESS_PHASE阶段后一个阶段的第一个处理元素。但这里我感觉有一个bug，就是在没有该阶段的时候，即用户未在该阶段注册函数时，就会导致NGX_HTTP_POST_ACCESS_PHASE后面阶段的next指向混乱，不知道是我哪里没看明白还是怎么回事。不过当前默认是存在该模块的（将框架里面的模块会向该模块增加处理函数）*/
        case NGX_HTTP_ACCESS_PHASE:
            checker = ngx_http_core_access_phase;
            n++;
            break;
        
        // 只有当存在NGX_HTTP_ACCESS_PHASE阶段时才会增加该阶段。由于在NGX_HTTP_ACCESS_PHASE阶段已经将n指向了该阶段后的一个阶段的第一个处理元素，因此其next直接使用n即可（正常来说，直接就返回了，也不会有后续处理了）。该阶段不允许用户增加处理函数。
        case NGX_HTTP_POST_ACCESS_PHASE:
            if (use_access) {
                ph->checker = ngx_http_core_post_access_phase;
                ph->next = n;
                ph++;
            }

            continue;
        
        // NGX_HTTP_CONTENT_PHASE阶段，不允许用户增加自定义处理函数
        case NGX_HTTP_CONTENT_PHASE:
            checker = ngx_http_core_content_phase;
            break;
        // 默认处理模块
        default:
            checker = ngx_http_core_generic_phase;
        }
        // n增加当前模块的处理函数数量，这时n就指向了下一个模块的第一个处理元素了（NGX_HTTP_ACCESS_PHASE阶段除外，其在之前先让那个n自增了1）
        n += cmcf->phases[i].handlers.nelts;
        // 遍历当前阶段的handlers数组，增加phase_engine元素，每个元素的next都设置为下一个阶段的第一个处理函数。但这里赋值顺序和添加顺序是相反的，目前也不知道具体原因
        for (j = cmcf->phases[i].handlers.nelts - 1; j >= 0; j--) {
            ph->checker = checker;
            ph->handler = h[j];
            ph->next = n;
            ph++;
        }
    }

    return NGX_OK;
}
```



### 每个阶段的checker函数

## 核心模块收尾

在执行完子模块的解析后，会执行核心模块的ctx的init_conf方法。其执行顺序完成ngx_conf_parse函数后，其代码如下：

```c
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->init_conf) {
            if (module->init_conf(cycle,
                                  cycle->conf_ctx[cycle->modules[i]->index])
                == NGX_CONF_ERROR)
            {
                environ = senv;
                ngx_destroy_cycle_pools(&conf);
                return NULL;
            }
        }
    }
```

这里执行核心模块的init_conf，对核心模块所关系的内容，根据之前的解析，进行完整的赋值。

因此各个模块到此，其解析过程为：

1. 首先执行核心模块的ctx成员的create_conf方法。
2. 在每个核心模块中，执行其管理的模块的create_*_conf类（如create_conf,create_main_conf,preconfiguration）方法。
3. 在每个核心模块中，执行其管理的模块的ini_*_conf类(如init_conf，init_main_conf，merge_srv_conf，merge_loc_conf)方法。
4. 最后执行核心模块的ctx成员的create_conf方法。

# 事件模块

## 解析配置

在linux下，默认的事件模块包含如下三部分：

```c
ngx_module_t *ngx_modules[] = {
    ...
    &ngx_events_module, // 核心模块，管理所有事件模块
    &ngx_event_core_module, // 事件的核心模块
    &ngx_epoll_module, // epoll模块
    ...
}
```

### ngx_events_module模块

其内容如下：

```c
static ngx_command_t  ngx_events_commands[] = {

    { ngx_string("events"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_events_block,
      0,
      0,
      NULL },

      ngx_null_command
};


static ngx_core_module_t  ngx_events_module_ctx = {
    ngx_string("events"),
    NULL,
    ngx_event_init_conf
};


ngx_module_t  ngx_events_module = {
    NGX_MODULE_V1,
    &ngx_events_module_ctx,                /* module context */
    ngx_events_commands,                   /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

其不存在module的create_conf方法。只存在init_conf方法。

因此其执行顺序为：ngx_events_commands指定的配置解析->init_conf。这两步均在init_cycle函数中执行。

#### comands数组处理函数

##### envents：ngx_events_block

执行模块的commands时执行函数：

```c
static char *
ngx_events_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    char                 *rv;
    void               ***ctx;
    ngx_uint_t            i;
    ngx_conf_t            pcf;
    ngx_event_module_t   *m;
    // 如果配置不为空，说明已经被解析过，发送了重复配置
    if (*(void **) conf) {
        return "is duplicate";
    }

    /* count the number of the event modules and set up their indices */
    // 获取事件模块数量，并对事件模块的ctx_index赋值
    ngx_event_max_module = ngx_count_modules(cf->cycle, NGX_EVENT_MODULE);

    ctx = ngx_pcalloc(cf->pool, sizeof(void *));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    *ctx = ngx_pcalloc(cf->pool, ngx_event_max_module * sizeof(void *));
    if (*ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    *(void **) conf = ctx;
    // 执行每个事件模块的ctx的create_conf方法。
    for (i = 0; cf->cycle->modules[i]; i++) {
        if (cf->cycle->modules[i]->type != NGX_EVENT_MODULE) {
            continue;
        }

        m = cf->cycle->modules[i]->ctx;

        if (m->create_conf) {
            (*ctx)[cf->cycle->modules[i]->ctx_index] =
                                                     m->create_conf(cf->cycle);
            if ((*ctx)[cf->cycle->modules[i]->ctx_index] == NULL) {
                return NGX_CONF_ERROR;
            }
        }
    }
    // 变更cf，使其解析事件块配置
    pcf = *cf;
    cf->ctx = ctx;
    cf->module_type = NGX_EVENT_MODULE;
    cf->cmd_type = NGX_EVENT_CONF;

    rv = ngx_conf_parse(cf, NULL);

    *cf = pcf;

    if (rv != NGX_CONF_OK) {
        return rv;
    }
    // 对于每个事件模块，执行其ctx的init_conf方法。
    for (i = 0; cf->cycle->modules[i]; i++) {
        if (cf->cycle->modules[i]->type != NGX_EVENT_MODULE) {
            continue;
        }

        m = cf->cycle->modules[i]->ctx;

        if (m->init_conf) {
            rv = m->init_conf(cf->cycle,
                              (*ctx)[cf->cycle->modules[i]->ctx_index]);
            if (rv != NGX_CONF_OK) {
                return rv;
            }
        }
    }

    return NGX_CONF_OK;
}
```

#### ctx的init_conf:ngx_event_init_conf

在执行完成所有事件配置解析后，会调用核心事件模块ngx_envents_module的init_conf方法，其处理逻辑如下：

```c
static char *
ngx_event_init_conf(ngx_cycle_t *cycle, void *conf)
{
#if (NGX_HAVE_REUSEPORT)
    ngx_uint_t        i;
    ngx_listening_t  *ls;
#endif

    if (ngx_get_conf(cycle->conf_ctx, ngx_events_module) == NULL) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                      "no \"events\" section in configuration");
        return NGX_CONF_ERROR;
    }
    // connection_n为每个work进程允许同时处理的的最大连接数（下文会详细介绍），如果该值小于需要监听的套接字数量，则报错。
    if (cycle->connection_n < cycle->listening.nelts + 1) {

        /*
         * there should be at least one connection for each listening
         * socket, plus an additional connection for channel
         */

        ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                      "%ui worker_connections are not enough "
                      "for %ui listening sockets",
                      cycle->connection_n, cycle->listening.nelts);

        return NGX_CONF_ERROR;
    }

#if (NGX_HAVE_REUSEPORT)
    // 如果系统支持端口复用，则对配置了端口复用的套接字进行拷贝，为每个worker进程拷贝一份监听地址
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        // 如果未设置端口复用或者本身是复制的地址，则跳过
        if (!ls[i].reuseport || ls[i].worker != 0) {
            continue;
        }
        // 将ls的地址复制worker子进程减一份（自身存在一份），复制后，每个复制的结构中worker不为0，因此在下次循环时直接跳过
        if (ngx_clone_listening(cycle, &ls[i]) != NGX_OK) {
            return NGX_CONF_ERROR;
        }

        /* cloning may change cycle->listening.elts */
        // 由于在array数组中添加元素，可能导致内存迁移（空间不够时，因此更新当前ls指向地址）
        ls = cycle->listening.elts;
    }

#endif

    return NGX_CONF_OK;
}
```

复制监听地址的逻辑如下：

```c
ngx_int_t
ngx_clone_listening(ngx_cycle_t *cycle, ngx_listening_t *ls)
{
#if (NGX_HAVE_REUSEPORT)

    ngx_int_t         n;
    ngx_core_conf_t  *ccf;
    ngx_listening_t   ols;
    // 如果监听地址未配置复用端口，或者本身就是复制的地址，则返回
    if (!ls->reuseport || ls->worker != 0) {
        return NGX_OK;
    }
    // ols存储ls的内容（不是地址）
    ols = *ls;
   // 获取worker进程数量
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 从1开始遍历，创建worker进程数减一的复制量
    for (n = 1; n < ccf->worker_processes; n++) {

        /* create a socket for each worker process */
        // 向数组中添加元素
        ls = ngx_array_push(&cycle->listening);
        if (ls == NULL) {
            return NGX_ERROR;
        }
        // 设置内容
        *ls = ols;
        // 变更worker
        ls->worker = n;
    }

#endif

    return NGX_OK;
}
```

注意，执行核心module的init_conf在打开监听套接字之前，即克隆监听套接字也是在实际监听之前。

### ngx_event_core_module模块

ngx_event_core_module模块为事件模块的核心模块。其定义如下：

```c
static ngx_str_t  event_core_name = ngx_string("event_core");

static ngx_command_t  ngx_event_core_commands[] = {

    { ngx_string("worker_connections"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_connections,
      0,
      0,
      NULL },

    { ngx_string("use"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_use,
      0,
      0,
      NULL },

    { ngx_string("multi_accept"),
      NGX_EVENT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_event_conf_t, multi_accept),
      NULL },

    { ngx_string("accept_mutex"),
      NGX_EVENT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_event_conf_t, accept_mutex),
      NULL },

    { ngx_string("accept_mutex_delay"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      0,
      offsetof(ngx_event_conf_t, accept_mutex_delay),
      NULL },

    { ngx_string("debug_connection"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_debug_connection,
      0,
      0,
      NULL },

      ngx_null_command
};


static ngx_event_module_t  ngx_event_core_module_ctx = {
    &event_core_name,
    ngx_event_core_create_conf,            /* create configuration */
    ngx_event_core_init_conf,              /* init configuration */

    { NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL }
};


ngx_module_t  ngx_event_core_module = {
    NGX_MODULE_V1,
    &ngx_event_core_module_ctx,            /* module context */
    ngx_event_core_commands,               /* module directives */
    NGX_EVENT_MODULE,                      /* module type */
    NULL,                                  /* init master */
    ngx_event_module_init,                 /* init module */
    ngx_event_process_init,                /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

#### 相关数据结构

##### ngx_event_module_t

ngx_event_module_t为事件模块（非核心模块）的ctx执行的结构体，其定义如下：

```c
typedef struct {
    ngx_str_t              *name; // 对应模块名称，用于根据配置查找

    void                 *(*create_conf)(ngx_cycle_t *cycle); // 模块的create_conf，创建存储在conf_ctx中的结构体
    char                 *(*init_conf)(ngx_cycle_t *cycle, void *conf); // 对创建的结构体初始化

    ngx_event_actions_t     actions; // 存储事件对应处理函数的结构体
} ngx_event_module_t;
```

其中ngx_event_actions_t定义如下：

```c
typedef struct {
    /* 添加事件方法，负责将一个事件添加到系统提供的事件驱动机制（epoll，select，poll等）中。后续通过在事件发生时通过process_events获取事件 */
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    /* 删除事件方法。将一个事件从系统提供的事件驱动机制中删除 */
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    /* 启用一个事件，目前事件框架不会调用该函数，大部分事件驱动模块对该方法的实现与add一致 */
    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    /* 禁用一个事件，目前事件框架不会调用该函数，大部分事件驱动模块对该方法的实现与delete一致 */
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    /* 向事件驱动机制中添加一个新的连接，意味着该连接上的读写事件都添加到事件驱动机制中了 */
    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    /* 向事件驱动机制中删除一个连接 */
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

    ngx_int_t  (*notify)(ngx_event_handler_pt handler);
    /* 正常工作循环中，通过调用process_events方法来处理事件，其为处理、分发事件的核心 */
    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                                 ngx_uint_t flags);
    /* 初始化事件驱动模块的方法 */
    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    /* 退出事件驱动模块前调用的方法 */
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;
```

##### ngx_event_conf_t

该类为ngx_event_core_module在conf_ctx中构建的结构体，其定义如下：

```c
typedef struct {
    // 每个worker进程最大同时处理的链接数量，对应worker_connections配置
    ngx_uint_t    connections;
    // 选择的事件驱动模块在所有事件模块中的序号，即ctx_index值，对应use配置
    ngx_uint_t    use;
    
    // 是否批量建立新连接，当事件模块通知有新连接时，尽可能对本次调度中客户端发起的所有TCP请求都建立连接。对应multi_accept配置
    ngx_flag_t    multi_accept;
    //是否打开accept锁，对应accept_mutex配置
    ngx_flag_t    accept_mutex;
    // 使用accept锁时，间隔获取锁的时间间隔。及如果一次没有获取到锁，则最少等待指定的时间才能再次尝试获取锁。对应accept_mutex_delay配置
    ngx_msec_t    accept_mutex_delay;
    // 所选用的事件驱动的名字，对应use配置
    u_char       *name;

#if (NGX_DEBUG)
    // 针对特定的IP地址或者CIDR地址的请求输出debug级别日志，其他请求依然使用error_log中配置的日志级别，对应debug_connection配置
    ngx_array_t   debug_connection;
#endif
} ngx_event_conf_t;
```



#### ctx的create_conf函数：ngx_event_core_create_conf

其逻辑如下：

```c
static void *
ngx_event_core_create_conf(ngx_cycle_t *cycle)
{
    ngx_event_conf_t  *ecf;
    // 分配ngx_event_conf_t结构体空间
    ecf = ngx_palloc(cycle->pool, sizeof(ngx_event_conf_t));
    if (ecf == NULL) {
        return NULL;
    }
    // 设置初值为未定义
    ecf->connections = NGX_CONF_UNSET_UINT;
    ecf->use = NGX_CONF_UNSET_UINT;
    ecf->multi_accept = NGX_CONF_UNSET;
    ecf->accept_mutex = NGX_CONF_UNSET;
    ecf->accept_mutex_delay = NGX_CONF_UNSET_MSEC;
    ecf->name = (void *) NGX_CONF_UNSET;

#if (NGX_DEBUG)

    if (ngx_array_init(&ecf->debug_connection, cycle->pool, 4,
                       sizeof(ngx_cidr_t)) == NGX_ERROR)
    {
        return NULL;
    }

#endif

    return ecf;
}
```

#### comands中对应配置的处理

这里只介绍非通用的方法，对于系统定义的一些通用解析配置方式，可参考配置解析一章中的详细说明。

##### worker_connections配置解析方法

```c
static char *
ngx_event_connections(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_event_conf_t  *ecf = conf;

    ngx_str_t  *value;
    // 如果已经定义过该值，则说明配置中重复定义了，返回错误
    if (ecf->connections != NGX_CONF_UNSET_UINT) {
        return "is duplicate";
    }
    // 获取配置解析中的数组
    value = cf->args->elts;
    // 设置对应的数量
    ecf->connections = ngx_atoi(value[1].data, value[1].len);
    if (ecf->connections == (ngx_uint_t) NGX_ERROR) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid number \"%V\"", &value[1]);

        return NGX_CONF_ERROR;
    }
    // 对全局的cycle中的connection_n进行设置
    cf->cycle->connection_n = ecf->connections;

    return NGX_CONF_OK;
}
```

##### use配置解析方法

```c
static char *
ngx_event_use(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_event_conf_t  *ecf = conf;

    ngx_int_t             m;
    ngx_str_t            *value;
    ngx_event_conf_t     *old_ecf;
    ngx_event_module_t   *module;
    // 判断是否重复配置
    if (ecf->use != NGX_CONF_UNSET_UINT) {
        return "is duplicate";
    }

    value = cf->args->elts;
    // 获取旧版本的cycle中对应的ngx_event_core_module创建的ngx_event_conf_t
    if (cf->cycle->old_cycle->conf_ctx) {
        old_ecf = ngx_event_get_conf(cf->cycle->old_cycle->conf_ctx,
                                     ngx_event_core_module);
    } else {
        old_ecf = NULL;
    }

    // 遍历所有模块，找到事件模块的名字与use后配置一致的一个模块
    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_EVENT_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;
        if (module->name->len == value[1].len) {
            if (ngx_strcmp(module->name->data, value[1].data) == 0) {
                // 设置use为对应模块的ctx_index和name值
                ecf->use = cf->cycle->modules[m]->ctx_index;
                ecf->name = module->name->data;
                // 如果老的事件模块和新的事件模块不一致，则报错（热加载配置时）
                if (ngx_process == NGX_PROCESS_SINGLE
                    && old_ecf
                    && old_ecf->use != ecf->use)
                {
                    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "when the server runs without a master process "
                               "the \"%V\" event type must be the same as "
                               "in previous configuration - \"%s\" "
                               "and it cannot be changed on the fly, "
                               "to change it you need to stop server "
                               "and start it again",
                               &value[1], old_ecf->name);

                    return NGX_CONF_ERROR;
                }

                return NGX_CONF_OK;
            }
        }
    }

    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                       "invalid event type \"%V\"", &value[1]);
    // 如果没有找到对应的配置（可能系统不支持），则报错返回
    return NGX_CONF_ERROR;
}
```

##### debug_connections配置解析方法



#### ctx的init_conf函数：ngx_event_core_init_conf

在执行完成commands方法后，会运行ctx的init_conf方法，对ngx_event_conf_t完成初始化。其方法如下：

```c
static char *
ngx_event_core_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_event_conf_t  *ecf = conf;

#if (NGX_HAVE_EPOLL) && !(NGX_TEST_BUILD_EPOLL)
    int                  fd;
#endif
    ngx_int_t            i;
    ngx_module_t        *module;
    ngx_event_module_t  *event_module;

    module = NULL;

#if (NGX_HAVE_EPOLL) && !(NGX_TEST_BUILD_EPOLL)
    // 尝试调用epoll事件驱动模块的create函数，验证是否可以使用epoll模块
    fd = epoll_create(100);

    if (fd != -1) {
        (void) close(fd);
        // 设置模块为ngx_epoll_module
        module = &ngx_epoll_module;

    } else if (ngx_errno != NGX_ENOSYS) {
        module = &ngx_epoll_module;
    }

#endif

#if (NGX_HAVE_DEVPOLL) && !(NGX_TEST_BUILD_DEVPOLL)

    module = &ngx_devpoll_module;

#endif

#if (NGX_HAVE_KQUEUE)

    module = &ngx_kqueue_module;

#endif

#if (NGX_HAVE_SELECT)

    if (module == NULL) {
        module = &ngx_select_module;
    }

#endif
    // 如果未找到事件驱动模块，则遍历所有事件驱动模块，找到第一个可用的事件驱动模块
    if (module == NULL) {
        for (i = 0; cycle->modules[i]; i++) {

            if (cycle->modules[i]->type != NGX_EVENT_MODULE) {
                continue;
            }

            event_module = cycle->modules[i]->ctx;
            // 核心事件模块负责管理，不能作为事件驱动模块
            if (ngx_strcmp(event_module->name->data, event_core_name.data) == 0)
            {
                continue;
            }

            module = cycle->modules[i];
            break;
        }
    }
    // 如果没有可用的，则报错
    if (module == NULL) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, 0, "no events module found");
        return NGX_CONF_ERROR;
    }
    // 对ngx_event_conf_t中未在配置中设置的值设置初值
    ngx_conf_init_uint_value(ecf->connections, DEFAULT_CONNECTIONS);
    cycle->connection_n = ecf->connections;

    ngx_conf_init_uint_value(ecf->use, module->ctx_index);

    event_module = module->ctx;
    ngx_conf_init_ptr_value(ecf->name, event_module->name->data);

    ngx_conf_init_value(ecf->multi_accept, 0);
    ngx_conf_init_value(ecf->accept_mutex, 0);
    ngx_conf_init_msec_value(ecf->accept_mutex_delay, 500);

    return NGX_CONF_OK;
}
```

#### module的init_module方法

在执行完成配置解析后，在init_cycle初始化cycle的结尾（关闭无用资源前）会执行每个模块的init_module方法（在进入worker子进程前）。事件核心模块具有该方法，其逻辑如下：

```c
static ngx_int_t
ngx_event_module_init(ngx_cycle_t *cycle)
{
    void              ***cf;
    u_char              *shared;
    size_t               size, cl;
    ngx_shm_t            shm;
    ngx_time_t          *tp;
    ngx_core_conf_t     *ccf;
    ngx_event_conf_t    *ecf;
    // 获取事件模块对应的ngx_core_conf_t
    cf = ngx_get_conf(cycle->conf_ctx, ngx_events_module);
    // 获取实际使用的事件驱动模块
    ecf = (*cf)[ngx_event_core_module.ctx_index];

    if (!ngx_test_config && ngx_process <= NGX_PROCESS_MASTER) {
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                      "using the \"%s\" event method", ecf->name);
    }

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 设置调用gettimeofday()来更新缓存时钟的事件间隔，由timer_resolution配置项在核心模块解析
    ngx_timer_resolution = ccf->timer_resolution;

#if !(NGX_WIN32)
    {
    ngx_int_t      limit;
    struct rlimit  rlmt;
    // 获取进程允许创建的文件描述符数量
    if (getrlimit(RLIMIT_NOFILE, &rlmt) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "getrlimit(RLIMIT_NOFILE) failed, ignored");

    } else {
        /* ccf->rlimit_nofile是配置中指定的worker子进程打开文件描述符大小限制，可以通过setrlimit设置该值，如果每个子进程设置的能够同时处理的连接数量，大于进程限制的描述符数量或者配置的描述符数量，则记录一个错误 */
        if (ecf->connections > (ngx_uint_t) rlmt.rlim_cur
            && (ccf->rlimit_nofile == NGX_CONF_UNSET
                || ecf->connections > (ngx_uint_t) ccf->rlimit_nofile))
        {
            limit = (ccf->rlimit_nofile == NGX_CONF_UNSET) ?
                         (ngx_int_t) rlmt.rlim_cur : ccf->rlimit_nofile;

            ngx_log_error(NGX_LOG_WARN, cycle->log, 0,
                          "%ui worker_connections exceed "
                          "open file resource limit: %i",
                          ecf->connections, limit);
        }
    }
    }
#endif /* !(NGX_WIN32) */

    // 如果是非master-worker模式运行，则直接返回
    if (ccf->master == 0) {
        return NGX_OK;
    }
    // 如果已经设置了accept指针（进程间实现负载均衡的锁），则直接返回。（在重新加载配置文件时，直接返回）
    if (ngx_accept_mutex_ptr) {
        return NGX_OK;
    }


    /* cl should be equal to or greater than cache line size */
    // 创建每个进程共用的变量，通过mmap（内存映射）创建公用空间
    cl = 128;
    // 申请内存映射的空间大小
    size = cl            /* ngx_accept_mutex */ // 负载均衡锁使用空间
           + cl          /* ngx_connection_counter */ // 连接数量使用空间
           + cl;         /* ngx_temp_number */ // temp number使用空间

#if (NGX_STAT_STUB)

    size += cl           /* ngx_stat_accepted */
           + cl          /* ngx_stat_handled */
           + cl          /* ngx_stat_requests */
           + cl          /* ngx_stat_active */
           + cl          /* ngx_stat_reading */
           + cl          /* ngx_stat_writing */
           + cl;         /* ngx_stat_waiting */

#endif
    // 初始化共享内存结构ngx_shm_t
    shm.size = size;
    ngx_str_set(&shm.name, "nginx_shared_zone");
    shm.log = cycle->log;
    // 按照size分配内存映射区域，具体参考共享内存部分
    if (ngx_shm_alloc(&shm) != NGX_OK) {
        return NGX_ERROR;
    }
    // 获取分配的mmap地址
    shared = shm.addr;
    // ngx_accept_mutex_ptr指向mmap分配的起始地址（使用第一个128空间）
    ngx_accept_mutex_ptr = (ngx_atomic_t *) shared;
    // 不使用信号量
    ngx_accept_mutex.spin = (ngx_uint_t) -1;
    // 创建负载均衡锁，具体参考锁机制章节
    if (ngx_shmtx_create(&ngx_accept_mutex, (ngx_shmtx_sh_t *) shared,
                         cycle->lock_file.data)
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // ngx_connection_counter使用第二个mmap分配的128空间
    ngx_connection_counter = (ngx_atomic_t *) (shared + 1 * cl);
    // 设置值初始值为1
    (void) ngx_atomic_cmp_set(ngx_connection_counter, 0, 1);

    ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "counter: %p, %uA",
                   ngx_connection_counter, *ngx_connection_counter);
    // ngx_temp_number使用第三个mmap分配的128空间
    ngx_temp_number = (ngx_atomic_t *) (shared + 2 * cl);
    // 通过时间和pid设置ngx_random_number值
    tp = ngx_timeofday();

    ngx_random_number = (tp->msec << 16) + ngx_pid;

#if (NGX_STAT_STUB)

    ngx_stat_accepted = (ngx_atomic_t *) (shared + 3 * cl);
    ngx_stat_handled = (ngx_atomic_t *) (shared + 4 * cl);
    ngx_stat_requests = (ngx_atomic_t *) (shared + 5 * cl);
    ngx_stat_active = (ngx_atomic_t *) (shared + 6 * cl);
    ngx_stat_reading = (ngx_atomic_t *) (shared + 7 * cl);
    ngx_stat_writing = (ngx_atomic_t *) (shared + 8 * cl);
    ngx_stat_waiting = (ngx_atomic_t *) (shared + 9 * cl);

#endif

    return NGX_OK;
}
```

关于进程资源限制的内容可以查看如下文档：[进程资源信息](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E5%87%BD%E6%95%B0getrlimit%E5%92%8Csetrlimit)。

#### module的init_process方法

在启动worker子进程后，会先执行相应的初始化工作，这时会调用所有模块的init_process方法。事件核心模块的该方法执行逻辑如下：这部分的代码涉及后续的内容较多，可以先查阅本章的事件处理。

```c
static ngx_int_t
ngx_event_process_init(ngx_cycle_t *cycle)
{
    ngx_uint_t           m, i;
    ngx_event_t         *rev, *wev;
    ngx_listening_t     *ls;
    ngx_connection_t    *c, *next, *old;
    ngx_core_conf_t     *ccf;
    ngx_event_conf_t    *ecf;
    ngx_event_module_t  *module;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    ecf = ngx_event_get_conf(cycle->conf_ctx, ngx_event_core_module);
    // 如果是master-workers模式运行，且子进程数量大于1，并且使用负载均衡锁，则对相应的全局变量赋值。
    if (ccf->master && ccf->worker_processes > 1 && ecf->accept_mutex) {
        ngx_use_accept_mutex = 1;
        ngx_accept_mutex_held = 0;
        ngx_accept_mutex_delay = ecf->accept_mutex_delay;

    } else {
        ngx_use_accept_mutex = 0;
    }

#if (NGX_WIN32)

    /*
     * disable accept mutex on win32 as it may cause deadlock if
     * grabbed by a process which can't accept connections
     */

    ngx_use_accept_mutex = 0;

#endif
    // 初始化全局变量,具体查看双向队列详细介绍
    ngx_queue_init(&ngx_posted_accept_events);
    ngx_queue_init(&ngx_posted_next_events);
    ngx_queue_init(&ngx_posted_events);
    
/*
ngx_int_t
ngx_event_timer_init(ngx_log_t *log)
{
    ngx_rbtree_init(&ngx_event_timer_rbtree, &ngx_event_timer_sentinel,
                    ngx_rbtree_insert_timer_value);

    return NGX_OK;
}
*/
    // 初始化全局变量ngx_event_timer_rbtree,具体参考对于红黑树的介绍
    if (ngx_event_timer_init(cycle->log) == NGX_ERROR) {
        return NGX_ERROR;
    }
    // 遍历每一个事件模块，执行选择的事件驱动模块的ctx的actions.init方法来初始化事件驱动模块。具体执行逻辑下文讲解
    for (m = 0; cycle->modules[m]; m++) {
        if (cycle->modules[m]->type != NGX_EVENT_MODULE) {
            continue;
        }

        if (cycle->modules[m]->ctx_index != ecf->use) {
            continue;
        }

        module = cycle->modules[m]->ctx;

        if (module->actions.init(cycle, ngx_timer_resolution) != NGX_OK) {
            /* fatal */
            exit(2);
        }

        break;
    }

#if !(NGX_WIN32)
    // ngx_event_flags为选择事件驱动的类型。这里如果设置了ngx_timer_resolution（即更新时间间隔），并且选择的事件驱动不是NGX_USE_TIMER_EVENT时,增加时钟信号处理
    if (ngx_timer_resolution && !(ngx_event_flags & NGX_USE_TIMER_EVENT)) {
        struct sigaction  sa;
        struct itimerval  itv;

        ngx_memzero(&sa, sizeof(struct sigaction));
        /*
static void
ngx_timer_signal_handler(int signo)
{
    ngx_event_timer_alarm = 1;

#if 1
    ngx_log_debug0(NGX_LOG_DEBUG_EVENT, ngx_cycle->log, 0, "timer signal");
#endif
}
        */
        // 信号处理函数设置ngx_event_timer_alarm为1.
        sa.sa_handler = ngx_timer_signal_handler;
        sigemptyset(&sa.sa_mask);
        // 设置信号处理函数
        if (sigaction(SIGALRM, &sa, NULL) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "sigaction(SIGALRM) failed");
            return NGX_ERROR;
        }
        // 使用setitimer定时触发时钟信号
        itv.it_interval.tv_sec = ngx_timer_resolution / 1000;
        itv.it_interval.tv_usec = (ngx_timer_resolution % 1000) * 1000;
        itv.it_value.tv_sec = ngx_timer_resolution / 1000;
        itv.it_value.tv_usec = (ngx_timer_resolution % 1000 ) * 1000;

        if (setitimer(ITIMER_REAL, &itv, NULL) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "setitimer() failed");
        }
    }
    // linux一般不会使用NGX_USE_FD_EVENT事件，这里不做介绍
    if (ngx_event_flags & NGX_USE_FD_EVENT) {
        struct rlimit  rlmt;

        if (getrlimit(RLIMIT_NOFILE, &rlmt) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "getrlimit(RLIMIT_NOFILE) failed");
            return NGX_ERROR;
        }

        cycle->files_n = (ngx_uint_t) rlmt.rlim_cur;

        cycle->files = ngx_calloc(sizeof(ngx_connection_t *) * cycle->files_n,
                                  cycle->log);
        if (cycle->files == NULL) {
            return NGX_ERROR;
        }
    }

#else

    if (ngx_timer_resolution && !(ngx_event_flags & NGX_USE_TIMER_EVENT)) {
        ngx_log_error(NGX_LOG_WARN, cycle->log, 0,
                      "the \"timer_resolution\" directive is not supported "
                      "with the configured event method, ignored");
        ngx_timer_resolution = 0;
    }

#endif
    // 为加速，预先按照connection_n（进程最大同时处理链接数量）分配connections数组
    cycle->connections =
        ngx_alloc(sizeof(ngx_connection_t) * cycle->connection_n, cycle->log);
    if (cycle->connections == NULL) {
        return NGX_ERROR;
    }

    c = cycle->connections;
    // 为加速，预先按照connection_n（进程最大同时处理链接数量）分配read_events数组，并设置初值
    cycle->read_events = ngx_alloc(sizeof(ngx_event_t) * cycle->connection_n,
                                   cycle->log);
    if (cycle->read_events == NULL) {
        return NGX_ERROR;
    }
    
    rev = cycle->read_events;
    for (i = 0; i < cycle->connection_n; i++) {
        rev[i].closed = 1;
        rev[i].instance = 1;
    }
    // 为加速，预先按照connection_n（进程最大同时处理链接数量）分配write_events数组，并设置初值
    cycle->write_events = ngx_alloc(sizeof(ngx_event_t) * cycle->connection_n,
                                    cycle->log);
    if (cycle->write_events == NULL) {
        return NGX_ERROR;
    }

    wev = cycle->write_events;
    for (i = 0; i < cycle->connection_n; i++) {
        wev[i].closed = 1;
    }

    i = cycle->connection_n;
    next = NULL;
    // 从后向前遍历每一个cycle->connections，初始化data，并将其与对应的read_events和write_events绑定
    do {
        i--;

        c[i].data = next;
        c[i].read = &cycle->read_events[i];
        c[i].write = &cycle->write_events[i];
        c[i].fd = (ngx_socket_t) -1;

        next = &c[i];
    } while (i);
    // 设置当前未使用connections的起始地址
    cycle->free_connections = next;
     // 设置当前未使用connections的数量
    cycle->free_connection_n = cycle->connection_n;

    /* for each listening socket */
    // 遍历每个监听端口，绑定一个connection
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {

#if (NGX_HAVE_REUSEPORT)
        /* 如果监听地址设置了可复用端口号，并且其worker不是当前进程的编号时，则忽略（对应支持端口复用的地址，为每个进程设置了一个专门的ls）*/
        if (ls[i].reuseport && ls[i].worker != ngx_worker) {
            continue;
        }
#endif
        // 获取第一个未使用的connection，并将其绑定到该ls上，具体查看连接池的介绍
        c = ngx_get_connection(ls[i].fd, cycle->log);

        if (c == NULL) {
            return NGX_ERROR;
        }
        // 设置connection
        c->type = ls[i].type;
        c->log = &ls[i].log;

        c->listening = &ls[i];
        ls[i].connection = c;
        // 获取连接对应的读事件
        rev = c->read;

        rev->log = c->log;
        // 设置读事件为监听套接字
        rev->accept = 1;

#if (NGX_HAVE_DEFERRED_ACCEPT)
        // 设置deferred_accept，详见监听端口处理，用于加速用的配置。在TCP连接发送请求前，不为其建立connection结构
        rev->deferred_accept = ls[i].deferred_accept;
#endif
        // 如果事件驱动不是NGX_USE_IOCP_EVENT
        if (!(ngx_event_flags & NGX_USE_IOCP_EVENT)) {
            // 如果ls是继承而来
            if (ls[i].previous) {

                /*
                 * delete the old accept events that were bound to
                 * the old cycle read events array
                 */
                // 获取旧版本的connection
                old = ls[i].previous->connection;
                // 从事件模块中去除对应的读事件
                if (ngx_del_event(old->read, NGX_READ_EVENT, NGX_CLOSE_EVENT)
                    == NGX_ERROR)
                {
                    return NGX_ERROR;
                }

                old->fd = (ngx_socket_t) -1;
            }
        }

#if (NGX_WIN32)

        if (ngx_event_flags & NGX_USE_IOCP_EVENT) {
            ngx_iocp_conf_t  *iocpcf;

            rev->handler = ngx_event_acceptex;

            if (ngx_use_accept_mutex) {
                continue;
            }

            if (ngx_add_event(rev, 0, NGX_IOCP_ACCEPT) == NGX_ERROR) {
                return NGX_ERROR;
            }

            ls[i].log.handler = ngx_acceptex_log_error;

            iocpcf = ngx_event_get_conf(cycle->conf_ctx, ngx_iocp_module);
            if (ngx_event_post_acceptex(&ls[i], iocpcf->post_acceptex)
                == NGX_ERROR)
            {
                return NGX_ERROR;
            }

        } else {
            rev->handler = ngx_event_accept;

            if (ngx_use_accept_mutex) {
                continue;
            }

            if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
                return NGX_ERROR;
            }
        }

#else
        // 设置读事件的处理函数，根据类型确定。具体事件处理函数见下文
        rev->handler = (c->type == SOCK_STREAM) ? ngx_event_accept
                                                : ngx_event_recvmsg;

#if (NGX_HAVE_REUSEPORT)
        /* 如果设置了端口号复用，则将读事件添加到事件驱动中，结束处理。对于支持端口号复用的监听地址来说，会为每一个进程设置一个监听的ls结构，每一个进程都会进行监听，由内核实现进程间的负载均衡，选择连接请求下发到哪个进程中，因此不需要考虑是否使用负载均衡锁，可以直接将事件添加到事件驱动中 */
        if (ls[i].reuseport) {
            if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
                return NGX_ERROR;
            }

            continue;
        }

#endif
        // 如果使用负载均衡锁，则跳过处理,对于使用负载均衡锁的一般监听地址来说，不能直接使用事件驱动。使用负载均衡锁时，只有在获取到锁时，才能将该读事件添加到事件驱动中。具体后文会详细介绍
        if (ngx_use_accept_mutex) {
            continue;
        }

#if (NGX_HAVE_EPOLLEXCLUSIVE)

        if ((ngx_event_flags & NGX_USE_EPOLL_EVENT)
            && ccf->worker_processes > 1)
        {
            if (ngx_add_event(rev, NGX_READ_EVENT, NGX_EXCLUSIVE_EVENT)
                == NGX_ERROR)
            {
                return NGX_ERROR;
            }

            continue;
        }

#endif
        // 对于不使用负载均衡锁来说，可以直接将事件添加到事件驱动中
        if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }

#endif

    }

    return NGX_OK;
}
```

### ngx_epoll_module模块

```c
static ngx_str_t      epoll_name = ngx_string("epoll");

static ngx_command_t  ngx_epoll_commands[] = {
    // 调用epoll_wait函数时返回的最多事件数
    { ngx_string("epoll_events"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      0,
      offsetof(ngx_epoll_conf_t, events),
      NULL },
    // 开启异步I/O并且使用io_setup系统调用初始化异步I/O上下文环境时，初始分配的异步i/O事件个数。后文详细介绍
    { ngx_string("worker_aio_requests"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      0,
      offsetof(ngx_epoll_conf_t, aio_requests),
      NULL },

      ngx_null_command
};


static ngx_event_module_t  ngx_epoll_module_ctx = {
    &epoll_name,
    ngx_epoll_create_conf,               /* create configuration */
    ngx_epoll_init_conf,                 /* init configuration */
    // 对于下面的函数，后续会详细介绍
    {
        ngx_epoll_add_event,             /* add an event */
        ngx_epoll_del_event,             /* delete an event */
        ngx_epoll_add_event,             /* enable an event */
        ngx_epoll_del_event,             /* disable an event */
        ngx_epoll_add_connection,        /* add an connection */
        ngx_epoll_del_connection,        /* delete an connection */
#if (NGX_HAVE_EVENTFD)
        ngx_epoll_notify,                /* trigger a notify */
#else
        NULL,                            /* trigger a notify */
#endif
        ngx_epoll_process_events,        /* process the events */
        ngx_epoll_init,                  /* init the events */
        ngx_epoll_done,                  /* done the events */
    }
};

ngx_module_t  ngx_epoll_module = {
    NGX_MODULE_V1,
    &ngx_epoll_module_ctx,               /* module context */
    ngx_epoll_commands,                  /* module directives */
    NGX_EVENT_MODULE,                    /* module type */
    NULL,                                /* init master */
    NULL,                                /* init module */
    NULL,                                /* init process */
    NULL,                                /* init thread */
    NULL,                                /* exit thread */
    NULL,                                /* exit process */
    NULL,                                /* exit master */
    NGX_MODULE_V1_PADDING
};
```

#### 相关结构

##### ngx_epoll_conf_t

该结构存储关于事件驱动和文件异步I/O的配置内容。

```c
typedef struct {
    ngx_uint_t  events; // 调用epoll_wait函数时返回的最多事件数
    ngx_uint_t  aio_requests;// 开启异步I/O并且使用io_setup系统调用初始化异步I/O上下文环境时，初始分配的异步i/O事件个数。后文详细介绍
} ngx_epoll_conf_t;
```

对其设置较为简单，都是调用系统通过的解析方法，这里就不做过多介绍了。

#### ctx的create_conf方法

```c
static void *
ngx_epoll_create_conf(ngx_cycle_t *cycle)
{
    ngx_epoll_conf_t  *epcf;

    epcf = ngx_palloc(cycle->pool, sizeof(ngx_epoll_conf_t));
    if (epcf == NULL) {
        return NULL;
    }

    epcf->events = NGX_CONF_UNSET;
    epcf->aio_requests = NGX_CONF_UNSET;

    return epcf;
}
```

#### ctx的init_conf方法

```c
static char *
ngx_epoll_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_epoll_conf_t *epcf = conf;

    ngx_conf_init_uint_value(epcf->events, 512);
    ngx_conf_init_uint_value(epcf->aio_requests, 32);

    return NGX_CONF_OK;
}
```

## 系统接口

事件处理主要涉及两个功能，一个是网络事件的epoll模块，一个是文件异步I/O模块。这里先对这两个功能在linux下如何使用进行介绍，再来看nginx如果使用。

### epoll事件驱动

#### epoll特点

讨论epoll事件驱动的特点，要先来讨论一下select和poll的问题。

1. select由于传参是最大的套接字值，因此其限制监听的套接字数量。
2. select和poll在收集事件时，将所有连接的套接字传递给内核（大量内存负责），内核遍历寻找这些连接上是否存在未处理的事件（复杂度O(n)）。

由于以上两个特性，导致select和poll事件驱动不适合处理高并发连接的情况。因此在2.6以后的linux内核支持了epoll事件驱动。很好的解决了上述问题。

对于第一个问题：用户态到内核态大量拷贝操作，epoll解决方式是：使用mmap内存映射，将内核态和用户态使用的内存统一，避免拷贝操作。

对于第二个问题：内核需要遍历所有连接，epoll解决方案是，对于每个连接，都会与设备（如网卡）驱动程序建立回调关系，即，相应的事件发送时会调用这里的回调方案。这个回调方法内核中叫做`ep_poll_callback`。回调方法会将触发的事件添加到一个链表中，在我们获取已经触发的事件时，直接返回链表即可，不用再进行遍历。

#### epoll使用

epoll只提供了三个函数，让我们来进行事件处理。

##### epoll_create

```
int epoll_create(int size);
```

epoll_create返回一个句柄，之后epoll的使用都依赖该句柄来标识。size是告诉epoll所处理的大致事件数目。不再使用epoll时，需要调用close关闭这个句柄。size参数只是告诉内核这个epoll对象会处理的事件的大致数目，而不是能够处理的事件的最大个数。新版的Linux内核版本中size已无意义。

##### epoll_ctl

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
```

epoll_ctl向epoll对象中添加、修改或者删除感兴趣的事件，成功返回0，否则返回-1，此时需要根据errno错误码判断错误类型。epoll_wait方法返回的事件必然是通过epoll_ctl添加到epoll中的。参数epfd是epoll_create创建的句柄。op对应操作类型，取值如下：

| op取值        | 含义                  |
| ------------- | --------------------- |
| EPOLL_CLT_ADD | 添加新的事件到epoll中 |
| EPOLL_CLT_MOD | 修改epoll中的事件     |
| EPOLL_CLT_DEL | 删除epoll中的事件     |

第三个参数fd是待检测的连接套接字。

第四个参数告诉epoll对什么样的事件感兴趣，其使用的是epoll_event结构体。epoll_event定义如下：

```c
struct epoll_event {
  __uint32_t events;
  epoll_data_t data;
}
```

其中event表示感兴趣事件，取值如下：

| events取值   | 含义                                                         |
| ------------ | ------------------------------------------------------------ |
| EPOLLIN      | 表示对应的连接上有数据可读（TCP连接的远端主动关闭事件，也相当于可读事件，因为需要处理发送来的FIN包。对应监听端口，远端发送来连接请求，也是可读事件，之后调用accept） |
| EPOLLOUT     | 表示对应连接上可以写入数据发送（主动向上游服务器发起非阻塞的TCP连接，连接建立成功的事件相当于可写事件） |
| EPOLLRDHUP   | 表示TCP连接的远端关闭或者半关闭连接                          |
| EPOLLPRI     | 表示对应的连接上有紧急数据要读                               |
| EPOLLERR     | 表示对应的连接发生错误                                       |
| EPOLLHUP     | 表示对应的连接被挂起                                         |
| EPOLLET      | 表示将触发方式设置为边缘触发（ET），系统默认为水平触发（LT） |
| EPOLLONESHOT | 表示对事件只处理一次，下次需要处理时需要重新加入epoll        |

data成员是一个epoll_data联合，其定义如下：

```c
struct union epoll_data{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t
```

可以看出，data成员与具体的使用方式有关。例如，ngx_epoll_module模块只使用了联合的ptr成员，作为指向ngx_connection_t连接的指针。

##### epoll_wait

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
```

收集epoll监控的事件中已经发生的事件，如果没有事件发生，则最多等待timeout秒返回。epoll_wait返回值表示当前发生的事件个数，如果返回0，表示本次调用中没有事件发生，如果返回-1，则表示错误，需要检查errno错误码来判断错误类型。第一个参数epfd为epoll的描述符。第二个参数events是分配好的epoll_event结构体数组，epoll将会把发生的事件复制到events数组中（events不能是空指针，内核只负责复制数据到events，不会帮助用户在用户态中分配内存。）第三个参数是maxevents表示本次可以返回的最大事件数目，通常maxevents参数与预分配的events数组的大小一致。timeout为-1表示一直阻塞，直到有事件触发。

#### epoll原理

介绍完epoll如何使用，这里再简单介绍一下epoll的原理。

在调用epoll_create时，Linux内核会创建一个eventpoll的结构体，其中有两个成员与epoll的使用方式密切相关。

```c
struct eventpoll {
  ...
  // 红黑树的根节点，这棵树存储着所有添加到epoll中的事件，即epoll监控的事件
  struct rb_tree rbr;
  // 双向链表rdllist报错着将要通过epoll_wait返回给用户的、满足条件的事件
  struct list_head rdllist;
}
```

每个epoll对象都有一个独立的eventpoll结构体，这个结构体会在内核空间中创造独立的内存，用于存储使用epoll_ctl方法向epoll对象中添加进来的事件。这些事件会被挂到rbr红黑树中，以支持快速的增加、删除、变更。rbr的红黑树和前文描述的nginx构建的红黑树类似。

所有添加到epoll中的事件都会与设备（如网卡）驱动程序建立回调关系，即相应事件发生时会调用这里的回调方法。这个回调方法在内核中叫做`ep_poll_callback`，它会把这样的事件放到rdllist双线链表中。rdllist双向链表与前文nginx创建的双向链表几乎一致。

在epoll中，对每个事件都会建立一个epitem结构体：

```c
struct epitem {
  ...
  // 红黑树节点，与ngx_rbtree_node_t红黑树节点类似
  struct rb_node rbn;
  
  // 双向链表节点，与ngx_queue_t双向链表节点类似
  struct list_head rdllink;
  
  // 事件句柄等信息
  struct epoll_filefd ffd;
  
  // 执行其所属eventpoll对象
  struct eventpoll *ep;
  
  // 期待的事件类型
  struct epoll_event event;
  ...
}
```

epitem包含每一个事件对应着的信息。

当调用epoll_wait检查是否有事件发生时，只是检查eventpoll对象中的rdllist双向链表是否有epitem元素而已。

epoll有两种工作模式：LT（水平触发）模式和ET（边缘触发）模式。默认情况下，epoll采用LT模式工作，这时可以处理阻塞和非阻塞套接字。而在epoll_ctl中的EPOLLET表明可以将一个事件改为ET模式，ET模式效率更高，其只能处理非阻塞套接字。ET模式和LT模式主要区别在于：当一个事件到来时，ET模式下从epoll_wait调用中获取到这个事件后，如果这次没有把事件对应的套接字缓冲区处理完，在这个套接字没有新的事件再次到来时，ET模式下无法再次从epoll_wait中获取到这个事件，而LT模式可以。因此在LT模式下开发基于epoll的应用要简单一些，不太容易出错，而在ET模式下事件发生时，如果没有彻底将缓冲区数据处理完全，会导致缓冲区中的用户请求得不到响应。默认情况下，nginx通过ET模式使用epoll。

### 文件异步I/O

epoll解决了网络事件的事件驱动。这里介绍一下Linux内核2.6.2x之后版本中支持的文件异步I/O操作。这里提到的文件异步I/O不是glibc库提供的文件异步io。glibc提供的文件异步I/O是基于多线程实现的，不是真正意义的异步I/O。这里说的文件异步I/O是由Linux内核实现，只有在内核完成了磁盘操作，内核才会通知进程，进而使得磁盘文件的处理与网络事件一样高效。

使用Linux内核版本的文件异步I/O前提是内核版本必须支持文件异步I/O。这样的好处是，nginx把读文件的操作异步地提交给内核后，内核会通知I/O设备独立执行操作，这样nginx进程可以继续充分利用cpu。当大量读事件堆积到I/O设备的队列中时，将会发挥出内核中电梯算法的优势，从而降低随机读取磁盘扇区的成本。

Linux内核级别的文件异步I/O是不支持缓存操作的，即，即使要操作的文件快在Linux文件缓存中，也不会通过读取、更改缓存中的文件块来代替实际对磁盘的操作。使用文件异步I/O虽然从阻塞worker进程的角度来说有很大好转，但对单个请求来说，有可能降低处理速度的，因为原本可以从内存中快速获取的文件块在使用了文件异步I/O后必须通过磁盘获取。如果大部分用户请求对文件的操作都会落到文件缓存中，那么就不应该使用异步I/O。

目前nginx仅支持在读文件时使用异步I/O，因为在写入文件时，往往是写入内存中就立即返回，速度很快，使用异步I/O反而会速度下降。

nginx默认不使用文件异步I/O，要向使用，则在执行config时，增加参数：

```
./configure --with-file-aio
```

文件异步I/O使用方式与epoll基本一致，下面进行简单介绍。

#### 初始化文件异步I/O上下文

使用如下函数初始化文件异步I/O上下文：

```c
int io_setup(unsigned nr_events, aio_context_t *ctxp)
```

`nr_events`表示需要初始化的异步I/O上下文可以处理的事件的最小个数，ctxp是文件异步I/O上下文描述符指针。执行成功后，ctxp就是分配的上下文描述符，这个异步I/O至少可以处理nr_events个事件，返回0表示成功。

该函数与epoll_create对应。

#### 向异步I/O上下文中增删文件操作事件

下面两个函数向异步I/O中增加、删除事件：

```c
// 增加事件,nr是一次提交事件的数量，cbp是提交事件数组的首地址。返回值表示成功提交的事件个数
int io_submit(aio_context_t ctx, long nr, struct iocb *cbp[]);

// 删除事件， iocb是要取消的异步I/O操作，而result表示操作的结果。返回0表示成功
int io_cancel(aio_context_t ctx, struct iocb *iocb, struct io_event *result);
```

其中使用到的iocb定义如下：

```c
struct iocb {
  // 用于存储业务需要的指针。例如在nginx中，该字段通常存储这对应的ngx_event_t事件的指针，其与io_getevents返回的IO_event结构体中的data成员一致。
  u_int64_t aio_data;
  
  // 不需要设置
  u_int32_t PADDED(aio_key, aio_reserved1);
  
  // 操作码，其取值范围为io_icob_cmd_t中枚举命令
  u_int32_t aio_lio_opcode;
  
  // 请求优先级
  int16_t aio_reqprio;
  
  // 文件描述符
  u_int32_t aio_fildes;
  
  // 读写对应的用户态缓冲区
  u_int64_t aio_buf;
  
  // 读写操作的字节长度
  u_int65_t aio_nbytes;
  
  // 读写对应文件中的偏移
  int64_t aio_offset;
  
  // 保留字段
  u_int64_t aio_reserved2;
  
  // 表示可以设置为IOCB_FLAG_RESFD,它会告诉内核，当有异步I/O请求完成时使用eventfd进行通知，可与epoll配合使用。
  u_int32_t aio_flags;
  
  // 表示当使用了IOCB_FLAG_RESFD标志位时，用于进行事件通知的句柄
  u_int32_t aio_resfd;
}
```

其中aio_lio_opcode取值范围如下：

```c
typedef enum io_iocb_cmd {
  // 异步读操作
  IO_CMD_PREAD = 0;
  // 异步写操作
  IO_CMD_PWRITE = 1;
  // 强制同步
  IO_CMD_FSYNC = 2;
  // 目前未使用
  IO_CMD_FDSYNC = 3;
  // 目前未使用
  IO_CMD_POLL = 5;
  // 不做任何事情
  IO_CMD_NOOP = 6;
}
```

nginx中仅使用了IO_CMD_PREAD。

对应io_event下面介绍。

#### 获取已完成事件

```c
int io_getevents(aio_context_t ctx, long min_nr, long nr, struct io_event *events, struct timespec *timeout);
```

从已完成文件异步I/O的操作队列中读取操作。min_nr是最小返回事件数量，nr表示最大返回事件数量，events是执行完成的事件数组。timeout是超时时间，即获取到min_nr事件前的等待事件。

这里使用到了io_event结构，定义如下：

```c
struct io_event{
  // 对应提交事件时iocb结构的aio_data
  uint64_t data;
  
  // 指向提交事件时对应的iocb结构
  uint64_t obj;
  
  // 异步I/O请求结果。res大于或等于0表示成功，否则表示失败。
  int64_t res;
  
  // 保留字段
  int64_t res2;
}
```

这样，根据获取到的io_event结构体数组，就可以获取已完成的异步I/O操作了。其中iocb和io_event可以传递指针，因此业务中数据结构、事件完成后的回调方法都在其中。

#### 关闭异步I/O上下文

进程退出时，需要关闭异步I/O上下文，方法如下：

```c
int io_destroy(aio_context_t ctx);
```

返回0表示成功。与epoll结束时close类似。

### 系统事件模块eventfd

在Linux系统中，eventfd是一个用来通知事件的文件描述符。可以使用eventfd来触发事件通知。在nginx中也会使用该方法，用于绑定消息通知和将文件异步I/O和epoll驱动。

#### 创建eventfd

```c
int eventfd(unsigned int initval, int flags);
```

创建一个eventfd对象，或者说打开一个eventfd的文件，类似普通文件的open操作。

该对象是一个内核维护的无符号的64位整型计数器。初始化为initval的值。

flags可以以下三个标志位的OR结果：

- EFD_CLOEXEC：FD_CLOEXEC，简单说就是fork子进程时不继承，对于多线程的程序设上这个值不会有错的。
- EFD_NONBLOCK：文件会被设置成O_NONBLOCK，一般要设置。
- EFD_SEMAPHORE：（2.6.30以后支持）支持semophore语义的read，简单说就值递减1。

对于eventfd就只是一个计数器，一般用于记录绑定到其上的事件触发数量。一般与epoll模块配合使用，后文会详细介绍nginx如果使用eventfd和epoll。

#### 读eventfd

```c
int read(int efd, void *buf, size_t nbytes);// 或者int eventfd_read (int __fd, eventfd_t *__value);
```

读取当前事件对应计数数值。在nginx中就是获取绑定到efd上的事件触发数量。这里如果eventfd计数不为0，并且未设置为EFD_SEMAPHORE则，计数清零，返回buf中存储计数数量，read返回读取字节数。如果eventfd计数不为0，并且设置了EFD_SEMAPHORE则，计数减一。如果eventfd为0，没有设置EFD_NONBLOCK，进程阻塞，直到计数不为0。如果eventfd为0，设置了EFD_NONBLOCK，则直接返回。

#### 写eventfd

```c
int write(int efd, void *buf, size_t nbytes);// 或者int eventfd_write (int __fd, eventfd_t __value);
```

这里的写并非直接设置eventfd值，而是对eventfd的引用计数进行增加。对于和epoll配合使用的eventfd，这一步一般不需要用户自己写入，而是在eventfd绑定的事件触发时由epoll自己写入。

## 关键结构

事件模块中存在一些关键结构，用于构建整个事件驱动的运行，这里进行详细介绍。

### ngx_event_s事件结构

ngx_event_s定义了事件，其内容如下：

```c
typedef void (*ngx_event_handler_pt)(ngx_event_t *ev);

struct ngx_event_s {
    // 事件相关的对象，通常是执行ngx_connection_t连接对象，开启文件异步I/O时也可指向ngx_event_aio_t
    void            *data;
    /* 标志位，为1时说明事件是可写的。通常表示TCP连接目前处于可写状态 */
    unsigned         write:1;
    /* 标注位，为1时表示为此事件可以建立新的连接。通常只有ngx_cycle_t中的listening动态数组中才，每一个监听对象ngx_listening_t对应的读事件中的accept标志位才会是1 */
    unsigned         accept:1;

    /* used to detect the stale events in kqueue and epoll */
    /* 区分当前事件是否过期，仅仅给事件驱动模块使用，事件消费模块可不关心。设计原因为：开始处理前面的事件时，可能会关闭一些连接，而这些连接有可能影响还未处理的事件。这是通过该标志位来避免处理后续已经过期的事件 */
    unsigned         instance:1;

    /*
     * the event was passed or would be passed to a kernel;
     * in aio mode - operation was posted.
     */
     /* 标志位，事件是否活跃 */
    unsigned         active:1;
    
    /*标志位，只在kqueue和rtsig事件中国有效，事件禁止*/
    unsigned         disabled:1;

    /* the ready event; in aio mode 0 means that no operation can be posted */
    /* 标志位，事件是否准备就绪，是否允许事件的消费模块处理该事件 */
    unsigned         ready:1;
    /* 对epoll无意义 */
    unsigned         oneshot:1;

    /* aio operation is complete */
    /* 标志位，用于AIO事件处理 */
    unsigned         complete:1;
    /* 标志位，1表示要处理的字节流已结束 */
    unsigned         eof:1;
    /* 标志位，1表示事件处理过程中出现错误 */
    unsigned         error:1;
    /* 标志位，1表示事件已超时，用来提醒事件消费模块做超时处理 */
    unsigned         timedout:1;
    /* 标志位，1表示事件存在于定时器中 */
    unsigned         timer_set:1;
    /* 标志位，1表示要延迟处理该事件，用于限速功能 */
    unsigned         delayed:1;
    /* 标志位，是否延迟建立TCP连接，即完成握手不建立连接，等到发送请求数据 */
    unsigned         deferred_accept:1;

    /* the pending eof reported by kqueue, epoll or in aio chain operation */
    /* 标志位，等待字符流结束 */
    unsigned         pending_eof:1;
    /*  */
    unsigned         posted:1;
    /* 标志位，1表示事件已关闭，epoll未使用 */
    unsigned         closed:1;

    /* to test on worker exit */
    unsigned         channel:1;
    unsigned         resolver:1;

    unsigned         cancelable:1;

#if (NGX_HAVE_KQUEUE)
    unsigned         kq_vnode:1;

    /* the pending errno reported by kqueue */
    int              kq_errno;
#endif

    /*
     * kqueue only:
     *   accept:     number of sockets that wait to be accepted
     *   read:       bytes to read when event is ready
     *               or lowat when event is set with NGX_LOWAT_EVENT flag
     *   write:      available space in buffer when event is ready
     *               or lowat when event is set with NGX_LOWAT_EVENT flag
     *
     * iocp: TODO
     *
     * otherwise:
     *   accept:     1 if accept many, 0 otherwise
     *   read:       bytes to read when event is ready, -1 if not known
     */
    /* 在epoll中表示一次尽可能多的建立连接，与multi_accept配置相关 */
    int              available;
    /* 事件对应的处理方法，每个事件消费模块会重新实现它 */
    ngx_event_handler_pt  handler;


#if (NGX_HAVE_IOCP)
    ngx_event_ovlp_t ovlp;
#endif
    /* epoll未使用 */
    ngx_uint_t       index;
    
    ngx_log_t       *log;
    /* 定时器节点，用于定时器红黑树中 */
    ngx_rbtree_node_t   timer;

    /* the posted queue */
    /* 在post队列中的节点 */
    ngx_queue_t      queue;

#if 0

    /* the threads support */

    /*
     * the event thread context, we store it here
     * if $(CC) does not understand __thread declaration
     * and pthread_getspecific() is too costly
     */

    void            *thr_ctx;

#if (NGX_EVENT_T_PADDING)

    /* event should not cross cache line in SMP */

    uint32_t         padding[NGX_EVENT_T_PADDING];
#endif
#endif
};
```

### ngx_connection_t连接

ngx_connection_t定义了连接，其内容如下：

```c
struct ngx_connection_s {
    /* 连接未使用时，充当连接池中空闲链表的next指针。连接使用时，由其使用的模块决定，如在http中，data指向ngx_http_request_t结构 */
    void               *data;
    /* 对应的读事件 */
    ngx_event_t        *read;
    /* 对应的写事件 */
    ngx_event_t        *write;
    /* 对应的套接字 */
    ngx_socket_t        fd;
    /* 直接接收网络字符流的方法 */
    ngx_recv_pt         recv;
    /* 直接发送网络字节流的方法 */
    ngx_send_pt         send;
    /* 使用ngx_chain_t链表为参数接收网络字符流的方法 */
    ngx_recv_chain_pt   recv_chain;
    /* 使用ngx_chain_t链表为参数接收网络字符流的方法 */
    ngx_send_chain_pt   send_chain;
    /* 连接对应的监听对象，此链接由listening监听端口的事件建立 */
    ngx_listening_t    *listening;
    /* 连接上已经发送的字节数 */
    off_t               sent;
    /* 记录日志的log */
    ngx_log_t          *log;
    /* 连接池。一版在accept一个新连接时，创建一个内存池，连接结束时销毁。内存池大小由listening监听对象的pool_size成员决定 */
    ngx_pool_t         *pool;
    /* 连接类型 */
    int                 type;
    /* 连接客户的地址信息 */
    struct sockaddr    *sockaddr;
    socklen_t           socklen;
    /* 连接客户端字符串形式ip地址 */
    ngx_str_t           addr_text;
    /* 为代理连接进行的扩招 */
    ngx_proxy_protocol_t  *proxy_protocol;

#if (NGX_SSL || NGX_COMPAT)
    /* ssl连接进行的扩展 */
    ngx_ssl_connection_t  *ssl;
#endif
    /* udp连接进行的扩展 */
    ngx_udp_connection_t  *udp;
    /* 本机监听端口对应的sockaddr，即listening监听对象的sockaddr */
    struct sockaddr    *local_sockaddr;
    socklen_t           local_socklen;
    /* 用于接收、缓存客户端发送来的字节流，每个事件消费模块可自由决定从连接池中分配多大空间给buffer，例如http模块中，buffer大小取决于client_header_buffer_size配置 */
    ngx_buf_t          *buffer;
    /* 该字段将当前连接以双向链表元素的形式添加到ngx_cycle_t中reusable_connection_t双向链表中，表示可以重用的连接 */
    ngx_queue_t         queue;
    /* 连接使用次数， ngx_connection_t结构体每建立一条来自客户端的连接，或者向后端发起连接时，number都会加1*/
    ngx_atomic_uint_t   number;
    /* 处理的请求次数 */
    ngx_uint_t          requests;
    /* 缓存中的业务类型 */
    unsigned            buffered:8;
    /* 日志级别 */
    unsigned            log_error:3;     /* ngx_connection_log_error_e */
    /* 连接是否超时 */
    unsigned            timedout:1;
    /* 连接是否发生错误 */
    unsigned            error:1;
    /* 连接是否已销毁。值TCP连接是否已销毁，对应的套接字和内存池不可用了 */
    unsigned            destroyed:1;
    /* 连接处于空闲状态，如keepalive请求中两次请求之间的状态 */
    unsigned            idle:1;
    /* 连接是否可重用的，与queue配合使用 */
    unsigned            reusable:1;
    /* 连接是否关闭 */
    unsigned            close:1;
    unsigned            shared:1;
    /* 为1表示正将文件中的数据发往连接的另一端 */
    unsigned            sendfile:1;
    /* 标志位，1表示只有在连接套接字对应的发送缓冲区必须满足最低设置的大小阈值时，事件驱动模块才会分发该事件 */
    unsigned            sndlowat:1;
    /* 如何使用TCP的nodelay属性，可取值：NGX_TCP_NODELAY_UNSET:0,NGX_TCP_NODELAY_SET:1,NGX_TCP_NODELAY_DISABLE:2 */
    unsigned            tcp_nodelay:2;   /* ngx_connection_tcp_nodelay_e */
    /* 如何使用TCP的nopush属性，可取值：NGX_TCP_NOPUSH_UNSET:0,NGX_TCP_NOPUSH_SET:1,NGX_TCP_NOPUSH_DISABLE:2 */
    unsigned            tcp_nopush:2;    /* ngx_connection_tcp_nopush_e */

    unsigned            need_last_buf:1;

#if (NGX_HAVE_AIO_SENDFILE || NGX_COMPAT)
    unsigned            busy_count:2;
#endif

#if (NGX_THREADS || NGX_COMPAT)
    ngx_thread_task_t  *sendfile_task;
#endif
};
```

其中接收发送字符流方法定义如下：

```c
typedef ssize_t (*ngx_recv_pt)(ngx_connection_t *c, u_char *buf, size_t size);
typedef ssize_t (*ngx_recv_chain_pt)(ngx_connection_t *c, ngx_chain_t *in,
    off_t limit);
typedef ssize_t (*ngx_send_pt)(ngx_connection_t *c, u_char *buf, size_t size);
typedef ngx_chain_t *(*ngx_send_chain_pt)(ngx_connection_t *c, ngx_chain_t *in,
    off_t limit);
```

### ngx_connection_t连接池

Nginx接受客户端连接时，所使用的ngx_connection_t结构体在启动阶段就域分配好的（详见ngx_event_core_module模块的init_process方法），使用时从连接池中获取即可。

连接池结构示意图：

[![5Mizo8.png](https://z3.ax1x.com/2021/10/13/5Mizo8.png)](https://imgtu.com/i/5Mizo8)

cycle中的connections和free_connections两个成员构成一个连接池，其中connections指向整个连接池数组的首部，free_connections指向第一个空闲的ngx_connection_t空闲连接。所有空闲连接通过ngx_connection_t的data成员作为next指针串联成一个单链表。当有用户发起连接时，就从free_connections指向的链表头部获取一个空闲连接，同时free_connections指向下一个空闲ngx_connection_t。归还连接时将对应的ngx_connection_t插入free_connections头部即可。

上图不止展示了连接池，同时还有事件池。Nginx认为每一个连接都至少需要一个读事件和写事件。有多少个连接就分配对应数量的读事件池和写事件池，并通过ngx_connection_t中的指针绑定对应的两个事件，ngx_event_t事件也存在指向ngx_connection_t的指针。这样将每个连接关联一个读事件和写事件。这一步也是在ngx_event_core_module模块的init_process方法进行设置的。

这里还有reusable_connections_queue可复用连接池，其是一个双向链表，其存在可以被复用的连接，当连接池剩余数量较少时，会优先释放可费用连接池中的连接，来进行建立新连接。

对于连接池，封装了如下方法。

#### ngx_get_connection获取空闲事件

```c
ngx_connection_t *
ngx_get_connection(ngx_socket_t s, ngx_log_t *log)
{
    ngx_uint_t         instance;
    ngx_event_t       *rev, *wev;
    ngx_connection_t  *c;

    /* disable warning: Win32 SOCKET is u_int while UNIX socket is int */
    // epoll不使用files，这里不做接收
    if (ngx_cycle->files && (ngx_uint_t) s >= ngx_cycle->files_n) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "the new socket has number %d, "
                      "but only %ui files are available",
                      s, ngx_cycle->files_n);
        return NULL;
    }
    // 加速释放可复用连接，下文会详细介绍
    ngx_drain_connections((ngx_cycle_t *) ngx_cycle);
    // 获取当前的空闲连接
    c = ngx_cycle->free_connections;
    // 如果没有空闲连接了，返回NULL
    if (c == NULL) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "%ui worker_connections are not enough",
                      ngx_cycle->connection_n);

        return NULL;
    }
    // 变更空闲连接池指针，后移一个
    ngx_cycle->free_connections = c->data;
    ngx_cycle->free_connection_n--;

    if (ngx_cycle->files && ngx_cycle->files[s] == NULL) {
        ngx_cycle->files[s] = c;
    }
    // 由于ngx_connection_t和ngx_event_t都已经在之前被使用过，因此要先清除已经使用的数据，重新赋值
    rev = c->read;
    wev = c->write;
    // 清除connection数据
    ngx_memzero(c, sizeof(ngx_connection_t));
    // 重新绑定事件
    c->read = rev;
    c->write = wev;
    // 设置描述符
    c->fd = s;
    c->log = log;

    instance = rev->instance;
    // 清除数据
    ngx_memzero(rev, sizeof(ngx_event_t));
    ngx_memzero(wev, sizeof(ngx_event_t));

    rev->instance = !instance;
    wev->instance = !instance;

    rev->index = NGX_INVALID_INDEX;
    wev->index = NGX_INVALID_INDEX;
    // 绑定事件到连接
    rev->data = c;
    wev->data = c;
    // 设置写事件可写
    wev->write = 1;

    return c;
}
```

#### ngx_free_connection释放连接

```c
void
ngx_free_connection(ngx_connection_t *c)
{
    // c添加到free_connections链表的头部
    c->data = ngx_cycle->free_connections;
    ngx_cycle->free_connections = c;
    ngx_cycle->free_connection_n++;

    if (ngx_cycle->files && ngx_cycle->files[c->fd] == c) {
        ngx_cycle->files[c->fd] = NULL;
    }
}
```

#### ngx_reusable_connection变更连接可复用性

```c
void
ngx_reusable_connection(ngx_connection_t *c, ngx_uint_t reusable)
{
    ngx_log_debug1(NGX_LOG_DEBUG_CORE, c->log, 0,
                   "reusable connection: %ui", reusable);
    // 如果连接原本是可复用的
    if (c->reusable) {
        // 从可复用双向队列中删除
        ngx_queue_remove(&c->queue);
        ngx_cycle->reusable_connections_n--;

#if (NGX_STAT_STUB)
        (void) ngx_atomic_fetch_add(ngx_stat_waiting, -1);
#endif
    }
    // 重新设置可复用性
    c->reusable = reusable;
    // 如果是可复用的
    if (reusable) {
        /* need cast as ngx_cycle is volatile */
        // 将连接添加到reusable_connections_queue头部
        ngx_queue_insert_head(
            (ngx_queue_t *) &ngx_cycle->reusable_connections_queue, &c->queue);
        ngx_cycle->reusable_connections_n++;

#if (NGX_STAT_STUB)
        (void) ngx_atomic_fetch_add(ngx_stat_waiting, 1);
#endif
    }
}
```

这里可能有个疑问，为啥要不考虑可复用性就先执行删除操作。个人理解这与其释放顺序有关，对于先加入的可复用连接，预期会是先释放的，因此这里重新设置可执行性时，相当于重新设置了舒服的优先顺序，因此即使可复用性是否变更，都要先执行删除操作。对于先加入的先删除，较为简单，添加的时候是向链表头部添加，删除的时候从链表的最后进行遍历。

#### ngx_drain_connections加速释放可复用连接

```c
static void
ngx_drain_connections(ngx_cycle_t *cycle)
{
    ngx_uint_t         i, n;
    ngx_queue_t       *q;
    ngx_connection_t  *c;
    // 如果还有充足的连接，或者可复用连接为0，则直接返回。
    if (cycle->free_connection_n > cycle->connection_n / 16
        || cycle->reusable_connections_n == 0)
    {
        return;
    }
    
    // 上一此加速释放连接事件与当前事件不一致。这里的时间是缓存中的事件，如果一致说明，还未触发缓存中事件的变更
    if (cycle->connections_reuse_time != ngx_time()) {
        cycle->connections_reuse_time = ngx_time();

        ngx_log_error(NGX_LOG_WARN, cycle->log, 0,
                      "%ui worker_connections are not enough, "
                      "reusing connections",
                      cycle->connection_n);
    }
    // 获取要释放连接的个数，最少1个
    n = ngx_max(ngx_min(32, cycle->reusable_connections_n / 8), 1);
    // 遍历
    for (i = 0; i < n; i++) {
        // 如果可复用连接已经为空，则break
        if (ngx_queue_empty(&cycle->reusable_connections_queue)) {
            break;
        }
        // 从双向链表的最后获取一个链表元素
        q = ngx_queue_last(&cycle->reusable_connections_queue);
        // 通过链表元素的地址，获取链接地址
        c = ngx_queue_data(q, ngx_connection_t, queue);

        ngx_log_debug0(NGX_LOG_DEBUG_CORE, c->log, 0,
                       "reusing connection");
        // 标志连接关闭
        c->close = 1;
        // 执行连接对应读事件的处理函数，注意，最终是否能够释放由对应的处理函数决定，即调用不一定保证一定能够释放当前连接
        c->read->handler(c->read);
    }
}

```

注意，这是静态方法，即是连接内部调用的方法。

## 事件处理方法

事件驱动，必然定义了很多对于事件的处理方法，这里进行详细介绍，主要包括epoll模块action中定义的一系列方法和一些主要的事件处理方法。

先来看一下epoll模块action定义的事件。

```
static ngx_event_module_t  ngx_epoll_module_ctx = {
    &epoll_name,
    ngx_epoll_create_conf,               /* create configuration */
    ngx_epoll_init_conf,                 /* init configuration */
    // 对于下面的函数，后续会详细介绍
    {
        ngx_epoll_add_event,             /* add an event */
        ngx_epoll_del_event,             /* delete an event */
        ngx_epoll_add_event,             /* enable an event */
        ngx_epoll_del_event,             /* disable an event */
        ngx_epoll_add_connection,        /* add an connection */
        ngx_epoll_del_connection,        /* delete an connection */
#if (NGX_HAVE_EVENTFD)
        ngx_epoll_notify,                /* trigger a notify */
#else
        NULL,                            /* trigger a notify */
#endif
        ngx_epoll_process_events,        /* process the events */
        ngx_epoll_init,                  /* init the events */
        ngx_epoll_done,                  /* done the events */
    }
};
```

### ngx_epoll_init初始化事件模块

首先被调用的方法就是初始化事件模块，其处理过程在ngx_event_core_module的init_process方法中，即worker子进程刚启动后。其逻辑如下：

```c
static ngx_int_t
ngx_epoll_init(ngx_cycle_t *cycle, ngx_msec_t timer)
{
    ngx_epoll_conf_t  *epcf;
    // 获取epoll事件模块创建的ngx_epoll_conf_t结构
    epcf = ngx_event_get_conf(cycle->conf_ctx, ngx_epoll_module);
    // 如果epoll模块的描述符还未创建
    if (ep == -1) {
        // 执行构建
        ep = epoll_create(cycle->connection_n / 2);

        if (ep == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "epoll_create() failed");
            return NGX_ERROR;
        }

#if (NGX_HAVE_EVENTFD)
        // 初始化通知机制
        if (ngx_epoll_notify_init(cycle->log) != NGX_OK) {
            ngx_epoll_module_ctx.actions.notify = NULL;
        }
#endif

#if (NGX_HAVE_FILE_AIO)
        // 初始化文件异步I/O
        ngx_epoll_aio_init(cycle, epcf);
#endif

#if (NGX_HAVE_EPOLLRDHUP)
        // 测试事件能否监控客户端主动关闭连接
        ngx_epoll_test_rdhup(cycle);
#endif
    }
    // 如果用于事件返回时存储事件的数组小于配置中设置的每次返回的数量，则重新分配空间
    if (nevents < epcf->events) {
        if (event_list) {
            ngx_free(event_list);
        }

        event_list = ngx_alloc(sizeof(struct epoll_event) * epcf->events,
                               cycle->log);
        if (event_list == NULL) {
            return NGX_ERROR;
        }
    }

    nevents = epcf->events;

    // 对全局变量ngx_io和ngx_event_actions赋值，读写描述符操作
    ngx_io = ngx_os_io;
    
    ngx_event_actions = ngx_epoll_module_ctx.actions;

#if (NGX_HAVE_CLEAR_EVENT)
    // 设置事件flags
    ngx_event_flags = NGX_USE_CLEAR_EVENT
#else
    ngx_event_flags = NGX_USE_LEVEL_EVENT
#endif
                      |NGX_USE_GREEDY_EVENT
                      |NGX_USE_EPOLL_EVENT;

    return NGX_OK;
}
```



#### ngx_epoll_notify_init初始化消息提醒

```c
static ngx_int_t
ngx_epoll_notify_init(ngx_log_t *log)
{
    struct epoll_event  ee;

    // 获取事件描述符
#if (NGX_HAVE_SYS_EVENTFD_H)
    notify_fd = eventfd(0, 0);
#else
    notify_fd = syscall(SYS_eventfd, 0);
#endif

    if (notify_fd == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "eventfd() failed");
        return NGX_ERROR;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, log, 0,
                   "notify eventfd: %d", notify_fd);
    // 设置全局变量notify_event（ngx_event_t）
    notify_event.handler = ngx_epoll_notify_handler;
    notify_event.log = log;
    notify_event.active = 1;
    // 设置全局变量notify_conn（ngx_connection_t）
    notify_conn.fd = notify_fd;
    notify_conn.read = &notify_event;
    notify_conn.log = log;
    // 设置epoll事件，监控可读，并且设置边缘触发
    ee.events = EPOLLIN|EPOLLET;
    ee.data.ptr = &notify_conn;
    // 将消息提醒提交到全局的epoll事件监控中
    if (epoll_ctl(ep, EPOLL_CTL_ADD, notify_fd, &ee) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "epoll_ctl(EPOLL_CTL_ADD, eventfd) failed");

        if (close(notify_fd) == -1) {
            ngx_log_error(NGX_LOG_ALERT, log, ngx_errno,
                            "eventfd close() failed");
        }

        return NGX_ERROR;
    }

    return NGX_OK;
}
```

##### notify事件触发的处理方法

对于该事件的处理函数为：

```c
static void
ngx_epoll_notify_handler(ngx_event_t *ev)
{
    ssize_t               n;
    uint64_t              count;
    ngx_err_t             err;
    ngx_event_handler_pt  handler;
    // 如果连接使用已经超过最大值了，则清除eventfd中的计数，重新对index计数
    if (++ev->index == NGX_MAX_UINT32_VALUE) {
        ev->index = 0;

        n = read(notify_fd, &count, sizeof(uint64_t));

        err = ngx_errno;

        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                       "read() eventfd %d: %z count:%uL", notify_fd, n, count);

        if ((size_t) n != sizeof(uint64_t)) {
            ngx_log_error(NGX_LOG_ALERT, ev->log, err,
                          "read() eventfd %d failed", notify_fd);
        }
    }
    // 执行事件对应的函数
    handler = ev->data;
    handler(ev);
}
```

由于epoll使用的是边缘触发，因此不需要每次都执行read操作，因此这里只在必须使用的时候，才进行了read的调用。

##### 向notify中写入事件

对应的，向notify_fd中写入事件的方法为：

ngx_epoll_module的action中的ngx_epoll_notify方法，其处理逻辑为：

```c
static ngx_int_t
ngx_epoll_notify(ngx_event_handler_pt handler)
{
    static uint64_t inc = 1;
    // 设置消息处理函数
    notify_event.data = handler;
    // 向notify_fd的计数器增加一个值，下次epoll_wait将会返回notify_fd对应的事件
    if ((size_t) write(notify_fd, &inc, sizeof(uint64_t)) != sizeof(uint64_t)) {
        ngx_log_error(NGX_LOG_ALERT, notify_event.log, ngx_errno,
                      "write() to eventfd %d failed", notify_fd);
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

#### ngx_epoll_aio_init初始化文件异步I/O

```c
static void
ngx_epoll_aio_init(ngx_cycle_t *cycle, ngx_epoll_conf_t *epcf)
{
    int                 n;
    struct epoll_event  ee;
    
    // 初始化全局变量，ngx_eventfd事件
#if (NGX_HAVE_SYS_EVENTFD_H)
    ngx_eventfd = eventfd(0, 0);
#else
    ngx_eventfd = syscall(SYS_eventfd, 0);
#endif
    // 如果出错，表示不支持异步I/O
    if (ngx_eventfd == -1) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                      "eventfd() failed");
        // 设置异步I/O标志位为0
        ngx_file_aio = 0;
        return;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "eventfd: %d", ngx_eventfd);

    n = 1;
    // 设置事件描述符为非阻塞的
    if (ioctl(ngx_eventfd, FIONBIO, &n) == -1) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                      "ioctl(eventfd, FIONBIO) failed");
        goto failed;
    }
    // 使用配置中的aio_requests初始化文件I/O描述符全局变量ngx_aio_ctx
    if (io_setup(epcf->aio_requests, &ngx_aio_ctx) == -1) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                      "io_setup() failed");
        goto failed;
    }
    
    // 设置全局变量ngx_eventfd_event，用于异步I/O的读事件
    ngx_eventfd_event.data = &ngx_eventfd_conn;
    // 设置处理函数
    ngx_eventfd_event.handler = ngx_epoll_eventfd_handler;
    ngx_eventfd_event.log = cycle->log;
    ngx_eventfd_event.active = 1;
    ngx_eventfd_conn.fd = ngx_eventfd;
    // 绑定全局的ngx_eventfd_conn异步I/O的连接
    ngx_eventfd_conn.read = &ngx_eventfd_event;
    ngx_eventfd_conn.log = cycle->log;
    // 添加可读事件并设置边缘触发到全局事件驱动中
    ee.events = EPOLLIN|EPOLLET;
    ee.data.ptr = &ngx_eventfd_conn;

    if (epoll_ctl(ep, EPOLL_CTL_ADD, ngx_eventfd, &ee) != -1) {
        return;
    }

    ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                  "epoll_ctl(EPOLL_CTL_ADD, eventfd) failed");
    // 如果没有成功添加文件异步I/O事件则销毁异步I/O描述符
    if (io_destroy(ngx_aio_ctx) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "io_destroy() failed");
    }

failed:

    if (close(ngx_eventfd) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "eventfd close() failed");
    }

    ngx_eventfd = -1;
    ngx_aio_ctx = 0;
    ngx_file_aio = 0;
}
```

这里将系统的eventfd和文件异步I/O描述符和epoll三者进行关联。在epoll中监控系统eventfd（ngx_eventfd），在异步I/O中添加读取事件时，利用iocb中的aio_resfd（异步通知的fd）绑定到eventfd（ngx_eventfd）上。当异步I/O事件后，epoll_wait()将会返回eventfd（ngx_eventfd）对应的事件，并将eventfd（ngx_eventfd）计数加一，其中事件的ptr指针，指向ngx_eventfd_conn，执行ngx_eventfd_conn的read事件中的方法ngx_epoll_eventfd_handler。ngx_epoll_eventfd_handler方法将通过异步I/O描述符全局变量ngx_aio_ctx获取已完成的事件。

下面分别看一下如何条件异步I/O事件和处理异步I/O事件。

##### 增加异步I/O事件

`ngx_file_aio_read`函数向ngx_aio_ctx描述符中条件异步I/O事件。

其中ngx_event_aio_t结构定义了添加异步I/O的相关结构，定义如下：

```c
struct ngx_event_aio_s {    void                      *data;    // 异步I/O事件完成后被调用的方法    ngx_event_handler_pt       handler;    ngx_file_t                *file;    ngx_fd_t                   fd;#if (NGX_HAVE_AIO_SENDFILE || NGX_COMPAT)    ssize_t                  (*preload_handler)(ngx_buf_t *file);#endif#if (NGX_HAVE_EVENTFD)    // 对应io_getevents返回的ngx_event_aio_t结构中的res，用于判断标志是否执行成功    int64_t                    res;#endif#if !(NGX_HAVE_EVENTFD) || (NGX_TEST_BUILD_EPOLL)    ngx_err_t                  err;    size_t                     nbytes;#endif    // 添加异步I/O事件的结构    ngx_aiocb_t                aiocb;    // 绑定的事件，用于完成异步I/O后执行相应操作    ngx_event_t                event;};
```

对于文件来说，nginx定义了ngx_file_t结构，其定义如下：

```c
typedef struct stat              ngx_file_info_t;

struct ngx_file_s {
    // 文件描述符
    ngx_fd_t                   fd;
    // 文件名
    ngx_str_t                  name;
    // 文件信息
    ngx_file_info_t            info;
    // 游标偏移
    off_t                      offset;
    off_t                      sys_offset;

    ngx_log_t                 *log;

#if (NGX_THREADS || NGX_COMPAT)
    ngx_int_t                (*thread_handler)(ngx_thread_task_t *task,
                                               ngx_file_t *file);
    void                      *thread_ctx;
    ngx_thread_task_t         *thread_task;
#endif

#if (NGX_HAVE_FILE_AIO || NGX_COMPAT)
    // 异步I/O结构
    ngx_event_aio_t           *aio;
#endif

    unsigned                   valid_info:1;
    unsigned                   directio:1;
};
```

具体添加函数为：

```c
ssize_t
ngx_file_aio_read(ngx_file_t *file, u_char *buf, size_t size, off_t offset,
    ngx_pool_t *pool)
{
    ngx_err_t         err;
    struct iocb      *piocb[1];
    ngx_event_t      *ev;
    ngx_event_aio_t  *aio;
    // 如果不支持异步I/O则使用线程读取的系统调用方式，这里不讨论
    if (!ngx_file_aio) {
        return ngx_read_file(file, buf, size, offset);
    }
    // 如果文件中的aio未设置，则初始化文件的aio
    if (file->aio == NULL && ngx_file_aio_init(file, pool) != NGX_OK) {
        return NGX_ERROR;
    }
    // 获取文件绑定的aio
    aio = file->aio;
    // 获取异步I/O对应的事件
    ev = &aio->event;
    // 如果事件是非可读的报错
    if (!ev->ready) {
        ngx_log_error(NGX_LOG_ALERT, file->log, 0,
                      "second aio post for \"%V\"", &file->name);
        return NGX_AGAIN;
    }

    ngx_log_debug4(NGX_LOG_DEBUG_CORE, file->log, 0,
                   "aio complete:%d @%O:%uz %V",
                   ev->complete, offset, size, &file->name);
    // 如果事件已经完成
    if (ev->complete) {
        ev->active = 0;
        ev->complete = 0;
        // 如果事件完成，并且res（异步事件完成返回）正确，则设置errno为0，即无错误，返回
        if (aio->res >= 0) {
            ngx_set_errno(0);
            return aio->res;
        }
        // 否则设置报错，返回error
        ngx_set_errno(-aio->res);

        ngx_log_error(NGX_LOG_CRIT, file->log, ngx_errno,
                      "aio read \"%s\" failed", file->name.data);

        return NGX_ERROR;
    }
    // 分配aiocb空间
    ngx_memzero(&aio->aiocb, sizeof(struct iocb));
    // 设置aiocb。绑定aio_data为事件ev
    aio->aiocb.aio_data = (uint64_t) (uintptr_t) ev;
    aio->aiocb.aio_lio_opcode = IOCB_CMD_PREAD;
    aio->aiocb.aio_fildes = file->fd;
    aio->aiocb.aio_buf = (uint64_t) (uintptr_t) buf;
    aio->aiocb.aio_nbytes = size;
    aio->aiocb.aio_offset = offset;
    // 设置异步读事件
    aio->aiocb.aio_flags = IOCB_FLAG_RESFD;
    // 绑定事件完成的通知到全局变量ngx_eventfd
    aio->aiocb.aio_resfd = ngx_eventfd;
    // 设置事件完成的执行函数
    ev->handler = ngx_file_aio_event_handler;

    piocb[0] = &aio->aiocb;
    // 向ngx_aio_ctx添加读事件
    if (io_submit(ngx_aio_ctx, 1, piocb) == 1) {
        ev->active = 1;
        ev->ready = 0;
        ev->complete = 0;

        return NGX_AGAIN;
    }

    err = ngx_errno;

    if (err == NGX_EAGAIN) {
        return ngx_read_file(file, buf, size, offset);
    }

    ngx_log_error(NGX_LOG_CRIT, file->log, err,
                  "io_submit(\"%V\") failed", &file->name);

    if (err == NGX_ENOSYS) {
        ngx_file_aio = 0;
        return ngx_read_file(file, buf, size, offset);
    }

    return NGX_ERROR;
}
```

这里初始化aio方法如下：

```
ngx_int_t
ngx_file_aio_init(ngx_file_t *file, ngx_pool_t *pool)
{
    ngx_event_aio_t  *aio;
    // 分配内存
    aio = ngx_pcalloc(pool, sizeof(ngx_event_aio_t));
    if (aio == NULL) {
        return NGX_ERROR;
    }
    // 绑定文件
    aio->file = file;
    aio->fd = file->fd;
    // 绑定event的data为aio本身
    aio->event.data = aio;
    aio->event.ready = 1;
    aio->event.log = file->log;
    // 设置文件aio为aio
    file->aio = aio;

    return NGX_OK;
}

```

注意，上述的处理中对ngx_event_aio_t结构的handler未进行处理，但该值是必须要有的，因为后续会使用（下一节介绍），这里对handler的赋值可能在执行ngx_file_aio_read前，即已经完成file中aio的分配，并设置了aio的handler。或者在函数返回后，设置file中的aio成员handler。因为在所有事件传递中都是传递的指针，因此在事件添加后再进行对aio的变更并不会有问题。

##### epoll事件异步I/O返回的处理

在初始化aio时，设置的事件触发后的执行函数为ngx_epoll_eventfd_handler，其执行逻辑为：

```c
static void
ngx_epoll_eventfd_handler(ngx_event_t *ev)
{
    int               n, events;
    long              i;
    uint64_t          ready;
    ngx_err_t         err;
    ngx_event_t      *e;
    ngx_event_aio_t  *aio;
    struct io_event   event[64];
    struct timespec   ts;

    ngx_log_debug0(NGX_LOG_DEBUG_EVENT, ev->log, 0, "eventfd handler");
    // 从ngx_eventfd中读取异步I/O事件完成数量，存到ready中。
    n = read(ngx_eventfd, &ready, 8);

    err = ngx_errno;

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, ev->log, 0, "eventfd: %d", n);
    // 如果读取到的字符数量不为8，则表示存在错误，进行错误处理
    if (n != 8) {
        if (n == -1) {
            if (err == NGX_EAGAIN) {
                return;
            }

            ngx_log_error(NGX_LOG_ALERT, ev->log, err, "read(eventfd) failed");
            return;
        }

        ngx_log_error(NGX_LOG_ALERT, ev->log, 0,
                      "read(eventfd) returned only %d bytes", n);
        return;
    }
    // 设置io_getevents超时时间为0，即立即返回
    ts.tv_sec = 0;
    ts.tv_nsec = 0;
    // 如果还有事件未处理完成
    while (ready) {
        // 从ngx_aio_ctx获取已经完成的异步事件，最少返回1个，最多返回64个
        events = io_getevents(ngx_aio_ctx, 1, 64, event, &ts);

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                       "io_getevents: %d", events);
        // 如果获取到事件大于0
        if (events > 0) {
            // 减去此次处理的事件
            ready -= events;
            // 遍历每个事件
            for (i = 0; i < events; i++) {

                ngx_log_debug4(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                               "io_event: %XL %XL %L %L",
                                event[i].data, event[i].obj,
                                event[i].res, event[i].res2);
                // 获取事件对应的data数据，其中data为添加事件时aiocb.aio_data，即aiocb对应的事件。
                e = (ngx_event_t *) (uintptr_t) event[i].data;
                // 设置异步事件为完成状态
                e->complete = 1;
                // 设置事件不活跃了
                e->active = 0;
                // 事件处于可处理状态
                e->ready = 1;

                aio = e->data;
                // 设置事件绑定的aiocb结构的res为事件返回的res，即是否成功（>=0表示成功）
                aio->res = event[i].res;
                // 向ngx_posted_events队列增加事件。后续执行事件e的handler方法
                ngx_post_event(e, &ngx_posted_events);
            }

            continue;
        }
        // 如果获取到的事件数量为0，则直接返回
        if (events == 0) {
            return;
        }
        // 如果出错，则报错返回
        /* events == -1 */
        ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                      "io_getevents() failed");
        return;
    }
}
```

这里虽然我们通过读取read(ngx_eventfd, &ready, 8)获取已经准备完成的异步I/O事件数量，但实际处理的数量不一定就是ready中的值。有可能大于，有可能小于，有可能等于。

这是因为，在调用read后，循环执行的过程中，可能还有新的异步I/O事件完成，此时，在while循环中可能将新的完成事件也返回，这时，本轮就会处理比从read中读取的异步I/O完成的事件多的时间。同时这可能导致下一轮循环时，处理的事件少于从read中读取的数量，因此在上一轮已经处理了一部分了。

这里执行的事件处理函数为在添加异步I/O事件时，事件绑定的handler：

```c
static void
ngx_file_aio_event_handler(ngx_event_t *ev)
{
    ngx_event_aio_t  *aio;
    // 获取事件指向的ngx_event_aio_t结构，这里是添加事件时的ngx_event_aio_t结构的ngx_aiocb_t成员aiocb
    aio = ev->data;

    ngx_log_debug2(NGX_LOG_DEBUG_CORE, ev->log, 0,
                   "aio event handler fd:%d %V", aio->fd, &aio->file->name);
    // 执行aiocb绑定的handler函数
    aio->handler(ev);
}
```

因为最终会执行aiocb绑定的handler方法，因此上文说过在添加异步I/O一定要保证设置了handler方法。

#### ngx_epoll_test_rdhup测试epoll监听事件

ngx_epoll_test_rdhup方法用来测试是否能够监听客户端主动关闭连接。

```c
static void
ngx_epoll_test_rdhup(ngx_cycle_t *cycle)
{
    int                 s[2], events;
    struct epoll_event  ee;
    // 建立一组unix域套接字
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, s) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "socketpair() failed");
        return;
    }
    // 事件建库，边缘触发，可读（主动关闭连接其实属于可读）和主动关闭连接
    ee.events = EPOLLET|EPOLLIN|EPOLLRDHUP;
    // 在第一个套接字中监听事件
    if (epoll_ctl(ep, EPOLL_CTL_ADD, s[0], &ee) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "epoll_ctl() failed");
        goto failed;
    }
    // 关闭第二个套接字
    if (close(s[1]) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "close() failed");
        s[1] = -1;
        goto failed;
    }

    s[1] = -1;
    // 获取触发的事件
    events = epoll_wait(ep, &ee, 1, 5000);

    if (events == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "epoll_wait() failed");
        goto failed;
    }
    // 如果正常获取到事件
    if (events) {
        // 是否存在建库主动关闭连接
        ngx_use_epoll_rdhup = ee.events & EPOLLRDHUP;

    } else {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, NGX_ETIMEDOUT,
                      "epoll_wait() timed out");
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "testing the EPOLLRDHUP flag: %s",
                   ngx_use_epoll_rdhup ? "success" : "fail");

failed:

    if (s[1] != -1 && close(s[1]) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "close() failed");
    }

    if (close(s[0]) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "close() failed");
    }
}
```



### ngx_epoll_process_events事件处理

```c
// timer为调用epoll_wait等待时间，flags为按位的flags
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    int                events;
    uint32_t           revents;
    ngx_int_t          instance, i;
    ngx_uint_t         level;
    ngx_err_t          err;
    ngx_event_t       *rev, *wev;
    ngx_queue_t       *queue;
    ngx_connection_t  *c;

    /* NGX_TIMER_INFINITE == INFTIM */

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "epoll timer: %M", timer);
    // 获取触发事件
    events = epoll_wait(ep, event_list, (int) nevents, timer);

    err = (events == -1) ? ngx_errno : 0;
    // 如果flags指示需要更新缓存中的时间，或者定义更新时间的信号已触发，则更新缓存中事件。时间相关内容在后续定时事件介绍
    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {
        ngx_time_update();
    }

    if (err) {
        // 错误时记录日志
        if (err == NGX_EINTR) {

            if (ngx_event_timer_alarm) {
                ngx_event_timer_alarm = 0;
                return NGX_OK;
            }

            level = NGX_LOG_INFO;

        } else {
            level = NGX_LOG_ALERT;
        }

        ngx_log_error(level, cycle->log, err, "epoll_wait() failed");
        return NGX_ERROR;
    }

    // 返回事件数量为0
    if (events == 0) {
        // 如果是非阻塞调用，返回0为正常
        if (timer != NGX_TIMER_INFINITE) {
            return NGX_OK;
        }

        ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                      "epoll_wait() returned no events without timeout");
        return NGX_ERROR;
    }

    // 处理每个事件
    for (i = 0; i < events; i++) {
        // 获取事件对应的ngx_connection_t结构
        c = event_list[i].data.ptr;
        // 获取连接地址中存储的instance标志位
        instance = (uintptr_t) c & 1;
        // 恢复正常的连接地址
        c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);
        // 获取连接对应的读事件
        rev = c->read;
        // 如果连接对应的套接字已经被关闭或者为过期事件，则跳过执行
        if (c->fd == -1 || rev->instance != instance) {

            /*
             * the stale event from a file descriptor
             * that was just closed in this iteration
             */

            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll: stale event %p", c);
            continue;
        }
        // 获取触发事件类型
        revents = event_list[i].events;

        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "epoll: fd:%d ev:%04XD d:%p",
                       c->fd, revents, event_list[i].data.ptr);
        /* 如果对应的连接发送错误或者被挂起，则赋值事件为可读和可写，让对应的读事件和写事件来进行对错误和挂起进行响应,因此读写事件对应的处理函数，不能单纯只处理读写事件，需要判断是否发送错误 */
        if (revents & (EPOLLERR|EPOLLHUP)) {
            ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll_wait() error on fd:%d ev:%04XD",
                           c->fd, revents);

            /*
             * if the error events were returned, add EPOLLIN and EPOLLOUT
             * to handle the events at least in one active handler
             */

            revents |= EPOLLIN|EPOLLOUT;
        }

#if 0
        if (revents & ~(EPOLLIN|EPOLLOUT|EPOLLERR|EPOLLHUP)) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                          "strange epoll_wait() events fd:%d ev:%04XD",
                          c->fd, revents);
        }
#endif
        // 事件可读处理，需要读事件依然活跃
        if ((revents & EPOLLIN) && rev->active) {

#if (NGX_HAVE_EPOLLRDHUP)
            // 如果客户端关闭连接，则标识位置1
            if (revents & EPOLLRDHUP) {
                rev->pending_eof = 1;
            }
#endif
            // 准备处理的标志位置1
            rev->ready = 1;
            rev->available = -1;
            // 如果flags标识位表明延后处理
            if (flags & NGX_POST_EVENTS) {
                // 如果事件是监听连接事件，则加入ngx_posted_accept_events队列，否则加入ngx_posted_events队列
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;

                ngx_post_event(rev, queue);

            } else {
                // 否则立即执行
                rev->handler(rev);
            }
        }
        // 获取写事件
        wev = c->write;
        // 事件捕获为写事件，并且写事件活跃
        if ((revents & EPOLLOUT) && wev->active) {
            // 如果写事件过期了，则跳过处理
            if (c->fd == -1 || wev->instance != instance) {

                /*
                 * the stale event from a file descriptor
                 * that was just closed in this iteration
                 */

                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                               "epoll: stale event %p", c);
                continue;
            }

            wev->ready = 1;
#if (NGX_THREADS)
            wev->complete = 1;
#endif
             // 判断是否立即执行
            if (flags & NGX_POST_EVENTS) {
                ngx_post_event(wev, &ngx_posted_events);

            } else {
                wev->handler(wev);
            }
        }
    }

    return NGX_OK;
}
```

这里有两点需要详细进行介绍，一个是延迟处理，另一个是事件过期标志。

事件处理支持将每个触发的事件进行延后处理，其实现是一个双向链表，这里将要延后处理的事件添加到上线链表中，在执行完成`ngx_epoll_process_events`事件后，再依次执行添加进双向链表中的事件，执行每个的handler函数。

`ngx_events_t`中存在一个`instance`成员用来和连接的地址的最后一位比对，判断是否为过期连接。一般我们在释放连接时，会将对应的fd置为-1。这样也可以作为事件过期标志，但是这无法解决所有问题。例如如下一个场景：有三个事件触发了，第一个事件处理中·关闭了一个连接，这个连接恰好对应第三个事件，这样的化，处理第三个事件就是过期事件了。第一个事件会可能会调用`ngx_free_connection`将fd置为-1，并将连接归还连接池中。但第二个事件建立一个新连接，会调用`ngx_get_connection`(查看连接池章节)从连接池中获取一个连接，这很可能会是我们上一个事件释放的连接。这时会重新对连接的fd赋值，此时只使用fd就无法判断过期状态了。

对于上述问题解决方案是，在`ngx_get_connection`中获取连接池时，会将连接对应读写事件的instance置反，这时就可以通过判断事件ptr指针的最后一位（利用指针最后两位一定为0，使用最后一位作为标志位）是否与事件的instance一致来判断是否为过期事件。正常情况下事件的instance和事件返回的ptr指针最后一位一致。

将事件的instance和事件的ptr指针设置为相同发生在添加事件时，下文会详细介绍。

因此整体流程是：添加事件时，连接绑定的事件的instance和事件返回的ptr指针（指向连接）的最后一位一致，之后释放连接，当再次使用连接时，将连接对应的读写事件结构的instance取反，此时事件的instance和ptr的最后一位不一致，用于判断为过期事件，最后再将事件添加到epoll驱动时，将监控事件的结构ptr（执行连接）的最后一位和连接绑定的事件的instance统一。

不失一般性，假设从开始将事件加入epoll时，ptr指针的最后一位和事件的instance都是0（也可以都是1，但要一致）开始，则这两个变量值的随事件流转变化可以表示为如下图形式：

[![5J4rrD.png](https://z3.ax1x.com/2021/10/16/5J4rrD.png)](https://imgtu.com/i/5J4rrD)

对于添加事件到epoll接下来详细介绍。

### ngx_epoll_add_event添加事件到epoll中

该方法向epoll中添加监听事件：

```c
// event表示是读事件还是写事件，flags标注关注的事件，即epoll_event结构的events
static ngx_int_t
ngx_epoll_add_event(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags)
{
    int                  op;
    uint32_t             events, prev;
    ngx_event_t         *e;
    ngx_connection_t    *c;
    struct epoll_event   ee;
    // 获取事件对应的连接
    c = ev->data;

    events = (uint32_t) event;
    /* 未避免同一个连接被加入两次到epoll中，在加入读事件或者写事件时，需要先判断另一个事件是否已经处于监听状态了 */
    // 读事件处理
    if (event == NGX_READ_EVENT) {
        // 获取读事件，并存储读事件监听的事件
        e = c->write;
        prev = EPOLLOUT;
// 不会执行
#if (NGX_READ_EVENT != EPOLLIN|EPOLLRDHUP)
        events = EPOLLIN|EPOLLRDHUP;
#endif

    } else {
        // 写事件处理
        // 获取读事件，记录读事件监听
        e = c->read;
        prev = EPOLLIN|EPOLLRDHUP;
// linux下不会执行
#if (NGX_WRITE_EVENT != EPOLLOUT)
        events = EPOLLOUT;
#endif
    }
    // 如果对应的另一个事件是活跃的，表示是只需要对事件进行变更，变更时要保留原本的监听事件
    if (e->active) {
        op = EPOLL_CTL_MOD;
        events |= prev;

    } else {
        // 否则添加事件
        op = EPOLL_CTL_ADD;
    }
// linux不会执行
#if (NGX_HAVE_EPOLLEXCLUSIVE && NGX_HAVE_EPOLLRDHUP)
    if (flags & NGX_EXCLUSIVE_EVENT) {
        events &= ~EPOLLRDHUP;
    }
#endif
    // 设置关注事件
    ee.events = events | (uint32_t) flags;
    // 设置执行连接的指针，并设置最后一位
    ee.data.ptr = (void *) ((uintptr_t) c | ev->instance);

    ngx_log_debug3(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                   "epoll add event: fd:%d op:%d ev:%08XD",
                   c->fd, op, ee.events);
    // 添加事件
    if (epoll_ctl(ep, op, c->fd, &ee) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                      "epoll_ctl(%d, %d) failed", op, c->fd);
        return NGX_ERROR;
    }
    // 设置事件为活跃状态
    ev->active = 1;
#if 0
    ev->oneshot = (flags & NGX_ONESHOT_EVENT) ? 1 : 0;
#endif

    return NGX_OK;
}
```



### ngx_epoll_del_event从epoll中删除事件

```c
static ngx_int_t
ngx_epoll_del_event(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags)
{
    int                  op;
    uint32_t             prev;
    ngx_event_t         *e;
    ngx_connection_t    *c;
    struct epoll_event   ee;

    /*
     * when the file descriptor is closed, the epoll automatically deletes
     * it from its queue, so we do not need to delete explicitly the event
     * before the closing the file descriptor
     */
    // 如果是关闭事件，则直接将active置为0即可，即使事件触发，也不会做任何操作。同时，如果文件描述符被关闭，事件将不会再触发
    if (flags & NGX_CLOSE_EVENT) {
        ev->active = 0;
        return NGX_OK;
    }
    // 获取对应连接结构
    c = ev->data;
    // 与添加事件时一致，删除事件时也需要先判断另一个事件是否在监听状态，如果是则只能变更监听事件
    // 写事件处理
    if (event == NGX_READ_EVENT) {
        // 获取对应读事件，并记录读事件监听标志
        e = c->write;
        prev = EPOLLOUT;

    } else {
        // 获取对写事件，记录写事件监听标志
        e = c->read;
        prev = EPOLLIN|EPOLLRDHUP;
    }
    // 如果对应的另一个事件依然存活，则变更监听
    if (e->active) {
        op = EPOLL_CTL_MOD;
        ee.events = prev | (uint32_t) flags;
        ee.data.ptr = (void *) ((uintptr_t) c | ev->instance);

    } else {
        // 否则关闭监听
        op = EPOLL_CTL_DEL;
        ee.events = 0;
        ee.data.ptr = NULL;
    }

    ngx_log_debug3(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                   "epoll del event: fd:%d op:%d ev:%08XD",
                   c->fd, op, ee.events);

    if (epoll_ctl(ep, op, c->fd, &ee) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                      "epoll_ctl(%d, %d) failed", op, c->fd);
        return NGX_ERROR;
    }
    // 设置对应事件不再活跃
    ev->active = 0;

    return NGX_OK;
}
```

### ngx_epoll_add_connection

```c
static ngx_int_t
ngx_epoll_add_connection(ngx_connection_t *c)
{
    struct epoll_event  ee;
    // 设置监听可读、可写、客户端主动关闭，和边缘触发
    ee.events = EPOLLIN|EPOLLOUT|EPOLLET|EPOLLRDHUP;
    // 设置过期指针
    ee.data.ptr = (void *) ((uintptr_t) c | c->read->instance);

    ngx_log_debug2(NGX_LOG_DEBUG_EVENT, c->log, 0,
                   "epoll add connection: fd:%d ev:%08XD", c->fd, ee.events);
    // 添加事件
    if (epoll_ctl(ep, EPOLL_CTL_ADD, c->fd, &ee) == -1) {
        ngx_log_error(NGX_LOG_ALERT, c->log, ngx_errno,
                      "epoll_ctl(EPOLL_CTL_ADD, %d) failed", c->fd);
        return NGX_ERROR;
    }
    // 设置读写均活跃
    c->read->active = 1;
    c->write->active = 1;

    return NGX_OK;
}
```



### ngx_epoll_del_connection客户端关闭连接

```c
static ngx_int_t
ngx_epoll_del_connection(ngx_connection_t *c, ngx_uint_t flags)
{
    int                 op;
    struct epoll_event  ee;

    /*
     * when the file descriptor is closed the epoll automatically deletes
     * it from its queue so we do not need to delete explicitly the event
     * before the closing the file descriptor
     */
    // 如果是关闭事件，则设置读写均不再活跃即可
    if (flags & NGX_CLOSE_EVENT) {
        c->read->active = 0;
        c->write->active = 0;
        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, c->log, 0,
                   "epoll del connection: fd:%d", c->fd);
    // 否则从epoll中删除事件
    op = EPOLL_CTL_DEL;
    ee.events = 0;
    ee.data.ptr = NULL;

    if (epoll_ctl(ep, op, c->fd, &ee) == -1) {
        ngx_log_error(NGX_LOG_ALERT, c->log, ngx_errno,
                      "epoll_ctl(%d, %d) failed", op, c->fd);
        return NGX_ERROR;
    }
    // 设置读写均不再活跃
    c->read->active = 0;
    c->write->active = 0;

    return NGX_OK;
}
```

`ngx_epoll_add_connection`和`ngx_epoll_del_connection`与`ngx_epoll_add_event`和`ngx_epoll_del_event`区别是前两者是在连接纬度(包括读写两个事件)进行添加和删除事件，而后两者是在事件纬度（单独的读或者写事件纬度）进行添加或删除事件。

### ngx_epoll_done结束事件处理

在关闭nginx前，会释放epoll模块创建的资源，这时会执行该操作。

```c
static void
ngx_epoll_done(ngx_cycle_t *cycle)
{
    // 关闭全局epoll事件描述符
    if (close(ep) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "epoll close() failed");
    }

    ep = -1;

#if (NGX_HAVE_EVENTFD)
    // 关闭全局消息提醒notify_fd事件描述符
    if (close(notify_fd) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "eventfd close() failed");
    }

    notify_fd = -1;

#endif

#if (NGX_HAVE_FILE_AIO)

    if (ngx_eventfd != -1) {
        // 关闭全局文件异步I/O ngx_aio_ctx 描述符
        if (io_destroy(ngx_aio_ctx) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "io_destroy() failed");
        }

        if (close(ngx_eventfd) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "eventfd close() failed");
        }

        ngx_eventfd = -1;
    }

    ngx_aio_ctx = 0;

#endif
    // 舒服全局epoll_wait返回事件数组分配的地址
    ngx_free(event_list);

    event_list = NULL;
    nevents = 0;
}
```



## 定时器事件

上文介绍的所有模块都是基于I/O操作的事件驱动，这样并不能解决所有问题。这样做不到每个连接内的多阶段处理，因此nginx架构自身实现了基于事件的定时器事件。

定时器的实现是利用一颗以时间作为key的红黑树。在worker进程处理循环中，查找该树中已经触发超时的节点，执行对应的处理函数。因此时间控制就显得尤为重要。

### 缓存时间管理

缓存时间在进程间是不共用的，也就是说，病危使用mmap内存映射将多个进程读写同一个地址的数据。每个worker要维护自己的时间缓存。

处于性能考虑，一般不会在每次需要使用时间时调用`gettimeofday`来更新时间，而是将时间缓存到一些全局变量中，在必要的时候再进行更新。

nginx缓存了如下关于时间的全局变量：

```c
typedef struct {
    // 时间戳，到秒
    time_t      sec;
    // mesc是相对sec的毫秒偏移
    ngx_uint_t  msec;
    // 时区
    ngx_int_t   gmtoff;
} ngx_time_t;

// 事件戳,毫秒级别
extern volatile ngx_msec_t  ngx_current_msec;
// ngx_time_t结构形式的当前事件
extern volatile ngx_time_t  *ngx_cached_time;
// 用于记录error_log的当前时间字符串，格式类似于"1997/09/28 12:09:87"
extern volatile ngx_str_t    ngx_cached_err_log_time;
// 用于http相关的时间字符串，格式类似于 "Mon, 28 Sep 1970 09:00:00 GMT"
extern volatile ngx_str_t    ngx_cached_http_time;
// 用于记录HTTP日志的当前事件字符串，格式类似于："28/Sep/1979:12:00:00 +0600"
extern volatile ngx_str_t    ngx_cached_http_log_time;
// 以ISO 8601标准格式记录下的字符串形式当前事件
extern volatile ngx_str_t    ngx_cached_http_log_iso8601;
extern volatile ngx_str_t    ngx_cached_syslog_time;
```

对于nginx自己定义的定时器事件来说，对于时间是十分敏感的，因此何时更新缓存的时间是是否重要的。

nginx更新缓存事件分为两种情况：

1. 配置了timer_resolution：如果配置中存在该配置，则会在每次worker进程的循环中，判断定时器是否已经触发，触发时进行时间更新。其中worker进程增加时钟信号在ngx_event_core_module模块的int_process方法中（详情可查看上文介绍），事件循环中，对应epoll_wait等待事件为-1，即直到等待有事件触发或者时钟信号触发时，就会更新时间。
2. 当没有配置timer_resolution时，则每次worker进程循环的时候，都会执行事件更新。这时epoll_wait等待事件则取决于定时器事件中当前是否存在超时事件，如果存在，则时间为0，即不等待，直接返回，如果没有，则最多等待最近的一个事件所对应的事件。具体查看ngx_process_events_and_timers函数。

#### ngx_time_init初始化时间

```c
void
ngx_time_init(void)
{
    // 分配缓存数据空间
    ngx_cached_err_log_time.len = sizeof("1970/09/28 12:00:00") - 1;
    ngx_cached_http_time.len = sizeof("Mon, 28 Sep 1970 06:00:00 GMT") - 1;
    ngx_cached_http_log_time.len = sizeof("28/Sep/1970:12:00:00 +0600") - 1;
    ngx_cached_http_log_iso8601.len = sizeof("1970-09-28T12:00:00+06:00") - 1;
    ngx_cached_syslog_time.len = sizeof("Sep 28 12:00:00") - 1;
    // cached_time是一个数组，用于多线程的处理，防止在一个线程写数据。另一个线程读取导致的错误
    ngx_cached_time = &cached_time[0];

    ngx_time_update();
}
```

#### ngx_time_update更新时间

```c
void
ngx_time_update(void)
{
    u_char          *p0, *p1, *p2, *p3, *p4;
    ngx_tm_t         tm, gmt;
    time_t           sec;
    ngx_uint_t       msec;
    ngx_time_t      *tp;
    struct timeval   tv;
    // 多线程获取锁，不用考虑
    if (!ngx_trylock(&ngx_time_lock)) {
        return;
    }
    /*
    #define ngx_gettimeofday(tp)  (void) gettimeofday(tp, NULL);
    获取当前时间
    */
    ngx_gettimeofday(&tv);

    sec = tv.tv_sec;
    msec = tv.tv_usec / 1000;
    // 计算缓存的时间戳
    ngx_current_msec = ngx_monotonic_time(sec, msec);
    // 获取线程对应的ngx_time_t
    tp = &cached_time[slot];
    // 如果秒纬度未变化，则不执行后续处理
    if (tp->sec == sec) {
        tp->msec = msec;
        ngx_unlock(&ngx_time_lock);
        return;
    }
    // 将写的数据后移一个slot
    if (slot == NGX_TIME_SLOTS - 1) {
        slot = 0;
    } else {
        slot++;
    }

    tp = &cached_time[slot];

    tp->sec = sec;
    tp->msec = msec;
    // 逐一处理每个缓存数据
    ngx_gmtime(sec, &gmt);


    p0 = &cached_http_time[slot][0];

    (void) ngx_sprintf(p0, "%s, %02d %s %4d %02d:%02d:%02d GMT",
                       week[gmt.ngx_tm_wday], gmt.ngx_tm_mday,
                       months[gmt.ngx_tm_mon - 1], gmt.ngx_tm_year,
                       gmt.ngx_tm_hour, gmt.ngx_tm_min, gmt.ngx_tm_sec);

#if (NGX_HAVE_GETTIMEZONE)

    tp->gmtoff = ngx_gettimezone();
    ngx_gmtime(sec + tp->gmtoff * 60, &tm);

#elif (NGX_HAVE_GMTOFF)

    ngx_localtime(sec, &tm);
    cached_gmtoff = (ngx_int_t) (tm.ngx_tm_gmtoff / 60);
    tp->gmtoff = cached_gmtoff;

#else

    ngx_localtime(sec, &tm);
    cached_gmtoff = ngx_timezone(tm.ngx_tm_isdst);
    tp->gmtoff = cached_gmtoff;

#endif


    p1 = &cached_err_log_time[slot][0];

    (void) ngx_sprintf(p1, "%4d/%02d/%02d %02d:%02d:%02d",
                       tm.ngx_tm_year, tm.ngx_tm_mon,
                       tm.ngx_tm_mday, tm.ngx_tm_hour,
                       tm.ngx_tm_min, tm.ngx_tm_sec);


    p2 = &cached_http_log_time[slot][0];

    (void) ngx_sprintf(p2, "%02d/%s/%d:%02d:%02d:%02d %c%02i%02i",
                       tm.ngx_tm_mday, months[tm.ngx_tm_mon - 1],
                       tm.ngx_tm_year, tm.ngx_tm_hour,
                       tm.ngx_tm_min, tm.ngx_tm_sec,
                       tp->gmtoff < 0 ? '-' : '+',
                       ngx_abs(tp->gmtoff / 60), ngx_abs(tp->gmtoff % 60));

    p3 = &cached_http_log_iso8601[slot][0];

    (void) ngx_sprintf(p3, "%4d-%02d-%02dT%02d:%02d:%02d%c%02i:%02i",
                       tm.ngx_tm_year, tm.ngx_tm_mon,
                       tm.ngx_tm_mday, tm.ngx_tm_hour,
                       tm.ngx_tm_min, tm.ngx_tm_sec,
                       tp->gmtoff < 0 ? '-' : '+',
                       ngx_abs(tp->gmtoff / 60), ngx_abs(tp->gmtoff % 60));

    p4 = &cached_syslog_time[slot][0];

    (void) ngx_sprintf(p4, "%s %2d %02d:%02d:%02d",
                       months[tm.ngx_tm_mon - 1], tm.ngx_tm_mday,
                       tm.ngx_tm_hour, tm.ngx_tm_min, tm.ngx_tm_sec);

    /*
    #define ngx_memory_barrier()        __sync_synchronize()
    防止读操作混乱有关，它告诉编译器不要将其后面的语句进行优化，不要打乱其执行顺序,保证最后对全局的缓存时间赋值
    */
    ngx_memory_barrier();

    ngx_cached_time = tp;
    ngx_cached_http_time.data = p0;
    ngx_cached_err_log_time.data = p1;
    ngx_cached_http_log_time.data = p2;
    ngx_cached_http_log_iso8601.data = p3;
    ngx_cached_syslog_time.data = p4;

    ngx_unlock(&ngx_time_lock);
}
```

这里比较有趣的就是slot的使用。对于写操作，是使用锁进行更新，保证数据不会被损坏，对于读操作，是并未加锁的，直接从缓存中读取的。如果在读取的时候，有别的线程在写入，将会导致读取出的数据是被损坏的，这里使用一个数组来进行写进行规避。

```
#define NGX_TIME_SLOTS   64

static ngx_uint_t        slot;
static ngx_atomic_t      ngx_time_lock;

volatile ngx_msec_t      ngx_current_msec;
volatile ngx_time_t     *ngx_cached_time;
volatile ngx_str_t       ngx_cached_err_log_time;
volatile ngx_str_t       ngx_cached_http_time;
volatile ngx_str_t       ngx_cached_http_log_time;
volatile ngx_str_t       ngx_cached_http_log_iso8601;
volatile ngx_str_t       ngx_cached_syslog_time;

static ngx_time_t        cached_time[NGX_TIME_SLOTS];
static u_char            cached_err_log_time[NGX_TIME_SLOTS]
                                    [sizeof("1970/09/28 12:00:00")];
static u_char            cached_http_time[NGX_TIME_SLOTS]
                                    [sizeof("Mon, 28 Sep 1970 06:00:00 GMT")];
static u_char            cached_http_log_time[NGX_TIME_SLOTS]
                                    [sizeof("28/Sep/1970:12:00:00 +0600")];
static u_char            cached_http_log_iso8601[NGX_TIME_SLOTS]
                                    [sizeof("1970-09-28T12:00:00+06:00")];
static u_char            cached_syslog_time[NGX_TIME_SLOTS]
                                    [sizeof("Sep 28 12:00:00")];
```

从上面的定义可以看出来，除了`ngx_current_msec`全局变量，其余每一个变量都有一个与之对应的64纬数组。`slot`变量记录了当前缓存中的全局变量的值是那个数组中存储的数据，每次更新的时候，都将slot增加1，这样保证更新的是缓存中的后一个数据，在更新完成后，统一将内容写入到全局变量中，以此来保证读操作时，不会有写线程来导致全局变量是被损坏的数据。

比如，在线程A在复制ngx_cached_http_time的data字段时（赋值了一半），这时被信号中断，或者别的线程重写了数据，如果没有slot，则复制过程中，后面地址的内容将会是错误数据，有了slot后，一般情况变更不会将原来地址的数据重写（除非一下有64个线程调用了时间更新），这样在一定程度上保证了读操作的安全性（不能完全避免）。目前未使用多线程，目前可以不用考虑。

### 定时器方法

#### ngx_event_timer_init初始化定时器红黑树

```c
ngx_rbtree_t              ngx_event_timer_rbtree;
static ngx_rbtree_node_t  ngx_event_timer_sentinel;

ngx_int_t
ngx_event_timer_init(ngx_log_t *log)
{
    ngx_rbtree_init(&ngx_event_timer_rbtree, &ngx_event_timer_sentinel,
                    ngx_rbtree_insert_timer_value);

    return NGX_OK;
}
```

事件时间红黑树中会存在key一致的问题，但并未做任何处理，因为对其中的元素来说，我们并不会要精确获取哪一个，只是在时间触发时，执行对应元素的处理即可。

初始化红黑树详见红黑树的介绍。

#### ngx_event_add_timer向事件树中增加元素

```c
static ngx_inline void
ngx_event_add_timer(ngx_event_t *ev, ngx_msec_t timer)
{
    ngx_msec_t      key;
    ngx_msec_int_t  diff;
    // 元素key为当前时间+过期时间
    key = ngx_current_msec + timer;
    // 如果事件已经被增加到时间树中
    if (ev->timer_set) {

        /*
         * Use a previous timer value if difference between it and a new
         * value is less than NGX_TIMER_LAZY_DELAY milliseconds: this allows
         * to minimize the rbtree operations for fast connections.
         */
        // 获取本次添加和之前添加的时间差值
        diff = (ngx_msec_int_t) (key - ev->timer.key);
        // 差值（绝对值）小于300ms，则不再重复添加
        if (ngx_abs(diff) < NGX_TIMER_LAZY_DELAY) {
            ngx_log_debug3(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                           "event timer: %d, old: %M, new: %M",
                            ngx_event_ident(ev->data), ev->timer.key, key);
            return;
        }
        /*
        #define ngx_add_timer        ngx_event_add_timer
        #define ngx_del_timer        ngx_event_del_timer
        */
        // 将当前事件从红黑树中删除
        ngx_del_timer(ev);
    }
    // 设置事件key
    ev->timer.key = key;

    ngx_log_debug3(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                   "event timer add: %d: %M:%M",
                    ngx_event_ident(ev->data), timer, ev->timer.key);
    // 推荐奖进红黑树
    ngx_rbtree_insert(&ngx_event_timer_rbtree, &ev->timer);
    // 设置已经在事件树中
    ev->timer_set = 1;
}
```

#### ngx_event_del_timer从事件树中删除元素

```c
static ngx_inline void
ngx_event_del_timer(ngx_event_t *ev)
{
    ngx_log_debug2(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                   "event timer del: %d: %M",
                    ngx_event_ident(ev->data), ev->timer.key);
    // 红黑树中删除元素
    ngx_rbtree_delete(&ngx_event_timer_rbtree, &ev->timer);

#if (NGX_DEBUG)
    ev->timer.left = NULL;
    ev->timer.right = NULL;
    ev->timer.parent = NULL;
#endif
    // 设置为未添加状态
    ev->timer_set = 0;
}
```

#### ngx_event_find_timer查找是否存在超时事件

```c
ngx_msec_t
ngx_event_find_timer(void)
{
    ngx_msec_int_t      timer;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    if (ngx_event_timer_rbtree.root == &ngx_event_timer_sentinel) {
        return NGX_TIMER_INFINITE;
    }

    root = ngx_event_timer_rbtree.root;
    sentinel = ngx_event_timer_rbtree.sentinel;
    // 查找key最小的节点，即最先超时节点
    node = ngx_rbtree_min(root, sentinel);
    // 对比和当前时间差距
    timer = (ngx_msec_int_t) (node->key - ngx_current_msec);
    // 返回是否超时，以及下一个将要超时的时间要等多久
    return (ngx_msec_t) (timer > 0 ? timer : 0);
}
```

#### ngx_event_expire_timers执行所有的超时事件

```c
void
ngx_event_expire_timers(void)
{
    ngx_event_t        *ev;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    sentinel = ngx_event_timer_rbtree.sentinel;

    for ( ;; ) {
        root = ngx_event_timer_rbtree.root;

        if (root == sentinel) {
            return;
        }
        // 获取最先超时的事件
        node = ngx_rbtree_min(root, sentinel);

        /* node->key > ngx_current_msec */
        // 如果最新超时的事件未超时，则返回
        if ((ngx_msec_int_t) (node->key - ngx_current_msec) > 0) {
            return;
        }
        // 获取对应的事件
        ev = (ngx_event_t *) ((char *) node - offsetof(ngx_event_t, timer));

        ngx_log_debug2(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                       "event timer del: %d: %M",
                       ngx_event_ident(ev->data), ev->timer.key);
        // 从事件树中删除
        ngx_rbtree_delete(&ngx_event_timer_rbtree, &ev->timer);

#if (NGX_DEBUG)
        ev->timer.left = NULL;
        ev->timer.right = NULL;
        ev->timer.parent = NULL;
#endif
        // 设置未在事件树中
        ev->timer_set = 0;
        // 设置事件为已超时
        ev->timedout = 1;
        // 执行事件的处理函数
        ev->handler(ev);
    }
}
```

因此事件一旦超时，将会执行处理函数，并从事件红黑树中删除。在nginx中，对于某些事件会同时添加到事件红黑树和epoll事件驱动模块中，例如，在建立连接后，接收请求有一个超时时间。因此向事件红黑树和epoll中均添加上对应等待请求的事件。如果在指定时间内还未收到请求，则事件红黑树中将执行超时处理，设置事件为超时事件，并执行相应的处理（关闭连接，从epoll中剔除事件）。如果在超时之前接收到了请求，则会先执行对应的处理，如果在指定时间内未完成获取请求头和解析，导致定时器红黑树中事件触发，依然会将事件移出，并关闭连接，如果在指定事件完成上述操作，则会将事件从定时器红黑树中移出。

#### ngx_event_no_timers_left检查是否所有事件均可忽略

在worker进程退出时，会判断是否所有事件均可以忽略，如果是，则直接退出。

```c
ngx_int_t
ngx_event_no_timers_left(void)
{
    ngx_event_t        *ev;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    sentinel = ngx_event_timer_rbtree.sentinel;
    root = ngx_event_timer_rbtree.root;

    if (root == sentinel) {
        return NGX_OK;
    }
    // 从最先超时的节点遍历整棵树，查找是否存在不可忽略的事件
    for (node = ngx_rbtree_min(root, sentinel);
         node;
         node = ngx_rbtree_next(&ngx_event_timer_rbtree, node))
    {
        ev = (ngx_event_t *) ((char *) node - offsetof(ngx_event_t, timer));

        if (!ev->cancelable) {
            return NGX_AGAIN;
        }
    }

    /* only cancelable timers left */

    return NGX_OK;
}
```



# 一些重要的模块

## pcre正则

在nginx中使用pcre正则来解析配置并进行处理。PCRE是 **‘Perl Compatible Regular Expressions’**(Perl兼容的正则表达式）的缩写。PCRE库由一系列函数组成，实现了与Perl5相同的语法、语义的正则表达式匹配功能。

正则需要进行模式匹配，其中包含命名模式匹配和匿名模式匹配，以如下为例：

```
(?<date> (?<year>(\d\d)?\d\d) - (?<month>\d\d) - (?<day>\d\d) )
```

这里存在5个匹配（包括匿名匹配和命名匹配）。其中`date`匹配`(?<year>(\d\d)?\d\d) - (?<month>\d\d) - (?<day>\d\d)`部分，`year`匹配`(\d\d)?\d\d`，`month`匹配`\d\d`，`day`匹配`\d\d`。还包含一个匿名匹配，即`year`后面的`(/d/d)`。

`PCRE`允许使用命名捕获分组，也允许使用匿名捕获分组（即分组用数字来表示），其实命名捕获分组只是用来标识分组的另一种方式，命名捕获分组也会获得一个数字分组名称。`PCRE`提供了一些方法可以通过命名捕获分组的名称来快速获取捕获分组内容的函数，比如：`pcre_get_named_substring()` .
也可以通过以下步骤来获取捕获分组的信息：

- 将命名捕获分组的名称转换为数字。

- 通过上一步的数字来获取分组的信息。
  这里就牵涉到了一个 `name to number` 的转换过程，PCRE维护了一个 `name-to-number` 的`map`，我们可以根据这个`map`完成转换功能，这个`map`有以下三个属性：

  ```
  PCRE_INFO_NAMECOUNT
  PCRE_INFO_NAMEENTRYSIZE
  PCRE_INFO_NAMETABLE
  ```

  这个`map`包含了若干个固定大小的记录，可以通过`PCRE_INFO_NAMECOUNT`参数来获取这个`map`的记录数量(其实就是命名捕获分组的数量)，通过`PCRE_INFO_NAMEENTRYSIZE`来获取每个记录的大小，这两种情况下，最后一个参数都是一个`int`类型的指针。其中每个每个记录的大小是由最长的捕获分组的名称来确立的。其中每个每个记录的大小是由最长的捕获分组的名称来确立的。这里的记录是说命名的大小。

  `PCRE_INFO_NAMETABLE` 返回一个指向这个`map`的第一条记录的指针（一个`char`类型的指针），每条记录的前两个字节是命名捕获分组所对应的数字分组值，剩下的内容是命名捕获分组的`name`，以`'\0'`结束。返回的`map`的顺序是命名捕获分组的字母顺序。

  对于上述`date`的例子来说，map中存储数据内容如下：

  ```
  00 01 d a t e 00 ??
  00 05 d a y 00 ?? ??
  00 04 m o n t h 00
  00 02 y e a r 00 ??
  ```

  其中`PCRE_INFO_NAMEENTRYSIZE`为8，因为最长的name是`moth`,加上前面两位表示对应的实际数字分组值，已经最后的`\0`，总共八位。对应长度小于8位的命名分组来说，超长的部分是未设置值的。在遍历map时，需要使用`PCRE_INFO_NAMEENTRYSIZE`作为偏移。

这里主要介绍一下nginx中使用的pcre方法。

### pcre_compile()

函数原型：

```c
pcre *pcre_compile(const char *pattern, int options, const char **errptr, int *erroffset, const unsigned char *tableptr)
```

功能： 将一个正则表达式编译成一个内部的`pcre`结构，在匹配多个字符串时，可以加速匹配。其中`pcre_compile2()`功能一样，只是缺少一个参数`errorcodeptr`。

`pattern`为字符串，表示待编译的正则表达式。

`options`为0或者其他可选的标志。

`errptr`为返回的出错信息。

`erroffset`为出错位置。

`tableptr`指向一个字符数组的指针，可以设置为NULL。

### pcre_study()函数

函数原型：

```c
pcre_extra *pcre_study(const pcre *code, int options, const char **errptr)
```

功能： 对编译的模式进行学习，提取可以加速匹配过程的信息。

`code`为已编译的模式，即`pcre_compile`函数的输出。

`options`为选项。目前只有一个options，即`PCRE_STUDY_JIT_COMPILE`表示如果可能，它会要求及时编译。

`errpetr`为出错信息。

### pcre_exec()函数

函数原型：

```c
int pcre_exec(const pcre *code, const pcre_extra *extra, const char *subject, int length, int startoffset, int options, int *ovector, int ovecsize)
```

功能： 使用编译好的模式进行匹配，采用与`Perl`相似的算法。返回值大于0，表示匹配到的pattern个数； 否则表示出错信息。`NGX_REGEX_NO_MATCHED`表示不匹配（-1），其他负数表示错误。

`code`为编译好的模式。

`extra`为pcre_extra结构体，可以为null。

`subject`为需要匹配的字符串。

`length`为匹配字符串长度。

`startoffset`为匹配的开始位置。

`options`为选项位。

`ovector`指向一个结果的整形数组。这里返回的是每个捕获相对于subject中的偏移，对于第$i$个捕获来说，其起始下标为$ovector[2*i]$，末尾下标为$ovector[2*i+1]$。其中$i$从0开始。

`ovecsize`数组大小。

### pcre_fullinfo函数

函数原型：

```c
int pcre_fullinfo(const pcre *code, const pcre_extra *extra,int what, void *where);
```

功能：返回一个被编译完成的pattern的相关信息。

`code`为编译完成的pattern。

`extra`为优化后的额外信息，可以为空。

`what`为需要获取的信息类型。

`where`返回对应的存放地址。

nginx中主要获取到如下信息：

| what                      | 含义             |
| ------------------------- | ---------------- |
| `PCRE_INFO_CAPTURECOUNT`  | 捕获子模式的数量 |
| `PCRE_INFO_NAMECOUNT`     | 命名子模式的数量 |
| `PCRE_INFO_NAMEENTRYSIZE` | 名称表条目的大小 |
| `PCRE_INFO_NAMETABLE`     | 指向名称表的指针 |
| `PCRE_INFO_JIT`           | 是否支持JIT      |



### pcre_config函数

函数原型：

```
int pcre_config(int what, void *where)
```

功能： 查询当前PCRE版本中使用的选项信息。

- what: 选项名
- where: 存储结果的位置

## nginx中使用pcre

`nginx`中使用正则主要通过`ngx_regex_module`模块处理，这里不详细介绍该模块的所有部分，只关注对正则的处理。

### 相关结构

```c
typedef struct {
    pcre        *code; // 被编译后的正则字段
    pcre_extra  *extra; // 编译后执行study后的输出
} ngx_regex_t;


typedef struct {
    ngx_str_t     pattern; // 被编译的字符串
    ngx_pool_t   *pool;
    ngx_int_t     options; // 编译选项

    ngx_regex_t  *regex; // 编译结果
    int           captures; // 正则中需要捕获的数量，包括匿名和非匿名
    int           named_captures; // 正则中命名捕获数量
    int           name_size; // 命名捕获大小
    u_char       *names; // 命名捕获表
    ngx_str_t     err;
} ngx_regex_compile_t;


typedef struct {
    ngx_regex_t  *regex;
    u_char       *name; // 存储原始正则语句
} ngx_regex_elt_t;
```

### ngx_regex_compile编译正则

`ngx_regex_compile`函数用来对正则进行编译并初始化对应的`ngx_regex_compile_t`结构：

```c
ngx_int_t
ngx_regex_compile(ngx_regex_compile_t *rc)
{
    int               n, erroff;
    char             *p;
    pcre             *re;
    const char       *errstr;
    ngx_regex_elt_t  *elt;

    ngx_regex_malloc_init(rc->pool);
    // 编译正则
    re = pcre_compile((const char *) rc->pattern.data, (int) rc->options,
                      &errstr, &erroff, NULL);

    /* ensure that there is no current pool */
    ngx_regex_malloc_done();
    // 如果返回为null，则是错误，输出日志
    if (re == NULL) {
        if ((size_t) erroff == rc->pattern.len) {
           rc->err.len = ngx_snprintf(rc->err.data, rc->err.len,
                              "pcre_compile() failed: %s in \"%V\"",
                               errstr, &rc->pattern)
                      - rc->err.data;

        } else {
           rc->err.len = ngx_snprintf(rc->err.data, rc->err.len,
                              "pcre_compile() failed: %s in \"%V\" at \"%s\"",
                               errstr, &rc->pattern, rc->pattern.data + erroff)
                      - rc->err.data;
        }

        return NGX_ERROR;
    }
    // 分配对应的regex结构
    rc->regex = ngx_pcalloc(rc->pool, sizeof(ngx_regex_t));
    if (rc->regex == NULL) {
        goto nomem;
    }

    rc->regex->code = re;

    /* do not study at runtime */
    // 如果要使用study，则将该正则添加进入ngx_pcre_studies中，在解析完成配置后，执行ngx_regex_module模块的init module方法时执行study来进行优化
    if (ngx_pcre_studies != NULL) {
        elt = ngx_list_push(ngx_pcre_studies);
        if (elt == NULL) {
            goto nomem;
        }

        elt->regex = rc->regex;
        elt->name = rc->pattern.data;
    }
    // 获取匹配数量
    n = pcre_fullinfo(re, NULL, PCRE_INFO_CAPTURECOUNT, &rc->captures);
    if (n < 0) {
        p = "pcre_fullinfo(\"%V\", PCRE_INFO_CAPTURECOUNT) failed: %d";
        goto failed;
    }
    // 如果匹配数量为0，不需要执行后续操作
    if (rc->captures == 0) {
        return NGX_OK;
    }
    // 获取命名匹配数量
    n = pcre_fullinfo(re, NULL, PCRE_INFO_NAMECOUNT, &rc->named_captures);
    if (n < 0) {
        p = "pcre_fullinfo(\"%V\", PCRE_INFO_NAMECOUNT) failed: %d";
        goto failed;
    }
    // 如果命名匹配为0，不需要执行后续操作
    if (rc->named_captures == 0) {
        return NGX_OK;
    }
    // 获取命名匹配大小
    n = pcre_fullinfo(re, NULL, PCRE_INFO_NAMEENTRYSIZE, &rc->name_size);
    if (n < 0) {
        p = "pcre_fullinfo(\"%V\", PCRE_INFO_NAMEENTRYSIZE) failed: %d";
        goto failed;
    }
    // 获取命名匹配表
    n = pcre_fullinfo(re, NULL, PCRE_INFO_NAMETABLE, &rc->names);
    if (n < 0) {
        p = "pcre_fullinfo(\"%V\", PCRE_INFO_NAMETABLE) failed: %d";
        goto failed;
    }

    return NGX_OK;

failed:

    rc->err.len = ngx_snprintf(rc->err.data, rc->err.len, p, &rc->pattern, n)
                  - rc->err.data;
    return NGX_ERROR;

nomem:

    rc->err.len = ngx_snprintf(rc->err.data, rc->err.len,
                               "regex \"%V\" compilation failed: no memory",
                               &rc->pattern)
                  - rc->err.data;
    return NGX_ERROR;
}
```

### ngx_regex_exec执行匹配

```
#define ngx_regex_exec(re, s, captures, size)                                \
    pcre_exec(re->code, re->extra, (const char *) (s)->data, (s)->len, 0, 0, \
              captures, size)
```

由于nginx中不需要额外指定其余参数，因此设置了一个宏定义来减少参数，执行匹配。

### ngx_regex_exec_array匹配正则数组

`ngx_regex_exec_array`函数查找一个字符串是否匹配一个正则数组中的任意一个：

```c
// a为ngx_regex_elt_t的数组
ngx_int_t
ngx_regex_exec_array(ngx_array_t *a, ngx_str_t *s, ngx_log_t *log)
{
    ngx_int_t         n;
    ngx_uint_t        i;
    ngx_regex_elt_t  *re;

    re = a->elts;
    // 遍历每一个被编译后的正则
    for (i = 0; i < a->nelts; i++) {
        // 执行匹配
        n = ngx_regex_exec(re[i].regex, s, NULL, 0);
        // 不匹配
        if (n == NGX_REGEX_NO_MATCHED) {
            continue;
        }
        // 出错
        if (n < 0) {
            ngx_log_error(NGX_LOG_ALERT, log, 0,
                          ngx_regex_exec_n " failed: %i on \"%V\" using \"%s\"",
                          n, s, re[i].name);
            return NGX_ERROR;
        }

        /* match */
        // 匹配返回
        return NGX_OK;
    }
    // 不匹配
    return NGX_DECLINED;
}
```

## rewrite模块

### 配置解析

```c
static ngx_command_t  ngx_http_rewrite_commands[] = {

    { ngx_string("rewrite"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_TAKE23,
      ngx_http_rewrite,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("return"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_TAKE12,
      ngx_http_rewrite_return,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("break"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_NOARGS,
      ngx_http_rewrite_break,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("if"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_BLOCK|NGX_CONF_1MORE,
      ngx_http_rewrite_if,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("set"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_TAKE2,
      ngx_http_rewrite_set,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("rewrite_log"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF
                        |NGX_HTTP_LIF_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_rewrite_loc_conf_t, log),
      NULL },

    { ngx_string("uninitialized_variable_warn"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF
                        |NGX_HTTP_LIF_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_rewrite_loc_conf_t, uninitialized_variable_warn),
      NULL },

      ngx_null_command
};
```

### 相关结构

#### ngx_http_rewrite_loc_conf_t

该结构为rewrite模块生成的loc下的配置数据，其定义如下：

```c
typedef struct {
    // codes为执行方法的列表
    ngx_array_t  *codes;        /* uintptr_t */

    ngx_uint_t    stack_size;

    ngx_flag_t    log;
    ngx_flag_t    uninitialized_variable_warn;
} ngx_http_rewrite_loc_conf_t;
```

codes数组设置比较独特。脚本指令都是实现了”接口ngx_http_script_code_pt”的各个不同的充当类的结构体，这些结构体可以是ngx_http_script_var_code_t、ngx_http_script_value_code_t等，其类型不同，占用的内存也不同，如何放入一个数组中呢（不是它们的指针）。通过如下三点做到：

1. codes数组设计每个元素仅占用1字节大小，即不奢望一个数组元素能够存放表示一个脚本指令的结构体。

2. 每次要将1个指令放入codes数组中时，将根据指令结构体的占用内存字节数N，在codes数组中分配N个元素存储这1个指令，再依次把指令结构体的内容都拷贝到这N个数组成员中。例如：

   ```
   void *
   ngx_http_script_start_code(ngx_pool_t *pool, ngx_array_t **codes, size_t size)
   {
       if (*codes == NULL) {
           *codes = ngx_array_create(pool, 256, 1);
           if (*codes == NULL) {
               return NULL;
           }
       }
   
       return ngx_array_push_n(*codes, size);
   }
   ```

3. HTTP请求到来，脚本指令执行时，每执行完一个脚本指令的ngx_http_script_code_pt方法后，该方法必须主动告知所属指令结构体占用的内存数N，这样从当前指令所在codes数组索引加上N后就是下一条指令。

#### ngx_regex_compile_t

`ngx_regex_compile_t`结构用于正则表达式的编译，定义如下：

```c
typedef struct {
    // 待编译的正则表达式
    ngx_str_t     pattern;
    ngx_pool_t   *pool;
    // 编译选项，即pcre_compile中的选项
    ngx_int_t     options;
    // 编译完成的结果
    ngx_regex_t  *regex;
    // 编译的pattern中模式匹配（命名+匿名）数量
    int           captures;
    // 编译的pattern中命名模式匹配数量
    int           named_captures;
    // 编译的pattern中命名模式匹配大小
    int           name_size;
    // 编译的pattern中命名模式匹配table表
    u_char       *names;
    // 错误信息
    ngx_str_t     err;
} ngx_regex_compile_t;
```



#### ngx_http_regex_t

`ngx_http_regex_t`结构存储http模块使用正则的相关内容：

```c
typedef struct {
    ngx_regex_t                  *regex; // 正则编译后结构
    ngx_uint_t                    ncaptures; // 匹配数量
    ngx_http_regex_variable_t    *variables; // 命名匹配变量
    ngx_uint_t                    nvariables; // 命名匹配变量数量
    ngx_str_t                     name; // 正则字符串
} ngx_http_regex_t;
```

#### ngx_http_regex_variable_t

对于正则表达式中使用的命名匹配来说，nginx会将其增加到变量数组中，`ngx_http_regex_variable_t`结构存储了相关结构：

```c
typedef struct {
    ngx_uint_t                    capture; // 存储其值在编译后返回的偏移数组中的位置，后续详细介绍
    ngx_int_t                     index; // 在变量中的index
} ngx_http_regex_variable_t;
```



#### ngx_http_script_engine_t

同一段脚本被编译进nginx中，在不同请求中执行效果是不一样的，所以每个请求都必须有其独特的脚本执行上下文，或者称为脚本引擎。这由ngx_http_script_engine_t类充当：

```
typedef struct {
    // 执行待执行的脚本指令
    u_char                     *ip;
    // 存储编译完成的值的空间
    u_char                     *pos;
    // 变量构成的栈
    ngx_http_variable_value_t  *sp;

    ngx_str_t                   buf;
    // 需要编译的字符串
    ngx_str_t                   line;

    /* the start of the rewritten arguments */
    u_char                     *args;

    unsigned                    flushed:1;
    unsigned                    skip:1;
    unsigned                    quote:1;
    unsigned                    is_args:1;
    unsigned                    log:1;
    // 脚本执行状态
    ngx_int_t                   status;
    // 执行当前脚本引擎所属的HTTP请求
    ngx_http_request_t         *request;
} ngx_http_script_engine_t;
```

sp是一个栈，作为编译工具。大小默认为10.

ip可以理解为IP寄存器，指向下一行将要执行的代码。对于u_char*类型来说，其指向类型是不定的。其指向的一定是待执行的脚本指令。用面向对象的语言来说，其指向的是实现了ngx_http_script_code_pt接口的类。但C语言没有接口的概念，在C语言实现上述目的，通常会使用嵌套结构体的方法，比如表示接口的结构体A，要放在表示实现接口的类-结构体B的第一个位置。这样一个指向B的指针，也可以强制转换为A，再调用A的成员。ngx_http_script_code_pt是一个指针函数：

```
typedef void (*ngx_http_script_code_pt) (ngx_http_script_engine_t *e);
```

ngx_http_script_code_pt参数ngx_http_script_engine_t，其表示当前指令的脚本上下文。

#### ngx_http_script_compile_t

`ngx_http_script_compile_t`结构用于执行脚本的解析编译。

```
typedef struct {
    ngx_conf_t                 *cf; // 配置
    ngx_str_t                  *source; // 需要compile的字符串

    ngx_array_t               **flushes; // 该结构也用于正则匹配，该成员，构建为变量为$1,$2...这种时，存储每个变量对应于哪一个，构成一个数组
    ngx_array_t               **lengths; //处理变量长度的处理子数组，需要执行ngx_http_script_code_pt构成的列表,这里是生成值时执行ngx_http_script_code_pt的列表
    ngx_array_t               **values; // 处理变量内容的处理子数组，对应于rewrite模块的locations层级下的codes执行数组

    ngx_uint_t                  variables; // 值中含有变量的数量
    ngx_uint_t                  ncaptures; // 当前处理时，出现的$n变量的最大值，如配置的最大为$3，那么ncaptures就等于3
    
    /*
     * 以位移的形式保存$1,$2...$9等变量，即响应位置上置1来表示，主要的作用是为dup_capture准备，
     * 正是由于这个mask的存在，才比较容易得到是否有重复的$n出现。
     */
    ngx_uint_t                  captures_mask;
    ngx_uint_t                  size;

    /* 
     * ngx_http_script_regex_code_t的结构
     */
    void                       *main;

    unsigned                    compile_args:1; // 是否需要处理请求参数
    unsigned                    complete_lengths:1; // 是否设置lengths数组的终止符，即NULL
    unsigned                    complete_values:1; // 是否设置values数组的终止符
    unsigned                    zero:1; // values数组运行时，得到的字符串是否追加'\0'结尾
    unsigned                    conf_prefix:1; // 是否在生成的文件名前，追加路径前缀
    unsigned                    root_prefix:1; // 同conf_prefix

    /*
     * 这个标记位主要在rewrite模块里使用，在ngx_http_rewrite中，
     * if (sc.variables == 0 && !sc.dup_capture) {
     *     regex->lengths = NULL;
     * }
     * 没有重复的$n，那么regex->lengths被置为NULL，这个设置很关键，在函数
     * ngx_http_script_regex_start_code中就是通过对regex->lengths的判断，来做不同的处理，
     * 因为在没有重复的$n的时候，可以通过正则自身的captures机制来获取$n，一旦出现重复的，
     * 那么pcre正则自身的captures并不能满足我们的要求，我们需要用自己handler来处理。
     */
    unsigned                    dup_capture:1;
    unsigned                    args:1; // 待compile的字符串中是否发现了'?'
} ngx_http_script_compile_t;
```

#### ngx_http_script_regex_code_t

`ngx_http_script_regex_code_t`用于存储rewrite中第一个正则表达式的相关信息及对应的处理方法。

```
typedef struct {
    ngx_http_script_code_pt     code;
    ngx_http_regex_t           *regex;
    ngx_array_t                *lengths;
    uintptr_t                   size;
    uintptr_t                   status; // 执行rewrite后，返回时，返回的http状态码
    uintptr_t                   next;

    unsigned                    test:1;
    unsigned                    negative_test:1;
    unsigned                    uri:1;
    // args表示是否在rewrite中用户自定义了相关参数
    unsigned                    args:1;

    /* add the r->args to the new arguments */
    unsigned                    add_args:1;
    // 是否为重定向
    unsigned                    redirect:1;
    // 是否执行完当前的函数，跳出循环，对应flag为break
    unsigned                    break_cycle:1;

    ngx_str_t                   name;
} ngx_http_script_regex_code_t;
```



#### ngx_http_script_regex_end_code_t

`ngx_http_script_regex_end_code_t`用于存储生成rewrite中第二个新的uri时需要的参数及方法

```c
typedef struct {
    ngx_http_script_code_pt     code;

    unsigned                    uri:1;
    unsigned                    args:1;

    /* add the r->args to the new arguments */
    unsigned                    add_args:1;

    unsigned                    redirect:1;
} ngx_http_script_regex_end_code_t;
```



### create location方法

```c
static void *
ngx_http_rewrite_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_rewrite_loc_conf_t  *conf;
    // 生成配置结构
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_rewrite_loc_conf_t));
    if (conf == NULL) {
        return NULL;
    }
    // 初始化值为未定义
    conf->stack_size = NGX_CONF_UNSET_UINT;
    conf->log = NGX_CONF_UNSET;
    conf->uninitialized_variable_warn = NGX_CONF_UNSET;

    return conf;
}
```

### commands方法

#### rewrite配置

语法：`rewrite regex replacement [flag];`
        作用域：`server` 、`location`、`if`
        功能：如果一个URI匹配指定的正则表达式regex，URI就按照 replacement 重写。

rewrite 按配置文件中出现的顺序执行。可以使用 flag 标志来终止指令的进一步处理。

如果 replacement 以 `http://` 、 `https://` 或 `$ scheme` 开始，将不再继续处理，这个重定向将返回给客户端。

**`flag`** 有四种参数可以选择：

1. **`last`** 停止处理后续 rewrite 指令集，然后对当前重写的新 URI 在 rewrite 指令集上重新查找。

2. **`break`** 停止处理后续 rewrite 指令集，并不再重新查找。一般break和代理转发进行配合使用，注意，如果没有break，则新生成的uri会重新执行NGX_HTTP_FIND_CONFIG_PHASE阶段，而使用了break，则不会再进行查找，例如：

   ```nginx
   location ~ ^/test/(.*)$ {
          rewrite /test/(.*) /test/index.php/$1 break;
          proxy_pass http://backend;
               
   }
   ```

   这时会按照新生成的uri进行代理转发。

3. **`redirect`** 如果 `replacement` 不是以 `http://` 或 `https://` 开始，返回 `302`临时重定向。

4. **`permanent`** 返回 `301`永久重定向。

rewrite对uri进行改写。

```
	{ ngx_string("rewrite"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_TAKE23,
      ngx_http_rewrite,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },
```

其对应方法函数如下：

```c
static char *
ngx_http_rewrite(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_rewrite_loc_conf_t  *lcf = conf;

    ngx_str_t                         *value;
    ngx_uint_t                         last;
    ngx_regex_compile_t                rc;
    ngx_http_script_code_pt           *code;
    ngx_http_script_compile_t          sc;
    ngx_http_script_regex_code_t      *regex;
    ngx_http_script_regex_end_code_t  *regex_end;
    u_char                             errstr[NGX_MAX_CONF_ERRSTR];
    // 分配处理rewrite中value1，即正则语句的结构,这里存储的原理是存储的一个数组中
    regex = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                       sizeof(ngx_http_script_regex_code_t));
    if (regex == NULL) {
        return NGX_CONF_ERROR;
    }

    ngx_memzero(regex, sizeof(ngx_http_script_regex_code_t));

    value = cf->args->elts;
    // 如果新的uri长度为0，则是配置错误
    if (value[2].len == 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "empty replacement");
        return NGX_CONF_ERROR;
    }

    // 分配进行正则解析的结构
    ngx_memzero(&rc, sizeof(ngx_regex_compile_t));
    // 设置对应的解析语句
    rc.pattern = value[1];
    rc.err.len = NGX_MAX_CONF_ERRSTR;
    rc.err.data = errstr;

    /* TODO: NGX_REGEX_CASELESS */
    // 执行解析
    regex->regex = ngx_http_regex_compile(cf, &rc);
    if (regex->regex == NULL) {
        return NGX_CONF_ERROR;
    }
    // 设置对执行该rewrite方法的uri的处理，进行编译，确定是否请求的uri匹配正则表达式，函数参数为regex本身
    regex->code = ngx_http_script_regex_start_code;
    regex->uri = 1;
    regex->name = value[1];
    // 如果value[2]最后存在?,说明需要元素的参数
    if (value[2].data[value[2].len - 1] == '?') {

        /* the last "?" drops the original arguments */
        value[2].len--;

    } else {
        // 否则标识需要增加参数
        regex->add_args = 1;
    }

    last = 0;
    // 如果转发到http或https，或者schema，则是临时重定向
    if (ngx_strncmp(value[2].data, "http://", sizeof("http://") - 1) == 0
        || ngx_strncmp(value[2].data, "https://", sizeof("https://") - 1) == 0
        || ngx_strncmp(value[2].data, "$scheme", sizeof("$scheme") - 1) == 0)
    {
        regex->status = NGX_HTTP_MOVED_TEMPORARILY;
        regex->redirect = 1;
        last = 1;
    }
    // 最后的参数含义，设置对应参数
    if (cf->args->nelts == 4) {
        if (ngx_strcmp(value[3].data, "last") == 0) {
            last = 1;

        } else if (ngx_strcmp(value[3].data, "break") == 0) {
            regex->break_cycle = 1;
            last = 1;

        } else if (ngx_strcmp(value[3].data, "redirect") == 0) {
            regex->status = NGX_HTTP_MOVED_TEMPORARILY;
            regex->redirect = 1;
            last = 1;

        } else if (ngx_strcmp(value[3].data, "permanent") == 0) {
            regex->status = NGX_HTTP_MOVED_PERMANENTLY;
            regex->redirect = 1;
            last = 1;

        } else {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid parameter \"%V\"", &value[3]);
            return NGX_CONF_ERROR;
        }
    }
    
    // 分配编译value2的结构
    ngx_memzero(&sc, sizeof(ngx_http_script_compile_t));
    // 设置配置
    sc.cf = cf;
    // 设置待解析参数
    sc.source = &value[2];
    // ngx_http_script_compile_t中的lengths（计算生成的uri长度的方法数组）取ngx_http_script_regex_code_t中的lengths，这样就是直接在ngx_http_script_regex_code_t的lengths中追加处理方法，之后在执行ngx_http_script_regex_code_t的code函数时，使用lengths来计算需要分配存储新的uri的空间长度
    sc.lengths = &regex->lengths;
    // ngx_http_script_compile_t中的value是ngx_http_rewrite_loc_conf_t中的codes，即直接在ngx_http_rewrite_loc_conf_t中添加处理方法，这样在完成长度计算后，就会允许变量赋值的处理方法。
    sc.values = &lcf->codes;
    // 获取需要使用的变量数量，通过$字符
    sc.variables = ngx_http_script_variables_count(&value[2]);
    sc.main = regex;
    sc.complete_lengths = 1;
    // 记录是否需要编译参数，如果是重定向，则不需要编译参数
    sc.compile_args = !regex->redirect;
    // 执行脚本编译，增加生成新的uri的各个子方法
    if (ngx_http_script_compile(&sc) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    regex = sc.main;

    regex->size = sc.size;
    regex->args = sc.args;

    if (sc.variables == 0 && !sc.dup_capture) {
        regex->lengths = NULL;
    }
    // 增加在解析完成后获取到新的uri的处理方法
    regex_end = ngx_http_script_add_code(lcf->codes,
                                      sizeof(ngx_http_script_regex_end_code_t),
                                      &regex);
    if (regex_end == NULL) {
        return NGX_CONF_ERROR;
    }
    // 方法函数，函数执行的参数即为regex_end本身
    regex_end->code = ngx_http_script_regex_end_code;
    regex_end->uri = regex->uri;
    regex_end->args = regex->args;
    regex_end->add_args = regex->add_args;
    regex_end->redirect = regex->redirect;
    // 如果是last的，则在末尾增加一个null，配合rewrit后的检查阶段，来选择是重新执行遍历，还是直接进行下一阶段
    if (last) {
        code = ngx_http_script_add_code(lcf->codes, sizeof(uintptr_t), &regex);
        if (code == NULL) {
            return NGX_CONF_ERROR;
        }

        *code = NULL;
    }
    // 如果在正则编译时未匹配，则跳过该rewrite后续方法的处理，这里计算跳到的位置
    regex->next = (u_char *) lcf->codes->elts + lcf->codes->nelts
                                              - (u_char *) regex;

    return NGX_CONF_OK;
}
```



##### ngx_http_regex_compile

该方法对正则表达试进行编译。

```c
ngx_http_regex_t *
ngx_http_regex_compile(ngx_conf_t *cf, ngx_regex_compile_t *rc)
{
    u_char                     *p;
    size_t                      size;
    ngx_str_t                   name;
    ngx_uint_t                  i, n;
    ngx_http_variable_t        *v;
    ngx_http_regex_t           *re;
    ngx_http_regex_variable_t  *rv;
    ngx_http_core_main_conf_t  *cmcf;

    rc->pool = cf->pool;
    // 执行正则编译
    if (ngx_regex_compile(rc) != NGX_OK) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "%V", &rc->err);
        return NULL;
    }

    re = ngx_pcalloc(cf->pool, sizeof(ngx_http_regex_t));
    if (re == NULL) {
        return NULL;
    }
    // 存储正则编译结果
    re->regex = rc->regex;
    // 存储捕获变量数量
    re->ncaptures = rc->captures;
    // 存储编译的正则语句
    re->name = rc->pattern;
    // 获取main级别的ngx_http_core_module生成结构
    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    // 设置捕获变量大小
    cmcf->ncaptures = ngx_max(cmcf->ncaptures, re->ncaptures);
    // 获取命名捕获数量
    n = (ngx_uint_t) rc->named_captures;

    if (n == 0) {
        return re;
    }
    // 分配对应数量的正则变量结构
    rv = ngx_palloc(rc->pool, n * sizeof(ngx_http_regex_variable_t));
    if (rv == NULL) {
        return NULL;
    }
    // 存储变量，记录变量数量
    re->variables = rv;
    re->nvariables = n;
    // 获取命名变量size，用于遍历name的table
    size = rc->name_size;
    p = rc->names;
    // 变量命名变量
    for (i = 0; i < n; i++) {
        // 用于获取变量所在值其实地址的数组的下标。由于table中前两位存储了对应于数字捕获的值，因此先计算出其原始值，再将值乘以2，详见上文中pcre_exec函数
        rv[i].capture = 2 * ((p[0] << 8) + p[1]);
        // 记录变量名
        name.data = &p[2];
        // 记录变量长度
        name.len = ngx_strlen(name.data);
       // 条件可变变量，详解变量章节
        v = ngx_http_add_variable(cf, &name, NGX_HTTP_VAR_CHANGEABLE);
        if (v == NULL) {
            return NULL;
        }
        // 获取变量所在数组中的index
        rv[i].index = ngx_http_get_variable_index(cf, &name);
        if (rv[i].index == NGX_ERROR) {
            return NULL;
        }
        // 设置变量对应的处理函数，该函数不会获取值，仅设置not_found为1，即该变量只有rewrite模块使用
        v->get_handler = ngx_http_variable_not_found;
        // 计算偏移，遍历下一个命名捕获
        p += size;
    }

    return re;
}
```



##### ngx_http_script_regex_start_code

该函数添加到了`ngx_http_rewrite_loc_conf_t`中的codes类的实际执行方法。该方法进行正则匹配，并进行相关设置。

```c
void
ngx_http_script_regex_start_code(ngx_http_script_engine_t *e)
{
    size_t                         len;
    ngx_int_t                      rc;
    ngx_uint_t                     n;
    ngx_http_request_t            *r;
    ngx_http_script_engine_t       le;
    ngx_http_script_len_code_pt    lcode;
    ngx_http_script_regex_code_t  *code;

    code = (ngx_http_script_regex_code_t *) e->ip;

    r = e->request;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http script regex: \"%V\"", &code->name);
    // 获取要执行解析的uri，根据是否有uri，这里在rewrite中使用的是uri
    if (code->uri) {
        e->line = r->uri;
    } else {
        e->sp--;
        e->line.len = e->sp->len;
        e->line.data = e->sp->data;
    }
    // 执行http正则解析
    rc = ngx_http_regex_exec(r, code->regex, &e->line);
    // 如果解析结果为不匹配
    if (rc == NGX_DECLINED) {
        if (e->log || (r->connection->log->log_level & NGX_LOG_DEBUG_HTTP)) {
            ngx_log_error(NGX_LOG_NOTICE, r->connection->log, 0,
                          "\"%V\" does not match \"%V\"",
                          &code->name, &e->line);
        }

        r->ncaptures = 0;
        // 测试？，
        if (code->test) {
            if (code->negative_test) {
                e->sp->len = 1;
                e->sp->data = (u_char *) "1";

            } else {
                e->sp->len = 0;
                e->sp->data = (u_char *) "";
            }

            e->sp++;

            e->ip += sizeof(ngx_http_script_regex_code_t);
            return;
        }
        
        // 跳过对该rewrite剩下的操作，next指向rewrite后的函数
        e->ip += code->next;
        return;
    }
    // 解析错误处理，设置status将会在该阶段立即结束请求
    if (rc == NGX_ERROR) {
        e->ip = ngx_http_script_exit;
        e->status = NGX_HTTP_INTERNAL_SERVER_ERROR;
        return;
    }

    if (e->log || (r->connection->log->log_level & NGX_LOG_DEBUG_HTTP)) {
        ngx_log_error(NGX_LOG_NOTICE, r->connection->log, 0,
                      "\"%V\" matches \"%V\"", &code->name, &e->line);
    }

    if (code->test) {
        if (code->negative_test) {
            e->sp->len = 0;
            e->sp->data = (u_char *) "";

        } else {
            e->sp->len = 1;
            e->sp->data = (u_char *) "1";
        }

        e->sp++;

        e->ip += sizeof(ngx_http_script_regex_code_t);
        return;
    }

    // 如果设置了返回状态码，即是重定向，则在e中设置对应的状态码，后续执行结束请求返回用户的状态码，设置返回状态码后会立即结束请求，返回响应
    if (code->status) {
        e->status = code->status;
        // 只有在重定向时才应该有状态码，否则是错误
        if (!code->redirect) {
            e->ip = ngx_http_script_exit;
            return;
        }
    }
    
    // 如果是重写uri
    if (code->uri) {
        // 请求变成内部请求
        r->internal = 1;
        r->valid_unparsed_uri = 0;
        // 如果是跳出循环，则设置获取有效的location为0
        if (code->break_cycle) {
            r->valid_location = 0;
            // 设置uri发送变更为0，在NGX_HTTP_POST_REWRITE_PHASE阶段将根据该值，决定直接进行下一阶段
            r->uri_changed = 0;

        } else {
            // 否则uri发生变更，NGX_HTTP_POST_REWRITE_PHASE阶段根据该值，重新执行find_config_index阶段
            r->uri_changed = 1;
        }
    }
    
    // 如果计算新生成的uri长度的数组为null
    if (code->lengths == NULL) {
        e->buf.len = code->size;

        if (code->uri) {
            if (r->ncaptures && (r->quoted_uri || r->plus_in_uri)) {
                e->buf.len += 2 * ngx_escape_uri(NULL, r->uri.data, r->uri.len,
                                                 NGX_ESCAPE_ARGS);
            }
        }

        for (n = 2; n < r->ncaptures; n += 2) {
            e->buf.len += r->captures[n + 1] - r->captures[n];
        }

    } else {
        // 长度不为空的处理，依次调用长度的处理函数，累加获取到新的uri长度
        ngx_memzero(&le, sizeof(ngx_http_script_engine_t));

        le.ip = code->lengths->elts;
        le.line = e->line;
        le.request = r;
        le.quote = code->redirect;

        len = 0;

        while (*(uintptr_t *) le.ip) {
            lcode = *(ngx_http_script_len_code_pt *) le.ip;
            len += lcode(&le);
        }

        e->buf.len = len;
    }

    // 如果需要增加参数，则buf中增加参数部分长度
    if (code->add_args && r->args.len) {
        e->buf.len += r->args.len + 1;
    }
    // 分配buf空间
    e->buf.data = ngx_pnalloc(r->pool, e->buf.len);
    if (e->buf.data == NULL) {
        e->ip = ngx_http_script_exit;
        e->status = NGX_HTTP_INTERNAL_SERVER_ERROR;
        return;
    }
    
    // quoto标识是否为重定向
    e->quote = code->redirect;
    // 初始变量赋值起始地址
    e->pos = e->buf.data;
    // 即ip向后移动，执行下一个处理类及其函数
    e->ip += sizeof(ngx_http_script_regex_code_t);
}
```

###### ngx_http_regex_exec

执行http正则匹配：

```c
ngx_int_t
ngx_http_regex_exec(ngx_http_request_t *r, ngx_http_regex_t *re, ngx_str_t *s)
{
    ngx_int_t                   rc, index;
    ngx_uint_t                  i, n, len;
    ngx_http_variable_value_t  *vv;
    ngx_http_core_main_conf_t  *cmcf;
    // 获取核心模块
    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    // 如果设置了捕获数量
    if (re->ncaptures) {
        // 存储捕获偏移数组长度，该长度为所有模块中捕获长度最大值+1 再乘以3的结果，可看ngx_http_core_module模块的init方法。
        len = cmcf->ncaptures;
        // 如果捕获数组为null，或者需要重新分配
        if (r->captures == NULL || r->realloc_captures) {
            r->realloc_captures = 0;
            // 分配地址空间
            r->captures = ngx_palloc(r->pool, len * sizeof(int));
            if (r->captures == NULL) {
                return NGX_ERROR;
            }
        }

    } else {
        len = 0;
    }
    // 执行正则匹配
    rc = ngx_regex_exec(re->regex, s, r->captures, len);
    // 不匹配
    if (rc == NGX_REGEX_NO_MATCHED) {
        return NGX_DECLINED;
    }
    // 出错
    if (rc < 0) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
                      ngx_regex_exec_n " failed: %i on \"%V\" using \"%V\"",
                      rc, s, &re->name);
        return NGX_ERROR;
    }
    // 变量每个命名命名捕获
    for (i = 0; i < re->nvariables; i++) {

        n = re->variables[i].capture;
        index = re->variables[i].index;
        vv = &r->variables[index];
        // 对变量赋值
        vv->len = r->captures[n + 1] - r->captures[n];
        vv->valid = 1;
        vv->no_cacheable = 0;
        vv->not_found = 0;
        vv->data = &s->data[r->captures[n]];

#if (NGX_DEBUG)
        ...
#endif
    }
    // 设置请求的ncaptures长度为获取得到的捕获数组*2
    r->ncaptures = rc * 2;
    // 捕获对应的值为进行正则匹配的字符串
    r->captures_data = s->data;

    return NGX_OK;
}
```



##### ngx_http_script_compile

该函数负责对新生成的uri的构建工作，对表达式进行解析，并在codes中增加响应的处理函数。

```c
ngx_int_t
ngx_http_script_compile(ngx_http_script_compile_t *sc)
{
    u_char       ch;
    ngx_str_t    name;
    ngx_uint_t   i, bracket;
    // 初始化相关参数，负责根据先前解析的变量数量对values和lengths进行初始化
    if (ngx_http_script_init_arrays(sc) != NGX_OK) {
        return NGX_ERROR;
    }
    // 解析
    for (i = 0; i < sc->source->len; /* void */ ) {
        // 记录命名变量的长度
        name.len = 0;
        // 如果为$,则表示是变量的开始位置
        if (sc->source->data[i] == '$') {
            // 如果下一个是结尾，说明配置错误
            if (++i == sc->source->len) {
                goto invalid_variable;
            }
            // 如果指为1-9之间的数字
            if (sc->source->data[i] >= '1' && sc->source->data[i] <= '9') {
#if (NGX_PCRE)
                ngx_uint_t  n;
                // 计算实际对应的捕获值数字
                n = sc->source->data[i] - '0';
                // 如果该值在之前已经使用过，则记录dup_capture复用捕获为1
                if (sc->captures_mask & ((ngx_uint_t) 1 << n)) {
                    sc->dup_capture = 1;
                }
                // 记录该值被使用过
                sc->captures_mask |= (ngx_uint_t) 1 << n;
                // 增加对应的处理函数，包含计算变量长度函数和变量值函数
                if (ngx_http_script_add_capture_code(sc, n) != NGX_OK) {
                    return NGX_ERROR;
                }
                // 向后移动i
                i++;

                continue;
#else
                ngx_conf_log_error(NGX_LOG_EMERG, sc->cf, 0,
                                   "using variable \"$%c\" requires "
                                   "PCRE library", sc->source->data[i]);
                return NGX_ERROR;
#endif
            }
            // 如果值为{,则记录获取到左括号
            if (sc->source->data[i] == '{') {
                bracket = 1;
                // 同样，如果立即结束了，则报错
                if (++i == sc->source->len) {
                    goto invalid_variable;
                }
                // 记录命名捕获的变量名
                name.data = &sc->source->data[i];

            } else {
                // 否则，仅记录变量开始位置，即变量配置允许$name和${name}两种
                bracket = 0;
                name.data = &sc->source->data[i];
            }
            // 获取变量名长度
            for ( /* void */ ; i < sc->source->len; i++, name.len++) {
                ch = sc->source->data[i];
                // 获取右括号，并且之前有左括号，则找到变量名长度了
                if (ch == '}' && bracket) {
                    i++;
                    bracket = 0;
                    break;
                }
                // 允许的合法值
                if ((ch >= 'A' && ch <= 'Z')
                    || (ch >= 'a' && ch <= 'z')
                    || (ch >= '0' && ch <= '9')
                    || ch == '_')
                {
                    continue;
                }

                break;
            }
            // 如果只有左括号，在找到右括号前break，则配置错误
            if (bracket) {
                ngx_conf_log_error(NGX_LOG_EMERG, sc->cf, 0,
                                   "the closing bracket in \"%V\" "
                                   "variable is missing", &name);
                return NGX_ERROR;
            }
            // 长度为0，配置错误
            if (name.len == 0) {
                goto invalid_variable;
            }
            // 变量值加1
            sc->variables++;
            // 增加对于命名变量的处理方法，包括计算程度和value
            if (ngx_http_script_add_var_code(sc, &name) != NGX_OK) {
                return NGX_ERROR;
            }

            continue;
        }
        /*
        如果是?则是变量的起始位置，即用户自定义了相关参数，如果设置了需要编译参数，则进行处理
        这里需要注意，其实后续的参数是通过剩余三种方式进行赋值的，常量拷贝，命名捕获和匿名捕获，这里只是将?本身剔除，用于在rewrite中包含用户新增参数的处理。
        */
        if (sc->source->data[i] == '?' && sc->compile_args) {
            sc->args = 1;
            sc->compile_args = 0;
            // 增加对变量的处理
            if (ngx_http_script_add_args_code(sc) != NGX_OK) {
                return NGX_ERROR;
            }

            i++;

            continue;
        }
        // 其他情况是简单的值拷贝，获取起始地址及其长度
        name.data = &sc->source->data[i];

        while (i < sc->source->len) {
            // 知道$或?都是需要拷贝的值
            if (sc->source->data[i] == '$') {
                break;
            }

            if (sc->source->data[i] == '?') {

                sc->args = 1;

                if (sc->compile_args) {
                    break;
                }
            }

            i++;
            name.len++;
        }

        sc->size += name.len;
        // 增加简单的值拷贝处理函数
        if (ngx_http_script_add_copy_code(sc, &name, (i == sc->source->len))
            != NGX_OK)
        {
            return NGX_ERROR;
        }
    }
    // 进行首位工作，主要包括对length中增加结尾，如果需要末尾增加\0,则增加对应的赋值处理，如果是获取文件，则根据相对目录，增加对最终的绝对路径的计算
    return ngx_http_script_done(sc);

invalid_variable:

    ngx_conf_log_error(NGX_LOG_EMERG, sc->cf, 0, "invalid variable name");

    return NGX_ERROR;
}
```

###### ngx_http_script_add_capture_code

添加匿名捕获处理函数：

```c
// 其中n为匿名捕获对应的数值
static ngx_int_t
ngx_http_script_add_capture_code(ngx_http_script_compile_t *sc, ngx_uint_t n)
{
    ngx_http_script_copy_capture_code_t  *code;
    // 向lengths中增加计算长度的处理
    code = ngx_http_script_add_code(*sc->lengths,
                                    sizeof(ngx_http_script_copy_capture_code_t),
                                    NULL);
    if (code == NULL) {
        return NGX_ERROR;
    }
    // 长度处理函数
    code->code = (ngx_http_script_code_pt) (void *)
                                         ngx_http_script_copy_capture_len_code;
    // 该值为相对位置数组中的偏移
    code->n = 2 * n;

    // 向values中增加计算值的处理函数
    code = ngx_http_script_add_code(*sc->values,
                                    sizeof(ngx_http_script_copy_capture_code_t),
                                    &sc->main);
    if (code == NULL) {
        return NGX_ERROR;
    }
    // 函数逻辑
    code->code = ngx_http_script_copy_capture_code;
    code->n = 2 * n;

    if (sc->ncaptures < n) {
        sc->ncaptures = n;
    }

    return NGX_OK;
}
```

其中ngx_http_script_copy_capture_code_t定义如下：

```c
typedef struct {
    ngx_http_script_code_pt     code; // 处理方法
    uintptr_t                   n; // 相对捕获数组的偏移
} ngx_http_script_copy_capture_code_t;
```

其中长度处理函数逻辑为：

```c
size_t
ngx_http_script_copy_capture_len_code(ngx_http_script_engine_t *e)
{
    int                                  *cap;
    u_char                               *p;
    ngx_uint_t                            n;
    ngx_http_request_t                   *r;
    ngx_http_script_copy_capture_code_t  *code;

    r = e->request;

    code = (ngx_http_script_copy_capture_code_t *) e->ip;
    // 将ip向后移动，用于下个计算
    e->ip += sizeof(ngx_http_script_copy_capture_code_t);
    // 获取长度其实位置在数组中的偏移
    n = code->n;
    // n应该小于捕获的最大长度
    if (n < r->ncaptures) {
        // 获取捕获的偏移数组
        cap = r->captures;
        // 复杂uri处理
        if ((e->is_args || e->quote)
            && (e->request->quoted_uri || e->request->plus_in_uri))
        {
            p = r->captures_data;

            return cap[n + 1] - cap[n]
                   + 2 * ngx_escape_uri(NULL, &p[cap[n]], cap[n + 1] - cap[n],
                                        NGX_ESCAPE_ARGS);
        } else {
            // 计算长度
            return cap[n + 1] - cap[n];
        }
    }

    return 0;
}
```

对应的值处理逻辑为：

```c
void
ngx_http_script_copy_capture_code(ngx_http_script_engine_t *e)
{
    int                                  *cap;
    u_char                               *p, *pos;
    ngx_uint_t                            n;
    ngx_http_request_t                   *r;
    ngx_http_script_copy_capture_code_t  *code;

    r = e->request;

    code = (ngx_http_script_copy_capture_code_t *) e->ip;
    // 将ip向后移动，用于下个计算
    e->ip += sizeof(ngx_http_script_copy_capture_code_t);

    n = code->n;

    pos = e->pos;

    if (n < r->ncaptures) {
        // 获取数组
        cap = r->captures;
        p = r->captures_data;

        if ((e->is_args || e->quote)
            && (e->request->quoted_uri || e->request->plus_in_uri))
        {
            e->pos = (u_char *) ngx_escape_uri(pos, &p[cap[n]],
                                               cap[n + 1] - cap[n],
                                               NGX_ESCAPE_ARGS);
        } else {
            // 进行值拷贝，并将pos向后移动响应长度
            e->pos = ngx_copy(pos, &p[cap[n]], cap[n + 1] - cap[n]);
        }
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                   "http script capture: \"%*s\"", e->pos - pos, pos);
}
```

###### ngx_http_script_add_var_code

该方法增加命名捕获变量的处理：

```c
static ngx_int_t
ngx_http_script_add_var_code(ngx_http_script_compile_t *sc, ngx_str_t *name)
{
    ngx_int_t                    index, *p;
    ngx_http_script_var_code_t  *code;
    // 获取变量对应的index，查看变量章节
    index = ngx_http_get_variable_index(sc->cf, name);

    if (index == NGX_ERROR) {
        return NGX_ERROR;
    }
    // 向flushes中添加索引
    if (sc->flushes) {
        p = ngx_array_push(*sc->flushes);
        if (p == NULL) {
            return NGX_ERROR;
        }

        *p = index;
    }
    // 增加长度计算处理函数
    code = ngx_http_script_add_code(*sc->lengths,
                                    sizeof(ngx_http_script_var_code_t), NULL);
    if (code == NULL) {
        return NGX_ERROR;
    }

    code->code = (ngx_http_script_code_pt) (void *)
                                             ngx_http_script_copy_var_len_code;
    code->index = (uintptr_t) index;
    // 增加值获取处理函数
    code = ngx_http_script_add_code(*sc->values,
                                    sizeof(ngx_http_script_var_code_t),
                                    &sc->main);
    if (code == NULL) {
        return NGX_ERROR;
    }

    code->code = ngx_http_script_copy_var_code;
    code->index = (uintptr_t) index;

    return NGX_OK;
}
```

其中ngx_http_script_var_code_t定义如下：

```c
typedef struct {
    ngx_http_script_code_pt     code; // 处理函数
    uintptr_t                   index; // 所在变量数组中的索引
} ngx_http_script_var_code_t;
```

其中计算长度函数为：

```c
size_t
ngx_http_script_copy_var_len_code(ngx_http_script_engine_t *e)
{
    ngx_http_variable_value_t   *value;
    ngx_http_script_var_code_t  *code;

    code = (ngx_http_script_var_code_t *) e->ip;
    // ip向后移动
    e->ip += sizeof(ngx_http_script_var_code_t);
    // 获取变量值
    if (e->flushed) {
        value = ngx_http_get_indexed_variable(e->request, code->index);

    } else {
        value = ngx_http_get_flushed_variable(e->request, code->index);
    }
    // 返回长度
    if (value && !value->not_found) {
        return value->len;
    }

    return 0;
}
```

获取值函数为：

```c
oid
ngx_http_script_copy_var_code(ngx_http_script_engine_t *e)
{
    u_char                      *p;
    ngx_http_variable_value_t   *value;
    ngx_http_script_var_code_t  *code;

    code = (ngx_http_script_var_code_t *) e->ip;
    // 对ip进行偏移
    e->ip += sizeof(ngx_http_script_var_code_t);

    if (!e->skip) {
        // 获取变量值
        if (e->flushed) {
            value = ngx_http_get_indexed_variable(e->request, code->index);

        } else {
            value = ngx_http_get_flushed_variable(e->request, code->index);
        }
        // 赋值，并移动pos
        if (value && !value->not_found) {
            p = e->pos;
            e->pos = ngx_copy(p, value->data, value->len);

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP,
                           e->request->connection->log, 0,
                           "http script var: \"%*s\"", e->pos - p, p);
        }
    }
```

###### ngx_http_script_add_copy_code

该函数增加简单的值拷贝处理逻辑。

```c
// last判断是否为值的末尾
static ngx_int_t
ngx_http_script_add_copy_code(ngx_http_script_compile_t *sc, ngx_str_t *value,
    ngx_uint_t last)
{
    u_char                       *p;
    size_t                        size, len, zero;
    ngx_http_script_copy_code_t  *code;
    // 如果要在文本末尾增加\0,并且到达了末尾，长度增加ngx_http_script_copy_len_code处理函数
    zero = (sc->zero && last);
    len = value->len + zero;
    // 在ngx_http_script_compile_t中的lengths增加
    code = ngx_http_script_add_code(*sc->lengths,
                                    sizeof(ngx_http_script_copy_code_t), NULL);
    if (code == NULL) {
        return NGX_ERROR;
    }

    code->code = (ngx_http_script_code_pt) (void *)
                                                 ngx_http_script_copy_len_code;
    code->len = len;
    // 申请空间包括一个存储ngx_http_script_copy_code_t结构的空间，和存储常量值长度的空间，并进行了内存对齐
    size = (sizeof(ngx_http_script_copy_code_t) + len + sizeof(uintptr_t) - 1)
            & ~(sizeof(uintptr_t) - 1);

    // 在ngx_http_rewrite_loc_conf_t中的codes增加ngx_http_script_copy_code
    code = ngx_http_script_add_code(*sc->values, size, &sc->main);
    if (code == NULL) {
        return NGX_ERROR;
    }

    code->code = ngx_http_script_copy_code;
    code->len = len;
    // 在ngx_http_script_copy_code_t的后面增加对应的常量值。
    p = ngx_cpymem((u_char *) code + sizeof(ngx_http_script_copy_code_t),
                   value->data, value->len);

    if (zero) {
        *p = '\0';
        // zero变更为0，表示末尾已正常添加\0,后续不用再添加了
        sc->zero = 0;
    }

    return NGX_OK;
}
```

其中ngx_http_script_copy_code_t定义如下：

```c
typedef struct {
    ngx_http_script_code_pt     code; // 函数方法
    uintptr_t                   len; // 值长度，这里并没有直接存储值，而是在该结构后分配对应len空间，来存储值
} ngx_http_script_copy_code_t;
```

计算长度和获取值处理如下：

```c
size_t
ngx_http_script_copy_len_code(ngx_http_script_engine_t *e)
{
    ngx_http_script_copy_code_t  *code;
    
    code = (ngx_http_script_copy_code_t *) e->ip;
    // ip指向下一个指令
    e->ip += sizeof(ngx_http_script_copy_code_t);
    // 返回常量对应的长度
    return code->len;
}

void
ngx_http_script_copy_code(ngx_http_script_engine_t *e)
{
    u_char                       *p;
    ngx_http_script_copy_code_t  *code;

    code = (ngx_http_script_copy_code_t *) e->ip;

    p = e->pos;
    // 把常量拷贝到pos中，将pos后移
    if (!e->skip) {
        e->pos = ngx_copy(p, e->ip + sizeof(ngx_http_script_copy_code_t),
                          code->len);
    }
    // 将ip执行下一个要执行的方法。这里和构建时一样，进行了内存对齐。
    e->ip += sizeof(ngx_http_script_copy_code_t)
          + ((code->len + sizeof(uintptr_t) - 1) & ~(sizeof(uintptr_t) - 1));

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                   "http script copy: \"%*s\"", e->pos - p, p);
}
```

###### ngx_http_script_add_args_code

`ngx_http_script_add_args_code`函数添加对参数的处理。

```c
static ngx_int_t
ngx_http_script_add_args_code(ngx_http_script_compile_t *sc)
{
    uintptr_t   *code;

    code = ngx_http_script_add_code(*sc->lengths, sizeof(uintptr_t), NULL);
    if (code == NULL) {
        return NGX_ERROR;
    }

    *code = (uintptr_t) ngx_http_script_mark_args_code;

    code = ngx_http_script_add_code(*sc->values, sizeof(uintptr_t), &sc->main);
    if (code == NULL) {
        return NGX_ERROR;
    }

    *code = (uintptr_t) ngx_http_script_start_args_code;

    return NGX_OK;
}
```

计算长度和值的函数如下：

```
size_t
ngx_http_script_mark_args_code(ngx_http_script_engine_t *e)
{
    e->is_args = 1;
    e->ip += sizeof(uintptr_t);

    return 1;
}


void
ngx_http_script_start_args_code(ngx_http_script_engine_t *e)
{
    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                   "http script args");

    e->is_args = 1;
    e->args = e->pos;
    e->ip += sizeof(uintptr_t);
}
```



##### ngx_http_script_add_code

该函数向codes数组中添加元素，但多了一个参数。

```c
void *
ngx_http_script_add_code(ngx_array_t *codes, size_t size, void *code)
{
    u_char  *elts, **p;
    void    *new;

    elts = codes->elts;
    // 向array中添加元素
    new = ngx_array_push_n(codes, size);
    if (new == NULL) {
        return NULL;
    }
    // 如果存在code
    if (code) {
        // 如果array数组发生了拷贝，即地址发生了偏移，则计算相关的偏移量，重新对code赋值，找到其所在的新地址
        if (elts != codes->elts) {
            p = code;
            *p += (u_char *) codes->elts - elts;
        }
    }

    return new;
}
```

该处理函数的code含义为，对于添加元素来说，当先前分配的地址不够直接添加时，会生成一个新的更大的地址，这时旧的code地址就无法继续使用了，但执行完该函数后，好需要继续使用code，这时就需要计算新的code所在地址，找到新的地址，继续使用。

##### ngx_http_script_regex_end_code

`ngx_http_script_regex_end_code`函数对完成rewrite方法后进行处理。

```c
void
ngx_http_script_regex_end_code(ngx_http_script_engine_t *e)
{
    u_char                            *dst, *src;
    ngx_http_request_t                *r;
    ngx_http_script_regex_end_code_t  *code;

    code = (ngx_http_script_regex_end_code_t *) e->ip;

    r = e->request;

    e->quote = 0;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http script regex end");
    // 如果是rewrite是重定向
    if (code->redirect) {
        // 对新生成的uri进一步归一化，检查
        dst = e->buf.data;
        src = e->buf.data;

        ngx_unescape_uri(&dst, &src, e->pos - e->buf.data,
                         NGX_UNESCAPE_REDIRECT);

        if (src < e->pos) {
            dst = ngx_movemem(dst, src, e->pos - src);
        }
        // pos存储新的uri
        e->pos = dst;
        // 如果需要增加参数
        if (code->add_args && r->args.len) {
             // 如果用户原本已经有新增的参数，则将原参数拼接的方式是使用&，否则使用?
            *e->pos++ = (u_char) (code->args ? '&' : '?');
            e->pos = ngx_copy(e->pos, r->args.data, r->args.len);
        }
        // 记录新的uri长度
        e->buf.len = e->pos - e->buf.data;

        if (e->log || (r->connection->log->log_level & NGX_LOG_DEBUG_HTTP)) {
            ngx_log_error(NGX_LOG_NOTICE, r->connection->log, 0,
                          "rewritten redirect: \"%V\"", &e->buf);
        }
        /*
        清理headers_out为空
        #define ngx_http_clear_location(r)                                            \
                                                                              \
        if (r->headers_out.location) {                                            \
            r->headers_out.location->hash = 0;                                    \
            r->headers_out.location = NULL;                                       \
        }
        */
        ngx_http_clear_location(r);
        // headers_out中增加一个Location头
        r->headers_out.location = ngx_list_push(&r->headers_out.headers);
        if (r->headers_out.location == NULL) {
            e->ip = ngx_http_script_exit;
            e->status = NGX_HTTP_INTERNAL_SERVER_ERROR;
            return;
        }

        r->headers_out.location->hash = 1;
        ngx_str_set(&r->headers_out.location->key, "Location");
        // 设置Location头值为新的uri，即重定向位置
        r->headers_out.location->value = e->buf;
        // 向后移动ip指针，一般后面是null，结束当前rewrite阶段。
        e->ip += sizeof(ngx_http_script_regex_end_code_t);
        return;
    }
    // 如果需要处理参数，即rewrite后有新增加参数
    if (e->args) {
        // 计算uri长度
        e->buf.len = e->args - e->buf.data;
        // 如果需要增加参数，并且原参数不为空，添加参数
        if (code->add_args && r->args.len) {
            *e->pos++ = '&';
            e->pos = ngx_copy(e->pos, r->args.data, r->args.len);
        }
        // 设置参数长度和值
        r->args.len = e->pos - e->args;
        r->args.data = e->args;

        e->args = NULL;

    } else {
        // 如果不需要单独处理参数，则参数值不变
        e->buf.len = e->pos - e->buf.data;

        if (!code->add_args) {
            r->args.len = 0;
        }
    }

    if (e->log || (r->connection->log->log_level & NGX_LOG_DEBUG_HTTP)) {
        ngx_log_error(NGX_LOG_NOTICE, r->connection->log, 0,
                      "rewritten data: \"%V\", args: \"%V\"",
                      &e->buf, &r->args);
    }

    if (code->uri) {
        // 新的uri为buf值
        r->uri = e->buf;
        // 长度为0则报错
        if (r->uri.len == 0) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "the rewritten URI has a zero length");
            e->ip = ngx_http_script_exit;
            e->status = NGX_HTTP_INTERNAL_SERVER_ERROR;
            return;
        }
        // 根据新的uri设置exten值
        ngx_http_set_exten(r);
    }
    // 向后移动ip，继续处理
    e->ip += sizeof(ngx_http_script_regex_end_code_t);
}
```

#### if 配置



### merge location方法

`ngx_http_rewrite_merge_loc_conf`为mergelocation方法。其执行逻辑为：

```c
static char *
ngx_http_rewrite_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_rewrite_loc_conf_t *prev = parent;
    ngx_http_rewrite_loc_conf_t *conf = child;

    uintptr_t  *code;

    ngx_conf_merge_value(conf->log, prev->log, 0);
    ngx_conf_merge_value(conf->uninitialized_variable_warn,
                         prev->uninitialized_variable_warn, 1);
    ngx_conf_merge_uint_value(conf->stack_size, prev->stack_size, 10);
    // 如果子层级的codes为空，则不需要merge
    if (conf->codes == NULL) {
        return NGX_CONF_OK;
    }
    // 如果子层级和父层级codes一致，（if配置），则跳过merge
    if (conf->codes == prev->codes) {
        return NGX_CONF_OK;
    }
    // 子层级增加一个null结尾
    code = ngx_array_push_n(conf->codes, sizeof(uintptr_t));
    if (code == NULL) {
        return NGX_CONF_ERROR;
    }

    *code = (uintptr_t) NULL;

    return NGX_CONF_OK;
}
```

### postconfiguration方法

```c
static ngx_int_t
ngx_http_rewrite_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_SERVER_REWRITE_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_rewrite_handler;

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_rewrite_handler;

    return NGX_OK;
}
```

向`NGX_HTTP_SERVER_REWRITE_PHASE`和`NGX_HTTP_REWRITE_PHASE`阶段增加处理函数。



## ngx_http_limit_req_module

该模块主要用于限流，在`NGX_HTTP_PREACCESS_PHASE`阶段执行。

```
static ngx_command_t  ngx_http_limit_req_commands[] = {

    { ngx_string("limit_req_zone"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE3,
      ngx_http_limit_req_zone,
      0,
      0,
      NULL },

    { ngx_string("limit_req"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE123,
      ngx_http_limit_req,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("limit_req_log_level"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_enum_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_limit_req_conf_t, limit_log_level),
      &ngx_http_limit_req_log_levels },

    { ngx_string("limit_req_status"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_limit_req_conf_t, status_code),
      &ngx_http_limit_req_status_bounds },

    { ngx_string("limit_req_dry_run"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_limit_req_conf_t, dry_run),
      NULL },

      ngx_null_command
};


static ngx_http_module_t  ngx_http_limit_req_module_ctx = {
    ngx_http_limit_req_add_variables,      /* preconfiguration */
    ngx_http_limit_req_init,               /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_limit_req_create_conf,        /* create location configuration */
    ngx_http_limit_req_merge_conf          /* merge location configuration */
};


ngx_module_t  ngx_http_limit_req_module = {
    NGX_MODULE_V1,
    &ngx_http_limit_req_module_ctx,        /* module context */
    ngx_http_limit_req_commands,           /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

### 相关结构

#### ngx_http_limit_req_conf_t

该结构为模块创建的location的配置。

```c
typedef struct {
    ngx_array_t                  limits; // 限制纬度和相关信息，存储的结构为ngx_http_limit_req_ctx_t
    ngx_uint_t                   limit_log_level; // 超过限制时log等级
    ngx_uint_t                   delay_log_level; // 延迟时log等级
    ngx_uint_t                   status_code; // 超过限制时的返回码
    ngx_flag_t                   dry_run; // 是否为测试运行
} ngx_http_limit_req_conf_t;
```



#### ngx_http_limit_req_ctx_t

该结构为限制的详细信息：

```c
typedef struct {
    ngx_http_limit_req_shctx_t  *sh; // 限制对应的红黑树及，对应限制纬度来说，使用红黑树存储来加快查找
    ngx_slab_pool_t             *shpool; // slab分配内存的结构
    /* integer value, 1 corresponds to 0.001 r/s */
    ngx_uint_t                   rate; // 限制的速率
    ngx_http_complex_value_t     key; // 限制的纬度，例如ip纬度，获取其他纬度，该方法用于结算请求的纬度
    ngx_http_limit_req_node_t   *node; // 红黑树节点信息
} ngx_http_limit_req_ctx_t;
```



#### ngx_http_limit_req_shctx_t

对于限制来说，我们炫耀快速超找到某个限制是否对当前请求是否存在。使用红黑树存储信息来加速查找。

```c
typedef struct {
    ngx_rbtree_t                  rbtree; // 红黑树阶段
    ngx_rbtree_node_t             sentinel; // 红黑树的哨兵节点
    ngx_queue_t                   queue; // 按照访问的先后顺序构建的队列，越近的访问在越前面，方便后续进行空间回收
} ngx_http_limit_req_shctx_t;
```

#### ngx_http_complex_value_t

对于限制的纬度来说，允许用户任意进行设置，因此可能是一个复杂的变量值，因此需要一个结构承接对请求中该纬度的计算。结构如下：

```c
typedef struct {
    ngx_str_t                   value; // 存储计算出的值
    ngx_uint_t                 *flushes; 
    void                       *lengths; // 获取值长度处理函数数组，参考rewrite部分
    void                       *values; // 计算值部分处理函数数组

    union {
        size_t                  size;
    } u;
} ngx_http_complex_value_t;
```



#### ngx_http_limit_req_node_t

该结构存储限制信息：

```c
typedef struct {
    u_char                       color;  // 对应红黑树的color
    u_char                       dummy;
    u_short                      len; 
    ngx_queue_t                  queue; // 构成ngx_http_limit_req_shctx_t的队列元素
    ngx_msec_t                   last; // 最后一次访问事件
    /* integer value, 1 corresponds to 0.001 r/s */
    ngx_uint_t                   excess; // 超过rate的限制数量
    ngx_uint_t                   count; // 当前对应的key，请求的数量
    u_char                       data[1]; // 对应的key值
} ngx_http_limit_req_node_t;
```



### create location

create location函数为ngx_http_limit_req_create_conf，其执行逻辑为：

```
static void *
ngx_http_limit_req_create_conf(ngx_conf_t *cf)
{
    ngx_http_limit_req_conf_t  *conf;
    
    // 生成配置
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_limit_req_conf_t));
    if (conf == NULL) {
        return NULL;
    }

    /*
     * set by ngx_pcalloc():
     *
     *     conf->limits.elts = NULL;
     */

    conf->limit_log_level = NGX_CONF_UNSET_UINT;
    conf->status_code = NGX_CONF_UNSET_UINT;
    conf->dry_run = NGX_CONF_UNSET;

    return conf;
}
```

### limit_req_zone配置

#### 语法及含义

配置语法为：

```nginx
limit_req_zone key zone=name:size rate=rate [sync];

context
http
```

该配置设置一个监控纬度，其中key可以是文本，变量或者联合值，name表示监控的名字，用于在`limit_req`中使用。size为允许为该监控创建的共享内存大小，rate为每一个key，请求的出现的频率。例如：

```
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```

这里的key是请求的二进制地址，这里使用二进制地址而不是`remote_addr`是为了节省空间考虑的，对于ipv4来说，二进制地址占用4byte空间，对于ipv6来说，二进制地址占用16bytes空间。1M大概能够存储16000个地址信息（这里不止有地址，还包括其状态信息）。

如果区域存储耗尽，则删除最近最少使用的状态。 如果即使在此之后无法创建新状态，请求也会因错误而终止。

速率以每秒请求数 (r/s) 为单位指定。 如果需要每秒小于一个请求的速率，则以每分钟请求数 (r/m) 指定。 例如，每秒半请求为 30r/m。

sync 参数启用共享内存区域的同步。

#### 解析配置方法

```c
static char *
ngx_http_limit_req_zone(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    u_char                            *p;
    size_t                             len;
    ssize_t                            size;
    ngx_str_t                         *value, name, s;
    ngx_int_t                          rate, scale;
    ngx_uint_t                         i;
    ngx_shm_zone_t                    *shm_zone;
    ngx_http_limit_req_ctx_t          *ctx;
    ngx_http_compile_complex_value_t   ccv;

    value = cf->args->elts;

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_limit_req_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }
    // 初始化解析复杂变量的结构
    ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));

    ccv.cf = cf;
    // 解析第一个值，即指定的key
    ccv.value = &value[1];
    ccv.complex_value = &ctx->key;
    // 执行解析
    if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    size = 0;
    rate = 1;
    scale = 1;
    name.len = 0;
    // 遍历剩下的配置
    for (i = 2; i < cf->args->nelts; i++) {
        
        // 如果是zone
        if (ngx_strncmp(value[i].data, "zone=", 5) == 0) {
            // 设置name
            name.data = value[i].data + 5;
            // 通过:获取name结尾
            p = (u_char *) ngx_strchr(name.data, ':');

            if (p == NULL) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid zone size \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

            name.len = p - name.data;
            // 获取内存大小
            s.data = p + 1;
            s.len = value[i].data + value[i].len - s.data;
            // 解析大小
            size = ngx_parse_size(&s);

            if (size == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid zone size \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }
            // 过小则报错
            if (size < (ssize_t) (8 * ngx_pagesize)) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "zone \"%V\" is too small", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }
        // 如果是速率
        if (ngx_strncmp(value[i].data, "rate=", 5) == 0) {
            // 获取速率值，即纬度（秒/分钟）
            len = value[i].len;
            p = value[i].data + len - 3;

            if (ngx_strncmp(p, "r/s", 3) == 0) {
                scale = 1;
                len -= 3;

            } else if (ngx_strncmp(p, "r/m", 3) == 0) {
                scale = 60;
                len -= 3;
            }
            // 获取速率
            rate = ngx_atoi(value[i].data + 5, len - 5);
            if (rate <= 0) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid rate \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid parameter \"%V\"", &value[i]);
        return NGX_CONF_ERROR;
    }

    if (name.len == 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "\"%V\" must have \"zone\" parameter",
                           &cmd->name);
        return NGX_CONF_ERROR;
    }
    
    // 计算每秒请求数量，由于直接除以60可能导致结果为0，所有这里乘以了1000
    ctx->rate = rate * 1000 / scale;
    // 添加分配共享内存的结构到cycle中的shared_memory中，会先检查shared_memory是否已经存在，不存在时将添加一个，并且设置对应的参数
    shm_zone = ngx_shared_memory_add(cf, &name, size,
                                     &ngx_http_limit_req_module);
    if (shm_zone == NULL) {
        return NGX_CONF_ERROR;
    }
    // 如果data已经包含值了，说明在这之前已经存在，则报错
    if (shm_zone->data) {
        ctx = shm_zone->data;

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "%V \"%V\" is already bound to key \"%V\"",
                           &cmd->name, &name, &ctx->key.value);
        return NGX_CONF_ERROR;
    }
    // 设置创建共享内存的init方法，这将在解析完配置后，在init cycle中执行，可看相关章节
    shm_zone->init = ngx_http_limit_req_init_zone;
    // 设置data为ngx_http_limit_req_ctx_t结构
    shm_zone->data = ctx;

    return NGX_CONF_OK;
}
```





#### 共享内存初始化

在执行该值前，会已经为`shm_zone->shm.addr`分配了对应size的内存映射(mmap),，可查看`ngx_init_cycle`函数（其中的`ngx_shm_alloc`，会按照shm的size分配大小）。

```c
static ngx_int_t
ngx_http_limit_req_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_http_limit_req_ctx_t  *octx = data;

    size_t                     len;
    ngx_http_limit_req_ctx_t  *ctx;

    ctx = shm_zone->data;
    // 先查进行校验
    if (octx) {
        if (ctx->key.value.len != octx->key.value.len
            || ngx_strncmp(ctx->key.value.data, octx->key.value.data,
                           ctx->key.value.len)
               != 0)
        {
            ngx_log_error(NGX_LOG_EMERG, shm_zone->shm.log, 0,
                          "limit_req \"%V\" uses the \"%V\" key "
                          "while previously it used the \"%V\" key",
                          &shm_zone->shm.name, &ctx->key.value,
                          &octx->key.value);
            return NGX_ERROR;
        }

        ctx->sh = octx->sh;
        ctx->shpool = octx->shpool;

        return NGX_OK;
    }

    // 将mmap生成的内存作为这个slab内存结构
    ctx->shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        ctx->sh = ctx->shpool->data;

        return NGX_OK;
    }
    // 从slab中分配sh
    ctx->sh = ngx_slab_alloc(ctx->shpool, sizeof(ngx_http_limit_req_shctx_t));
    if (ctx->sh == NULL) {
        return NGX_ERROR;
    }

    ctx->shpool->data = ctx->sh;
    
    // 初始化红黑树，红黑树增加方式较为简单，直接进行比较，不用判断是否存在data一致的节点，因为在增加前会先查找
    ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel,
                    ngx_http_limit_req_rbtree_insert_value);
    // 初始化队列
    ngx_queue_init(&ctx->sh->queue);

    // 设置log信息
    len = sizeof(" in limit_req zone \"\"") + shm_zone->shm.name.len;

    ctx->shpool->log_ctx = ngx_slab_alloc(ctx->shpool, len);
    if (ctx->shpool->log_ctx == NULL) {
        return NGX_ERROR;
    }

    ngx_sprintf(ctx->shpool->log_ctx, " in limit_req zone \"%V\"%Z",
                &shm_zone->shm.name);

    ctx->shpool->log_nomem = 0;

    return NGX_OK;
}
```



###  limit_req配置

#### 语法及含义

语法为：

```
limit_req zone=name [burst=number] [nodelay | delay=number];

Context:	http, server, location
```

设置共享内存区域和请求的最大突发大小。 如果请求速率超过为区域配置的速率，则它们的处理将被延迟，以便以定义的速率处理请求。 过多的请求会被延迟，直到它们的数量超过最大突发大小，在这种情况下，请求会因错误而终止。 默认情况下，最大突发大小为零。 例如，指令：

```
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

server {
    location /search/ {
        limit_req zone=one burst=5;
    }
```

平均每秒允许不超过 1 个请求，突发不超过 5 个请求。超过5个的请求会直接丢弃。

如果不希望在请求受到限制时延迟过多的请求，则应使用参数 nodelay：

```
limit_req zone=one burst=5 nodelay;
```

delay 参数指定过多请求延迟的限制。 默认值为零，即所有过多的请求都被延迟。只有在设置了burst时才有意义，delay表示超过设置的速率，但没有超过burst的请求中，前k个会被立即处理，超过delay的数量才会被延迟处理。

可能有几个 limit_req 指令。 例如，以下配置将限制来自单个 IP 地址的请求的处理速率，同时限制虚拟服务器的请求处理速率：

```
limit_req_zone $binary_remote_addr zone=perip:10m rate=1r/s;
limit_req_zone $server_name zone=perserver:10m rate=10r/s;

server {
    ...
    limit_req zone=perip burst=5 nodelay;
    limit_req zone=perserver burst=10;
}
```

#### 配置解析方法

```c
static char *
ngx_http_limit_req(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_limit_req_conf_t  *lrcf = conf;

    ngx_int_t                    burst, delay;
    ngx_str_t                   *value, s;
    ngx_uint_t                   i;
    ngx_shm_zone_t              *shm_zone;
    ngx_http_limit_req_limit_t  *limit, *limits;

    value = cf->args->elts;

    shm_zone = NULL;
    burst = 0;
    delay = 0;
    
    // 遍历参数
    for (i = 1; i < cf->args->nelts; i++) {
        // 获取限制的纬度
        if (ngx_strncmp(value[i].data, "zone=", 5) == 0) {

            s.len = value[i].len - 5;
            s.data = value[i].data + 5;
            // 添加共享内存，这里应该是已经在解析limit_req_zone中已经添加过了，直接返回对应的zone
            shm_zone = ngx_shared_memory_add(cf, &s, 0,
                                             &ngx_http_limit_req_module);
            if (shm_zone == NULL) {
                return NGX_CONF_ERROR;
            }

            continue;
        }
        
        // 获取突发流量的最大限制
        if (ngx_strncmp(value[i].data, "burst=", 6) == 0) {

            burst = ngx_atoi(value[i].data + 6, value[i].len - 6);
            if (burst <= 0) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid burst value \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        // 获取delay数量
        if (ngx_strncmp(value[i].data, "delay=", 6) == 0) {

            delay = ngx_atoi(value[i].data + 6, value[i].len - 6);
            if (delay <= 0) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid delay value \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }
        // 当是nodelay时，delay为无限大，即对所有在burst内的请求都不延迟处理
        if (ngx_strcmp(value[i].data, "nodelay") == 0) {
            delay = NGX_MAX_INT_T_VALUE / 1000;
            continue;
        }

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid parameter \"%V\"", &value[i]);
        return NGX_CONF_ERROR;
    }

    if (shm_zone == NULL) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "\"%V\" must have \"zone\" parameter",
                           &cmd->name);
        return NGX_CONF_ERROR;
    }
    // 在limits中添加一个限制
    limits = lrcf->limits.elts;

    if (limits == NULL) {
        if (ngx_array_init(&lrcf->limits, cf->pool, 1,
                           sizeof(ngx_http_limit_req_limit_t))
            != NGX_OK)
        {
            return NGX_CONF_ERROR;
        }
    }
    // 查看是否已经使用了该限制
    for (i = 0; i < lrcf->limits.nelts; i++) {
        if (shm_zone == limits[i].shm_zone) {
            return "is duplicate";
        }
    }

    limit = ngx_array_push(&lrcf->limits);
    if (limit == NULL) {
        return NGX_CONF_ERROR;
    }
    // 设置信息信息，其中burst和delay都乘以1000，因为rate也乘以了1000
    limit->shm_zone = shm_zone;
    limit->burst = burst * 1000;
    limit->delay = delay * 1000;

    return NGX_CONF_OK;
}
```

### 其他配置

剩下的配置较为简单，这里进行简单介绍.

`limit_req_log_level`该配置设置在由于超过burst限制时拒绝服务的log等级。

```
Syntax:	limit_req_log_level info | notice | warn | error;
Default:	
limit_req_log_level error;
Context:	http, server, location
```

`limit_req_status`设置拒绝服务时返回的状态码。

```
Syntax:	limit_req_status code;
Default:	
limit_req_status 503;
Context:	http, server, location
```

`limit_req_dry_run`启用试运行模式。 在这种模式下，请求处理速率不受限制，但是在共享内存区域中，过度请求的数量照常计算。

```
Syntax:	limit_req_dry_run on | off;
Default:	
limit_req_dry_run off;
Context:	http, server, location
```

### merge location方法

merge方法较为简单：

```
static char *
ngx_http_limit_req_merge_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_limit_req_conf_t *prev = parent;
    ngx_http_limit_req_conf_t *conf = child;

    if (conf->limits.elts == NULL) {
        conf->limits = prev->limits;
    }
    // merge日志等级
    ngx_conf_merge_uint_value(conf->limit_log_level, prev->limit_log_level,
                              NGX_LOG_ERR);

    conf->delay_log_level = (conf->limit_log_level == NGX_LOG_INFO) ?
                                NGX_LOG_INFO : conf->limit_log_level + 1;
    // merge返回的状态码
    ngx_conf_merge_uint_value(conf->status_code, prev->status_code,
                              NGX_HTTP_SERVICE_UNAVAILABLE);
    // merge是否试运行
    ngx_conf_merge_value(conf->dry_run, prev->dry_run, 0);

    return NGX_CONF_OK;
}
```

### preconfiguration方法

该方法添加一个变量。

```
static ngx_http_variable_t  ngx_http_limit_req_vars[] = {

    { ngx_string("limit_req_status"), NULL,
      ngx_http_limit_req_status_variable, 0, NGX_HTTP_VAR_NOCACHEABLE, 0 },

      ngx_http_null_variable
};

static ngx_int_t
ngx_http_limit_req_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var, *v;

    for (v = ngx_http_limit_req_vars; v->name.len; v++) {
        var = ngx_http_add_variable(cf, &v->name, v->flags);
        if (var == NULL) {
            return NGX_ERROR;
        }

        var->get_handler = v->get_handler;
        var->data = v->data;
    }

    return NGX_OK;
}
```



### postconfiguration方法

该方法向`NGX_HTTP_PREACCESS_PHASE`阶段增加一个处理函数：

```c
static ngx_int_t
ngx_http_limit_req_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_limit_req_handler;

    return NGX_OK;
}
```

### 处理函数ngx_http_limit_req_handler

```c
static ngx_int_t
ngx_http_limit_req_handler(ngx_http_request_t *r)
{
    uint32_t                     hash;
    ngx_str_t                    key;
    ngx_int_t                    rc;
    ngx_uint_t                   n, excess;
    ngx_msec_t                   delay;
    ngx_http_limit_req_ctx_t    *ctx;
    ngx_http_limit_req_conf_t   *lrcf;
    ngx_http_limit_req_limit_t  *limit, *limits;
    
    // 如果已经设置了limit_req_status，则执行下一个处理函数
    if (r->main->limit_req_status) {
        return NGX_DECLINED;
    }
    // 获取location层级的ngx_http_limit_req_limit_t
    lrcf = ngx_http_get_module_loc_conf(r, ngx_http_limit_req_module);
    limits = lrcf->limits.elts;

    excess = 0;

    rc = NGX_DECLINED;

#if (NGX_SUPPRESS_WARN)
    limit = NULL;
#endif
    // 遍历每一个限制纬度
    for (n = 0; n < lrcf->limits.nelts; n++) {
        // 获取对应的限制纬度ngx_http_limit_req_ctx_t
        limit = &limits[n];
        // 获取限制纬度对应的
        ctx = limit->shm_zone->data;
        // 获取请求对应该纬度的key，例如，获取二进制地址
        if (ngx_http_complex_value(r, &ctx->key, &key) != NGX_OK) {
            // 如果计算key错误，则拒绝请求，并将之前的zone中关于该次请求的记录删除
            ngx_http_limit_req_unlock(limits, n);
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        // key为0，则跳过
        if (key.len == 0) {
            continue;
        }
        // key过长，报错
        if (key.len > 65535) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "the value of the \"%V\" key "
                          "is more than 65535 bytes: \"%V\"",
                          &ctx->key.value, &key);
            continue;
        }
        // 使用crc32算法计算hash值
        hash = ngx_crc32_short(key.data, key.len);
        // 将该zone结构整体锁住
        ngx_shmtx_lock(&ctx->shpool->mutex);
        // 查到对应的key在红黑树中的位置，如果没有，则新增
        rc = ngx_http_limit_req_lookup(limit, hash, &key, &excess,
                                       (n == lrcf->limits.nelts - 1));
        // 释放锁
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        ngx_log_debug4(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "limit_req[%ui]: %i %ui.%03ui",
                       n, rc, excess / 1000, excess % 1000);
        // 如果返回值不是NGX_AGAIN继续超找，则跳出循环
        if (rc != NGX_AGAIN) {
            break;
        }
    }
    // 如果返回NGX_DECLINED，则执行该阶段下一个处理函数
    if (rc == NGX_DECLINED) {
        return NGX_DECLINED;
    }
    // 如果返回NGX_BUSY，超过了最大限制，或者错误
    if (rc == NGX_BUSY || rc == NGX_ERROR) {

        if (rc == NGX_BUSY) {
            ngx_log_error(lrcf->limit_log_level, r->connection->log, 0,
                        "limiting requests%s, excess: %ui.%03ui by zone \"%V\"",
                        lrcf->dry_run ? ", dry run" : "",
                        excess / 1000, excess % 1000,
                        &limit->shm_zone->shm.name);
        }
        // 删除计算到的n个zone中关于该请求的值。
        ngx_http_limit_req_unlock(limits, n);

        if (lrcf->dry_run) {
            r->main->limit_req_status = NGX_HTTP_LIMIT_REQ_REJECTED_DRY_RUN;
            return NGX_DECLINED;
        }
        
        r->main->limit_req_status = NGX_HTTP_LIMIT_REQ_REJECTED;
        // 返回状态码
        return lrcf->status_code;
    }

    /* rc == NGX_AGAIN || rc == NGX_OK */
    // 如果返回NGX_AGAIN，则设置超过rate的请求数量为0
    if (rc == NGX_AGAIN) {
        excess = 0;
    }
    // 计算所有纬度下，该请求需要延迟的大小，返回最大值
    delay = ngx_http_limit_req_account(limits, n, &excess, &limit);
    // 如果不需要延迟，则直接执行下一阶段
    if (!delay) {
        r->main->limit_req_status = NGX_HTTP_LIMIT_REQ_PASSED;
        return NGX_DECLINED;
    }

    ngx_log_error(lrcf->delay_log_level, r->connection->log, 0,
                  "delaying request%s, excess: %ui.%03ui, by zone \"%V\"",
                  lrcf->dry_run ? ", dry run" : "",
                  excess / 1000, excess % 1000, &limit->shm_zone->shm.name);

    if (lrcf->dry_run) {
        r->main->limit_req_status = NGX_HTTP_LIMIT_REQ_DELAYED_DRY_RUN;
        return NGX_DECLINED;
    }
    // 设置为请求被delayed
    r->main->limit_req_status = NGX_HTTP_LIMIT_REQ_DELAYED;
    // 由于请求被delay，所以客户端可能会关闭请求，设置对应的事件处理函数，将read事件添加到epoll中
    if (ngx_handle_read_event(r->connection->read, 0) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    
    // 设置请求对应的读写事件
    r->read_event_handler = ngx_http_test_reading;
    r->write_event_handler = ngx_http_limit_req_delay;
    // 设置写事件被delayed中
    r->connection->write->delayed = 1;
    // 条件写事件到事件红黑树中
    ngx_add_timer(r->connection->write, delay);

    return NGX_AGAIN;
}
```

#### ngx_http_complex_value计算请求在该zone的key

```c
ngx_int_t
ngx_http_complex_value(ngx_http_request_t *r, ngx_http_complex_value_t *val,
    ngx_str_t *value)
{
    size_t                        len;
    ngx_http_script_code_pt       code;
    ngx_http_script_len_code_pt   lcode;
    ngx_http_script_engine_t      e;

    if (val->lengths == NULL) {
        *value = val->value;
        return NGX_OK;
    }

    ngx_http_script_flush_complex_value(r, val);

    ngx_memzero(&e, sizeof(ngx_http_script_engine_t));
    // 遍历lengths获取key长度
    e.ip = val->lengths;
    e.request = r;
    e.flushed = 1;

    len = 0;

    while (*(uintptr_t *) e.ip) {
        lcode = *(ngx_http_script_len_code_pt *) e.ip;
        len += lcode(&e);
    }

    value->len = len;
    value->data = ngx_pnalloc(r->pool, len);
    if (value->data == NULL) {
        return NGX_ERROR;
    }
    // 遍历values获取值
    e.ip = val->values;
    e.pos = value->data;
    e.buf = *value;

    while (*(uintptr_t *) e.ip) {
        code = *(ngx_http_script_code_pt *) e.ip;
        code((ngx_http_script_engine_t *) &e);
    }

    *value = e.buf;

    return NGX_OK;
}
```



#### ngx_http_limit_req_unlock删除当前请求在前n个zone中的记录

```c
static void
ngx_http_limit_req_unlock(ngx_http_limit_req_limit_t *limits, ngx_uint_t n)
{
    ngx_http_limit_req_ctx_t  *ctx;
    // 遍历前n个zone
    while (n--) {
        ctx = limits[n].shm_zone->data;

        if (ctx->node == NULL) {
            continue;
        }
        // 获取锁
        ngx_shmtx_lock(&ctx->shpool->mutex);
        // 将该节点中count减去1
        ctx->node->count--;
        // 释放锁
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        // 设置当前为查询到节点
        ctx->node = NULL;
    }
}
```



#### ngx_http_limit_req_lookup查找节点

该函数赋值查找节点，如果不存在，则新建节点。

```c
/*
hash为计算出的hash值
key实际值
ep为超过zone限制的rate大小，account为是否是最后一个zone
*/

static ngx_int_t
ngx_http_limit_req_lookup(ngx_http_limit_req_limit_t *limit, ngx_uint_t hash,
    ngx_str_t *key, ngx_uint_t *ep, ngx_uint_t account)
{
    size_t                      size;
    ngx_int_t                   rc, excess;
    ngx_msec_t                  now;
    ngx_msec_int_t              ms;
    ngx_rbtree_node_t          *node, *sentinel;
    ngx_http_limit_req_ctx_t   *ctx;
    ngx_http_limit_req_node_t  *lr;
    // 获取当前时间
    now = ngx_current_msec;

    ctx = limit->shm_zone->data;
    // 获取红黑树根节点
    node = ctx->sh->rbtree.root;
    // 获取红黑树哨兵节点
    sentinel = ctx->sh->rbtree.sentinel;
    // 红黑树超找
    while (node != sentinel) {
        // 找左子树
        if (hash < node->key) {
            node = node->left;
            continue;
        }
        // 找右子树
        if (hash > node->key) {
            node = node->right;
            continue;
        }

        /* hash == node->key */
        // 获取实际节点值
        lr = (ngx_http_limit_req_node_t *) &node->color;
        // 计算是否key一致
        rc = ngx_memn2cmp(key->data, lr->data, key->len, (size_t) lr->len);
        // key一致的处理
        if (rc == 0) {
            // 先从队列移出
            ngx_queue_remove(&lr->queue);
            // 再将节点加入队列头部，表示最近访问
            ngx_queue_insert_head(&ctx->sh->queue, &lr->queue);
            // 计算时间差，这里的last为上次该key对应的请求的时间戳
            ms = (ngx_msec_int_t) (now - lr->last);
            // 如果差值小于1分钟，记录为1毫秒，这里未负值的原因是，多进程情况下可能别的进程在该进程之前获取到了该zone下该key的另一个请求的锁。一般情况下都应该是正数
            if (ms < -60000) {
                ms = 1;
            // 如果二者差值小于1分钟，则任务两个请求同时到达，即不存在时间差
            } else if (ms < 0) {
                ms = 0;
            }
            // 计算当前这个请求到了，导致该zone下该key超过限制rate的请求数量
            excess = lr->excess - ctx->rate * ms / 1000 + 1000;
            // 如果未超过，则是0
            if (excess < 0) {
                excess = 0;
            }

            *ep = excess;
            // 如果超出的数量已经超过了允许突发请求的最大值，则返回busy，拒绝请求
            if ((ngx_uint_t) excess > limit->burst) {
                return NGX_BUSY;
            }
            /*
            如果是最后一个zone，则设置其超过限制rate的数量，并且如果ms大于0，则设置最后一次请求达到事件，返回ok
            这里需要注意的是，为啥只有最后一个请求进行设置，并且没有将count++。
            首先看为啥最后一个请求设置了excess，这是因为在之后超找最大延迟时间时，有一个基准，即以最后一个zone需要延时的时间作为基准，来查看在最后一个zone之前的超时时间是否有超过该值的，并设置最大值作为请求的实际需要的延迟事件。
            再来看为啥不对count++：这是因为，请求速率限制其实是使用excess计算是否超限的，因此计算完成excess后就不需要关注实际count数量了，该值仅会用来作为空间回收时的参考，即该值不为0，表示对应的key上存在请求，不能进行空间回收，但对于最后一个zone来说，所有的cunt都是0，不存在该问题，并且这里已经完成了excess的计算，因此不必再进行+1的操作，后续计算前面zone的excess时也可以看出来，再完成了excess的计算后，就会将count--。
            */ 
            if (account) {
                lr->excess = excess;

                if (ms) {
                    lr->last = now;
                }

                return NGX_OK;
            }
            // 非最后一个zone，对count++
            lr->count++;

            ctx->node = lr;

            return NGX_AGAIN;
        }
        // 如果对比key不一致，则继续查找左/右子树
        node = (rc < 0) ? node->left : node->right;
    }
    // 如果没有超找到，则新增一个节点
    *ep = 0;
    // 计算占用空间大小
    size = offsetof(ngx_rbtree_node_t, color)
           + offsetof(ngx_http_limit_req_node_t, data)
           + key->len;
    // 从队列中释放空闲空间
    ngx_http_limit_req_expire(ctx, 1);
    // 使用slab分配一个siez大小空间
    node = ngx_slab_alloc_locked(ctx->shpool, size);
    // 如果返回为空，表示空间使用完全
    if (node == NULL) {
        // 强制是否空间
        ngx_http_limit_req_expire(ctx, 0);
        // 分配空间
        node = ngx_slab_alloc_locked(ctx->shpool, size);
        // 如果依然没有空间，则返回错误
        if (node == NULL) {
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0,
                          "could not allocate node%s", ctx->shpool->log_ctx);
            return NGX_ERROR;
        }
    }
    // 设置key
    node->key = hash;
    // 设置color为对应数据
    lr = (ngx_http_limit_req_node_t *) &node->color;
    // 设置key长度
    lr->len = (u_short) key->len;
    // 设置超限请求数量为0
    lr->excess = 0;

    ngx_memcpy(lr->data, key->data, key->len);
    // 条件进入红黑树中
    ngx_rbtree_insert(&ctx->sh->rbtree, node);
    // 条件到队列头部
    ngx_queue_insert_head(&ctx->sh->queue, &lr->queue);
    // 如果是最后一个，则设置事件和count，这里count不为1和上面的原因一致
    if (account) {
        lr->last = now;
        lr->count = 0;
        return NGX_OK;
    }

    lr->last = 0;
    lr->count = 1;

    ctx->node = lr;

    return NGX_AGAIN;
}
```

这里需要专门介绍一个超过限制的计算，即代理的：

```
excess = lr->excess - ctx->rate * ms / 1000 + 1000;
```

首先计算是累加的，即计算时使用节点之前的`lr->excess`进行累加，由于超额请求可能是负数，当是负数时，设置的excess为0。

后面的先看`ctx->rate * ms/ 1000`，其中`ctx->rate`为1000*允许速率（即 `k r/s`，每秒k个请求），`ms/1000`为距离上次请求的秒数，因此`ctx->rate * ms / 1000`为对应上次请求与该次请求的这段时间，允许的最大请求数量（再乘以1000），而这段时间的实际到底的请求数量为1个请求（这里为了和前面的1000平衡，因此后面也是1000），因此超过允许请求速率的请求数量为`1000- ctx->rate * ms / 1000`。这样便计算出了超过请求速率的请求数量（乘以1000）。

#### ngx_http_limit_req_expire释放空闲空间

该函数对长时间无请求的节点空间进行释放：

```c
/*
这里n=1表示删除一个或两个请求速率为0的节点
n=0表示强制删除一个最旧未访问的节点和一个或两个请求速率为0的节点
*/

static void
ngx_http_limit_req_expire(ngx_http_limit_req_ctx_t *ctx, ngx_uint_t n)
{
    ngx_int_t                   excess;
    ngx_msec_t                  now;
    ngx_queue_t                *q;
    ngx_msec_int_t              ms;
    ngx_rbtree_node_t          *node;
    ngx_http_limit_req_node_t  *lr;

    now = ngx_current_msec;

    /*
     * n == 1 deletes one or two zero rate entries
     * n == 0 deletes oldest entry by force
     *        and one or two zero rate entries
     */

    while (n < 3) {
        // 如果队列为空，则直接返回。
        if (ngx_queue_empty(&ctx->sh->queue)) {
            return;
        }
        // 从队列中取出一个最老未服务的节点
        q = ngx_queue_last(&ctx->sh->queue);
        // 获取节点对应的信息
        lr = ngx_queue_data(q, ngx_http_limit_req_node_t, queue);
        // 如果最老的节点上还有正在访问的请求，则直接返回，不需要再进行查看了
        if (lr->count) {

            /*
             * There is not much sense in looking further,
             * because we bump nodes on the lookup stage.
             */

            return;
        }
        // 将n增加
        if (n++ != 0) {
            // 获取当前时间与该节点最后一个请求的时间差值
            ms = (ngx_msec_int_t) (now - lr->last);
            ms = ngx_abs(ms);
            // 如果查找小于1分钟，则也直接返回，即最旧的节点也很新，因此没有可以直接删除的节点
            if (ms < 60000) {
                return;
            }
            // 计算该节点超过限制速率的请求数量，这里后面未加1，表示这段时间，没有请求到达
            excess = lr->excess - ctx->rate * ms / 1000;
            // 如果过去了ms毫秒后，该节点未收到请求但依然存在超限的请求，则直接返回
            if (excess > 0) {
                return;
            }
        }
        // 如果节点足够旧，并且请求数量未超限，则直接删除节点
        ngx_queue_remove(q);

        node = (ngx_rbtree_node_t *)
                   ((u_char *) lr - offsetof(ngx_rbtree_node_t, color));
        // 从红黑树中删除
        ngx_rbtree_delete(&ctx->sh->rbtree, node);
        // 释放节点空间
        ngx_slab_free_locked(ctx->shpool, node);
    }
}
```



#### ngx_http_limit_req_account获取最大延迟处理时间

该函数遍历所有限制zone，判断是否需要对请求进行延迟处理，并获取延迟时间：

```c
/*
n为限制数量
ep为最后一个限制超过rate的请求数量
limit为最后一个限制结构
*/

static ngx_msec_t
ngx_http_limit_req_account(ngx_http_limit_req_limit_t *limits, ngx_uint_t n,
    ngx_uint_t *ep, ngx_http_limit_req_limit_t **limit)
{
    ngx_int_t                   excess;
    ngx_msec_t                  now, delay, max_delay;
    ngx_msec_int_t              ms;
    ngx_http_limit_req_ctx_t   *ctx;
    ngx_http_limit_req_node_t  *lr;

    excess = *ep;
    //验证请求是否超过最后一个限制zone设置的delay数量，并根据该值设置延时时间（如果未超过设置的delay数量，则不需要延迟）
    if ((ngx_uint_t) excess <= (*limit)->delay) {
        max_delay = 0;

    } else {
        ctx = (*limit)->shm_zone->data;
        //延迟时间设置
        max_delay = (excess - (*limit)->delay) * 1000 / ctx->rate;
    }
    //遍历每一个限制
    while (n--) {
        ctx = limits[n].shm_zone->data;
        lr = ctx->node;

        if (lr == NULL) {
            continue;
        }
        //获取限制的锁
        ngx_shmtx_lock(&ctx->shpool->mutex);
        //计算本次请求与上次请求的时间间隔
        now = ngx_current_msec;
        ms = (ngx_msec_int_t) (now - lr->last);

        if (ms < -60000) {
            ms = 1;

        } else if (ms < 0) {
            ms = 0;
        }
        //计算超过rate的请求数量
        excess = lr->excess - ctx->rate * ms / 1000 + 1000;

        if (excess < 0) {
            excess = 0;
        }

        if (ms) {
            lr->last = now;
        }
        //设置请求超过数量，并将请求数量减1
        lr->excess = excess;
        lr->count--;
        //释放锁
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        
        ctx->node = NULL;
        //验证请求是否超过限制zone设置的delay数量，并根据该值设置延时时间（如果未超过设置的delay数量，则不需要延迟）
        if ((ngx_uint_t) excess <= limits[n].delay) {
            continue;
        }
        //如果超过delay限制，计算需要的延迟时间
        delay = (excess - limits[n].delay) * 1000 / ctx->rate;
        //更新max_delay
        if (delay > max_delay) {
            max_delay = delay;
            *ep = excess;
            *limit = &limits[n];
        }
    }
    //返回需要的最大延迟时间
    return max_delay;
}
```

这里着重介绍一下请求延迟时间的计算：

```
delay = (excess - limits[n].delay) * 1000 / ctx->rate;
```

其中excess为请求超过rate的数量，`excess - limits[n].delay`是请求超过delay限制的数量，可类比为距离，` ctx->rate`为请求限制速度，相除即得延迟时间，这里乘以1000，是因为ngix时间维度以毫秒为单位。

#### ngx_http_test_reading延迟处理期间读事件处理

```c
void
ngx_http_test_reading(ngx_http_request_t *r)
{
    int                n;
    char               buf[1];
    ngx_err_t          err;
    ngx_event_t       *rev;
    ngx_connection_t  *c;

    c = r->connection;
    rev = c->read;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "http test reading");

#if (NGX_HTTP_V2)

   ...

#endif

#if (NGX_HAVE_KQUEUE)
   ...
#endif

#if (NGX_HAVE_EPOLLRDHUP)

    if ((ngx_event_flags & NGX_USE_EPOLL_EVENT) && ngx_use_epoll_rdhup) {
        socklen_t  len;

        if (!rev->pending_eof) {
            return;
        }

        rev->eof = 1;
        c->error = 1;

        err = 0;
        len = sizeof(ngx_err_t);

        /*
         * BSDs and Linux return 0 and set a pending error in err
         * Solaris returns -1 and sets errno
         */

        if (getsockopt(c->fd, SOL_SOCKET, SO_ERROR, (void *) &err, &len)
            == -1)
        {
            err = ngx_socket_errno;
        }

        goto closed;
    }

#endif
    // 从套接字中读取数据
    n = recv(c->fd, buf, 1, MSG_PEEK);
    //如果获取数据为0，表示客户端已关闭，如果是-1，表示出错
    if (n == 0) {
        rev->eof = 1;
        c->error = 1;
        err = 0;

        goto closed;

    } else if (n == -1) {
        err = ngx_socket_errno;

        if (err != NGX_EAGAIN) {
            rev->eof = 1;
            c->error = 1;

            goto closed;
        }
    }

    /* aio does not call this handler */

    if ((ngx_event_flags & NGX_USE_LEVEL_EVENT) && rev->active) {

        if (ngx_del_event(rev, NGX_READ_EVENT, 0) != NGX_OK) {
            ngx_http_close_request(r, 0);
        }
    }

    return;

closed:

    if (err) {
        rev->error = 1;
    }

#if (NGX_HTTP_SSL)
    if (c->ssl) {
        c->ssl->no_send_shutdown = 1;
    }
#endif

    ngx_log_error(NGX_LOG_INFO, c->log, err,
                  "client prematurely closed connection");

    ngx_http_finalize_request(r, NGX_HTTP_CLIENT_CLOSED_REQUEST);
}
```

#### ngx_http_limit_req_delay请求延时期间写事件处理

```c
static void
ngx_http_limit_req_delay(ngx_http_request_t *r)
{
    ngx_event_t  *wev;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "limit_req delay");

    wev = r->connection->write;
    //如果写事件处于delay状态，将写事件加入epoll事件驱动模块中
    if (wev->delayed) {
        if (ngx_handle_write_event(wev, 0) != NGX_OK) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        }

        return;
    }
    
    //将读事件加入epoll事件驱动模块中
    if (ngx_handle_read_event(r->connection->read, 0) != NGX_OK) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }
    //设置请求读写事件为继续执行剩余模块
    r->read_event_handler = ngx_http_block_reading;
    r->write_event_handler = ngx_http_core_run_phases;

    ngx_http_core_run_phases(r);
}
```

这里需要参考`ngx_http_request_handler`处理函数，即写事件触发的处理函数。在写事件被触发时，仅有delayed是1，并且timeout为1时才会将delayed置为0。因此写事件同时被加入的事件红黑树中了，这时，在红黑树超时之前，epoll触发的写事件都不会设置请求为继续向下执行，在红黑树中的事件超时时，写事件的delayed将被设置为0，这时请求将会被向下继续执行，以此来达到限制请求处理的速率。

至此完成了`ngx_http_limit_req_module`模块的详细介绍。

## ngx_http_limit_conn_module模块

除了上述的模块限速以外，还有`ngx_http_limit_conn_module`限速模块，这里不详细对该模块的原理进行介绍。注意讲一下相关配置和大概的实现原理。ngx_http_limit_conn_module 模块用于限制每个定义的键的连接数，特别是来自单个 IP 地址的连接数。

### 配置

#### limit_conn_zone配置

该配置与`limit_req_zone`配置类似。

```
Syntax:	limit_conn_zone key zone=name:size;
Default:	—
Context:	http
```

设置限制纬度，和为限制分配的共享内存大小。

#### limit_conn

限制纬度下，最大允许的连接数量：

```
Syntax:	limit_conn zone number;
Default:	—
Context:	http, server, location
```

### 实现原理

与`ngx_http_limit_req_module`模块类似，使用共享内存加锁来实现计数，不过区别是，一个请求会将记录的数组增加1，并注册一个cleanup函数，在结束请求时将计数减1，当减到0时就会清除该记录。

# 变量

在读取配置项的时候，会对变量进行初始化，包括nginx提供的内置变量和用户配置文件中的变量。

## 相关结构

```c
// 参数分别为请求r，表示变量值的v，以及一个可能使用到的参数data。
typedef void (*ngx_http_set_variable_pt) (ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
typedef ngx_int_t (*ngx_http_get_variable_pt) (ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);

struct ngx_http_variable_s {
    // 变量名
    ngx_str_t                     name;   /* must be first to build the hash */
    // 如果需要变量最初赋值时就进行变量值的设置，需要实现set_handler方法。内部变量允许在配置中以set的方式重新设置值时，可以实现该方法。
    ngx_http_set_variable_pt      set_handler;
    // 获取变量值方法。nginx官方模块变量解析大都使用该方法。
    ngx_http_get_variable_pt      get_handler;
    // 作为参数传递给get_handler，set_handler方法
    uintptr_t                     data;
    // 变量特性
    ngx_uint_t                    flags;
    // 变量值在请求的缓存数组中索引
    ngx_uint_t                    index;
};

typedef struct {
    // 变量值是一段连续内存中的字符串，len表示长度
    unsigned    len:28;
    // valid表示变量已解析过，数据可用
    unsigned    valid:1;
    // no_cacheable为1表示变量值不可被缓存，和ngx_http_variable_s中的flag相关
    unsigned    no_cacheable:1;
    // no_cacheable为1表示变量值解析过，但未找到相应的值
    unsigned    not_found:1;
    // ngx_http_log_module模块使用，用于日志格式字符转义，其他模块忽略该字段
    unsigned    escape:1;
    // data指向变量值起始地址
    u_char     *data;
} ngx_variable_value_t;
```

其中ngx_http_variable_s中的flag相关取值是如下值的1按位与：

| 值                             | 含义                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| `NGX_HTTP_VAR_CHANGEABLE   1`  | 表示变量值可变，即对同一个请求来说，变量可以被反复修改其值。因此以上定义同一变量名与修改变量等价。 |
| `NGX_HTTP_VAR_NOCACHEABLE  2`  | 不能缓存改变量的值，每次使用变量时都需要重新解析。有些请求的变量会在执行中随着url跳转等动作反复改变。如$uri，如果取上次缓存值，是无法知道是否正确的。 |
|                                |                                                              |
| `NGX_HTTP_VAR_NOHASH       8`  | 不要将该变量hash到散列表。散列表需要消耗空间，如果某个模块设计了一个可选变量提供给其他模块使用，并且要求有其他模块使用该变量时必须索引化再使用（即不能调用ngx_http_get_variable方法获取），这样，这个变量就不用浪费散列表的空间了。 |
| `NGX_HTTP_VAR_WEAK         16` | 弱变量，对于set设置的变量都是弱变量，如果该变量是前缀变量，则会使用前缀变量的get_handler解析方法。 |
| `NGX_HTTP_VAR_PREFIX       32` | 前缀变量，如args_、cookie_、http_等，一般是nginx内置的一系列变量。 |

### 存储变量名的数据结构

变量被存储在main级别下的ngx_http_core_main_conf_t中：

```
typedef struct {
    ...

    ngx_hash_t                 variables_hash; // 存储hash过的变量

    ngx_array_t                variables;         /* ngx_http_variable_t */ // 存储索引过的变量。通过索引获取
    ngx_array_t                prefix_variables;  /* ngx_http_variable_t */ // 前缀变量存储

    ngx_hash_keys_arrays_t    *variables_keys;   // 构建过程中临时存储，用于方便构建hash表的，hash管理结构，用于构建variables_hash

    ...
} ngx_http_core_main_conf_t;
```

对于一个变量，可能存在于多个结构中，即variables_hash或variables或prefix_variables中。这时一个变量可能存在多份ngx_http_variable_t结构，但是需要保证，每个的都要要相等的。保证相等的操作在ngx_http_variables_init_vars方法中完成。具体见下文。

## 解析变量

解析变量使用get_handler方法，其原型如下：

```c
typedef ngx_int_t (*ngx_http_get_variable_pt) (ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
```

其中r和data是用来帮助生成变量值，而v是存放值的载体。结构体v已经分配好内存（调用get_handler的函数负责），分配好的内存不包括字符串变量值。可以使用请求r的内存池来分配新的内存放置变量值，这样请求结束，内存就被释放。参数v的data和len成员指向变量值字符串即完成解析。对于data参数来说，存在多种场景。

### data参数不起作用

如果只是生成一些和用户请求无关的变量值，例如时间、系统负载，那么使用各种方法获得变量后赋值给参数v的data和len即可。或者ngx_http_request_t *r中成员已经足够解析出变量值了，data参数不用也可以。

例如，HTTP架构提供的一个变量body_bytes_sent,表示一个请求的响应包体长度，常用在日志中，其解析方法如下：

```c
static ngx_http_variable_t  ngx_http_core_variables[] = { 
    ...
    { ngx_string("body_bytes_sent"), NULL, ngx_http_variable_body_bytes_sent,
      0, 0, 0 },
    ...
}

static ngx_int_t
ngx_http_variable_body_bytes_sent(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    off_t    sent;
    u_char  *p;
    // 大小直接通过r就能获取到，不需要data，因此请求是data为0
    sent = r->connection->sent - r->header_size;

    if (sent < 0) {
        sent = 0;
    }

    p = ngx_pnalloc(r->pool, NGX_OFF_T_LEN);
    if (p == NULL) {
        return NGX_ERROR;
    }

    v->len = ngx_sprintf(p, "%O", sent) - p;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;
    v->data = p;

    return NGX_OK;
}
```

### data参数作为指针

uintptr_t是一个可以放置指针的整型，所以，uintptr_t被设计为即可以用来做整型偏移，也可以做指针。

对于5类前缀字符串，如http_、args_等，实际每个这样的变量其解析方式都大同小异，遍历解析出来的r->headers_in.headers或者r->headers_in.headers数组，找到变量名再返回其值。为了设计更加通用，就使用data作为指针指向实际的变量名字符串。如下，当出现如http_这样的变量被模块使用时，就把data作为指针来保存实际的变量名字符串v[i].name（ngx_http_variables_init_vars初始化特殊变量时的代码段）。

```c
if (ngx_strncmp(src[i].key.data, "HTTP_", sizeof("HTTP_") - 1) == 0)
{
    v[i].get_hander = ngx_http_variable_unknown_header_in;
    v[i].data = (uintptr_t)&v[i].name;
 }
```

解析变量的方法get_hander将data转换为ngx_str_t*

```c
static ngx_int_t
ngx_http_variable_unknown_header_in(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    return ngx_http_variable_unknown_header(v, (ngx_str_t *) data,
                                            &r->headers_in.headers.part,
                                            sizeof("http_") - 1);
}
```

ngx_http_variable_unknown_header遍历ngx_list_t链表类型的headers数组，找到符合变量名的头部后，将其值作为变量返回。

### data作为内存的相对偏移量

很多时候，变量值很可能是原始HTTP字符流中的一部分连续字符串，如果能够复用，就不用再为变量分配，拷贝内存了。而且HTTP模块在使用get_handler时，HTTP框架可能在请求的自动解析过程中已经得到了需要的变量值。这就是data参数作为整数设计的目的。例如：

```
{ ngx_string("http_host"), NULL, ngx_http_variable_header,
      offsetof(ngx_http_request_t, headers_in.host), 0, 0 },

    { ngx_string("http_user_agent"), NULL, ngx_http_variable_header,
      offsetof(ngx_http_request_t, headers_in.user_agent), 0, 0 },
```

http_host变量对应于ngx_http_request_t结构体里的headers_in成员的host成员，http_user_agent变量对应ngx_http_request_t结构体里的headers_in成员的user_agent成员。对应的处理函数为ngx_http_variable_header：

```c
static ngx_int_t
ngx_http_variable_header(ngx_http_request_t *r, ngx_http_variable_value_t *v,
    uintptr_t data)
{
    ngx_table_elt_t  *h;
    // 找到对应的内存地址
    h = *(ngx_table_elt_t **) ((char *) r + data);
    // 赋值
    if (h) {
        v->len = h->value.len;
        v->valid = 1;
        v->no_cacheable = 0;
        v->not_found = 0;
        v->data = h->value.data;

    } else {
        v->not_found = 1;
    }

    return NGX_OK;
}
```

## 查找变量

### ngx_http_get_variable_index

设置变量被索引，并获得索引号，这是使用ngx_http_get_indexed_variable、ngx_http_get_flushed_variable方法的前置条件。调用它意味着这个变量会被频繁的使用，希望Nginx处理这个变量时效率更加高，体现在：

1. 变量值可以被缓存，重复读取时不用每次都解析。
2. 定义变量的解析方法时，可以通过索引直接查找到该方法进行解析，而不是通过操作散列表。
3. nginx在初始化http请求时，就需要为这个变量预分配好ngx_http_variable_value_t变量结构体。

函数执行逻辑如下：

```c
ngx_int_t
ngx_http_get_variable_index(ngx_conf_t *cf, ngx_str_t *name)
{
    ngx_uint_t                  i;
    ngx_http_variable_t        *v;
    ngx_http_core_main_conf_t  *cmcf;

    if (name->len == 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid variable name \"$\"");
        return NGX_ERROR;
    }
    // 获取main级别下的ngx_http_core_main_conf_t
    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    v = cmcf->variables.elts;
    // 在variables查找是否存在，如果还没有variables，则分配空间
    if (v == NULL) {
        if (ngx_array_init(&cmcf->variables, cf->pool, 4,
                           sizeof(ngx_http_variable_t))
            != NGX_OK)
        {
            return NGX_ERROR;
        }

    } else {
        // 如果查找到相同的变量（不区分大小写），则返回当前已经存在的变量的索引
        for (i = 0; i < cmcf->variables.nelts; i++) {
            if (name->len != v[i].name.len
                || ngx_strncasecmp(name->data, v[i].name.data, name->len) != 0)
            {
                continue;
            }

            return i;
        }
    }
    // 如果为查找到相同的变量，添加到variables，返回对于的索引。
    v = ngx_array_push(&cmcf->variables);
    if (v == NULL) {
        return NGX_ERROR;
    }

    v->name.len = name->len;
    v->name.data = ngx_pnalloc(cf->pool, name->len);
    if (v->name.data == NULL) {
        return NGX_ERROR;
    }

    ngx_strlow(v->name.data, name->data, name->len);

    v->set_handler = NULL;
    v->get_handler = NULL;
    v->data = 0;
    v->flags = 0;
    v->index = cmcf->variables.nelts - 1;

    return v->index;
}
```

### ngx_http_get_indexed_variable

根据ngx_http_get_variable_index得到的索引号，获取被索引过的变量的值。若变量被解析过一次后，其值会被缓存，这样该方法再次调用会直接获取缓存过的值，而不是重新解析。该方法忽略NGX_HTTP_VAR_NOCACHEABLE标识。

其方法如下：

```c
ngx_http_variable_value_t *
ngx_http_get_indexed_variable(ngx_http_request_t *r, ngx_uint_t index)
{
    ngx_http_variable_t        *v;
    ngx_http_core_main_conf_t  *cmcf;
    // 获取这次请求分配的ngx_http_core_main_conf_t
    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    if (cmcf->variables.nelts <= index) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
                      "unknown variable index: %ui", index);
        return NULL;
    }
    // 变量被解析过，无论是否查找到值，都直接返回
    if (r->variables[index].not_found || r->variables[index].valid) {
        return &r->variables[index];
    }
    // 防止循环死链，使用ngx_http_variable_depth记录，起始100
    v = cmcf->variables.elts;

    if (ngx_http_variable_depth == 0) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "cycle while evaluating variable \"%V\"",
                      &v[index].name);
        return NULL;
    }

    ngx_http_variable_depth--;
    // 执行变量的解析函数成功
    if (v[index].get_handler(r, &r->variables[index], v[index].data)
        == NGX_OK)
    {
        ngx_http_variable_depth++;
        // 如果变量是不允许缓存，则增加标志
        if (v[index].flags & NGX_HTTP_VAR_NOCACHEABLE) {
            r->variables[index].no_cacheable = 1;
        }
        // 返回数据
        return &r->variables[index];
    }

    ngx_http_variable_depth++;
    // 解析未查找数据，记录状态
    r->variables[index].valid = 0;
    r->variables[index].not_found = 1;

    return NULL;
}
```

### ngx_http_get_flushed_variable

该方法与ngx_http_get_indexed_variable类似，不过其不忽略NGX_HTTP_VAR_NOCACHEABLE标识。

```c
ngx_http_variable_value_t *
ngx_http_get_flushed_variable(ngx_http_request_t *r, ngx_uint_t index)
{
    ngx_http_variable_value_t  *v;

    v = &r->variables[index];
    // 如果变量被解析过
    if (v->valid || v->not_found) {
        // 如果变量可以被缓存，直接返回
        if (!v->no_cacheable) {
            return v;
        }
        // 否则，表示变量未被解析过
        v->valid = 0;
        v->not_found = 0;
    }
    // 执行解析
    return ngx_http_get_indexed_variable(r, index);
}
```

### ngx_http_get_value

根据变量名称，从hash过的散列表中找到对应的变量，并调用其解析方法，获得值，这里不存在缓存变量值的可能。同时前缀变量也可以从该方法中获取解析值。

```c
ngx_http_variable_value_t *
ngx_http_get_variable(ngx_http_request_t *r, ngx_str_t *name, ngx_uint_t key)
{
    size_t                      len;
    ngx_uint_t                  i, n;
    ngx_http_variable_t        *v;
    ngx_http_variable_value_t  *vv;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    // 在散列表中查找对于的变量
    v = ngx_hash_find(&cmcf->variables_hash, key, name->data, name->len);

    if (v) {
        // 如果变量被找到，并且已经被索引，则优先使用索引列表里面的值
        if (v->flags & NGX_HTTP_VAR_INDEXED) {
            return ngx_http_get_flushed_variable(r, v->index);
        }

        if (ngx_http_variable_depth == 0) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "cycle while evaluating variable \"%V\"", name);
            return NULL;
        }

        ngx_http_variable_depth--;

        vv = ngx_palloc(r->pool, sizeof(ngx_http_variable_value_t));
        // 调用变量的解析方法
        if (vv && v->get_handler(r, vv, v->data) == NGX_OK) {
            ngx_http_variable_depth++;
            return vv;
        }

        ngx_http_variable_depth++;
        return NULL;
    }

    vv = ngx_palloc(r->pool, sizeof(ngx_http_variable_value_t));
    if (vv == NULL) {
        return NULL;
    }

    len = 0;
    // 在前缀变量中查找
    v = cmcf->prefix_variables.elts;
    n = cmcf->prefix_variables.nelts;
    // 找到最长匹配的一个变量（这里只有5个http_,arg_,cookie_,sent_trailer_,sent_http_）
    for (i = 0; i < cmcf->prefix_variables.nelts; i++) {
        if (name->len >= v[i].name.len && name->len > len
            && ngx_strncmp(name->data, v[i].name.data, v[i].name.len) == 0)
        {
            len = v[i].name.len;
            n = i;
        }
    }
    // 调用对应的解析方法
    if (n != cmcf->prefix_variables.nelts) {
        if (v[n].get_handler(r, vv, (uintptr_t) name) == NGX_OK) {
            return vv;
        }

        return NULL;
    }
    // 未查到
    vv->not_found = 1;

    return vv;
}
```



## 添加系统变量

内部变量会在系统初始化是定义。赋值是在需要使用该变量时才赋值，而不是请求到达后就优先赋值。查找变量的方式为根据索引找到系统中相应的变量或者通过hash表来进行查找。

系统变量定义都是在执行模块的`proconfiguration`阶段。该方法添加变量到main级别创建的ngx_http_core_main_conf_t的variables_keys（散列表）或prefix_variables（数组）中。例如，对于核心模块ngx_http_core_module对应的方法ngx_http_core_preconfiguration：

```c
static ngx_int_t
ngx_http_core_preconfiguration(ngx_conf_t *cf)
{
    return ngx_http_variables_add_core_vars(cf);
}

ngx_int_t
ngx_http_variables_add_core_vars(ngx_conf_t *cf)
{
    ngx_http_variable_t        *cv, *v;
    ngx_http_core_main_conf_t  *cmcf;
    // 获取main级别创建的ngx_http_core_main_conf_t
    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    // 分配variables_keys内存
    cmcf->variables_keys = ngx_pcalloc(cf->temp_pool,
                                       sizeof(ngx_hash_keys_arrays_t));
    if (cmcf->variables_keys == NULL) {
        return NGX_ERROR;
    }

    cmcf->variables_keys->pool = cf->pool;
    cmcf->variables_keys->temp_pool = cf->pool;
    // 初始化散列表管理结构ngx_hash_keys_arrays_t
    if (ngx_hash_keys_array_init(cmcf->variables_keys, NGX_HASH_SMALL)
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // 初始化前缀变量的列表
    if (ngx_array_init(&cmcf->prefix_variables, cf->pool, 8,
                       sizeof(ngx_http_variable_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    // 遍历每一个nginx预设的系统变量，将其添加到对应的结构中（前缀列表或者散列表管理结构）.ngx_http_core_variables包含了大量nginx预设变量，如http_host、http_cookie、cookie_
    for (cv = ngx_http_core_variables; cv->name.len; cv++) {
        v = ngx_http_add_variable(cf, &cv->name, cv->flags);
        if (v == NULL) {
            return NGX_ERROR;
        }

        *v = *cv;
    }

    return NGX_OK;
}

//部分ngx_http_core_variables
static ngx_http_variable_t  ngx_http_core_variables[] = {

    { ngx_string("http_host"), NULL, ngx_http_variable_header,
      offsetof(ngx_http_request_t, headers_in.host), 0, 0 },

    { ngx_string("http_user_agent"), NULL, ngx_http_variable_header,
      offsetof(ngx_http_request_t, headers_in.user_agent), 0, 0 },

    { ngx_string("http_referer"), NULL, ngx_http_variable_header,
      offsetof(ngx_http_request_t, headers_in.referer), 0, 0 }
}
```

### ngx_http_add_variable

其中ngx_http_add_variable函数如下：

```c
ngx_http_variable_t *
ngx_http_add_variable(ngx_conf_t *cf, ngx_str_t *name, ngx_uint_t flags)
{
    // name为变量名称，flags为ngx_http_variable_s中的flag。返回值为已经准备好的ngx_http_variable_t。这时需要在返回值在添加解析方法set/get_handler,h和data。
    ngx_int_t                   rc;
    ngx_uint_t                  i;
    ngx_hash_key_t             *key;
    ngx_http_variable_t        *v;
    ngx_http_core_main_conf_t  *cmcf;

    if (name->len == 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid variable name \"$\"");
        return NULL;
    }
    // 如果是前缀变量，则使用ngx_http_add_prefix_variable添加
    if (flags & NGX_HTTP_VAR_PREFIX) {
        return ngx_http_add_prefix_variable(cf, name, flags);
    }

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    key = cmcf->variables_keys->keys.elts;
    for (i = 0; i < cmcf->variables_keys->keys.nelts; i++) {
        if (name->len != key[i].key.len
            || ngx_strncasecmp(name->data, key[i].key.data, name->len) != 0)
        {
            continue;
        }

        v = key[i].value;
        // 如果变量是不可变更的，并且现在已经存在该变量，即出现重复添加，则报错
        if (!(v->flags & NGX_HTTP_VAR_CHANGEABLE)) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "the duplicate \"%V\" variable", name);
            return NULL;
        }

        if (!(flags & NGX_HTTP_VAR_WEAK)) {
            v->flags &= ~NGX_HTTP_VAR_WEAK;
        }

        return v;
    }

    // 分配空间，赋值
    v = ngx_palloc(cf->pool, sizeof(ngx_http_variable_t));
    if (v == NULL) {
        return NULL;
    }

    v->name.len = name->len;
    v->name.data = ngx_pnalloc(cf->pool, name->len);
    if (v->name.data == NULL) {
        return NULL;
    }
    // 使用小写存储变量名
    ngx_strlow(v->name.data, name->data, name->len);

    v->set_handler = NULL;
    v->get_handler = NULL;
    v->data = 0;
    v->flags = flags;
    v->index = 0;
    // 将变量添加到variables_keys散列表中,散列表添加元素见上文,这里添加的变量是不带通配符的变量，这里增加的变量也不允许重复
    rc = ngx_hash_add_key(cmcf->variables_keys, &v->name, v, 0);

    if (rc == NGX_ERROR) {
        return NULL;
    }

    if (rc == NGX_BUSY) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "conflicting variable name \"%V\"", name);
        return NULL;
    }

    return v;
}
```

其中ngx_http_add_prefix_variable如下：

```c
static ngx_http_variable_t *
ngx_http_add_prefix_variable(ngx_conf_t *cf, ngx_str_t *name, ngx_uint_t flags)
{
    ngx_uint_t                  i;
    ngx_http_variable_t        *v;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    v = cmcf->prefix_variables.elts;
    for (i = 0; i < cmcf->prefix_variables.nelts; i++) {
        if (name->len != v[i].name.len
            || ngx_strncasecmp(name->data, v[i].name.data, name->len) != 0)
        {
            continue;
        }

        v = &v[i];
        // 对于已定义过的，并且为不可变的变量，重复定义报错
        if (!(v->flags & NGX_HTTP_VAR_CHANGEABLE)) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "the duplicate \"%V\" variable", name);
            return NULL;
        }

        if (!(flags & NGX_HTTP_VAR_WEAK)) {
            v->flags &= ~NGX_HTTP_VAR_WEAK;
        }

        return v;
    }
    // 添加到表中（这时可能一个变量在表中有两个元素）
    v = ngx_array_push(&cmcf->prefix_variables);
    if (v == NULL) {
        return NULL;
    }

    v->name.len = name->len;
    v->name.data = ngx_pnalloc(cf->pool, name->len);
    if (v->name.data == NULL) {
        return NULL;
    }

    ngx_strlow(v->name.data, name->data, name->len);

    v->set_handler = NULL;
    v->get_handler = NULL;
    v->data = 0;
    v->flags = flags;
    v->index = 0;

    return v;
}
```

从上可以看出，只有前缀并且设置可以变更的变量，可以重复添加。



## 外部变量和脚本引擎

出了上文提到的使用模块的`preconfiguration`方法来添加变量以外，还可以使用ngx_http_rewrite_moduel模块提供的set配置。其使用了Nginx的脚本引擎。

变量名称在nginx.conf的配置文件里声明，且在配置文件中确定了变量的赋值。模块定义外部变量的格式为：

```
set $variable value
```

配置通过set关键字定义了一个在nginx.conf中指定的新变量variable，并将其赋值为value。value可以是一个文本字符串，还可以包含多个变量，也可以是变量与文本字符串的组合。

### 相关数据结构

同一段脚本被编译进nginx中，在不同请求中执行效果是不一样的，所以每个请求都必须有其独特的脚本执行上下文，或者称为脚本引擎。这由ngx_http_script_engine_t类充当：

```c
typedef struct {
    // 执行待执行的脚本指令
    u_char                     *ip;
    // 
    u_char                     *pos;
    // 变量构成的栈
    ngx_http_variable_value_t  *sp;

    ngx_str_t                   buf;
    ngx_str_t                   line;

    /* the start of the rewritten arguments */
    u_char                     *args;

    unsigned                    flushed:1;
    unsigned                    skip:1;
    unsigned                    quote:1;
    unsigned                    is_args:1;
    unsigned                    log:1;
    // 脚本执行状态
    ngx_int_t                   status;
    // 执行当前脚本引擎所属的HTTP请求
    ngx_http_request_t         *request;
} ngx_http_script_engine_t;
```

sp是一个栈，作为编译工具。大小默认为10.

ip可以理解为IP寄存器，指向下一行将要执行的代码。对于u_char*类型来说，其指向类型是不定的。其指向的一定是待执行的脚本指令。用面向对象的语言来说，其指向的是实现了ngx_http_script_code_pt接口的类。但C语言没有接口的概念，在C语言实现上述目的，通常会使用嵌套结构体的方法，比如表示接口的结构体A，要放在表示实现接口的类-结构体B的第一个位置。这样一个指向B的指针，也可以强制转换为A，再调用A的成员。ngx_http_script_code_pt是一个指针函数：

```
typedef void (*ngx_http_script_code_pt) (ngx_http_script_engine_t *e);
```

ngx_http_script_code_pt参数ngx_http_script_engine_t，其表示当前指令的脚本上下文。

ngx_http_script_code_pt相当于抽象基类的一个接口，会有相应的结构体担当类的角色。对于"set"配置来说，编译变量名（即第一个参数）由一个实现了ngx_http_script_code_pt接口的类担当，这个类为ngx_http_script_var_code_t：

```c
typedef struct {
    // 本例中指向方法为ngx_http_script_set_var_code
    ngx_http_script_code_pt     code;
    // 表示ngx_http_request_t中被索引、缓存的变量数组值variables中，当前解析的、set设置的外部变量是在的索引
    uintptr_t                   index;
} ngx_http_script_var_code_t;
```

第一个成员是ngx_http_script_code_pt，因此可以把ngx_http_script_var_code_t强制转换为ngx_http_script_code_pt执行。

set的第2个参数是变量值，其也需要一个结构体ngx_http_script_value_code_t来编译，其定义如下：

```c
typedef struct {
    // 本例中执行的方法为ngx_http_script_value_code
    ngx_http_script_code_pt     code;
    // 若外部变量是整数，则转换为整数赋值给value，否则value为0
    uintptr_t                   value;
    // 外部变量值（set的第二个参数）长度
    uintptr_t                   text_len;
    // 外部变量值的起始地址
    uintptr_t                   text_data;
} ngx_http_script_value_code_t;
```

为何一行set脚本分别由编译变量名、编译变量值的2个结构来表示呢。因为对于set有很多不同的使用场景，对变量名来说，就存在变量名首次出现与非首次出现，而变量值有纯文字、字符串与其他变量组合等情况。把变量名的编译提取为ngx_http_script_var_code_t结构体，使所以变量名的编译可以复用其index成员，而具体的ngx_http_script_code_pt指令则可以各组实现。把变量值的编译提取为ngx_http_script_value_code_t结构体，则可以复用text_len、text_data成员。

ngx_http_script_engine_t随着HTTP请求到来才创建，所以其无法保存Nginx启动时就编译出的脚本。保存编译后的脚本工作由ngx_http_rewrite_loc_conf_t结构承担。其是ngx_http_rewrite_module_ctx模块在location级别的配置结构体。其定义如下：

```
typedef struct {
    // 保存所属location下的所有编译后的脚本（按照顺序）
    ngx_array_t  *codes;        /* uintptr_t */
    // 每一个请求的ngx_http_script_engine_t脚本引擎中都有一个变量值栈，即上文的ngx_http_script_engine_t的sp，其大小为stack_size
    ngx_uint_t    stack_size;

    ngx_flag_t    log;
    ngx_flag_t    uninitialized_variable_warn;
} ngx_http_rewrite_loc_conf_t;
```

如果location下没有脚本式配置，那么其成员codes数组就是空的，否则codes数组会放置承载者被解析后的脚本指令的结构体。

codes数组设置比较独特。脚本指令都是实现了"接口ngx_http_script_code_pt"的各个不同的充当类的结构体，这些结构体可以是ngx_http_script_var_code_t、ngx_http_script_value_code_t等，其类型不同，占用的内存也不同，如何放入一个数组中呢（不是它们的指针）。通过如下三点做到：

1. codes数组设计每个元素仅占用1字节大小，即不奢望一个数组元素能够存放表示一个脚本指令的结构体。

2. 每次要将1个指令放入codes数组中时，将根据指令结构体的占用内存字节数N，在codes数组中分配N个元素存储这1个指令，再依次把指令结构体的内容都拷贝到这N个数组成员中。例如：

   ```c
   void *
   ngx_http_script_start_code(ngx_pool_t *pool, ngx_array_t **codes, size_t size)
   {
       if (*codes == NULL) {
           *codes = ngx_array_create(pool, 256, 1);
           if (*codes == NULL) {
               return NULL;
           }
       }
   
       return ngx_array_push_n(*codes, size);
   }
   ```

3. HTTP请求到来，脚本指令执行时，每执行完一个脚本指令的ngx_http_script_code_pt方法后，该方法必须主动告知所属指令结构体占用的内存数N，这样从当前指令所在codes数组索引加上N后就是下一条指令。

下图展示了两个请求到来时，解析外部变量的流程：

[![WzSSNq.png](https://z3.ax1x.com/2021/08/01/WzSSNq.png)](https://imgtu.com/i/WzSSNq)

图中以`set $variable value`作为例子，脚本由右向左解析为ngx_http_script_value_code_t,ngx_http_script_var_code_t。

两个http请求（A和B）同时执行到该脚本，其中A请求正准备执行value指令的ngx_http_script_value_code_t，而B请求已经执行完值的入栈，正要执行ngx_http_script_var_code_t。ngx_http_script_engine_t脚本引擎的sp成员始终指向变量值正要操作的值，而ip成员则始终指向将要执行的下一条指令结构体。

### 编译set脚本

编译流程如下：

![image-20210801153020166](/Users/wangyinkui/Library/Application Support/typora-user-images/image-20210801153020166.png)

执行流程为：

[![fFttns.png](https://z3.ax1x.com/2021/08/03/fFttns.png)](https://imgtu.com/i/fFttns)

set在ngx_http_rewrite_module模块的配置如下：

```c
    { ngx_string("set"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_TAKE2,
      ngx_http_rewrite_set,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },
```

其中ngx_http_rewrite_set函数逻辑如下：

```c
static char *
ngx_http_rewrite_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_rewrite_loc_conf_t  *lcf = conf;

    ngx_int_t                            index;
    ngx_str_t                           *value;
    ngx_http_variable_t                 *v;
    ngx_http_script_var_code_t          *vcode;
    ngx_http_script_var_handler_code_t  *vhcode;

    value = cf->args->elts;
    // 判断第一个参数第一个字符是否为$
    if (value[1].data[0] != '$') {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid variable name \"%V\"", &value[1]);
        return NGX_CONF_ERROR;
    }
    // 去除$字符
    value[1].len--;
    value[1].data++;
    // 将变量添加到variable_keys中
    v = ngx_http_add_variable(cf, &value[1],
                              NGX_HTTP_VAR_CHANGEABLE|NGX_HTTP_VAR_WEAK);
    if (v == NULL) {
        return NGX_CONF_ERROR;
    }
    // 将变量索引化，返回变量索引的下标
    index = ngx_http_get_variable_index(cf, &value[1]);
    if (index == NGX_ERROR) {
        return NGX_CONF_ERROR;
    }
    // 如果get_handler为空，就重新设置变量的get_handler函数
    if (v->get_handler == NULL) {
        v->get_handler = ngx_http_rewrite_var;
        v->data = index;
    }
    // 处理变量值
    if (ngx_http_rewrite_value(cf, lcf, &value[2]) != NGX_CONF_OK) {
        return NGX_CONF_ERROR;
    }
    // 如果变量存在set_handler函数，则设置对应的ngx_http_script_var_handler_code_t来对变量赋值
    if (v->set_handler) {
        vhcode = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                   sizeof(ngx_http_script_var_handler_code_t));
        if (vhcode == NULL) {
            return NGX_CONF_ERROR;
        }

        vhcode->code = ngx_http_script_var_set_handler_code;
        vhcode->handler = v->set_handler;
        // 这里的data是set设置后的data，而非原始的data。
        vhcode->data = v->data;

        return NGX_CONF_OK;
    }
    // 如果变量没有设置set_handler函数，则使用ngx_http_script_var_code_t结构对变量赋值
    vcode = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                       sizeof(ngx_http_script_var_code_t));
    if (vcode == NULL) {
        return NGX_CONF_ERROR;
    }

    vcode->code = ngx_http_script_set_var_code;
    vcode->index = (uintptr_t) index;

    return NGX_CONF_OK;
}
```

在第四步中，内部变量的get_handler方法是必须实现的，因此通常都是采用惰性求值，即只有读取到这个变量时才会调用get_handler计算出这个值。然而外部变量是不同的。每一次set都会立即给变量重新赋值，同时读取变量时，因为变量值被索引化的，所有可以直接从请求的variables数组里取到set后的值。这样get_handler似乎是没有用的。然而，可能有某些模块会在set脚本执行前就使用外部变量了，此时外部变量的值是不存在的，即缓存的variables数组里变量是空的。因此这里将get_handler定义为ngx_http_rewrite_var，其功能就是在调用时将变量设置为空值。

```c
ngx_http_variable_value_t  ngx_http_variable_null_value =
    ngx_http_variable("");
```

对于处理变量值，即value[2]较为复杂。首先定义了两个新的结构：

```c
typedef struct {
    ngx_conf_t                 *cf; // 配置
    ngx_str_t                  *source; // 需要compile的字符串

    ngx_array_t               **flushes; // 该结构也用于正则匹配，该成员，构建为变量为$1,$2...这种时，存储每个变量对应于哪一个，构成一个数组
    ngx_array_t               **lengths; //处理变量长度的处理子数组，需要执行ngx_http_script_code_pt构成的列表,这里是生成值时执行ngx_http_script_code_pt的列表
    ngx_array_t               **values; // 处理变量内容的处理子数组，对应于rewrite模块的locations层级下的codes执行数组

    ngx_uint_t                  variables; // 值中含有变量的数量
    ngx_uint_t                  ncaptures; // 当前处理时，出现的$n变量的最大值，如配置的最大为$3，那么ncaptures就等于3
    
    /*
     * 以位移的形式保存$1,$2...$9等变量，即响应位置上置1来表示，主要的作用是为dup_capture准备，
     * 正是由于这个mask的存在，才比较容易得到是否有重复的$n出现。
     */
    ngx_uint_t                  captures_mask;
    ngx_uint_t                  size;

    /* 
     * ngx_http_script_regex_code_t的结构
     */
    void                       *main;

    unsigned                    compile_args:1; // 是否需要处理请求参数
    unsigned                    complete_lengths:1; // 是否设置lengths数组的终止符，即NULL
    unsigned                    complete_values:1; // 是否设置values数组的终止符
    unsigned                    zero:1; // values数组运行时，得到的字符串是否追加'\0'结尾
    unsigned                    conf_prefix:1; // 是否在生成的文件名前，追加路径前缀
    unsigned                    root_prefix:1; // 同conf_prefix

    /*
     * 这个标记位主要在rewrite模块里使用，在ngx_http_rewrite中，
     * if (sc.variables == 0 && !sc.dup_capture) {
     *     regex->lengths = NULL;
     * }
     * 没有重复的$n，那么regex->lengths被置为NULL，这个设置很关键，在函数
     * ngx_http_script_regex_start_code中就是通过对regex->lengths的判断，来做不同的处理，
     * 因为在没有重复的$n的时候，可以通过正则自身的captures机制来获取$n，一旦出现重复的，
     * 那么pcre正则自身的captures并不能满足我们的要求，我们需要用自己handler来处理。
     */
    unsigned                    dup_capture:1;
    unsigned                    args:1; // 待compile的字符串中是否发现了'?'
} ngx_http_script_compile_t;

typedef struct {
    ngx_str_t                   value;
    ngx_uint_t                 *flushes;
    void                       *lengths; // 需要执行ngx_http_script_code_pt构成的列表,这里是生成值时执行ngx_http_script_code_pt的列表
    void                       *values;

    union {
        size_t                  size;
    } u;
} ngx_http_complex_value_t;
```

具体代码如下：

```c
static char *
ngx_http_rewrite_value(ngx_conf_t *cf, ngx_http_rewrite_loc_conf_t *lcf,
    ngx_str_t *value)
{
    ngx_int_t                              n;
    ngx_http_script_compile_t              sc;
    ngx_http_script_value_code_t          *val;
    ngx_http_script_complex_value_code_t  *complex;
    // 根据$数量计算其中含有多少个变量
    n = ngx_http_script_variables_count(value);
    // 如果是纯文本，则只需要增加ngx_http_script_value_code_t处理方法即可
    if (n == 0) {
        val = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                         sizeof(ngx_http_script_value_code_t));
        if (val == NULL) {
            return NGX_CONF_ERROR;
        }
        // 尝试转换为整数
        n = ngx_atoi(value->data, value->len);

        if (n == NGX_ERROR) {
            n = 0;
        }

        val->code = ngx_http_script_value_code;
        val->value = (uintptr_t) n;
        val->text_len = (uintptr_t) value->len;
        val->text_data = (uintptr_t) value->data;

        return NGX_CONF_OK;
    }
    // 如果值中存在变量，则需要使用ngx_http_script_complex_value_code_t结构来承接
    complex = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                 sizeof(ngx_http_script_complex_value_code_t));
    if (complex == NULL) {
        return NGX_CONF_ERROR;
    }
    // 其方法为ngx_http_script_complex_value_code
    complex->code = ngx_http_script_complex_value_code;
    complex->lengths = NULL;

    ngx_memzero(&sc, sizeof(ngx_http_script_compile_t));

    sc.cf = cf;
    sc.source = value;
    sc.lengths = &complex->lengths;
    sc.values = &lcf->codes;
    sc.variables = n;
    sc.complete_lengths = 1;
    // 使用ngx_http_script_compile_t构建ngx_http_script_complex_value_code_t的lengths参数
    if (ngx_http_script_compile(&sc) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
```

其中ngx_http_script_compile函数如下：

```c
ngx_int_t
ngx_http_script_compile(ngx_http_script_compile_t *sc)
{
    u_char       ch;
    ngx_str_t    name;
    ngx_uint_t   i, bracket;
    // 初始化sc中的各个数组
    if (ngx_http_script_init_arrays(sc) != NGX_OK) {
        return NGX_ERROR;
    }
    // 遍历需要进行编码的字符串
    for (i = 0; i < sc->source->len; /* void */ ) {

        name.len = 0;
        // 找到变量
        if (sc->source->data[i] == '$') {
            // $是结尾，说明表达式错误
            if (++i == sc->source->len) {
                goto invalid_variable;
            }
            // 如果是$后面跟着数字，即$1,$2这种，则如下处理
            if (sc->source->data[i] >= '1' && sc->source->data[i] <= '9') {
#if (NGX_PCRE)
                ngx_uint_t  n;
                // 获取变量值
                n = sc->source->data[i] - '0';
                // 使用captures_mask查看是否当前变量被使用过
                if (sc->captures_mask & ((ngx_uint_t) 1 << n)) {
                    // 存在重复使用的变量时，dup_capture置1
                    sc->dup_capture = 1;
                }
                // 记录被使用的变量
                sc->captures_mask |= (ngx_uint_t) 1 << n;
                // 添加对应的处理逻辑
                if (ngx_http_script_add_capture_code(sc, n) != NGX_OK) {
                    return NGX_ERROR;
                }

                i++;

                continue;
#else
                ngx_conf_log_error(NGX_LOG_EMERG, sc->cf, 0,
                                   "using variable \"$%c\" requires "
                                   "PCRE library", sc->source->data[i]);
                return NGX_ERROR;
#endif
            }
            // 如果变量后跟着{的处理
            if (sc->source->data[i] == '{') {
                // 记录查找到左括号，需要一个右括号
                bracket = 1;
                // 同样，如果{是末尾，则报错
                if (++i == sc->source->len) {
                    goto invalid_variable;
                }

                name.data = &sc->source->data[i];

            } else {
                bracket = 0;
                name.data = &sc->source->data[i];
            }
            // 查找到当前变量的全名
            for ( /* void */ ; i < sc->source->len; i++, name.len++) {
                ch = sc->source->data[i];

                if (ch == '}' && bracket) {
                    i++;
                    bracket = 0;
                    break;
                }

                if ((ch >= 'A' && ch <= 'Z')
                    || (ch >= 'a' && ch <= 'z')
                    || (ch >= '0' && ch <= '9')
                    || ch == '_')
                {
                    continue;
                }

                break;
            }
            // 如果未找到又括号，则报错
            if (bracket) {
                ngx_conf_log_error(NGX_LOG_EMERG, sc->cf, 0,
                                   "the closing bracket in \"%V\" "
                                   "variable is missing", &name);
                return NGX_ERROR;
            }
            // 变量长度为0报错
            if (name.len == 0) {
                goto invalid_variable;
            }
            // 记录增加一个变量
            sc->variables++;
            // 增加解析时对应的处理逻辑
            if (ngx_http_script_add_var_code(sc, &name) != NGX_OK) {
                return NGX_ERROR;
            }

            continue;
        }
        // 如果查找到了?,并且需要解析请求参数，则执行如下处理
        if (sc->source->data[i] == '?' && sc->compile_args) {
            sc->args = 1;
            sc->compile_args = 0;
            // 增加解析参数对应的处理逻辑
            if (ngx_http_script_add_args_code(sc) != NGX_OK) {
                return NGX_ERROR;
            }

            i++;

            continue;
        }

        name.data = &sc->source->data[i];
        // 如果无上述情况，即是简单的常量字符串部分时
        while (i < sc->source->len) {
            // 找到变量起始标识，break
            if (sc->source->data[i] == '$') {
                break;
            }
            // 找到?，并且需要解析请求参数时，break。不解析请求参数时，?仅仅当做一个简单的字符串处理
            if (sc->source->data[i] == '?') {

                sc->args = 1;

                if (sc->compile_args) {
                    break;
                }
            }

            i++;
            name.len++;
        }

        sc->size += name.len;
        // 增加对应的解析常量的处理
        if (ngx_http_script_add_copy_code(sc, &name, (i == sc->source->len))
            != NGX_OK)
        {
            return NGX_ERROR;
        }
    }
    // 解析完成后的最后处理
    return ngx_http_script_done(sc);

invalid_variable:

    ngx_conf_log_error(NGX_LOG_EMERG, sc->cf, 0, "invalid variable name");

    return NGX_ERROR;
}
```

对于set的脚本变量，ngx_http_script_complex_value_code_t结构的lengths是用来计算变量值所占空间大小，code成员用来在ngx_http_script_engine_t的sp中分配对应的空间。之后使用ngx_http_rewrite_loc_conf_t中的codes成员（一般包含多个）来对分配的内容进行赋值。而后使用ngx_http_rewrite_loc_conf_t中的codes中对变量名的解析方法，来将对应的值赋值给对应的变量。

总体上构建处理函数的顺序为：

1. 先将ngx_http_script_complex_value_code_t添加到ngx_http_rewrite_loc_conf_t中的codes成员中。
2. 将变量解析的函数的每一部分（一般会含有多个部分）的处理函数（2种处理函数）依次添加到ngx_http_script_complex_value_code_t的lengths成员（计算每一部分生成的值长度）和ngx_http_rewrite_loc_conf_t的codes中（实际执行变量赋值的操作）
3. 将变量名解析的函数增加到ngx_http_rewrite_loc_conf_t中的codes成员中。

执行变量构建的顺序为：

1. 执行ngx_http_rewrite_loc_conf_t中的codes的ngx_http_script_complex_value_code_t成员的code方法，该方法会调用所有的ngx_http_script_complex_value_code_t中的lengths成员的方法，计算变量值总体占用空间，并在ngx_http_script_engine_t的当前sp中分配对应的空间。
2. 接着执行ngx_http_rewrite_loc_conf_t中的codes后续的方法，对分配到sp的空间进行填充，构建完整的变量值。
3. 执行变量名的解析函数，将sp中存储的变量值赋值到对应的变量上。

其中ngx_http_script_compile_t中的lengths对应于ngx_http_script_complex_value_code_t中的lengths成员，ngx_http_script_compile_t中的values对应于ngx_http_rewrite_loc_conf_t中的codes成员。

在此基础上再来查看上述代码。首先来看ngx_http_script_complex_value_code_t的code方法，即ngx_http_rewrite_value方法中的

```c
complex->code = ngx_http_script_complex_value_code;
```

其中ngx_http_script_complex_value_code方法如下：

```c
void
ngx_http_script_complex_value_code(ngx_http_script_engine_t *e)
{
    size_t                                 len;
    ngx_http_script_engine_t               le;
    ngx_http_script_len_code_pt            lcode;
    ngx_http_script_complex_value_code_t  *code;
    // 恢复ngx_http_script_complex_value_code_t类
    code = (ngx_http_script_complex_value_code_t *) e->ip;
    // 将ip执行下一个要执行的方法，这里相当于是移动到ngx_http_rewrite_loc_conf_t中的codes下一个成员
    e->ip += sizeof(ngx_http_script_complex_value_code_t);

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                   "http script complex value");

    ngx_memzero(&le, sizeof(ngx_http_script_engine_t));

    le.ip = code->lengths->elts;
    le.line = e->line;
    le.request = e->request;
    le.quote = e->quote;
    // 依次执行ngx_http_script_complex_value_code_t的lengths函数，计算变量值占用空间大小
    for (len = 0; *(uintptr_t *) le.ip; len += lcode(&le)) {
        lcode = *(ngx_http_script_len_code_pt *) le.ip;
    }
    // 在ngx_http_script_engine_t的buf中分配该空间
    e->buf.len = len;
    e->buf.data = ngx_pnalloc(e->request->pool, len);
    if (e->buf.data == NULL) {
        e->ip = ngx_http_script_exit;
        e->status = NGX_HTTP_INTERNAL_SERVER_ERROR;
        return;
    }
    // 让ngx_http_script_engine_t的pos执行该空间，后续赋值操作都是在依据pos指针
    e->pos = e->buf.data;
    // 在ngx_http_script_engine_t的当前sp的成员中初始化变量值
    e->sp->len = e->buf.len;
    e->sp->data = e->buf.data;
    // 将sp栈上升。这里虽然sp进行了变更，但是由于赋值是，使用的pos成员，因此赋值依旧是原始的sp
    e->sp++;
}
```

再来看一下对不同类型的变量增加相应处理函数的逻辑，这里对于$1类变量较为复杂，目前没有完全理解，暂时先不考虑。这里主要看一下变量名和常量的部分，即如下形式的部分：

```c
set $value "$valueName text"
```

其中$valueName部分对应与变量名，" text"对应与常量表达式。其处理函数分别是ngx_http_script_add_var_code和ngx_http_script_add_copy_code。

先来看一下ngx_http_script_add_var_code：

```c
static ngx_int_t
ngx_http_script_add_var_code(ngx_http_script_compile_t *sc, ngx_str_t *name)
{
    ngx_int_t                    index, *p;
    ngx_http_script_var_code_t  *code;
    // 找到变量的索引
    index = ngx_http_get_variable_index(sc->cf, name);

    if (index == NGX_ERROR) {
        return NGX_ERROR;
    }

    if (sc->flushes) {
        p = ngx_array_push(*sc->flushes);
        if (p == NULL) {
            return NGX_ERROR;
        }

        *p = index;
    }
    // 在ngx_http_script_compile_t中的lengths增加ngx_http_script_copy_var_len_code方法
    code = ngx_http_script_add_code(*sc->lengths,
                                    sizeof(ngx_http_script_var_code_t), NULL);
    if (code == NULL) {
        return NGX_ERROR;
    }
    
    code->code = (ngx_http_script_code_pt) (void *)
                                             ngx_http_script_copy_var_len_code;
    code->index = (uintptr_t) index;
    // 在ngx_http_rewrite_loc_conf_t中的codes增加ngx_http_script_copy_var_code
    code = ngx_http_script_add_code(*sc->values,
                                    sizeof(ngx_http_script_var_code_t),
                                    &sc->main);
    if (code == NULL) {
        return NGX_ERROR;
    }

    code->code = ngx_http_script_copy_var_code;
    code->index = (uintptr_t) index;

    return NGX_OK;
}

size_t
ngx_http_script_copy_var_len_code(ngx_http_script_engine_t *e)
{
    ngx_http_variable_value_t   *value;
    ngx_http_script_var_code_t  *code;

    code = (ngx_http_script_var_code_t *) e->ip;
    // 将ip执行下一个要执行的方法
    e->ip += sizeof(ngx_http_script_var_code_t);
    // 根据index获取变量长度
    if (e->flushed) {
        value = ngx_http_get_indexed_variable(e->request, code->index);

    } else {
        value = ngx_http_get_flushed_variable(e->request, code->index);
    }

    if (value && !value->not_found) {
        return value->len;
    }

    return 0;
}

void
ngx_http_script_copy_var_code(ngx_http_script_engine_t *e)
{
    u_char                      *p;
    ngx_http_variable_value_t   *value;
    ngx_http_script_var_code_t  *code;

    code = (ngx_http_script_var_code_t *) e->ip;
    // 将ip执行下一个要执行的方法
    e->ip += sizeof(ngx_http_script_var_code_t);
    // 通过index获取对应的值
    if (!e->skip) {

        if (e->flushed) {
            value = ngx_http_get_indexed_variable(e->request, code->index);

        } else {
            value = ngx_http_get_flushed_variable(e->request, code->index);
        }

        if (value && !value->not_found) {
        // 把变量值拷贝到pos中，将pos后移
            p = e->pos;
            e->pos = ngx_copy(p, value->data, value->len);

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP,
                           e->request->connection->log, 0,
                           "http script var: \"%*s\"", e->pos - p, p);
        }
    }
}
```

再看一下ngx_http_script_add_copy_code：

```c
static ngx_int_t
ngx_http_script_add_copy_code(ngx_http_script_compile_t *sc, ngx_str_t *value,
    ngx_uint_t last)
{
    u_char                       *p;
    size_t                        size, len, zero;
    ngx_http_script_copy_code_t  *code;
    // 如果要在文本末尾增加\0,并且到达了末尾，长度增加ngx_http_script_copy_len_code处理函数
    zero = (sc->zero && last);
    len = value->len + zero;
    // 在ngx_http_script_compile_t中的lengths增加
    code = ngx_http_script_add_code(*sc->lengths,
                                    sizeof(ngx_http_script_copy_code_t), NULL);
    if (code == NULL) {
        return NGX_ERROR;
    }

    code->code = (ngx_http_script_code_pt) (void *)
                                                 ngx_http_script_copy_len_code;
    code->len = len;
    // 申请空间包括一个存储ngx_http_script_copy_code_t结构的空间，和存储常量值长度的空间，并进行了内存对齐
    size = (sizeof(ngx_http_script_copy_code_t) + len + sizeof(uintptr_t) - 1)
            & ~(sizeof(uintptr_t) - 1);

    // 在ngx_http_rewrite_loc_conf_t中的codes增加ngx_http_script_copy_code
    code = ngx_http_script_add_code(*sc->values, size, &sc->main);
    if (code == NULL) {
        return NGX_ERROR;
    }

    code->code = ngx_http_script_copy_code;
    code->len = len;
    // 在ngx_http_script_copy_code_t的后面增加对应的常量值。
    p = ngx_cpymem((u_char *) code + sizeof(ngx_http_script_copy_code_t),
                   value->data, value->len);

    if (zero) {
        *p = '\0';
        // zero变更为0，表示末尾已正常添加\0,后续不用再添加了
        sc->zero = 0;
    }

    return NGX_OK;
}

size_t
ngx_http_script_copy_len_code(ngx_http_script_engine_t *e)
{
    ngx_http_script_copy_code_t  *code;
    
    code = (ngx_http_script_copy_code_t *) e->ip;
    // ip指向下一个指令
    e->ip += sizeof(ngx_http_script_copy_code_t);
    // 返回常量对应的长度
    return code->len;
}

void
ngx_http_script_copy_code(ngx_http_script_engine_t *e)
{
    u_char                       *p;
    ngx_http_script_copy_code_t  *code;

    code = (ngx_http_script_copy_code_t *) e->ip;

    p = e->pos;
    // 把常量拷贝到pos中，将pos后移
    if (!e->skip) {
        e->pos = ngx_copy(p, e->ip + sizeof(ngx_http_script_copy_code_t),
                          code->len);
    }
    // 将ip执行下一个要执行的方法。这里和构建时一样，进行了内存对齐。
    e->ip += sizeof(ngx_http_script_copy_code_t)
          + ((code->len + sizeof(uintptr_t) - 1) & ~(sizeof(uintptr_t) - 1));

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                   "http script copy: \"%*s\"", e->pos - p, p);
}
```

再来看编译变量名的处理逻辑，对应于ngx_http_rewrite_set函数如下代码段：

```c
    // 对于系统内置变量，且支持用户重新使用set设置值的变量来说，会提供set_handler接口，这里使用    
    if (v->set_handler) {
        vhcode = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                   sizeof(ngx_http_script_var_handler_code_t));
        if (vhcode == NULL) {
            return NGX_CONF_ERROR;
        }
        // code为其执行函数
        vhcode->code = ngx_http_script_var_set_handler_code;
        // handler被设置为变量原来的set_handler函数。
        vhcode->handler = v->set_handler;
        vhcode->data = v->data;

        return NGX_CONF_OK;
    }
    // 如果该变量不存在set_handler，则增加ngx_http_script_var_code_t结构来设置变量
    vcode = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                       sizeof(ngx_http_script_var_code_t));
    if (vcode == NULL) {
        return NGX_CONF_ERROR;
    }

    vcode->code = ngx_http_script_set_var_code;
    vcode->index = (uintptr_t) index;
```

ngx_http_script_var_handler_code_t结构专门支持内部变量能够被set重新设置，其定义如下：

```
typedef struct {
    // 变量执行的方法
    ngx_http_script_code_pt     code;
    // 对应于原变量的set_handler方法
    ngx_http_set_variable_pt    handler;
    // 对应数据
    uintptr_t                   data;
} ngx_http_script_var_handler_code_t;
```

先来看一下函数ngx_http_script_var_set_handler_code，其执行逻辑为：

```c
void
ngx_http_script_var_set_handler_code(ngx_http_script_engine_t *e)
{
    ngx_http_script_var_handler_code_t  *code;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                   "http script set var handler");

    code = (ngx_http_script_var_handler_code_t *) e->ip;
    // 将ip执行下一个要执行的方法
    e->ip += sizeof(ngx_http_script_var_handler_code_t);
    // 由于在ngx_http_script_complex_value_code_t方法中将sp上移了一个，但赋值是原来的，因此这里将栈下移，获取该变量的数据
    e->sp--;
    // 执行原变量的set_handler方法
    code->handler(e->request, e->sp, code->data);
}
```

看一个具有set_handler方法的例子，如agrs：

```c
    { ngx_string("args"),
      ngx_http_variable_set_args,
      ngx_http_variable_request,
      offsetof(ngx_http_request_t, args),
      NGX_HTTP_VAR_CHANGEABLE|NGX_HTTP_VAR_NOCACHEABLE, 0 },
      
static void
ngx_http_variable_set_args(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    // 这里v即为sp栈数据，这里直接赋值给args
    r->args.len = v->len;
    r->args.data = v->data;
    r->valid_unparsed_uri = 0;
}
```

同样，其取变量的方式是通过offsetof(ngx_http_request_t, args)，即r->args里面进行字符串匹配查找数据。

对应不包含set_handler的变量来说，其处理逻辑如下：

```c
void
ngx_http_script_set_var_code(ngx_http_script_engine_t *e)
{
    ngx_http_request_t          *r;
    ngx_http_script_var_code_t  *code;
    // 由于在ngx_http_script_complex_value_code_t方法中将sp上移了一个，但赋值是原来的，因此这里将栈下移，获取该变量的数据
    code = (ngx_http_script_var_code_t *) e->ip;
    // 将ip执行下一个要执行的方法
    e->ip += sizeof(ngx_http_script_var_code_t);

    r = e->request;

    e->sp--;
    // 直接使用索引对数据赋值
    r->variables[code->index].len = e->sp->len;
    r->variables[code->index].valid = 1;
    r->variables[code->index].no_cacheable = 0;
    r->variables[code->index].not_found = 0;
    r->variables[code->index].data = e->sp->data;

#if (NGX_DEBUG)
    {
    ngx_http_variable_t        *v;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    v = cmcf->variables.elts;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                   "http script set $%V", &v[code->index].name);
    }
#endif
}
```

## 变量间merge

在http层的set方法ngx_http_block中的ngx_http_variables_init_vars函数中会对三种变量（hash到变量，索引的变量，前缀变量）进行merge。其方法如下：

```c
ngx_int_t
ngx_http_variables_init_vars(ngx_conf_t *cf)
{
    size_t                      len;
    ngx_uint_t                  i, n;
    ngx_hash_key_t             *key;
    ngx_hash_init_t             hash;
    ngx_http_variable_t        *v, *av, *pv;
    ngx_http_core_main_conf_t  *cmcf;

    /* set the handlers for the indexed http variables */
    // 获取mian层级ngx_http_core_module创建的ngx_http_core_main_conf_t
    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    // 索引后的变量数组
    v = cmcf->variables.elts;
    // 前缀变量数组
    pv = cmcf->prefix_variables.elts;
    // 即将要被hash化的数组
    key = cmcf->variables_keys->keys.elts;
    // 遍历索引化的数据
    for (i = 0; i < cmcf->variables.nelts; i++) {
        // 遍历将要hash化的数组，找到与索引化的元素一样的元素
        for (n = 0; n < cmcf->variables_keys->keys.nelts; n++) {

            av = key[n].value;

            if (v[i].name.len == key[n].key.len
                && ngx_strncmp(v[i].name.data, key[n].key.data, v[i].name.len)
                   == 0)
            {
                // 设置索引化的数组的get_handler为，待hash数组对应元素的get_handler
                v[i].get_handler = av->get_handler;
                v[i].data = av->data;
                // 设置hash化数组中对应元素为已hash化。这里对应与ngx_http_get_value函数中，先查找hash表，找到后查看该变量是否已索引化，对于索引化时，使用索引数组中该变量处理方法
                av->flags |= NGX_HTTP_VAR_INDEXED;
                v[i].flags = av->flags;
                // 设置hash数组中该变量index为在索引化数组中索引，方便查找
                av->index = i;
                // 对于NGX_HTTP_VAR_WEAK变量，需要再查看是否属于前缀变量，否则直接进行下一个hash元素的查找
                if (av->get_handler == NULL
                    || (av->flags & NGX_HTTP_VAR_WEAK))
                {
                    break;
                }

                goto next;
            }
        }

        len = 0;
        av = NULL;
        // 遍历前缀变量，使用最长匹配查找是否为前缀变量，这里有个小bug，v[i].name.len > len应该是pv[n].name.len>len才对（个人理解）
        for (n = 0; n < cmcf->prefix_variables.nelts; n++) {
            if (v[i].name.len >= pv[n].name.len && v[i].name.len > len
                && ngx_strncmp(v[i].name.data, pv[n].name.data, pv[n].name.len)
                   == 0)
            {
                av = &pv[n];
                len = pv[n].name.len;
            }
        }
        // 如果是前缀变量，则设置变量的解析方式为前缀变量的解析方式，设置data为变量名（这是前缀变量get_handler使用方式要求）
        if (av) {
            v[i].get_handler = av->get_handler;
            v[i].data = (uintptr_t) &v[i].name;
            v[i].flags = av->flags;

            goto next;
        }

        if (v[i].get_handler == NULL) {
            ngx_log_error(NGX_LOG_EMERG, cf->log, 0,
                          "unknown \"%V\" variable", &v[i].name);

            return NGX_ERROR;
        }

    next:
        continue;
    }

    // 设置不需要hash化的变量data为null，这时构建hash表就会跳过该变量
    for (n = 0; n < cmcf->variables_keys->keys.nelts; n++) {
        av = key[n].value;

        if (av->flags & NGX_HTTP_VAR_NOHASH) {
            key[n].key.data = NULL;
        }
    }

    // 构建variables_hash表，存储hash后变量。
    hash.hash = &cmcf->variables_hash;
    hash.key = ngx_hash_key;
    hash.max_size = cmcf->variables_hash_max_size;
    hash.bucket_size = cmcf->variables_hash_bucket_size;
    hash.name = "variables_hash";
    hash.pool = cf->pool;
    hash.temp_pool = NULL;

    if (ngx_hash_init(&hash, cmcf->variables_keys->keys.elts,
                      cmcf->variables_keys->keys.nelts)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    cmcf->variables_keys = NULL;

    return NGX_OK;
}

// 这里看一下前缀变量get_handler方法
      { ngx_string("arg_"), NULL, ngx_http_variable_argument,
      0, NGX_HTTP_VAR_NOCACHEABLE|NGX_HTTP_VAR_PREFIX, 0 },

static ngx_int_t
ngx_http_variable_argument(ngx_http_request_t *r, ngx_http_variable_value_t *v,
    uintptr_t data)
{
    // data存储的是变量名
    ngx_str_t *name = (ngx_str_t *) data;

    u_char     *arg;
    size_t      len;
    ngx_str_t   value;
    // 去除前缀arg_
    len = name->len - (sizeof("arg_") - 1);
    arg = name->data + sizeof("arg_") - 1;
    // 在r->args中查找变量
    if (ngx_http_arg(r, arg, len, &value) != NGX_OK) {
        v->not_found = 1;
        return NGX_OK;
    }

    v->data = value.data;
    v->len = value.len;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;

    return NGX_OK;
}
```

## 举例

对于如下配置，输出为：

```
location /url {

    set   $name   “$arg_name”;

    set   $args   “name=jikui”;

    return 200   “$name = $arg_name”;

}

访问url输出结果是:
curl http://127.0.0.1/url?name=test

test = jikui
```

对于`set   $name   “$arg_name”;`来说，`$arg_name`作为前缀变量，不可缓存，因此从请求的r->args中获取。这是$name值为test。对于`set   $args   “name=jikui”;`来说，由于args允许用户set值，因此其存在set_handler函数。ngx_http_script_var_handler_code_t结构的code方法调用其set_handler函数，设置r->args为`"name=jikui"`。由于这两步是顺序执行的，因此此时name是test。args是`name=jikui`。最后的`$name = $arg_name”`。由于set设置的name变量是可缓存的，因此会直接使用执行set脚本时生成的test值，而对于`$arg_name`来说，其是不可缓存的变量，因此会再次读取`r->args`来获取值，因此其值变更为jikui。

# 共享内存

https://zhuanlan.zhihu.com/p/93684036

## ngx_shm_zone_t

其中`ngx_shm_zone_t`结构如下：

```
struct ngx_shm_zone_s {
    void                     *data; // 分配的共享内存
    ngx_shm_t                 shm; // 
    ngx_shm_zone_init_pt      init; // 初始化函数
    void                     *tag; // 绑定的模块
    void                     *sync;
    ngx_uint_t                noreuse;  /* unsigned  noreuse:1; */
};
```



## ngx_shm_t结构

```
typedef struct {
    u_char      *addr; 
    size_t       size; // 需要分配的大小
    ngx_str_t    name; // 绑定的key
    ngx_log_t   *log;
    ngx_uint_t   exists;   /* unsigned  exists:1;  */
} ngx_shm_t;
```





# 锁机制

由于nginx是多进程异步处理，因此很多时候进程间需要使用锁来确保进程安全，进行进程间同步。nginx使用的锁主要涉及三部分内容。方便为[文件锁](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E8%AE%B0%E5%BD%95%E9%94%81)、[信号量](http://www.yinkuiwang.cn/2019/12/18/unix%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/#%E4%BF%A1%E5%8F%B7%E9%87%8F)、原子变量。对于原子变量，可参考如下两篇文章的介绍：

https://www.infoq.cn/article/atomic-operation

https://www.jianshu.com/p/cb7b726e943c

这里主要讲解使用原子变量来构建锁。

## 相关数据结构

首先来看一下相关数据结构。

### ngx_shmtx_t

nginx封装的锁的类。其定义如下：

```c
typedef struct {
#if (NGX_HAVE_ATOMIC_OPS)
    // 如果系统支持原子变量，则优先使用原子变量
    ngx_atomic_t  *lock;
#if (NGX_HAVE_POSIX_SEM)
    // 如果星图支持信号量，则可选信号量
    ngx_atomic_t  *wait; // 当前在等待的进程数量
    ngx_uint_t     semaphore; // 是否正常开启了信号量
    sem_t          sem; // 信号量
#endif
#else
    // 如果不支持原子变量，则使用文件锁来进行
    ngx_fd_t       fd;
    u_char        *name;
#endif
    ngx_uint_t     spin; // 重复尝试获取锁间隔
} ngx_shmtx_t;
```

### ngx_shmtx_sh_t

由于进程间通讯需要各个进程访问同一个地址，因此该类是nginx封装的一个共享内存创建的锁地址。其由对应的使用方来创建。具体可以参考事件模块构建 accept锁的过程。其结构如下：

```c
typedef struct {
    ngx_atomic_t   lock; // 支持原子变量时，对应的原子变量地址
#if (NGX_HAVE_POSIX_SEM)
    ngx_atomic_t   wait; // 支持信号量时，对应的信号量有多少进程在等待的数量
#endif
} ngx_shmtx_sh_t;
```

## 创建锁

使用ngx_shmtx_create方法来创建锁，其逻辑如下：

```c
ngx_int_t
ngx_shmtx_create(ngx_shmtx_t *mtx, ngx_shmtx_sh_t *addr, u_char *name)
{
    mtx->lock = &addr->lock;
    // 如果spin为-1，表示不能使用信号量（默认是不使用的，对于accept锁）
    if (mtx->spin == (ngx_uint_t) -1) {
        return NGX_OK;
    }
    // 尝试获取锁的次数（并非准确值，具体下面介绍）
    mtx->spin = 2048;

#if (NGX_HAVE_POSIX_SEM)
    // 如果支持信号量，则使用wait记录等待信号量的进程数
    mtx->wait = &addr->wait;
    // 初始化信号量为0
    if (sem_init(&mtx->sem, 1, 0) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                      "sem_init() failed");
    } else {
        // 标识正确初始化了信号量
        mtx->semaphore = 1;
    }

#endif

    return NGX_OK;
}
```

## 销毁锁

使用ngx_shmtx_destroy方法来销毁锁：

```c
void
ngx_shmtx_destroy(ngx_shmtx_t *mtx)
{
#if (NGX_HAVE_POSIX_SEM)

    if (mtx->semaphore) {
        if (sem_destroy(&mtx->sem) == -1) {
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                          "sem_destroy() failed");
        }
    }

#endif
}
```

即仅在支持信号量，并且成功初始化了信号量时，需要执行一步信号量销毁操作。

## 非阻塞获取锁

使用ngx_shmtx_trylock函数来非阻塞的获取锁，即如果不能获取锁，则直接返回。

```c
ngx_uint_t
ngx_shmtx_trylock(ngx_shmtx_t *mtx)
{
    return (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid));
}

#define ngx_atomic_cmp_set(lock, old, set)                                    \
    __sync_bool_compare_and_swap(lock, old, set)
```

*mtx->lock表示当前未被其他进程占用，但是在执行ngx_atomic_cmp_set的过程前（\*mtx->lock == 0后），可能已经被其他进程占用了，因此在原子变量中操作获取锁时，只有当前锁的值依然是0，即在该进程锁住变量前，没有其他进程已经对其进行设置，获取到该锁时，才对其进行赋值，表示当前进程占用该锁。当其他进程希望获取该锁时，由于lock值不为0，返回false。

## 阻塞式获取锁

使用ngx_shmtx_lock函数来阻塞式获取锁，即直到获取到该锁才返回。

```c
#define ngx_cpu_pause()             __asm__ ("pause")

#if (NGX_HAVE_SCHED_YIELD)
#define ngx_sched_yield()  sched_yield()
#else
#define ngx_sched_yield()  usleep(1)
#endif

void
ngx_shmtx_lock(ngx_shmtx_t *mtx)
{
    ngx_uint_t         i, n;

    ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0, "shmtx lock");

    for ( ;; ) {
        // 重试获取锁，获取成功立即返回
        if (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid)) {
            return;
        }
        // 多处理器状态下spin才有效
        if (ngx_ncpu > 1) {
            // 循环调用pause，并重试获取锁，pause的次数随着尝试次数指数增加
            for (n = 1; n < mtx->spin; n <<= 1) {

                for (i = 0; i < n; i++) {
                    ngx_cpu_pause();
                }
                // 重复重试获取锁
                if (*mtx->lock == 0
                    && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid))
                {
                    return;
                }
            }
        }

#if (NGX_HAVE_POSIX_SEM)
        // 如果支持信号量，则尝试获取信号量
        if (mtx->semaphore) {
            // 使用原子变量将等待的进程数量+1
            (void) ngx_atomic_fetch_add(mtx->wait, 1);
            // 尝试获取锁，如果成功获取，则将等待进程数减一，返回
            if (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid)) {
                (void) ngx_atomic_fetch_add(mtx->wait, -1);
                return;
            }

            ngx_log_debug1(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                           "shmtx wait %uA", *mtx->wait);
            // 等待其他进程释放信号量（执行减一操作，同一时刻，只有一个进程持有信号量）
            while (sem_wait(&mtx->sem) == -1) {
                // 出错处理
                ngx_err_t  err;

                err = ngx_errno;

                if (err != NGX_EINTR) {
                    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, err,
                                  "sem_wait() failed while waiting on shmtx");
                    break;
                }
            }

            ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                           "shmtx awoke");
            // 获取到信号量，则执行下一次循环，再次尝试获取锁
            continue;
        }

#endif
        // 如果没有信号量，则执行sched_yield()
        ngx_sched_yield();
    }
}
```

这里ngx_cpu_pause()执行的实际是汇编中pause指令，其会减少CPU的消耗，节省电量。指令的本质功能是：让加锁失败的CPU睡眠大约30个clock，从而使得读操作的频率低很多，流水线重排的代价也会很小。

ngx_sched_yield()执行的是sched_yield()函数，sched_yield()会让出当前线程的CPU占有权，然后把线程放到静态优先队列的尾端，然后一个新的线程会占用CPU。sched_yield()这个函数可以使用另一个级别等于或高于当前线程的线程先运行。如果没有符合条件的线程，那么这个函数将会立刻返回然后继续执行当前线程的程序。 这是其于sleep系列函数的本质区别，而sleep则是等待一定时间后等待CPU的调度，然后去获得CPU资源。

对于阻塞的sem_wait函数，其会阻塞进程。

对于使用信号量时，在将信号量-1后，没有对等待进程数量执行减一操作，该操作在释放信号量时进行。

## 释放锁

```c
void
ngx_shmtx_unlock(ngx_shmtx_t *mtx)
{
    if (mtx->spin != (ngx_uint_t) -1) {
        ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0, "shmtx unlock");
    }
    // 释放锁，将原子变量设置回0，该进程获取锁时，原子变量值为该进程的id
    if (ngx_atomic_cmp_set(mtx->lock, ngx_pid, 0)) {
        ngx_shmtx_wakeup(mtx);
    }
}

```

## 释放信号

在使用信号量时，释放信号后，会释放信号量来唤醒其他等待进程。

```c
static void
ngx_shmtx_wakeup(ngx_shmtx_t *mtx)
{
#if (NGX_HAVE_POSIX_SEM)
    ngx_atomic_uint_t  wait;

    if (!mtx->semaphore) {
        return;
    }
    // 这里将等待的进程数量减一。这是由于在阻塞式获取锁时，获取信号量时，并未将该值减一，这样进行减一
    for ( ;; ) {

        wait = *mtx->wait;

        if ((ngx_atomic_int_t) wait <= 0) {
            return;
        }

        if (ngx_atomic_cmp_set(mtx->wait, wait, wait - 1)) {
            break;
        }
    }

    ngx_log_debug1(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                   "shmtx wake %uA", wait);
    // 释放信号量来唤醒其余等待进程。
    if (sem_post(&mtx->sem) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                      "sem_post() failed while wake shmtx");
    }

#endif
}
```

## 强制释放锁

当某些进程意外退出时，如果其已经获取到了某个锁，这时需要有一个机制能够让其他进程（master）进程来强制释放其所持有的锁。

```c
ngx_uint_t
ngx_shmtx_force_unlock(ngx_shmtx_t *mtx, ngx_pid_t pid)
{
    ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                   "shmtx forced unlock");
    /* 如果释放锁返回为0，则表面出错进程确实拥有该锁，这时需要执行唤醒其余进程的操作，否则表示要释放的进程并不持有该锁，无需额外操作。*/
    if (ngx_atomic_cmp_set(mtx->lock, pid, 0)) {
        ngx_shmtx_wakeup(mtx);
        return 1;
    }

    return 0;
}
```

