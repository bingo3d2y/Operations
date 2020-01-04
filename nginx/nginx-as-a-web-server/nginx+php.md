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
* PATH\_INFO  PATHINFO模式下的URL访问地址是： `http://localhost/index.php/user/login/var/value/`

#### TP框架

* 普通模式  **普通模式**也就是传统的GET传参方式来指定当前访问的模块和操作，例如： `http://localhost/?m=home&c=user&a=login&var=value m标识模块，c表示控制器，a表示action，var就是其他的参数了`
* PATHINFO模式  
  THINFO模式下面的URL访问地址是： `http://localhost/index.php/home/user/login/var/value/`

  PATHINFO地址的前三个参数分别表示模块/控制器/操作

* REWRITE模式  **REWRITE模式**是在PATHINFO模式的基础上添加了重写规则的支持，可以去掉URL地址里面的入口文件index.php，但是需要额外配置WEB服务器的重写规则。
* 兼容模式  
  **兼容模式**是用于不支持PATHINFO的特殊环境，URL地址是： `http://localhost/?s=/home/user/login/var/value`

  兼容模式配合Web服务器重写规则的定义，可以达到和REWRITE模式一样的URL效果。

#### 最简单配置

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

#### Pathinfo





