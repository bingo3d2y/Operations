---
description: Nginx && SSL && OpenSSL
---

# https and SNI

**Problem**

接管的所有nginx服务器，关于https的配置都是下面这些，所以弄清这些参数的具体意义，从而判断又没有可以优化的空间。

```text
   ssl                     on;
   ssl_certificate         /usr/local/nginx/ssl/XXX.pem;
   ssl_certificate_key     /usr/local/nginx/ssl/XXX.key;
   ssl_session_timeout     5m;
   ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
   ssl_ciphers             ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
   ssl_prefer_server_ciphers       on;  
```

**宏观配置理解**

Nginx 配置 HTTPS 并不复杂，主要有两个步骤：签署第三方可信任的 SSL 证书和配置支持HTTPS

 Nginx 下配置 HTTPS 的关键要点：

* **获得 SSL 证书**
  * 自己签发证书

    通过 OpenSSL 命令获得 example.key 和 example.csr 文件

    > 怪不得安装nginx时，要安装openssl。 nginx的https需要用到openssl工具验证证书。

  * 购买权威证书

    提供 example.csr 文件给第三方可靠证书颁发机构，选择适合的安全级别证书并签署，获得 example.crt 文件。
* **通过 listen 命令 SSL 参数以及引用 example.key 和 example.crt 文件完成 HTTPS 基础配置**
* **HTTPS优化**
  * 减少 CPU 运算量：仅前端的Loadbalance Nginx配置https，后端Proxy-Serve nginx不用配置https
    * 使用 keepalive 长连接

      > 设置长连接,这里是http请求的长连接设置，不是tcp的
      >
      > ```text
      > http {
      >     keepalive_timeout  120s 120s;
      >     keepalive_requests 10000;
      > }
      > ```
      >
      > keepalive\_timeout timeout \[header\_timeout\];
      >
      > 1. 第一个参数：设置keep-alive客户端连接在服务器端保持开启的超时值（默认75s）；值为0会禁用keep-alive客户端连接；
      > 2. 第二个参数：可选、在响应的header域中设置一个值“Keep-Alive: timeout=time”；通常可以不用设置；
      >
      > keepalive\_requests指令用于设置一个keep-alive连接上可以服务的请求的最大数量，当最大请求数量达到时，连接被关闭。默认是100。这个参数的真实含义，是指一个keep alive建立之后，nginx就会为这个连接设置一个计数器，记录这个keep alive的长连接上已经接收并处理的客户端请求的数量。如果达到这个参数设置的最大值时，则nginx会强行关闭这个长连接，逼迫客户端不得不重新建立新的长连接。

    * 复用 SSL 会话参数

      > 这些会话会存储在一个 SSL 会话缓存里面，通过命令 [ssl\_session\_cache](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_cache) 配置，可以使缓存在机器间共享，然后利用客户端在『握手』阶段使用的 `seesion id` 去查询服务端的 session cathe\(如果服务端设置有的话\)，简化『握手』阶段。
      >
      > 1M 的会话缓存大概包含 4000 个会话，默认的缓存超时时间为 5 分钟，可以通过使用 [ssl\_session\_timeout](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_timeout) 命令设置缓存超时时间。下面是一个拥有 10M 共享会话缓存的多核系统优化配置例子：
      >
      > ```text
      > #配置共享会话缓存大小
      > ssl_session_cache   shared:SSL:10m;
      > #配置会话超时时间
      > ssl_session_timeout 10m;
      > ```
  * 使用 HSTS 策略强制浏览器使用 HTTPS 连接

    * 添加 Strict-Transport-Security 头部信息
    * 使用 HSTS 预加载列表（HSTS Preload List）

    > ```text
    > server {
    >     add_header Strict-Transport-Security "max-age=31536000; includeSubDomains;preload" always;
    > }
    > ```
    >
    > * max-age：设置单位时间内强制使用 HTTPS 连接
    > * includeSubDomains：可选，所有子域同时生效
    > * preload：可选，_非规范值_，用于定义使用『HSTS 预加载列表』
    > * always：可选，保证所有响应都发送此响应头，包括各种內置错误响应

    **当用户进行 HTTPS 连接的时候**，服务器会发送一个 **Strict-Transport-Security** 响应头：

    ![HSTS](file://D:/data_files/MarkDown/Images/HSTS.jpg?lastModify=1583480013)

    浏览器在获取该响应头后，在 `max-age` 的时间内，如果遇到 HTTP 连接，就会通过 307 跳转强制使用 HTTPS 进行连接，并忽略其它的跳转设置（如 301 重定向跳转）：

    ![307](file://D:/data_files/MarkDown/Images/307.jpg?lastModify=1583480013)

    Note: 307 跳转的前提：第一次需要手动访问https且在max-age内才会307

    由于 HSTS 需要用户经过一次安全的 HTTPS 连接后才会在 max-age 的时间內生效，因此HSTS 策略并不能完美防止 [HTTP 会话劫持（HTTP session hijacking）](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/06.4.md)。

    所以，有了Google HSTS 预加载列表（HSTS Preload List）

  * 加强 HTTPS 安全性
    * 使用迪菲-赫尔曼密钥交换（D-H，Diffie–Hellman key exchange）方案
    * 添加 X-Frame-Options 头部信息，减少点击劫持
    * 添加 X-Content-Type-Options 头部信息，禁止服务器自动解析资源类型
    * 添加 X-Xss-Protection 头部信息，防XSS攻击

      > ```text
      > #减少点击劫持
      > add_header X-Frame-Options DENY;
      > #禁止服务器自动解析资源类型
      > add_header X-Content-Type-Options nosniff;
      > #防XSS攻击
      > add_header X-Xss-Protection 1;
      > ​
      > ```

**SSL**

 首先简单的介绍一下 SSL 协议建立连接的过程。

![ssl](file://D:/data_files/MarkDown/Images/ssl.gif?lastModify=1583480013)

* 客户端发起请求，包含一个`hello`消息，**并附上客户端支持的密码算法和 SSL 协议的版本消息**以及用于生成密钥的随机数。
* 服务器收到消息后，服务器端选择加密压缩算法并生成服务器端的随机数，将该信息反馈给客户端；接着服务器端将自身的数字证书（在图 1 中使用了一个 X.509 数字证书）发送到客户端；完成上述动作后后服务器端会发送“`hello done`”消息给客户端。此外**如果**服务器需要对客户端进行身份认证，服务器端还会发送一个请求客户端证书的消息。（一般不要client's certificate ）

  > 如果服务器需要确认客户端的身份，就会再包含一项请求，要求客户端提供"客户端证书"。比如，金融机构往往只允许认证客户连入自己的网络，就会向正式客户提供USB密钥，里面就包含了一张客户端证书。
  >
  > 0.0 U盾

* 一旦客户端收到”`hello done`” , 就开始对服务器端的数字证书进行认证并检查服务器端选中的算法是可行的。**如果**服务器要求认证客户端身份，客户端还会发送自己的公钥证书。
* 如果对服务器的身份认证通过，客户端会发起密钥交换的请求。
* 服务器端和客户端根据先前**协商的算法和交换随机数生成对称密钥进行后续的通信**。

  > 0.0 把三个随机数，一下子加密，用于该会话。
  >
  > 三个伪随机数已经很接近一个正在的随机数了。
  >
  > [http://www.ruanyifeng.com/blog/2014/02/ssl\_tls.html](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)

```text
​
 # 使用openssl命令查看ssl协议握手过程 
 $ openssl s_client -showcerts -state  -connect www.XXX.com:443
  CONNECTED(00000003)
  SSL_connect:before/connect initialization
  SSL_connect:SSLv2/v3 write client hello A  ## 发送hello 
  SSL_connect:SSLv3 read server hello A
  depth=3 C = US, O = Equifax, OU = Equifax Secure Certificate Authority
  verify return:1
  depth=2 C = US, O = GeoTrust Inc., CN = GeoTrust Global CA
  verify return:1
  depth=1 C = US, O = GeoTrust Inc., CN = RapidSSL SHA256 CA
  verify return:1
  depth=0 CN = *.fapiao56.com
  verify return:1
  SSL_connect:SSLv3 read server certificate A
  SSL_connect:SSLv3 read server key exchange A  ## exchange 交换
  SSL_connect:SSLv3 read server done A
  SSL_connect:SSLv3 write client key exchange A
  SSL_connect:SSLv3 write change cipher spec A
  SSL_connect:SSLv3 write finished A
  SSL_connect:SSLv3 flush data
  SSL_connect:SSLv3 read server session ticket A
  SSL_connect:SSLv3 read finished A
  ---
  Certificate chain  //显示证书链 -showcerts
   0 s:/CN=*.fapiao56.com
     i:/C=US/O=GeoTrust Inc./CN=RapidSSL SHA256 CA
  -----BEGIN CERTIFICATE-----
  # web 域名证书
  -----END CERTIFICATE-----
   1 s:/C=US/O=GeoTrust Inc./CN=RapidSSL SHA256 CA
     i:/C=US/O=GeoTrust Inc./CN=GeoTrust Global CA
  -----BEGIN CERTIFICATE-----
  # 中间证书
  -----END CERTIFICATE-----
   2 s:/C=US/O=GeoTrust Inc./CN=GeoTrust Global CA
     i:/C=US/O=Equifax/OU=Equifax Secure Certificate Authority
  -----BEGIN CERTIFICATE-----
  # CA根证书
  -----END CERTIFICATE-----
  ---
  Server certificate
  subject=/CN=*.fapiao56.com
  issuer=/C=US/O=GeoTrust Inc./CN=RapidSSL SHA256 CA
  ---
  No client certificate CA names sent
  Peer signing digest: SHA512
  Server Temp Key: ECDH, P-256, 256 bits
  ---
  SSL handshake has read 4200 bytes and written 415 bytes
  ---
  New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES128-GCM-SHA256  ## cipher 密码
  Server public key is 2048 bit
  Secure Renegotiation IS supported
  Compression: NONE
  Expansion: NONE
  No ALPN negotiated
  SSL-Session:
      Protocol  : TLSv1.2
      Cipher    : ECDHE-RSA-AES128-GCM-SHA256
      Session-ID: 99B9EBFD3835A73F544B13B9343D04E0DBDD14A64D05FAE2DF0892E3F06907EE
      Session-ID-ctx: 
      Master-Key: 659DA0DF425CC41E4E31E2003C2BA17411DAB1FD19399BE63F99D963A109367AC80622C90F4C2A38571C887C65DA4A62
      Key-Arg   : None
      Krb5 Principal: None
      PSK identity: None
      PSK identity hint: None
      TLS session ticket lifetime hint: 300 (seconds)
      TLS session ticket:
      0000 - e5 f1 58 e8 52 f3 73 0e-f8 94 3b 39 2b 0c 1d bd   ..X.R.s...;9+...
      0010 - 21 95 af c8 e9 57 ae 1b-2d ff a2 54 91 52 17 0d   !....W..-..T.R..
      0020 - a8 a6 41 08 52 84 1e 2b-56 69 d6 db 65 65 64 7d   ..A.R..+Vi..eed}
      0030 - 35 a6 e5 39 9b a6 6e 5d-fe 80 5f 87 2b 44 6b bd   5..9..n].._.+Dk.
      0040 - 2b c9 4f b8 c3 29 c8 51-1c 4e 3b b8 23 2a aa 82   +.O..).Q.N;.#*..
      0050 - cf a8 35 5d 50 de f2 26-43 c6 0c 94 c5 a5 2a 0b   ..5]P..&C.....*.
      0060 - bc d9 a6 4a 40 2d 72 7b-86 bf 68 eb dc 3c 27 76   ...J@-r{..h..<'v
      0070 - f0 7c 79 ce 8c c8 53 fc-60 e5 2d 2a 32 ae ea 62   .|y...S.`.-*2..b
      0080 - 00 ae ba bd 05 e7 bb 93-1f 21 47 b9 7c 83 76 26   .........!G.|.v&
      0090 - f2 35 8b 07 d4 d8 56 c0-80 5b 93 57 cb 05 60 80   .5....V..[.W..`.
      00a0 - 82 83 13 71 d8 1c a2 fd-2f 33 7f 2f 6a b9 b1 8e   ...q..../3./j...
  
      Start Time: 1514259754
      Timeout   : 300 (sec)
      Verify return code: 0 (ok)
  ---
  SSL3 alert read:warning:close notify
  closed
  SSL3 alert write:warning:close notify
  
  PS: -state：打印出 SSL 会话的状态。检查是否在握手协议的证书认证时失败
  $ cat  /usr/local/nginx/ssl/XXXcom.pem   //Nginx服务器上面的证书文件
  -----BEGIN CERTIFICATE-----
  web域名证书
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  中级证书
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  CA根证书
  -----END CERTIFICATE-----
  
​
```

**HTTPS**

**SNI**

**基于服务器名称（name-based）的 HTTPS 服务器**

**负载平衡器使用服务器名称指示 \(SNI\)，根据 TLS 握手中的域名确定要向客户端提供哪个证书。如果客户端不使用 SNI，或者客户端使用的域名与其中一个证书中的公用名 \(CN\) 不匹配，则负载平衡器将使用 Ingress 中列出的第一个证书。下图展示了根据请求中使用的域名将流量发送到不同后端的负载平衡器。**

* 为每个 HTTPS 服务器分配独立的 IP 地址
* 泛域证书
* 域名标识（SNI）

默认情况下，配置多个https虚拟主机，不论浏览器请求哪个主机，都只会收到默认主机的证书\(即第一个https主机的证书\)。这是由SSL协议本身的行为引起的—--—先建立SSL连接，再发送HTTP请求，所以nginx建立SSL连接时不知道所请求主机的名字，因此，它只会返回默认主机的证书。

> 这里其实就是Nginx default server的影响了，如果匹配不到server\_name就去default server block.
>
> 具体怎么设置default server 详见 Nginx Virtual host priority

nginx支持TLS协议的SNI扩展（Server Name Indication，简单地说这个扩展使得在同一个IP上可以以不同的证书serv不同的域名）。 专业的说SNI（　在一个IP上运行多个HTTPS主机的更通用的方案是TLS主机名指示扩展（SNI，RFC6066），它允许浏览器和服务器进行SSL握手时，将请求的主机名传递给服务器，因此服务器可以知道使用哪一个证书来服务这个连接。但SNI只得到有限的浏览器的支持。下面列举支持SNI的浏览器最低版本和平台信息：）不过，SNI扩展还必须有客户端的支持，另外本地的OpenSSL必须支持它。如果启用了SSL支持，nginx便会自动识别OpenSSL并启用SNI。 是否启用SNI支持，是在编译时由当时的 ssl.h 决定的。（SSL\_CTRL\_SET\_TLSEXT\_HOSTNAME），如果编译时使用的OpenSSL库支持SNI，

则目标系统的OpenSSL库只要支持它就可以正常使用SNI了 nginx在默认情况下是TLS SNI support disabled。

```text
启用方法：
需要重新编译nginx并启用TLS。步骤如下：
# wget http://www.openssl.org/source/openssl-1.0.1e.tar.gz
# tar zxvf openssl-1.0.1e.tar.gz 
# ./configure --prefix=/usr/local/nginx --with-http_ssl_module \
--with-openssl=/tmp/openssl-1.0.1e \
--with-openssl-opt="enable-tlsext" 
# make （编译之后/tmp/nginx-1.10.1/objs下回生成目标文件nginx）
# make install
PS：完成编译安装后删掉这个/tmp/openssl-1.0.1e也不影响。
​
查看是否启用：
# /usr/local/nginx/sbin/nginx -V
TLS SNI support enabled
​
这样就可以在 同一个IP上配置多个HTTPS虚拟主机了
```

**Certificate**

这里介绍如何将从厂家拿到的证书转换成Nginx服务器可以使用的格式。

从厂家申请过来的证书包含3个文件或两个，需要自己合并证书给Nginx服务器使用。

一份完整的 SSL 证书由以下几个部分构成：

1. CA 根证书 \(root CA\)
2. 中级证书 \(Intermediate Certificate\)
3. 域名证书：由证书厂商颁发
4. 证书私钥：只有你自己知道

有些厂商会将CA证书链合并和域名证书一起

* CA 证书串 - www\_bagesoft\_cn.ca-bundle
* 您的域名证书 - www\_bagesoft\_cn.crt

需要依照按此顺序合： 域名证书 -&gt; 中级证书 -&gt; 根证书 的顺序串联为证书链，才能被绝大多数浏览器信任。

> 通过`openssl s_client -showcerts -state -connect Domian_name:443`，也可以看到证书的顺序、

可以手动通过记事本合并，也可以使用linux命令 cat串联证书：

```text
$ cat example.crt COMODORSADomainValidationSecureServerCA.crt COMODORSAAddTrustCA.crt AddTrustExternalCARoot.crt > example_com.bundle.crt
```

如果您收到的是 www\_bagesoft\_cn.ca-bundle 的形式，请直接手动将CA证书链和 www\_bagesoft\_cn.crt 串联。或linux中命令为：

```text
$ cat www_bagesoft_cn.crt www_bagesoft_cn.ca-bundle > www_bagesoft_cn.bundle.crt
```

**最终配置**

配置https server有三步，必须：

1. listen dierctive添加ssl参数

   `ssl on|off` 这个ssl指令已经不推荐使用了，所以上面的`ssl on`。太low，推荐：`listen port ssl;`

   > 之前nginx版本低，所以增加了`ssl` 指令，现在高版本的nginx都通过`listen` 指令的ssl 参数来替代之前的`ssl` 指令。

   即直接在`listen`中将一个端口声明为ssl 、

2. 指定证书： `ssl_certificate`

   The server certificate is a public entity. It is sent to every client that connects to the server.

   服务器证书是个公共的实体，它会发送给每个连接server的client。

3. 指定私钥: `ssl_certificate_key`

   The private key is a secure entity and should be stored in a file with restricted access, however, it must be readable by nginx’s master process.

   Although the certificate and the key are stored in one file, only the certificate is sent to a client.

   > 私钥是个安全实体需要被存储在一个限制随便访问的文件里，但是它必须能被nginx's master process访问。
   >
   > 有点意思啊，怪不得Nginx's master process 都是root。

4.  `ssl_protocols` 和 `ssl_ciphers` \(没必要则不需要明显设定\)

   这两个命令的默认值已经好几次发生了改变，因此不建议显性定义，除非有需要额外定义的值，如定义 D-H 算法：

   The directives ssl\_protocols can be used to limit connections to include only the strong versions and ciphers of SSL/TLS. By default nginx uses “`ssl_protocols TLSv1 TLSv1.1 TLSv1.2`” and “`ssl_ciphers HIGH:!aNULL:!MD5`”, so configuring them explicitly is generally not needed. Note that default values of these directives were [changed](http://nginx.org/en/docs/http/configuring_https_servers.html#compatibility) several times.

   ```text
   #使用DH文件
   ssl_dhparam /etc/ssl/certs/dhparam.pem;
   ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
   #定义算法
   ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
   #...
   ```

   所以，这两个指令不用写，使用默认值即可。

综上，最终的https配置可以写成下面这样的即可。

```text
http {
    ...
    server {
        listen              443 ssl;
        server_name         www.example.com;
        
        keepalive_timeout   70;
		
		ssl_session_cache   shared:SSL:10m;
    	ssl_session_timeout 10m;
    
        ssl_certificate     www.example.com.crt;
        ssl_certificate_key www.example.com.key;
        ...
        #减少点击劫持
        add_header X-Frame-Options DENY;
        #禁止服务器自动解析资源类型
        add_header X-Content-Type-Options nosniff;
        #防XSS攻擊
        add_header X-Xss-Protection 1;
 }
```

参考：

[https://aotu.io/notes/2016/08/16/nginx-https/index.html](https://aotu.io/notes/2016/08/16/nginx-https/index.html)

