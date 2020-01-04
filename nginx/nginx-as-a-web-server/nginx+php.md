# Nginx + PHP

阅读要求：已熟悉Nginx常用模块与指令,还有一些正则知识。

Nginx 利用 PHP 为client 提供动态数据。

In a web browser, the **address bar** \(also **location bar** or **URL bar**\) is a GUI widget that shows the current URL.

 A URL is one type of _**U**niform **R**esource **I**dentifier \(**URI**\)_;

### PHP 框架常用的URL 模式

#### CI框架

* REQUEST\_URI  
  R**EQUEST\_URI**模式是CI框架默认URL模式，采用分析$\_SERVER\[‘REQUEST\_URI’\]来区分控制器/动作/参数。例如：`http://localhost/index.php?/user/login?var=value`$\_SERVER\[‘REQUEST\_URI’\]值：**index.php?/user/login?var=value**

  在**REQUEST\_URI**模式的基础上添加了重写规则的支持，可以去掉URL地址里面的入口文件index.php**.**

  **这样**，就可以直接访问：`http://localhost/user/login?var=value`

* QUERY\_STRING  **QUERT\_STRING**模式就是传统GET传参数来指定当前访问的模块和操作，例如： [`http://localhost/?c=controller&m=method&var=value`](http://localhost/?c=controller&m=method&var=value)
* PATH\_INFO PATH\_INFO提供了最好的SEO支持。  PATHINFO模式下的URL访问地址是： `http://localhost/index.php/user/login/var/value/`

#### TP框架

* 普通模式  **普通模式**也就是传统的GET传参方式来指定当前访问的模块和操作，例如： `http://localhost/?m=home&c=user&a=login&var=value m标识模块，c表示控制器，a表示action，var就是其他的参数了`
* PATHINFO模式  
  THINFO模式下面的URL访问地址是： `http://localhost/index.php/home/user/login/var/value/`

  PATHINFO地址的前三个参数分别表示模块/控制器/操作

* REWRITE模式  **REWRITE模式**是在PATHINFO模式的基础上添加了重写规则的支持，可以去掉URL地址里面的入口文件index.php，但是需要额外配置WEB服务器的重写规则。
* 兼容模式  
  **兼容模式**是用于不支持PATHINFO的特殊环境，URL地址是： `http://localhost/?s=/home/user/login/var/value`

  兼容模式配合Web服务器重写规则的定义，可以达到和REWRITE模式一样的URL效果。

### Nginx配置

总结PHP常用框架的URL模式，与它对应的Nginx配置其实就三种：

* 简单模式
* PATHINFO模式
* REWRITE模式

#### 简单配置

默认配置中仅能匹配到以`.php`为结尾的uri交给php-fpm去解析。

```text
  server {    
          location ~ \.php$ {
            # 根据php listen 填入TCP Socket 或者是 Unix Socket
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
  
```

上面配置的适用场景：

1. `http://www.dodo.com/index.php` 匹配
2. `http://www.dodo.com/admin/index.php?age=18` 匹配，注意这里url中有Get变量，nginx中location匹配的路径是uri，也就是虚拟路径部分，本例也就是：`/admin/index.php`
3. `http://www.dodo.com/admin/index.php/Index/index` 不匹配，因为`$` 的存在`index`结尾不符合`php`结尾的要求

#### PATHINFO模式

根据PATHINFO的URL:`http://localhost/index.php/home/user/login/var/value/`

Nginx要做两件事情：

1. 能匹配到pathinfo请求 `location  ~  \.php  {}  //改"php$"为"php"` 或者改成`location  ~  ^(.+\.php)(.*)  {}` 这两种区别只体现在location的优先级上。
2. 将url中的php入口文件和参数切分并传递给php-fpm

   `fastcgi_split_path_info  ^(.+\.php)(\/?.*)$;`

   Defines a regular expression that captures a value for the `$fastcgi_path_info` variable. The regular expression should have two captures: the first becomes a value of the `$fastcgi_script_name` variable, the second becomes a value of the `$fastcgi_path_info` variable. For example, with these settings

   > ```text
   > location ~ ^(.+\.php)(.*)$ {
   >     ## Sets a parameter that should be passed to the FastCGI server
   >     fastcgi_split_path_info       ^(.+\.php)(.*)$;
   >     fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
   >     fastcgi_param PATH_INFO       $fastcgi_path_info;
   > ```
   >
   > **fastcgi\_param指令产生的key-value键值对最后都（未确认，暂时这么理解吧~）转换成了超全局数组变量`$_SERVER`的键值对**。
   >
   > PS:`$_SERVER[]`取的环境变量是常用的CGI环境变量

Nginx基于ngx\_http\_fastcgi\_module的本质和配置实现

```text
##匹配nginx需要交给php-fpm执行的URI，先要允许pathinfo格式的URL能够被匹配到
##所以要去掉$
location ~ ^(.+\.php)(.*)$ {
     root              /var/www/;
     fastcgi_pass   127.0.0.1:9000;
     fastcgi_index  index.php;
     ##增加 fastcgi_split_path_info指令，拆分pathinfo和script_name
     fastcgi_split_path_info       ^(.+\.php)(.*)$;
     ##PHP中要能读取到pathinfo这个变量
     ##就要通过fastcgi_param指令将fastcgi_split_path_info指令匹配到的pathinfo部分赋值给PATH_INFO
     ##这样PHP中$_SERVER['PATH_INFO']才会存在值
     ##fastcgi_param: Sets a parameter that should be passed to the FastCGI server
     fastcgi_param PATH_INFO $fastcgi_path_info;
     ##在将这个请求的URI匹配完毕后，检查这个绝对地址的PHP脚本[$fastcgi_script_name]文件是否存在
     ##如果这个PHP脚本文件不存在就不用交给php-fpm来执行了
     ##否则页面将出现由php-fpm返回的:`File not found.`的提示
     if (!-e $document_root$fastcgi_script_name) {
         ##此处直接返回404错误
         ##你也可以rewrite 到新地址去，然后break;
         return 404;
     }
     fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
     include        fastcgi_params;
}
​
```

既然，Nginx的`fastcgi_split_path_info`指令就是使用正则表达式将url拆分成两部分，那么不通过这个指令自己写正则也是可以实现的

```text
##因为nginx中$fastcgi_script_name内建变量无法通过set赋值
##所有通过设置$real_script_name这个自定义nginx变量来做中间值
location ~ \.php {
    root           /var/www/;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    ##先加载默认的fastcgi配置项
    include        fastcgi_params;
    ##正则解析路径，先使用set指令产生两个nginx变量并赋值
    ##此处先将$path_info值赋值为空
    set $path_info "";
    set $real_script_name $fastcgi_script_name;
    ##正则匹配URI，若能匹配将产生两个子组
    if ($fastcgi_script_name ~ "^(.+?\.php)(/.+)$") {
        ##将两个子组赋值给刚生成的两个nginx变量
        set $real_script_name $1;
        set $path_info $2;
    }
    ##将可能匹配到的$path_info值通过fastcgi_param指令设置进去
    fastcgi_param PATH_INFO       $path_info;
    fastcgi_param SCRIPT_FILENAME $document_root$real_script_name;
    ##覆盖fastcgi_params文件中默认的SCRIPT_NAME配置项
    fastcgi_param SCRIPT_NAME     $real_script_name;
}
​
```

PS：`fastcgi_split_path_info ^((?U).+\.php)(/?.+)$;` 这个操作有点坑。

U 非贪婪匹配，有些坑。eg：请求的url是test.17win.com/a/b/c/d/test.php。

由于采用非贪婪匹配会导致，fastcgi\_script\_name变成/d/test.php ~~

#### REWRITE模式

即利用nginx的`try_files`指令屏蔽掉入口php文件。

```text
server {
listen       80;
server_name  xxx;


rewrite ^/app\.php/?(.*)$  /$1 permanent;


location / {
    index app.php;
    try_files $uri @rewriteapp;
}
location @rewriteapp {
    rewrite ^(.*)$ /app.php/$1 last;
}

location ~ ^/(app|app_dev|test)\.php(/|$) {
    fastcgi_split_path_info  ^(.+\.php)(/.*)$;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include  fastcgi_params;
    fastcgi_pass  unix:/run/php-cgi.sock;
}

 
    
}    
```



### 总结

在实现PATHINFO时，要注意根据实际业务需求，写出正确的正则表达式。





