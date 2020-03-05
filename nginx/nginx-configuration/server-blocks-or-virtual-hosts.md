# Nginx Virtual Servers priority

**Problem:**

访问nginx 指定server\_name却被解析到其他server\_name下。

eg：一个配置文件如下：

```text
upstream test {
    server 192.168.2.91:80
}
​
server {
    listen 192.168.2.91:80
    server www.lol.com
    ...
}
server {
    listen 80
    server www.lpl.com
}
```

**当你访问** [**www.lpl.com**](www.lpl.com) **是它匹配到的是**[**www.lol.com**](www.lol.com)**而不是lpl**

**Knowledge:**

**Nginx：listen指令**

不同listen的优先级：Port最优先，然后明确IP优先级高于default

The `listen` directive can be set to:

* An IP address/port combo.

  > `listen 192.168.97.54:80;`

* A lone IP address which will then listen on the default port 80.

  > `listen 192.168.97.54;`
  >
  > 只有一个ip，默认listen 80端口

* A lone port which will listen to every interface on that port.

  > `listen 80;`
  >
  > 只有一个port，listen to every interface on that port.
  >
  > every interface（即改机器上的所有ip）
  >
  > listen to every interface on that port相当于 `0.0.0.0:80`

* The path to a Unix socket.

  > `listen unix:/var/run/nginx.sock;`
  >
  > The option will generally only have implications when passing requests between different servers.
  >
  > 这个配置常用于同一服务器上不同服务之间的交流。

**server\_name directive**

Each IP address/port combo has a default server block that will be used when a course of action can not be determined with the above methods. For an IP address/port combo, this will either be the first block in the configuration or the block that contains the default\_server option as part of the listen directive \(which would override the first-found algorithm\). There can be only one default\_server declaration per each IP address/port combination.

每个IP地址/端口组合都有一个默认的server block，当通过server\_name无法匹配到server时，将使用这个block。 default server block，是第一个发现的server\_name的server或者是通过listen指令的default\_server参数什么的server。`default_server`每个IP地址/端口组合只能有一个声明。

> 如果有`default` 参数声明，则该server block为 default server（和默认路由一个功能）。
>
> 如果没有`default`来声明，则第一个出现的server block为default server.

nginx仅仅检查请求的“Host”头以决定该请求应由哪个虚拟主机来处理。如果Host头没有匹配任意一个虚拟主机，或者请求中根本没有包含Host头，那nginx会将请求分发到定义在此端口上的默认虚拟主机。在以上配置中，第一个被列出的虚拟主机即nginx的默认虚拟主机——这是nginx的默认行为。而且，可以显式地设置某个主机为默认虚拟主机，即在"`listen`"指令中置"`default_server`"参数

```text
# head -5  default.conf  znbsy.conf 
==> default.conf <==
#
# The default server
#
server {
    listen       80 default_server;
​
==> znbsy.conf <==
server {
        listen 80;
        server_name activity.17win.com;
​
        location /znbsy {
    
由于指定了default_server，这导致ip:80访问这个Nginx web server时，日志都记录到default.conf这个配置文件指定的access_log了
# 100次的访问全部去了 default.conf
# 因为 IP作为domain Name没有对应的server_name 所以他就去访问 default server了
$ for i in {1..100}; do curl -I  http://192.168.9.202/learning/index.html;done
```

Nginx使用的正则表达式与Perl编程语言（PCRE）使用的正则表达式兼容。

Nginx 的 server\_name 匹配时也是有优先级的。

nginx将按照1,2,3,4的顺序对server name进行匹配，只有有一项匹配以后就会停止搜索，所以我们在使用这个指令的时候一定要分清楚它的匹配顺序。

```text
# Nginx中的server_name指令主要用于配置基于名称的虚拟主机，server_name指令在接到请求后的匹配顺序为：
1、准确的server_name匹配，例如：
​
server {
     listen       80;
     server_name  domain.com  www.domain.com;
     ...
}
 
​
2、以*通配符开始的字符串：
​
server {
     listen       80;
     server_name  *.domain.com;
     ...
}
3、以*通配符结束的字符串：
​
server {
     listen       80;
     server_name  www.*;
     ...
}
4、匹配正则表达式：
​
server {
     listen       80;
     server_name  ~^(?.+)\.domain\.com$;
     ...
}
​
```

**Nginx：如何分配请求**

按照如下顺序进行路由和分发：

1.listen directive

2.server\_name directive

When trying to determine which server block to send a request to, Nginx will first try to decide based on the specificity of the listen directive using the following rules:

> Nginx 决定将请求发送到哪个后端时，先根据`listen directive` 来判断。判断规则如下：

