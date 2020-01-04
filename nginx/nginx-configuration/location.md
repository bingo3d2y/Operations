# nginx location

location的优先级

$uri： www.baidu.com/document

$request\_uri： www.baidu.com/document?x=1





```text

#手机端跳转
if ( $http_user_agent ~ ".*(iPhone|iPod|Android).*$" )
{
    rewrite  ^/(.*)$   'http://m.ask.17win.com/bundles/userMobile/index.html'  redirect;
}
```



1. try\_files和Named location

   ```text
    try_files $uri $uri/ /index.php?s=$uri&$args
   ```

   Checks for the existence of files in order, and returns the first file that is found. A trailing slash indicates a directory - `$uri /`. In the event that no file is found, an internal redirect to the last parameter is invoked. Do note that only the last parameter causes an internal redirect,former ones just sets the internal URI pointer. The last parameter is the fallback URI and _must_ exist, or else an internal error will be raised. Named locations can be used. Unlike with rewrite, $args are not automatically preserved if the fallback is not a named location.

   按顺序检查文件是否存在，返回第一个找到的文件。结尾的斜线表示为文件夹 -$uri/。如果所有的文件都找不到，会进行一个内部重定向到最后一个参数。

   务必确认只有最后一个参数可以引起一个内部重定向，之前的参数只设置内部URI的指向。 最后一个参数是回退URI且必须存在，否则将会出现内部500错误。

   命名的location也可以使用在最后一个参数中。与rewrite指令不同，如果回退URI不是命名的location那么、、$args不会自动保留，如果你想保留$args，必须明确声明。如上所示：`/index.php?s=​$uri&$args`

   named location

   @ ”是用来定义“Named Location ”的（你可以理解为独立于“普通location （location using literal strings ）”和“正则location （location using regular expressions ）”之外的第三种类型），这种“Named Location ”不是用来处理普通的HTTP 请求的，它是专门用来处理“内部重定向（internally redirected ）”请求的。注意：这里说的“内部重定向（internally redirected ）”或许说成“forward ”会好点，以为内internally redirected 是不需要跟浏览器交互的，纯粹是服务端的一个转发行为。

   ```text
       location / { 
           try_files $uri $uri/ /index.php?s=$uri&$args @rewrite;
       }   
       #从上面来看，try_files 按顺序寻找，也就是当/index.php也不存在时才会用到@rewrite --哈哈，
        所以说几乎是用不着了
       location @rewrite {
           set $static 0;
           if  ($uri ~ \.(css|js|jpg|jpeg|png|gif|ico|woff|eot|svg|css\.map|min\.map)$) {
               set $static 1;
           }   
           if ($static = 0) {
               rewrite ^/(.*)$ /index.php?s=/$1;
           }   
       } 
   ​
   ```

   `try_files` directive only accepts one named location, so apparently it goes for the last one. This [blog post](http://linuxplayer.org/2013/06/nginx-try-files-on-multiple-named-location-or-server) proposes a workaround that works in your case. In case you don't won't read the whole post, you can add the following lines at the end of `@local` block: \#这个好骚，算是变相实现try\_files后面加两个Named localtion

   ```text
   proxy_intercept_errors on;
   recursive_error_pages on;
   error_page 404 = @remote;
   ​
   ```

   and change your `try_files` to this:

   ```text
   try_files /$uri @local;
   ​
   ```

 PS：框架内部有函数可以取到经nginx转换后的uri，llicat说框架其实底层也是用的$server\[request\_uri\].

