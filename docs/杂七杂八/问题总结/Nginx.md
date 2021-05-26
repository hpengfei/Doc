> Nginx优缺点 常用功能

Nginx是异步框架的网页服务器，也可以用作反向代理、负载平衡器和HTTP缓存。

优点：使用的内存比 Apache 少得多，每秒可以处理大约四倍于 Apache 的请求。在高并发下 Nginx 能保持低资源低消耗高性能。高度模块化的设计，模块编写简单，以及配置文件简洁。

> nginx 有哪些负载均衡算法

轮询、加权轮询、ip_hash

>nginx 反向代理保持长连接如何配置

**http段中的配置**： keepalive_timeout   keepalive_requests  

**upstream中的配置**： keepalive 300; 

**location中的配置**： proxy_http_version 1.1;  proxy_set_header Connection "";  

https://www.cnblogs.com/kevingrace/p/9364404.html

> 如何获取用户的IP

```
proxy_set_header  X-Real-IP  $remote_addr;
proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
```

**remote_addr** 

代表客户端的IP，但它的值不是由客户端提供的，而是服务端根据客户端的ip指定的，当你的浏览器访问某个网站时，假设中间没有任何代理，那么网站的web服务器（Nginx，Apache等）就会把remote_addr设为你的机器IP，如果你用了某个代理，那么你的浏览器会先访问这个代理，然后再由这个代理转发到网站，这样web服务器就会把remote_addr设为这台代理机器的IP,除非代理将你的IP附在请求header中一起转交给web服务器。 

**proxy_add_x_forwarded_for**