* Nginx translates all "incomplete" listen directives by substituting missing values with their default values so that each block can be evaluated by its IP address and port. Some examples of these translations are:

  > listen 会通过各种缺省值，将listen 监听的 都变成 ip:port 形式。便于比较优先级。例子如下：
  >
  > default ip: `0.0.0.0` default port: `80`
  >
  > 在服务器中，0.0.0.0指的是本机上的所有IPV4地址，如果一个主机有两个IP地址，192.168.1.1 和 10.1.2.1，并且该主机上的一个服务监听的地址是0.0.0.0,那么通过两个ip地址都能够访问该服务。

  * A block with no listen directive uses the value 0.0.0.0:80.

    > 没有listen指令是默认是0.0.0.0:80

  * A block set to an IP address 111.111.111.111 with no port becomes 111.111.111.111:80

    > 只设置IP没port，则为ip:80

  * A block set to port 8888 with no IP address becomes 0.0.0.0:8888

    > 只设置port没ip，则为0.0.0.0:80

* Nginx then attempts to collect a list of the server blocks that match the request most specifically based on the IP address and port. This means that any block that is functionally using 0.0.0.0 as its IP address \(to match any interface\), will not be selected if there are matching blocks that list a specific IP address. In any case, the port must be matched exactly.
* If there is only one most specific match, that server block will be used to serve the request. If there are multiple server blocks with the same level of specificity matching, Nginx then begins to evaluate the server\_name directive of each server block.

  > 如果在listen directive 没有精确匹配到server。则再根据server\_name directive 继续匹配server。

重点来了， Nginx到底如何匹配请求。

首先，根据端口筛选port。ip可以"不对",但是port必须匹配正确。

其次，精确匹配ip

最后，使用0.0.0.0模糊匹配（一个网卡可以有多个vip，对应不同的公网ip）

这意味着，优先级：192.168.97.54:80 &gt; 80\(即0.0.0.0:80\)

在`listen`判断出优先级之后（即在ip+port层面只判断出一个virtual server），就不再管server\_name了、

只有当`ip+port`判断完之后，还有多个`server_names`时，才会判断`incomplete`具体属于哪个`server_name`

举个栗子：

```text
###  配置 两个 virtual server
##  一个 listen IP:Port
##  另一个 listen： lone_port
## 已知存在文件：/servyouapp/yqzs2/test/a.html
 server {
   listen 192.168.97.54:80;
   server_name a.b.c;
​
        location / {
            ##root directory不同
            root /servyouapp/yqzs2/test;  
            index  index.html index.htm;
          }
 }
​
 server {
        
        listen 80;
        server_name 17zs.servyou.com.cn;
        access_log    /backup/applogs/access.log log_json;
        error_log    /backup/applogs/error.log error;
    
        location / {
           ##root directory不同
            root /servyouapp/yqzs2;  
            index  index.html index.htm;
    
        }
    
}
​
# 如果没有说，则该请求，应该报404、因为这个域名的root在/servyouapp/yqzs2
# 而文件路径是/servyouapp/yqzs2/test/a.html
$ curl  17zs.servyou.com.cn/a.html  
lol
# 由此可见，访问17zs.servyou.com.cn时实际被location到a.b.c了。
$ curl  -I 17zs.servyou.com.cn/test/a.html  
HTTP/1.1 404 Not Found
Server: nginx
Date: Wed, 13 Dec 2017 06:25:21 GMT
Content-Type: text/html
Content-Length: 162
Connection: keep-alive
​
 
# 修改配置将listen都换成 ip: port
# nginx  -s reload
# killall nginx  //需要kill掉，reload 配置没生效
# nginx   //启动
修改后配置如下：
 server {
   listen 19.1.97.54:80;
   server_name a.b.c;
    
        location / {
            root /servyouapp/yqzs2/test;
            index  index.html index.htm;
          }
​
 }
​
 server {
        listen 19.1.97.54:80;
        server_name 17zs.servyou.com.cn;
        access_log    /backup/applogs/access.log log_json;
        error_log    /backup/applogs/error.log error;
    
        location / {
            root /servyouapp/yqzs2;
            index  index.html index.htm;
    
        }
    
}
    
$ curl  -I 17zs.servyou.com.cn/a.html
HTTP/1.1 404 Not Found
Server: nginx
Date: Wed, 13 Dec 2017 06:22:02 GMT
Content-Type: text/html
Content-Length: 162
Connection: keep-alive
​
# 17zs.servyou.com.cn的请求终于被分配到server_name 17zs.servyou.com.cn了
$ curl  17zs.servyou.com.cn/test/a.html 
lol
```

0.0.0.0 ip地址

* IPV4中，0.0.0.0地址被用于表示一个无效的，未知的或者不可用的目标。
* 在服务器中，0.0.0.0指的是本机上的所有IPV4地址，如果一个主机有两个IP地址，192.168.1.1 和 10.1.2.1，并且该主机上的一个服务监听的地址是0.0.0.0,那么通过两个ip地址都能够访问该服务。
* 在路由中，0.0.0.0表示的是默认路由，即当路由表中没有找到完全匹配的路由的时候所对应的路由。



