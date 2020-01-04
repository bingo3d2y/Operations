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

 `fastcgi_param`：这个指令配置了fastcgi的一些参数，传递给php-fpm，这个指令是3段式：

第一段fastcgi\_param 指令名称

第二段传递给php-fpm的参数的名称

第三段传递给php-fpm参数的值

也就是说fastcgi\_param配置了一系列的key-value类型的值；**对PHP来说fastcgi\_param指令产生的key-value键值对最后都（未确认，暂时这么理解吧~）转换成了超全局数组变量`$_SERVER`的键值对**。

PS:`$_SERVER[]`取的环境变量是常用的CGI环境变量

上述示例中`fastcgi_param SCRIPT_FILENAME /scripts$fastcgi_script_name`

就配置了一个SCRIPT\_FILENAME的fastcgi参数，转换成PHP中的变量就是`$_SERVER['SCRIPT_FILENAME']` 。

PHP参考手册中对`$_SERVER['SCRIPT_FILENAME']`的说明是：“当前执行脚本的绝对路径”。对nginx来说，将请求正确的交给php-fpm来执行正确的php脚本就是由fastcgi\_param指令配置的SCRIPT\_FILENAME来决定的，所以nginx能默契的与php-fpm协作，fastcgi\_param指令正确的配置了SCRIPT\_FILENAME值是关键。（parameter pə'ræmɪtə n.参数\] ）

#### PATHINFO模式

根据PATHINFO的URL:`http://localhost/index.php/home/user/login/var/value/`

Nginx要做两件事情：

1. 能匹配到pathinfo请求 `location  ~  \.php  {}  //改"php$"为"php"` 或者改成`location  ~  ^(.+\.php)(.*)  {}`
2. 将url中的php入口文件和参数切分并传递给php-fpm

   `fastcgi_split_path_info  ^(.+\.php)(\/?.*)$;`

   Defines a regular expression that captures a value for the `$fastcgi_path_info` variable. The regular expression should have two captures: the first becomes a value of the `$fastcgi_script_name` variable, the second becomes a value of the `$fastcgi_path_info` variable. For example, with these settings

   > ```text
   > location ~ ^(.+\.php)(.*)$ {
   >     fastcgi_split_path_info       ^(.+\.php)(.*)$;
   >     fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
   >     fastcgi_param PATH_INFO       $fastcgi_path_info;
   >
   > ```

nginx支持pathinfo的本质和配置实现

1. nginx需要正确将请求交给php-fpm来执行php脚本，nginx先得正确分析出URI中是否要去请求某个PHP脚本；

2、当php-fpm正确执行某个PHP脚本后，PHP中pathinfo模式实现单一入口需要PHP中`$_SERVER['PATH_INFO']`包含了正确的pathinfo值；而PHP中的$\_SERVER变量由nginx的`fastcgi_param`指令来决定；

所以让nginx支持pathinfo的配置中要修改内容也围绕这个两个点来展开。

第一、nginx的location能匹配到pathinfo格式的URI，去掉URI必须是`.php`结尾的限定，修改如下：

```text
location  ~  \.php  {}  //改"php$"为"php"
    
}
```

第二、需要将URI进行正则切割，产生正确的PHP脚本文件路径和pathinfo值；

nginx的0.7.31以上版本以后就可以使用`fastcgi_split_path_info`指令了，这个指令的参数为一个正则表达式，这个正则表示必须有两个捕获子组，从左往右捕获的第一子组自动赋值给nginx的`$fastcgi_script_name`变量，第二个捕获的子组自动赋值给nginx的`$fastcgi_path_info`变量。 子组即正则匹配的结果。

通常情况下，也就是在没有使用`fastcgi_split_path_info`指令时nginx的`$fastcgi_script_name`变量保存着相对PHP脚本的URI，这个URI相对于web根目录就是实际PHP脚本的路径，所以下方的关于`SCRIPT_FILENAME`的配置很常见。

```text
fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
$document_root即nginx的root指令指定的路径。
​
```

这样在高版本的nginx支持php的pathinfo配置就出来了，这种方式是正规且推荐的：其原理就是nginx正则分析好需要执行的PHP脚本路径和PATH\_INFO变量。

```text
##匹配nginx需要交给php-fpm执行的URI，先要允许pathinfo格式的URL能够被匹配到
##所以要去掉$
##nginx文档中的匹配规则为：^(.+\.php)(.*)$
##还有~ \.php这种写法 和 ~ \.php($|/)这种写法
##都是差不多意思没啥严格区别
##唯一区别就是有多个匹配php的location的话需要留意权重差异(即location的优先级)
location ~ ^(.+\.php)(.*)$ {
     root              /var/www/www.jjonline.cn/wwwRoot;
     fastcgi_pass   127.0.0.1:9000;
     fastcgi_index  index.php;
     ##增加 fastcgi_split_path_info指令，将URI匹配成PHP脚本的URI和pathinfo两个变量
     ##即$fastcgi_script_name 和$fastcgi_split_path_info  ^(.+\.php)(.*)$;
     ##PHP中要能读取到pathinfo这个变量
     ##就要通过fastcgi_param指令将fastcgi_split_path_info指令匹配到的pathinfo部分赋值给PATH_INFO
     ##这样PHP中$_SERVER['PATH_INFO']才会存在值
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

PS:`fastcgi_split_path_info ^(.+\.php)(.*)$;`,棒棒哒~

`fastcgi_split_path_info` 将正则的$1给`fastcgi_script_name`；$2给`pathinfo` 。

由于php配置文件`php.ini`中的`cgi.fix_pathinfo`配置项处于开启状态，php-fpm接收到这些有问题的SCRIPT\_FILENAME和PATH\_INFO后会内部自动修正，所以这种情况在PHP代码中`$_SERVER['SCRIPT_FILENAME']`和`$_SERVER['PATH_INFO']`是可以正确的修正解析的，这样配置nginx相当于把URI匹配出正确的SCRIPT\_FILENAME和PATH\_INFO值交给了php-fpm来执行，这种情况下你会发现PHP中存在`$_SERVER['ORIG_SCRIPT_FILENAME']`和`$_SERVER['ORIG_PATH_INFO']`这两个变量，或许还存在`$_SERVER['ORIG_SCRIPT_NAME']`。

最后将不再推荐的配置方式贴出来，贴出来的目的是分析下配置原理，加深nginx的配置指令理解. 即不使用`fastcgi_split_path_info` 指令，而是通过正则自己手动切分匹配。

```text
##因为nginx中$fastcgi_script_name内建变量无法赋值
##所有通过设置$real_script_name这个自定义nginx变量来做中间值
location ~ \.php {
    root           /var/www/www.jjonline.cn/wwwRoot;
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







