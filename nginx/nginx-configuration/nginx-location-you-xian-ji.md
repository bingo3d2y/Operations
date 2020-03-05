# Nginx Location

**Problems：Nginx 增加很简单的location后，访问验证发现不生效**

即期望client请求`/hswy-basic-web/basic/app/`时，可以转到`app-centerbackend`

> **Nginx 配置如下：**
>
> ```text
> ##nginx错误配置如下：两个类似的location，反向代理到不同的backend
> ​
> location ~ /hswy-basic-web/basic/ {
>     proxy_pass       http://basicbackend;（97.18 and 97.17）
>     。。。 
> }
> ​
> location ~ /hswy-basic-web/basic/app/{ 
>     proxy_pass       http://app-centerbackend; (97.56 and  97.72 and 97.73)
>     。。。
> }
> ```

**Actions：瞎比比想了半天后，就是想不通，最后才想起来去看日志（早该如此）**

> ```text
> # 呐，日志如下：  
> ​
> {"logtime": "29/Aug/2017:21:59:42 +0800", "client_ip": "120.239.88.139", "remote_user": "-", "resp_len": "957", "waster_time": "0.003", "status": "404", "request_path": "GET /hswy-basic-web/basic/app/query?&_1504015359770&accountId=&appType=&groupId=&locationCode=440703&terminalCode=092801GD010000&userInfo=%7B%22locationCode%22:%22440703%22,%22nsrsbh%22:%2291440703688620696H%22%7D HTTP/1.1", "request_method": "GET", "request_url": "https://17hs.servyou.com.cn/17hs/gs/gdMain/view/index.html", "body_bytes_sent":"957", "http_forwarded": "-", "user_agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/31.0.1650.57 Safari/537.36" ,"upstream":  "X.X.97.18:8080, X.X.97.17:8080","upstream_status": "404, 404"}
> ​
> ```
>
> **由上可知：get请求"GET /hswy-basic-web/basic/app/query?..."时，`upstream_status` :404**
>
> `upstream_status` 404则说明是后端的问题了。
>
> **且upstream到了97.18上面** ，按照我们的一厢情愿觉得应该到97.40 97.56 97.72 97.73上面的。

**Results：调整配置文件位置，请求访问成功。**

> **修改后如下**，仅仅调换了顺序
>
> ```text
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

**Knowledge**：Nginx的localtion 匹配规则不是我臆测的那样会“精确匹配”

> 我的臆测是：当请求的uri是`/hswy-basic-web/basic/app/query?...` 时`~ /hswy-basic-web/basic/app/`比`~ /hswy-basic-web/basic/` 更精确所以优先级也搞，从而认为它们的顺序无所谓....
>
> 其实，臆测的全错了-- 我根本不懂nginx的location 模块

[http://nginx.org/en/docs/http/ngx\_http\_core\_module.html\#location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) （官方文档）

**Understanding NGINX location rules**

这里要理解两个概念： 匹配顺序和匹配优先级

**location** \[ `=` \| `~` \| `~*` \| `^~` \] uri { ... }

location只分为两种：普通\(prefix location\) 和 正则 \(regular expression location\)。

以`~`或`^~`开头的都是正则location，其余的都是普通location（包含`=`、`^~`）。

下面是Nginx官网对location匹配顺序的描述：

To find location matching a given request, nginx first checks locations defined using the prefix strings \(prefix locations\). Among them, the location with the longest matching prefix is selected and remembered. Then regular expressions are checked, in the order of their appearance in the configuration file. The search of regular expressions terminates on the first match, and the corresponding configuration is used. If no match with a regular expression is found then the configuration of the prefix location remembered earlier is used.

Nginx匹配location进行转发，匹配顺序和优先级如下：

①先检测所有的prefix location（普通location），记录下最长匹配结果，但不接受匹配，转向regular location。

②如果prefix中存在`=`，存在精确匹配结果就结束匹配，直接转发.

③如果prefix中存在`^~`则也结束匹配，选择`^~`开头的最长匹配结果进行转发

④匹配 regular expressions location，匹配到第一个就停止匹配进行转发。这里是先匹配到哪个哪个优先级最高

⑤ regular expressions 若没有匹配，则使用第①步中记录的最长匹配结果，进行转发

There are **4 types of location** rule, and are applied with the following priorities:

**1: Exact matches**： `=` 精确匹配，优先级最高

There can be only one exact match – the clue is in the name!

> 独一无二的，按名字匹配，优先级最高

`location = /foo/bar {# exact match}`

**2: High priority prefix** ：`^~` 前缀高优先级

There can be more than one match, the longest one takes priority

> 匹配以`/foo` 开头的uri，能匹配多个uri，匹配的uri越长优先级越高
>
> **soga，我把`High priority prefix` 的特性强行记错到`Regex`上面了**

`location ^~ /foo {# request beginning with /foo}`

**3: Regex**: `~` or `~*`

There can be more than one match, the first one found takes priority. There are two variants.即`~` 和 `~*`

> `Regex` 的特性就是先匹配到哪个，哪个的优先级就高.
>
> 所以，如下配置client请求`/hswy-basic-web/basic/app/`时会被转发到`basicbackend`
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
> ```

`location ~ .foo {# case-sensitive regex}` `location ~* .foo​ { # case-insensitive regex}`

> `~` 区分大小写 `~*` 不区分大小写

**4: Low priority prefix**

There can be more than one match, the longest one takes priority.

`location /foo {# request beginning with /foo}`

> 上面的错误，第二种改正方法就是使用Low priority prefix：
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

prefix location 是比 regular expression先进行匹配的。所以Nginx官方也给出了一些优化建议：

**使用符号“=”修饰符可以定义一个精确匹配的URI和位置，如果找到了一个精确的匹配，则搜索终止，例如，如果一个”/”请求频繁发生，定义“location =/”将加快这些请求的处理，一旦精确匹配只有就结束，这样的location显然不能包含嵌套location**

**这里说一下location / {} 和location =/ {}的区别：**

`location / {}`是普通的最大前缀匹配，任何的uri肯定是以`/`开头，所以`location / {}`可以说是默认匹配，当其他都不匹配了，则匹配默认匹配.