X-Forwarded-For 是一个 HTTP 扩展头部。HTTP/1.1（RFC 2616）协议并没有对它的定义，它最开始是由 Squid 这个缓存代理软件引入，用来表示 HTTP 请求端真实 IP。如今它已经成为事实上的标准，被各大 HTTP 代理、负载均衡等转发服务广泛使用，并被写入 [RFC 7239](http://tools.ietf.org/html/rfc7239)（Forwarded HTTP Extension）标准之中。

XFF的格式为：

```
X-Forwarded-For: client, proxy1, proxy2
```

XFF 的内容由「英文逗号 + 空格」隔开的多个部分组成，最开始的是离服务端最远的设备 IP，然后是每一级代理设备的 IP。（注意：如果未经严格处理，可以被伪造）如果一个 HTTP 请求到达服务器之前，经过了三个代理 Proxy1、Proxy2、Proxy3，IP 分别为 IP1、IP2、IP3，用户真实 IP 为 IP0，那么按照 XFF 标准，服务端最终会收到以下信息：

```
X-Forwarded-For: IP0, IP1, IP2
```

 Proxy3 直连服务器，它会给 XFF 追加 IP2，表示它是在帮 Proxy2 转发请求。列表中并没有 IP3，IP3 可以在服务端通过 Remote Address 字段获得。我们知道 HTTP 连接基于 TCP 连接，HTTP 协议中没有 IP 的概念，Remote Address 来自 TCP 连接，表示与服务端建立 TCP 连接的设备 IP，在这个例子里就是 IP3。Remote Address 无法伪造，因为建立 TCP 连接需要三次握手，如果伪造了源 IP，无法建立 TCP 连接，更不会有后面的 HTTP 请求。但是在正常情况下，web服务器获取Remote Address只会获取到上一级的IP，本例里则是proxy3 的 IP3，这里先埋个伏笔。 

> Nginx 如何解决跨域问题

原因：浏览器限制、跨域、XHR（XMLHttpRequest）请求

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

Access-Control-Allow-Origin：服务器默认是不被允许跨域的。给Nginx服务器配置`Access-Control-Allow-Origin *`后，表示服务器可以接受所有的请求源（Origin）,即接受所有跨域的请求。

> Nginx 优化

**系统方面**

文件句柄数的修改

```
echo "session    required     pam_limits.so">>/etc/pam.d/login
echo "*                hard    nofile          65536">>/etc/security/limits.conf
echo "*                soft    nofile          65536">>/etc/security/limits.conf
echo "*                hard    nproc          4096">>/etc/security/limits.conf
echo "*                soft    nproc          4096">>/etc/security/limits.conf
```

内核参数优化

**使用优化**

自定义返回给客户端的4xx 5xx 错误页面

**安全方面**

server_tokens ： 关闭显示nginx 的版本信息。

防盗链：

```
location ~* .*\.(gif|jpg|ico|png|css|svg|js)$ {
	root /usr/local/nginx/static;
	valid_referers none blocked  *.gupao.com ; // 有效的来源
	if ($invalid_referer) { // 无效的来源的话就给404
		#rewrite ^/ http://www.youdomain.com/404.jpg;
		return 403;
		break;
	 }
	 access_log off;
}
```

none：  “Referer” 来源头部为空的情况 

blocked： “Referer”来源头部不为空，但是里面的值被代理或者防火墙删除了，这些值都不以http://或者https://开头.

add_header X-Frame-Options SAMEORIGIN; 用来指定此网页是否允许被 iframe 嵌套。

当设置为 DENY 时，站点禁止任何页面被嵌入。

当设置为 SAMEORIGIN 时，只允许加载同源的 fram/iframe/object。

当设置为 ALLOW-FROM 时，只允许加载指定的源。

**性能方面**

worker_processes : Nginx worker 服务的进程数。  一般和CPU核心数相同

worker_cpu_affinity：CPU亲缘性绑定，1.10+ 设置成 auto。

worker_connections： 设置一个进程理论允许的最大连接数，一般设置 65535 

worker_rlimit_nofile：毎个进程的最大文件打开数。如果不设的话上限就是系统的ulimit –n的数字，一般为65535。

use：设置为 epoll，epoll是Nginx支持的高性能事件驱动库之一。

sendfile：普通应用应该设为on，下载等IO重负荷的应用应该设为off，因为大文件不适合放到buffer中。

tcp_nopush：sendfile为on时这里也应该设为on，数据包会累积一下再一起传输，可以提高一些传输效率。

client_max_body_size：客户端上传的body的最大值。超过最大值就会发生413(Request Entity Too Large)错误。

gzip：文本内容开启gzip压缩，` gzip_disable       "msie6"; `

**开启缓存**

1.open_file_cache：配置nginx存储下面内容的缓存 文件描述符、文件大小和最近一次的修改时间；打开的目录结构；没有找到的或者没有权限操作的文件信息。
2.open_file_cache_errors：是否缓存找不到路径的文件，或者没有权限访问的文件相关信息。
3.open_file_cache_min_uses：设置由open_file_cache指令的inactive参数配置的时间段内文件访问的最小数量。
4.open_file_cache_valid：每隔多久检查一次缓存项的有效性。

expires：根据静态资源不同调整在浏览器中的缓存。图片一般30天，css js 这种一般7天。

> Nginx 信号

TERM，INT： 快速关闭 　　　　
QUIT ：从容关闭（优雅的关闭进程,即等请求结束后再关闭）
HUP ：平滑重启，重新加载配置文件 （平滑重启，修改配置文件之后不用重启服务器。直接kill -PUT 进程号即可）
USR1 ：重新读取日志文件，在切割日志时用途较大（停止写入老日志文件，打开新日志文件，之所以这样是因为老日志文件就算修改的文件名，由于inode的原因，nginx还会一直往老的日志文件写入数据）
USR2 ：平滑升级可执行程序 ，nginx升级时候用 　　　　
WINCH ：从容关闭工作进程 

> rewrite 规则

```
location /ecshop {
    # 商品详情页 http://192.168.0.200/ecshop/goods.php?id=9 -> http://192.168.0.200/ecshop/goods-9.html
    rewrite goods-(\d+)-.*\.html /ecshop/goods.php?id=$1;

    # 文章详情页 http://192.168.0.200/ecshop/article.php?id=12 -> http://192.168.0.200/ecshop/article-12.html
    rewrite article-(\d+)-.*\.html /ecshop/article.php?id=$1;

    # 复杂栏目检索页 http://192.168.0.200/ecshop/category.php?category=3&display=list&brand=0&price_min=0&price_max=0&filter_attr=0&page=1&sort=goods_id&order=ASC#goods_list -> http://192.168.0.200/ecshop/category-3-b0-min0-max0-attr0-1-goods_id-ASC.html
    rewrite category-(\d+)-b(\d+)-min(\d+)-max(\d+)-attr([\d\.]+)-(\d+)-(\w+)-(\w+)\.html /ecshop/category.php?id=$1&brand=$2&price_min=$3&price_max=$4&filter_attr=$5&page=$6&sort=$7&order=$8#goods_list;

    # 栏目检索页 http://192.168.0.200/ecshop/category-3-b1-min200-max1700-attr167.229.202.199.html -> http://192.168.0.200/ecshop/category.php?id=3&brand=1&price_min=200&price_max=1700&filter_attr=167.229.202.199
    rewrite category-(\d+)-b(\d+)-min(\d+)-max(\d+)-attr([\d\.]+).*\.html /ecshop/category.php?id=$1&brand=$2&price_min=$3&price_max=$4&filter_attr=$5;

    # 栏目页面 http://192.168.0.200/ecshop/category-2-b0.html -> http://192.168.0.200/ecshop/category.php?id=3&brand=1
    rewrite category-(\d+)-b(\d+)-.*\.html /ecshop/category.php?id=$1&brand=$2;
}
```

多目录转成参数：
`abc.domian.com/sort/2 => abc.domian.com/index.php?act=sort&name=abc&id=2`

```
if ($host ~* (.*)\.domain\.com) {
set $sub_name $1;   
rewrite ^/sort\/(\d+)\/?$ /index.php?act=sort&cid=$sub_name&id=$1 last;
}
```

目录对换
`/123456/xxxx -> /xxxx?id=123456`

```
rewrite ^/(\d+)/(.+)/ /$2?id=$1 last;
```

例如下面设定nginx在用户使用ie的使用重定向到/nginx-ie目录下：

```
if ($http_user_agent ~ MSIE) {
rewrite ^(.*)$ /nginx-ie/$1 break;
}
```

目录自动加“/”

```
if (-d $request_filename){
rewrite ^/(.*)([^/])$ http://$host/$1$2/ permanent;
}
```

将多级目录下的文件转成一个文件，增强seo效果
`/job-123-456-789.html 指向/job/123/456/789.html`

```
rewrite ^/job-([0-9]+)-([0-9]+)-([0-9]+)\.html$ /job/$1/$2/jobshow_$3.html last;
```

> httpd 三种工作模型

**prefork---多进程I/O模型，每个进程响应一个请求**

预派生模式，有一个主控制进程，然后生成多个子进程,每个子进程有一个独立的线程响应用户请求，相对比较占用内存，但是比较稳定，可以设置最大和最小进程数，是最古老的一种模式，也是最稳定的模式，适用于访问量不是很大的场景 

**worker---复用的多进程I/O模型，多进程多线程**

 一个主进程：生成m个子进程，每个子进程负责生个n个线程，每个线程响应一个请求，并发响应请求：m*n

**event---事件驱动模型，增加了一个监听线程**