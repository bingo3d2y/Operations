# Nginx Location

1. Nginx--location优先级

   错误总在志得意满的时候。（碰到经验解决不了的问题还是看日志最直接）

   **Problems：Nginx 增加很简单的location后，访问验证发现不生效**

   > **Nginx 配置如下：**
   >
   > ```text
   > ##nginx错误配置如下：两个类似的location，反向代理到不同的backend
   > ​
   > location ~ /hswy-basic-web/basic/ {
   >     proxy_pass       http://basicbackend;（97.18 97.17）
   >     。。。 
   > }
   > ​
   > location ~ /hswy-basic-web/basic/app/{ 
   >     proxy_pass       http://app-centerbackend; (97.40 97.56 97.72 97.73)
   >     。。。
   > }
   > ```

   **Actions：瞎比比想了半天后，就是想不通，最后才想起来去看日志（早该如此）**

   > ```text
   > # 呐，日志如下：  
   > ​
   > {"logtime": "29/Aug/2017:21:59:42 +0800", "client_ip": "120.239.88.139", "remote_user": "-", "resp_len": "957", "waster_time": "0.003", "status": "404", "request_path": "GET /hswy-basic-web/basic/app/query?&_1504015359770&accountId=&appType=&groupId=&locationCode=440703&terminalCode=092801GD010000&userInfo=%7B%22locationCode%22:%22440703%22,%22nsrsbh%22:%2291440703688620696H%22%7D HTTP/1.1", "request_method": "GET", "request_url": "https://17hs.servyou.com.cn/17hs/gs/gdMain/view/index.html", "body_bytes_sent":"957", "http_forwarded": "-", "user_agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/31.0.1650.57 Safari/537.36" ,"upstream":  "192.168.97.18:8080, 192.168.97.17:8080","upstream_status": "404, 404"}
   > ​
   > ```
   >
   > **由上可知：get请求"GET /hswy-basic-web/basic/app/query?..."时，`upstream_status` :404**
   >
   > `upstream_status` 404则说明是后端的问题了。
   >
   > **且upstream到了97.18上面** ，按照我们的一厢情愿应该到97.40 97.56 97.72 97.73上面的。
   >
   > 哈哈哈，找到问题了。

   **Results：调整配置文件位置，请求访问成功。**

   > **修改后如下**:
   >
   > ```text
   > ​
   > location ~ /hswy-basic-web/basic/app/{ 
   >     proxy_pass       http://app-centerbackend; (97.40 97.56 97.72 97.73)
   >     。。。
   > }
   > ​
   > location ~ /hswy-basic-web/basic/ {
   >     proxy_pass       http://basicbackend;（97.18 97.17）
   >     。。。 
   > }
   > ```
   >
   > end

   Knowledge：Nginx的localtion 匹配规则不是我臆测的那样会“精确匹配”

   > 我的臆测是：当请求的uri是`/hswy-basic-web/basic/app/query?...` 时`/hswy-basic-web/basic/app/`比`/hswy-basic-web/basic/` 更精确所以优先级也搞，从而任务顺序无所谓....
   >
   > 其实，臆测的全错了，哈哈哈、
   >
   > **location** \[ `=` \| `~` \| `~*` \| `^~` \] `*uri*` { ... }

   [http://nginx.org/en/docs/http/ngx\_http\_core\_module.html\#location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) （官方文档）

   **Understanding NGINX location rules**

   NGINX is great. Fast, efficient, etc. But the “location” rules are a bit cryptic and not very well explained in the manuals.

   There are **4 types of location** rule, and are applied with the following priorities:

   **1: Exact matches** 精确匹配，优先级最高

   There can be only one exact match – the clue is in the name!

   > 独一无二的，按名字匹配，优先级最高

   `location = /foo/bar {# exact match}`

   **2: High priority prefix** 前缀高优先级

   There can be more than one match, the longest one takes priority

   > 匹配以`/foo` 开头的uri，能匹配多个uri，匹配的uri越长优先级越高
   >
   > **soga，我把`High priority prefix` 的特性强行记错到`Regex`上面了**

   `location ^~ /foo {# request beginning with /foo}`

   **3: Regex**

   There can be more than one match, the first one found takes priority. There are two variants

   > 哈哈哈，`Regex` 的特性就是先匹配到哪个，哪个的优先级就高哈哈哈
   >
   > ```text
   > location ~ /hswy-basic-web/basic/ {
   >     proxy_pass       http://basicbackend;（97.18 97.17）
   >     。。。 
   > }
   > ​
   > location ~ /hswy-basic-web/basic/app/{ 
   >     proxy_pass       http://app-centerbackend; (97.40 97.56 97.72 97.73)
   >     。。。
   > }
   > ​
   > 哈哈哈，怪不得这样导致访问 /hswy-basic-web/basic/app/时会被upstream到97.18 97.17上面。
   > ```

   location ~ .foo { \# case-insensitive regex }

   > There are two variants.
   >
   > 哈哈哈, `~` 区分大小写 `~*` 不区分大小写

   **4: Low priority prefix**

   There can be more than one match, the longest one takes priority

   `location /foo {# request beginning with /foo}`

   > 上面的错误，第二种改正方法是：
   >
   > ```text
   > location  /hswy-basic-web/basic/ {
   >     proxy_pass       http://basicbackend;（97.18 97.17）
   >     。。。 
   > }
   > ​
   > location /hswy-basic-web/basic/app/{ 
   >     proxy_pass       http://app-centerbackend; (97.40 97.56 97.72 97.73)
   >     。。。
   > }
   > ```
   >
   > 这样，就行了。

   **下次，对知识一定要认真不然会被开发看不起的。**

   0.0 nginx使用的正则表达式与Perl编程语言（PCRE）使用的正则表达式兼容。

   ```text
   Nginx中的server_name指令主要用于配置基于名称的虚拟主机，server_name指令在接到请求后的匹配顺序分别为：
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
   nginx将按照1,2,3,4的顺序对server name进行匹配，只有有一项匹配以后就会停止搜索，所以我们在使用这个指令的时候一定要分清楚它的匹配顺序（类似于location指令）。
   ```

   end

