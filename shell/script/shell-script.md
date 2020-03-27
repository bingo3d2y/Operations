# Shell Script（待整理细分）

### Shell Scripts

```text
- If you find you need to use arrays for anything more than assignment of ${PIPESTATUS}, you should use Python.
- If you are writing a script that is more than 100 lines long, you should probably be writing it in Python instead. Bear in mind that scripts grow. Rewrite your script in another language early to avoid a time-consuming rewrite at a later date.
​
0.0
```

end

**shell script 原则： fail fast**

```text
以后记住，这就是国际惯例了
#!/bin/bash
set -o nounset
set -o errexit
​
## genius
set -o nounset
在默认情况下，遇到不存在的变量，会忽略并继续执行，而这往往不符合预期，加入该选项，可以避免恶果扩大，终止脚本的执行。
$ (echo $a;echo 'test')
​
test
$ (set -o nounset ;echo $a;echo 'test')
-bash: a: unbound variable
​
set -o errexit
在默认情况下，遇到执行出错，会跳过并继续执行，而这往往不符合预期，加入该选项，可以避免恶果扩大，终止脚本的执行。
PS：有些Linux命令，例如rm的-f参数可以强制忽略错误，此时脚本便无法捕捉到errexit，这样的参数在脚本里是不推荐使用的。
$ (ls /tmp/a.lolp; echo 'error is not exit')
ls: cannot access '/tmp/a.lolp': No such file or directory
error is not exit
$ (set -o errexit;ls /tmp/a.lolp; echo 'error is not exit')
ls: cannot access '/tmp/a.lolp': No such file or directory
```

end

**shell的不足：sshpass**

> 异常难以捕捉及异常提示不明显

2019/06/13 ,今天`sshpass`命令击溃了我对`shell > python > ansible`的心里。

原因：批量机器给机器添加互信和一些有初始化配置。

> 没立马定位到时sshpass执行的错误，而且没对sshpass的执行结果做判断。

0.0 `StrictHostKeyChecking no` 写入，配置文件中谢谢！！！

PS： ssh配置免密登录时必须保证，user的uid和gid相同，不然ssh做互相，无法登录。

root都是0不会有什么问题，但是其他user直接的互相，一定要保证user的id和权限相同。

因为，sshd的默认参数中`StrictModes yes`，则ssh在接收登录请求之前是否检查用户家目录和rhosts文件的权限和所有权，因此必需保证存放公钥的文件夹的拥有与登陆用户名是相同的.

若，建用户时，uid和gid这些没有保证一致，则可以通过将`StrictModes`改成`no`即可。

0.0 还是不行的话只能掏出绝招了`ssh -vv host_ip`

Specifies whether sshd\(8\) should check file modes and ownership of the user’s files and home directory before accepting login. This is normally desirable because novices sometimes accidentally leave their directory or files world-writable. The default is “yes”. Note that this does not apply to ChrootDirectory, whose permissions and ownership are checked unconditionally

[https://bingozb.github.io/33.html](https://bingozb.github.io/33.html)

```text
## 期望：先免交互添加ssh互信，然后远程过去执行command.
ssh-keygen -t rsa  -f ~/.ssh/id_rsa q -N ""
for host in 192.168.1.{1.100}
do
  sshpass -p 'zpepc001@' ssh-copy-id -p 10022  $host
  ssh -o StrictHostKeyChecking=no $host "command"
done
​
## 结果： 出现提示： root@ip's password: 
## 哈哈哈，当时不冷静，我以为是sshpass命令 -p 参数无效了。
## 后来冷静了，才发现是sshpass 执行失败了，然后想象开始执行ssh $host "command" 提示需要 password
​
## 科普 sshpass
$ man sshpass
...
RETURN VALUES
       As with any other program, sshpass returns 0 on success. In case of failure, the following return codes are used:
​
       1      Invalid command line argument
​
       2      Conflicting arguments given
​
       3      General runtime error
​
       4      Unrecognized response from ssh (parse error)
​
       5      Invalid/incorrect password
​
       6      Host public key is unknown. sshpass exits without confirming the new key.
...
$ sshpass -p 'zpepc001@' ssh-copy-id -p 10022  $host
# 啧啧啧，返回值是6. 经验证，这个error(6)的错误提示，是因为目标host，是未知主机--
# 关了StrictHostKeyChecking即可。
$ echo $?
6
​
丢人了：
sshpass -p 'zpepc001@' ssh-copy-id -p 10022 -o StrictHostKeyChecking=no  $host （错误）
emmm ， sshpass -p '' ssh -o StrictHostKeyChecking=no $host Command (√)
这个StrictHostKeyChecking no 还是写在config吧
​
## 校正版
修改`/etc/ssh/ssh_config` ,将StrictHostKeyChecking ask改成StrictHostKeyChecking no
---
ssh-keygen -t rsa  -f ~/.ssh/id_rsa q -N ""
for host in 192.168.1.{1.100}
do
  sshpass -p 'zpepc001@' ssh-copy-id -p 10022  $host
 # sshpass -p 'zpepc001@' ssh-copy-id -p 10022 -o StrictHostKeyChecking=no $host
  ssh -o StrictHostKeyChecking=no $host "command"
done
​
## 严谨版
创建 ~/.ssh/config，写入StrictHostKeyChecking no
---
ssh-keygen -t rsa  -f ~/.ssh/id_rsa q -N ""
for host in 192.168.1.{1.100}
do
  ## shoot，这个ssh-copy-id -p 10022 -o StrictHostKeyChecking=no 错的
  ##sshpass -p 'zpepc001@' ssh-copy-id -p 10022 -o StrictHostKeyChecking=no $host
  sshpass -p 'zpepc001@' ssh-copy-id -p 10022  $host
  if [ $? -ne 0 ]; then echo 'sshpass error'; exit(1);fi
  ssh -o StrictHostKeyChecking=no $host "command"
done
​
额，，，  if [ $? -ne 0 ]; then echo 'sshpass error'; exit(1);fi这个我写的有问题了
```

0..0

感悟： 一半由于我的问题，另一半就是`sshpass`的错误了，因为它没有执行成功只返回了错误码，没在`stderr`输出任何信息，让我以为...`sshpass`执行没有问题...

以前别人告诉我，shell脚本对异常处理不好，我嗤之以鼻，现在终于为自己的年少无知买单了。

shell 是方便快捷，但是确实有不好的地方，比如这个`sshpass`就对新手的支持没那么好，出现问题不抛出`stderr`,只给一个返回码，留个使用者判断，不妥当。

Python或者Ansible就好很多了，对每一个执行步骤都有错误异常抛出或者错误提示。Ansible的模块操作还类似k8s的声明式API，保障执行命令处于用户的期望值。

啧啧，要是这里我用Ansible的`authoried_keys`模块，是不是早搞定了--

**& and nohup**

0.0 nohup到底干嘛的

 nohup 和 & 都是挂在当前bash的后端运行，只是nohup 可以忽视SIGHUP信号量

The nohup utility shall invoke the utility named by the utility operand with arguments supplied as the argument operands. At the time the named utility is invoked, the SIGHUP signal shall be set to be ignored.

```text
# 0.0 nohup 和 & 都是挂在当前bash的后端运行，只是nohup 可以忽视SIGHUP信号量
[root@vultr ~]# nohup  sleep 1000 &
[1] 29168
[root@vultr ~]# sleep 1500 &
[2] 29169
[root@vultr ~]# nohup  sleep 1000 &^C
[root@vultr ~]#  ps -ef|grep sleep |grep -v grep
root     29168 29140  0 04:45 pts/0    00:00:00 sleep 1000
root     29169 29140  0 04:46 pts/0    00:00:00 sleep 1500
## 此时，从xshell或者mobaX关掉当前标签页，新开标签页可看到
[root@vultr ~]# ps -ef|grep sleep
root     29168     1  0 04:45 ?        00:00:00 sleep 1000 ## 通过nohup方式运行的
​
## nohup挡不住kill -9 哈哈
## 0.0 不是一个信号量了--
$ nohup sleep 2200 &
[1] 29394
$ ps -ef|grep sleep
root     29394 29363  0 04:57 pts/0    00:00:00 sleep 2200
root     29397 29363  0 04:58 pts/0    00:00:00 grep --color=auto sleep
[root@vultr ~]# kill -9 29394
$ ps -ef|grep sleep
root     29401 29363  0 04:58 pts/0    00:00:00 grep --color=auto slee
​
0.0 
# man 7 signal
...
       Signal     Value     Action   Comment
       ──────────────────────────────────────────────────────────────────────
       SIGHUP        1       Term    Hangup detected on controlling terminal
                                     or death of controlling process
       SIGINT        2       Term    Interrupt from keyboard
       SIGQUIT       3       Core    Quit from keyboard
       SIGILL        4       Core    Illegal Instruction
       SIGABRT       6       Core    Abort signal from abort(3)
       SIGFPE        8       Core    Floating point exception
       SIGKILL       9       Term    Kill signal
       SIGSEGV      11       Core    Invalid memory reference
       SIGPIPE      13       Term    Broken pipe: write to pipe with no
                                     readers
       SIGALRM      14       Term    Timer signal from alarm(2)
       SIGTERM      15       Term    Termination signal
       SIGUSR1   30,10,16    Term    User-defined signal 1
       SIGUSR2   31,12,17    Term    User-defined signal 2
       SIGCHLD   20,17,18    Ign     Child stopped or terminated
       SIGCONT   19,18,25    Cont    Continue if stopped
       SIGSTOP   17,19,23    Stop    Stop process
       SIGTSTP   18,20,24    Stop    Stop typed at terminal
       SIGTTIN   21,21,26    Stop    Terminal input for background process
       SIGTTOU   22,22,27    Stop    Terminal output for background process
​
```

end

**※变量与双引号**

What is the difference between $var and "$var"? \[duplicate\]

In short: 有`""`会被视为一个整体，啧啧，这个特性真棒。

```text
In echo $var, since the expansion of $var does not occur within double-quotes, so it does undergo splitting. 即不带引号，变量会被分隔。
​
$ a='''a
> n
> b
> '''
​
$ echo $a
a n b
​
$ echo "$a"
a
n
b
​
## $variabel without double-quote
HOSTS="10.1.1.1 10.1.1.2 10.1.1.3"
for host in $HOSTS ;do echo $host ;done
10.1.1.1
10.1.1.2
10.1.1.3
​
## $variable with double-quote
for host in "$HOSTS" ;do echo $host ;done
10.1.1.1 10.1.1.2 10.1.1.3
​
​
## $$variable with double-quote
## 厉害了，肖公子
command='''
cat << EOF > /etc/docker/daemon.json 
{
  "storage-driver": "overlay2",
  "live-restore": true,
  "log-opts": {
    "max-size": "20m",
    "max-file": "10"
  },
  "insecure-registries": []
}
EOF
echo "test"
echo "genius"
'''
## It's failure that use variable without double-quote.
ssh IP "$command"  
请注意这个双引号
​
### 在来个例子
func_a(){
    echo a b c d
}
func_b(){
    for i in $1
    do 
      echo $i
    done
}
​
## $variable with double-quote
func_b "$(func_a)"
a
b
c
d
​
## $variable without double-quote
## 就一个a
func_b $(func_a)
a
-- 
```

单引号和双引号的区别：

* literal：也就是普通的纯文字，对`shell`来说没特殊功能；
* meta: 对`shell`来说，具有特定功能的特殊保留元字符。

单引号会使`meta character` 失效。

**空格**

shell中最神奇的转义字符： space

`{}` --&gt; 前后加上空格变成函数体了：`foo() { a=3; }`

`[]` --&gt; 前后加上空格变成判断体了： `if [ $a -eq 3 ]`

**变量/环境变量 检测**

子shell访问当前shell中的变量

* export ，`export test=0`, sub-shell 可以继承
* `test=0 echo $test or (test=0 echo $test)`: 在同一个shell层执行、

  systemd 的 Unit File ，加载Environmental 也是再fork 子进程时，同时加载的Environmental，所启动的进程可以读到env

变量访问：

```text
$ hello="this is some text"   # we set $hello
$ var="hello"                 # $var is "hello"
$ echo "${!var}"              # we print the variable linked by $var's content
this is some text
$ echo ${var}                # 0.0
hello
​
```

> If the first character of parameter is an exclamation point \(**!**\), and _parameter_ is not a _nameref_, it introduces a level of variable indirection. Bash uses the value of the variable formed from the rest of parameter as the name of the variable; this variable is then expanded and that value is used in the rest of the substitution, rather than the value of _parameter_ itself. This is known as indirect expansion. If parameter is a _nameref_, this expands to the name of the variable referenced by parameter instead of performing the complete indirect expansion. The exceptions to this are the expansions of ${!prefix\*} and ${!name\[@\]} described below. The exclamation point must immediately follow the left brace in order to introduce indirection.
>
> 0.0

end --' --' --'

```text
function require_ev_all() {  //全部参数都不为空，才会正常返回，否则指出哪个变量没设置，并报错。
        for rev in $@ ; do
                if [[ -z "${!rev}" ]]; then
                        echo "${rev}" is not set
                        exit 1
                fi
        done
}
​
function require_ev_one() {  //只要有一个参数不为空，就正常返回，结束函数。
        for rev in $@ ; do
                if [[ ! -z "${!rev}" ]]; then
                        return
                fi
        done
        echo One of $@ must be set
        exit 1
}
​
### test
$ tail -n 1 test.sh
require_ev_all  $@
$ lol=1  bash   test.sh  lol  lpl
lpl is not set
$ tail -n 1 test.sh
require_ev_one  $@
$ lol=1  bash   test.sh  lol  lpl   ## lol 声明了，所以就不报错了
$ bash   test.sh  lol  lpl
One of lol lpl must be set
```

end

**将两行合并成一行**

```text
## awk
$ awk 'ORS=NR%2?" ":"\n" {print}' /tmp/a.txt
NR        current record number in the total input stream. 当前行数
ORS       terminates each record on output, initially = "\n".  默认输出行已什么结尾 
?: 三目运算符 --
​
## sed
$ sed 'N;s/\n/ /g' /tmp/a.txt
n N    Read/append the next line of input into the pattern space.
n: the pattern space 有两行数据
N：the pattern space 将两行数据视为一行
```

end

**间接变量**

[https://blog.csdn.net/hepeng597/article/details/8057692](https://blog.csdn.net/hepeng597/article/details/8057692)

* {!}

  ```text
  [node2 ~]$ a=3
  [node2 ~]$ b=a
  [node2 ~]$ echo $b
  a
  [node2 ~]$ echo  ${!b}
  3
  ```

  end

* eval -- 这个东西比较好玩

  ```text
  [node2 ~]$ a=3
  [node2 ~]$ b=a
  [node2 ~]$ eval c=\${$b}
  [node2 ~]$ echo $c
  3
  ```

  end

**统计glusterfs使用情况**

```text
# 思路，挂载volume 然后df统计
$ a=1; for i in `gluster volume list`;do echo $i; echo $a; mount -t glusterfs gluster_ser:$i /tmp/test$a; sleep 1; let a++; done

# 清理
$ a=1; for i in `gluster volume list`;do echo $i; echo $a; umount /tmp/test$a; sleep 1; let a++; done
```

end

**监控CPU的每个core**

思路：

1. 拿出每个core的元负载数据
2. 聚合每个core的负载数据
3. 拿出其中的最大值，告警即可

```text
# sar 换成 mpstat 一样
# 聚合5s
$ sar -P ALL 1 5 | grep Average |grep -v CPU|awk '{printf "%d\n",$3}'|sort -rn|head -n1
```

end

**gawk**

0.0 awk 是门编程语言咯

1. gawk 进行运算

   ```text
   $ cat  access.log |awk -F '[",]+' '$20 > 50 && $28~/lib\/angular\/angular.min.js/ {print $20,$8,$28}'|sort -rnk 1 |head -100
   $ cat  access.log |awk -F '[",]+' '$20 > 2 && $28~/lib\/angular\/angular.min.js/ {print $20,$8,$28}'|wc -l 
   啧啧，这小操作$20 > 2 && $28~/lib\/angular\/angular.min.js/  ,666
   $ cat /tmp/a.txt
   900=9
   $ cat /tmp/a.txt|gawk -F "=" '{i=$1;print i/3}'  //受超神启迪。
   30
   ```

   end

2. gawk数组:这个6

   ```text
   $ ss -n |gawk '/^tcp/ {++Socket[$2]} END {for( i in Socket) print i, Socket[i]}'  //不加-a参数不行啊
   CLOSE-WAIT 1
   ESTAB 18
   $ ss -an |gawk '/^tcp/ {++Socket[$2]} END {for( i in Socket) print i, Socket[i]}'
   LISTEN 16
   CLOSE-WAIT 1
   ESTAB 41
   TIME-WAIT 62
   UNCONN 9
   ```

   end

3. printf: 格式控制

   ```text
   ## 啧啧，格式控制的真好
   $ cat /tmp/a.txt| awk '{printf ("%-20s %-6s %-6s\n",$1,$2,$3)}'
   VolumeName           Size   Used  
   95598yyglxt          2.0T   456G  
   amrdb_arch           11T    126G  
   amrtest              1.9T   539G  
   consdb_arch          830G   106G 

   ## 源文件
   VolumeName                           Size  Used Avail Use% Mounted on
   95598yyglxt        2.0T  456G  1.6T  23% /tmp/test1
   amrdb_arch          11T  126G   11T   2% /tmp/test2
   amrtest            1.9T  539G  1.3T  29% /tmp/test3
   consdb_arch        830G  106G  724G  13% /tmp/test4
   cwdzbz             1.9T   90G  1.8T   5% /tmp/test5
   ```

   end

4. 正则匹配`/regular expression/`

   ```text
   root@fishong:~# cat /tmp/a.txt|awk "/.*/"
   lll
   111
   bbb
   ccc
   aaa aaa
   kkk
   kll

   root@fishong:~# cat /tmp/a.txt|awk "/a+/"
   aaa aaa

   ## 高级玩法
   $ free -m | awk '/Mem/{sum=$4+$6}END{printf "%d\n", sum/1024}'
   ```

   end

5. awk + 系统命令调用

   ```text
   先打印测试： ps -ef |grep worker| awk '{print "kill -9" $2}'
   直接执行：ps -ef |grep worker| awk '{system("kill -9" $2)}'
   ```

6. awk：基于某列求和

   emm, awk是个编程语言，真棒

   ```text
   ## 利用lsof查看已删除但是为释放空间的文件
   ## 单位bytes
   $ lsof  -n|grep deleted|awk '{sum += $7};END{print sum}'
   ```

   0.0

7. end

**shell -- array**

* 索引数组： 索引只能是整数。 `declare -a words`
* 关联数组：索引可以是字符串、整数及其他类型。`declare -A words`

众所周知，Linux是C语言编写的。那么，啧啧我是不是可以认为Linux的变量和C语言的一样。

变量是内存盒子即多次声明，只要变量名字不变，则指向同一内存。

```text
$ declare -a a
$ a[0]=1
$ declare -a a  ## 多次声明并没有覆盖前面的a
$ echo ${a[0]}
1
```

具体实用例子：

* `${array{@}}` : 数组的全部值
* `${!array{@}}`: 数组的全部索引
* `${#arrry{@}}`: 索引个数
* `@`和`*` 的区别：${数组\[@\]}得到是以空格隔开的元素，而${数组\[\*\]}得到一整个字符串。（这个区别和$@、$\* 表示脚本的传参列表类似的）

```text
$ cat /tmp/a.txt 
1
lol
2
lol
3
lol
4
## Declare idnex array : declare -a array_name.
## Declare Associative array: declare -A array_name.
## math operator: let Expression
## < <(command)  # <{space_character}<
$ declare -A words;while read line;do  let words[$line]++ ; done < <(cat /tmp/a.txt)
$ echo ${!words[@]}  ## index
lol 1 2 3 4
$ echo ${words[@]} ## value
3 1 1 1 1
$ echo ${#words[@]}  ## number of index
5

## traversal array
$ for i in ${!words[@]};do echo "words[$i]: ${words[$i]}";done 
words[lol]: 3
words[1]: 1
words[2]: 1
words[3]: 1
words[4]: 1

### 这样写也没关系，因为C语言声明的变量是内存的盒子--
### 多次重复声明的同名变量是一样的
while read line ;do declare -A array ; let array[$line]++; done< <(cat /tmp/a.txt)
```

end

**pipe with fork**

管道使变量失效。

理解Pipe ： [https://brandonwamboldt.ca/how-linux-pipes-work-under-the-hood-1518/](https://brandonwamboldt.ca/how-linux-pipes-work-under-the-hood-1518/)

Problem: 管道符后面的变量无法被引用，如下：

because pipes create forks, you have to use $foo before the pipe ends, so...

```text
$ echo "abc" | read foo ;echo "foo: ${foo}"  # foo变量没有值
foo: 
#because pipes create forks, you have to use $foo before the pipe ends, so...
$ echo "abc" | (read foo ;echo "foo: ${foo}")
foo: abc
```

**（） VS { } VS $（） VS ${ }**

`$()` 子命令

```text
 Command Substitution
       Command substitution allows the output of a command to replace the command name.  There are two forms:

              $(command)
       or
              `command`

Bash performs the expansion by executing command and replacing the command substitution with the standard output of  the  command,  with  any  trailing  newlines  deleted.Embedded  newlines  are  not deleted, but they may be removed during word splitting.  The command substitution $(cat file) can be replaced by the equivalent but faster $(<file).

推荐是使用，
（1）$()能够支持内嵌；

（2）$()不用转义；

（3）有些字体，`(反单引号)和’(单引号)很像，容易把人搞晕；

$ echo "A-$(echo B-$(echo C-))D"
A-B-C-D
```

end

If you want the side-effects of the command list to affect your **current** shell, use `{...}` If you want to discard any side-effects, use `(...)`

For example, I might use a subshell if I:

* want to alter `$IFS` for a few commands, but I don't want to alter `$IFS` globally for the current shell
* `cd` somewhere, but I don't want to change the `$PWD` for the current shell

It's worthwhile to note that parentheses can be used in a function definition:

* normal usage: braces: function body executes in current shell; side-effects remain after function completes

  > 卧槽这个`(*)`好强啊--直接生产一个数组
  >
  > 将当前路径下所有文件放到一个数组里-

  ```text
  $ count_tmp() { cd /tmp; files=(*); echo "${#files[@]}"; }
  $ pwd; count_tmp; pwd
  /home/jackman
  11
  /tmp
  $ echo "${#files[@]}"
  11    
  ```

* unusual usage: parentheses: function body executes in a subshell; side-effects disappear when subshell exits

  ```text
  $ cd ; unset files
  $ count_tmp() (cd /tmp; files=(*); echo "${#files[@]}")
  $ pwd; count_tmp; pwd
  /home/jackman
  11
  /home/jackman
  $ echo "${#files[@]}"
  0
  ```

  end

\(\) -- 在子shell中执行并产生影响

这个好像有点用耶

```text
if (ip netns) > /dev/null 2>&1; then :; else
    echo >&2 "$UTIL: ip utility not found (or it does not support netns),"\
             "cannot proceed"
    exit 1
fi

## 这个 if (ip netns) 检测了 ip command 是否存在。
```

0.0 下面详细的

> ```text
> ( list )
> ```
>
> Placing a list of commands between parentheses causes a subshell environment to be created, and each of the commands in list to be executed in that subshell. Since the list is executed in a subshell, variable assignments do not remain in effect after the subshell completes.

{} -- 在当前shell中执行

> ```text
> { list; }
> ```
>
> Placing a list of commands between curly braces causes the list to be executed in the current shell context. No subshell is created. The semicolon \(or newline\) following list is required.

区别：shell 函数的定义

```text
root@fishong:/tmp# foo() { a=3; } ## 定义函数
root@fishong:/tmp# foo
root@fishong:/tmp# echo  $a
3
root@fishong:/tmp# unset a
root@fishong:/tmp# echo  $a

root@fishong:/tmp# foo() ( a=3; ) ## 定义函数
root@fishong:/tmp# foo
root@fishong:/tmp# echo  $a  ## no output

root@fishong:/tmp#
```

end

终极BOSS,好好好分析下

```text
## logger 命令将日志发送到/var/log/message
export PROMPT_COMMAND='{ msg=$(history 1 | { read x y; echo $y; });user=$(whoami); logger  -p local0.info  -t history  \{\"time\":\"$(date "+%Y-%m-%d %H:%M:%S")\",\"user\":\"$user\",\"msg\":\""$msg"\"\}; }'
```

end

${} -- 提取和修改字符串

```text
$ id=`uuidgen | sed 's/-//g'`
$ echo $id
8b1064f100cf4977a40516c0f244c111
$ echo ${id:0:13}
8b1064f100cf4

假设我们定义了一个变量为：
file=/dir1/dir2/dir3/my.file.txt
我们可以用 ${ } 分别替换获得不同的值：
${file#*/}：拿掉第一条 / 及其左边的字符串：dir1/dir2/dir3/my.file.txt
${file##*/}：拿掉最后一条 / 及其左边的字符串：my.file.txt
${file#*.}：拿掉第一个 . 及其左边的字符串：file.txt
${file##*.}：拿掉最后一个 . 及其左边的字符串：txt
${file%/*}：拿掉最后条 / 及其右边的字符串：/dir1/dir2/dir3
${file%%/*}：拿掉第一条 / 及其右边的字符串：(空值)
${file%.*}：拿掉最后一个 . 及其右边的字符串：/dir1/dir2/dir3/my.file
${file%%.*}：拿掉第一个 . 及其右边的字符串：/dir1/dir2/dir3/my

记忆的方法为：  #和## 及 %和%% 是贪婪匹配和非贪婪匹配的区别。

# 是去掉左边(在鉴盘上 # 在 $ 之左边)

% 是去掉右边(在鉴盘上 % 在 $ 之右边)

单一符号是最小匹配﹔两个符号是最大匹配。
```

end

**sed and gawk**

* gawk

  1\) 西津的：将在一个master节点上操作全部slave节点

  ```text
  for i in seq 1 8; do echo kube-nodei; ssh kube-nodei "$@"; done 
  ```

  ```text
  s1.sh 'docker images' |grep xxb|gawk 'OFS=":" {print $1,$2}'|grep -v xxb_base > /tmp/xxb.txt
  啧啧，gawk OFS  parameter 也挺不错的~~
  ```

  2\) 超神的操作：gawk是个编程语言哦

  ```text
  # cat  access.log |awk -F '[",]+' '$20 > 50 && $28~/lib\/angular\/angular.min.js/ {print $20,$8,$28}'|sort -rnk 1 |head -100
  # cat  access.log |awk -F '[",]+' '$20 > 2 && $28~/lib\/angular\/angular.min.js/ {print $20,$8,$28}'|wc -l 
  啧啧，这小操作$20 > 2 && $28~/lib\/angular\/angular.min.js/  ,666
  #cat /tmp/a.txt
  900=9
  #cat /tmp/a.txt|gawk -F "=" '{i=$1;print i/3}'  //受超神启迪。
  30
  ```

  end

  AWK运算符

  **运算符介绍**

  | 运算符 | 描述 |
  | :--- | :--- |
  | **赋值运算符** |  |
  | = += -= \*= /= %= ^= \*\*= | 赋值语句 |
  | **逻辑运算符** |  |
  | \|\| | 逻辑或 |
  | && | 逻辑与 |
  | **正则运算符** |  |
  | ~ ~! | 匹配正则表达式和不匹配正则表达式 |
  | **关系运算符** |  |
  | &lt; &lt;= &gt; &gt;= != == | 关系运算符 |
  | **算术运算符** |  |
  | + - | 加，减 |
  | \* / & | 乘，除与求余 |
  | + - ! | 一元加，减和逻辑非 |
  | ^ \*\*\* | 求幂 |
  | ++ -- | 增加或减少，作为前缀或后缀 |
  | **其它运算符** |  |
  | $ | 字段引用 |
  | 空格 | 字符串连接符 |
  | ?: | C条件表达式 |
  | in | 数组中是否存在某键值 |

   end

  3\) awk 还有过滤的作用

  ```text
  # netstat -n  | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
  TIME_WAIT 156
  ESTABLISHED 100

  # man gawk
  ...
     Patterns
         AWK patterns may be one of the following:

                BEGIN
                END
                /regular expression/
                relational expression
                pattern && pattern
                pattern || pattern
                pattern ? pattern : pattern
                (pattern)
                ! pattern
                pattern1, pattern2

  $ ss -n |gawk '/^tcp/ {++Socket[$2]} END {for( i in Socket) print i, Socket[i]}'  //不加-a参数不行啊
  CLOSE-WAIT 1
  ESTAB 18
  $ ss -an |gawk '/^tcp/ {++Socket[$2]} END {for( i in Socket) print i, Socket[i]}'
  LISTEN 16
  CLOSE-WAIT 1
  ESTAB 41
  TIME-WAIT 62
  UNCONN 9
  ```

  4\)内置变量：OFS Eg: `'{OFS=":"}'`

  ```text
  # man gawk 
  。。。
  Built-in Variables
         Gawk's built-in variables are:
  OFS         The output field separator, a space by default.
  ...

  # echo "lol ll l"|gawk   'BEGIN{OFS=":"}{print $1,$2}'
  PS: gawk用双引号，表示其里面是变量值。因为单引号有其他作用了。

  延伸，清理docker镜像（因为镜像可能有多个tag导入无法用docker rmi images_id删除）
  # while read line;do docker rmi `echo $line |gawk '{OFS=":"} {print $1,$2}'`;done < <(docker images |grep -v ^[A-Z])
  ```

  end

* sed

  sed 在指定行或者匹配行前\(insert\)后\(append\)追加内容

  * 在匹配内容所在行处追加

    ```text
    ## 在#bbb 行后append lpl
    $ sed -i '/#bbb/a lpl'  /tmp/a.txt 
    $ cat /tmp/a.txt 
    new a.txt
    #bbb
    lpl
    ## 在#bbb 行前insert lol
    $ sed -i '/bbb/i lol'  /tmp/a.txt
    $ cat /tmp/a.txt
    new a.txt
    lol
    #bbb
    lpl
    ```

    end

  * 在指定行处追加

    ```text
    $ sed -i '2 a lol' /tmp/a.txt
    $ cat /tmp/a.txt
    1
    2
    lol
    3
    4
    $ sed -i '3 i lol' /tmp/a.txt
    $ cat /tmp/a.txt
    1
    2
    lol
    lol
    3
    4
    ## 在1,3 行后，分别插入lol
    $sed -i '1,3 a lol' /tmp/a.txt
    $ cat /tmp/a.txt
    1
    lol
    2
    lol
    3
    lol
    4
    ```

    end

  * sed 指定行替换和运算操作

    ```text
    ## metadata 
    $ cat a.txt 
    >lll
    >111
    >bbb
    >ccc
    >aaa aaa
    >kkk
    >kll
    ## data with handler
    $ cat a.txt
    >1
    lll
    >2
    111
    >3
    bbb
    >4
    ccc
    >5
    aaa aaa
    >6
    kkk
    >7
    kll
    ## sed  command
    $ s=`cat a.txt|wc -l` && echo $s && for (( i=1; i<=$s*2;i=i+2 )) do sed  -i "${i}s/>/>$((i/2+1))\n/" a.txt ;done
    ```

    end

  * end

**set -- 很棒**

The `set` command \(when not setting options\) sets the positional parameters。

```text
$ set a b c
$ echo $1
a
$ echo $2
b
$ echo $3
c
```

The `--` is the standard "don't treat anything following this as an option"

The `"$@"` are all the existing position paramters.

So the sequence

```text
set -- haproxy "$@"
```

Will put the word `haproxy` in front of `$1` `$2` etc.

```text
$ echo $1,$2,$3
a,b,c
$ set -- haproxy "$@"
$ echo $1,$2,$3,$4   
haproxy,a,b,c
```

end

**history in shell script**

0.0 `history`在shell script 中 失效--， 但是`source shell_name.sh` 。

```text
$man bash
...
set 
 ...
 history Enable command history, as described above under HISTORY.  This option is on by default in interactive shells.
 ...

#!/bin/bash
HISTFILE=~/.bash_history
set -o history
```

end

**jq**

格式化数据，`jq` 命令没有时，可以用....`python -m json.tool`

**source command vs bash command**

The source command can be used to load any functions file into the current shell script or a command prompt.

啧啧，在当前shell中执行脚本，运用当前shell的全部参数和选项设置。

当前shell 有什么好处呢？

* 更多的set 和shopt选项开启
* 更多的环境变量

  > ```text
  > ## current shell
  > $ echo $HISTFILE
  > /root/.bash_history
  >
  > ## bash shell
  > $ cat a.sh 
  > #!/bin/bash
  > #HISTFILE=~/.bash_history
  > echo "HISTFILE: ${HISTFILE}"
  > $ bash a.sh
  > HISTFILE: 
  >
  > ## diff current and new shell
  > $ shopt  > /tmp/current.txt
  > $ bash -c  "shopt  > /tmp/new-bash.txt" 
  > $ diff /tmp/new-bash.txt /tmp/current.txt
  > ```
  >
  > end

啧啧，总结

**Sourcing** a script will run the commands in the _current_ shell process.

**Executing** a script will run the commands in a _new_ shell proces.

Both sourcing and executing the script will run the commands in the script line by line, as if you typed those commands by hand line by line.

The differences are:

* When you _execute_ the script you are opening a _new_ shell, type the commands in the new shell, copy the output back to your current shell, then close the new shell. Any changes to environment will take effect only in the new shell and will be lost once the new shell is closed.
* When you _source_ the script you are typing the commands in your _current_ shell. Any changes to the environment will take effect and stay in your current shell.

**function**

全世界的函数都一样，减少重复性的代码。

```text
$ cat test.sh
#!/bin/bash
set -o nounset
set -o errexit

log() {
# local variable
  local prefix="[$(date +%Y/%m/%d\ %H:%M:%S)]: "
  echo  "${prefix} $@" >&2
}

log $@

$ ./test.sh "error" "Not found"
[2019/06/18 10:19:42]:  error Not found
```

end

**\[\] vs \[\[\]\]**

0.0 烦人耶，这两个细微的差别

在使用符号`=~`去匹配正则表达式时，只能使用`[[ ]]`，当使用`>`;或者`<`;判断字符串的ASCII值大小时，如果结合`[]`使用，则必须对`>`或者`>`进行转义。

**Linux SIGNAL 与 trpa（nohup）**

0.0 启动服务记得用`systemctl`或者`service`

**Problem:** 关闭正在运行脚本的Xshel终端导致通过脚本启动tomcat退出。

```text
# cat /home/admin/test.sh 
#!/bin/bash
cd /home/admin/taobao-tomcat-production-7.0.59.3/bin/
./startup.sh
tail -f ../logs/catalina.out
```

不清楚，关闭Xshell终端具体 相当于不可trap的signal。

```text
# ps -ef |grep test.sh
...
admin    13180 13066  0 11:16 pts/2    00:00:00 bash test.sh
<truncate>
# ps -ef |grep tomcat
admin    13198     1  64 11:16 pts/2    00:00:16 //bin/java -Djava.util.logging.config.file=/home/admin/taobao-tomcat-production-7.0.59.3/conf/logging.properties -
<truncate>
# ps -ef |grep tail
admin    13199 13180  0 11:16 pts/2    00:00:00 tail -f ../logs/catalina.out
root     13472 31272  0 11:17 pts/0    00:00:00 grep --color=auto tail
## 关闭终端后再查看tomcat，已经灰飞烟灭了
# ps -ef |grep tomcat
root     13625 31272  0 11:18 pts/0    00:00:00 grep --color=auto tomcat
```

```text
trap [-lp] [[arg] sigspec ...]
 The command arg is to be read and executed when the shell receives signal(s) sigspec. 
```

上面的例子，可以看出`tail` 的父进程还是 `test.sh` ,当关闭终端时，`tail` 随着`bash test.sh` 终止，我可以理解。因为bash有清理子进程的功能。但是`tomcat` 的 `PPID` 已经是 `1` 。你告诉我，它随着`bash test.sh` 被关闭了？？？ **Actions**：同样的方法测试Nginx是不是会被干掉

```text
# ps -ef |grep test.sh
root     13401 13349  0 12:11 pts/0    00:00:00 bash test.sh
root     13452 13436  0 12:11 pts/1    00:00:00 grep --color=auto test.sh
# ps -ef |grep nginx
root     13405     1  0 12:11 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
admin    13406 13405  0 12:11 ?        00:00:00 nginx: worker process
admin    13407 13405  0 12:11 ?        00:00:00 nginx: worker process
admin    13409 13405  0 12:11 ?        00:00:00 nginx: worker process
admin    13410 13405  0 12:11 ?        00:00:00 nginx: worker process
root     13457 13436  0 12:12 pts/1    00:00:00 grep --color=auto nginx
# ps -ef |grep tail
root     13408 13401  0 12:11 pts/0    00:00:00 tail -f /backup/applogs/access.log
root     13459 13436  0 12:12 pts/1    00:00:00 grep --color=auto tail
    
### 关闭运行bash test.sh 的终端。 观察发现nginx进程还在
# ps -ef |grep test.sh
root     13471 13436  0 12:12 pts/1    00:00:00 grep --color=auto test.sh
# ps -ef |grep tail
root     13476 13436  0 12:12 pts/1    00:00:00 grep --color=auto tail
# ps -ef |grep nginx
root     13405     1  0 12:11 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
admin    13406 13405  0 12:11 ?        00:00:00 nginx: worker process
admin    13407 13405  0 12:11 ?        00:00:00 nginx: worker process
admin    13409 13405  0 12:11 ?        00:00:00 nginx: worker process
admin    13410 13405  0 12:11 ?        00:00:00 nginx: worker process
root     13478 13436  0 12:12 pts/1    00:00:00 grep --color=auto nginx
```

测试到底是哪个信号量:

```text
# ps -ef |grep test.sh
root     29140 31272  0 12:15 pts/0    00:00:00 bash -x /home/admin/test.sh
root     29393 29269  0 12:16 pts/1    00:00:00 grep --color=auto test.sh
# kill -9 29140
# ps -ef |grep tomcat
root     29158     1 33 12:15 pts/0  <truncate>
    
ts/1    00:00:00 grep --color=auto test.sh\|tomcat
    
# ps -ef |grep 'test.sh\|tomcat' 
root     31470 31272  0 12:24 pts/0    00:00:00 bash -x /home/admin/test.sh
root     31488     1 99 12:24 pts/0    00:00:05 /usr/bin/java - <truncate>
root     31514 29269  0 12:24 pts/1    00:00:00 grep --color=auto test.sh\|tomcat
# ps -ef |grep 31272
root     31272 31270  0 10:26 pts/0    00:00:00 -bash  
root     31470 31272  0 12:24 pts/0    00:00:00 bash -x /home/admin/test.sh
root     31578 29269  0 12:24 pts/1    00:00:00 grep --color=auto 31272
这个 -bash 有意思
# kill -9 31270  //哈哈哈，神来之笔神来之笔哈哈哈
//另个一终端出现下面
Connection to 192.168.97.67 closed.
# ps -ef |grep 'test.sh\|tomcat' 
root     32592 29269  0 12:28 pts/1    00:00:00 grep --color=auto test.sh\|tomcat
    
再看下面的nginx，啧啧啧，好强啊nginx。
# ps -ef |grep 'test.sh\|nginx'
root     14065 13531  0 12:30 pts/0    00:00:00 bash -x test.sh
root     14068     1  0 12:30 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
admin    14069 14068  0 12:30 ?        00:00:00 nginx: worker process
admin    14071 14068  0 12:30 ?        00:00:00 nginx: worker process
admin    14072 14068  0 12:30 ?        00:00:00 nginx: worker process
admin    14073 14068  0 12:30 ?        00:00:00 nginx: worker process
root     14084 13436  0 12:30 pts/1    00:00:00 grep --color=auto test.sh\|nginx
# kill -9 13531
# ps -ef |grep 'test.sh\|nginx'
root     14068     1  0 12:30 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
admin    14069 14068  0 12:30 ?        00:00:00 nginx: worker process
admin    14071 14068  0 12:30 ?        00:00:00 nginx: worker process
admin    14072 14068  0 12:30 ?        00:00:00 nginx: worker process
admin    14073 14068  0 12:30 ?        00:00:00 nginx: worker process
root     14092 13436  0 12:30 pts/1    00:00:00 grep --color=auto test.sh\|nginx
```

end **Result:** 确定了是Tomcat运行机制的问题 。猜测关闭终端相当于`kill -bash` end **Knowledge:** `-bash` 、`signal` 和`trap` en

```text
#!/bin/bash
# trap1a
trap 'my_exit' SIGINT SIGQUIT SIGKILL   //捕捉到指定的signal后做指定操作覆盖原有操作
count=0
 
my_exit()
{
echo "you hit Ctrl-C/Ctrl-\, now exiting.. hahaha" > /tmp/a.txt
 # cleanp commands here if any
}
 
while :
 do
   sleep 1
   count=$(expr $count + 1)
   echo $count
 done
# man 7 signal
...
       Signal     Value     Action   Comment
       ──────────────────────────────────────────────────────────────────────
       SIGHUP        1       Term    Hangup detected on controlling terminal
                                     or death of controlling process
       SIGINT        2       Term    Interrupt from keyboard
       SIGQUIT       3       Core    Quit from keyboard
       SIGILL        4       Core    Illegal Instruction
       SIGABRT       6       Core    Abort signal from abort(3)
       SIGFPE        8       Core    Floating point exception
       SIGKILL       9       Term    Kill signal
       SIGSEGV      11       Core    Invalid memory reference
       SIGPIPE      13       Term    Broken pipe: write to pipe with no
                                     readers
       SIGALRM      14       Term    Timer signal from alarm(2)
       SIGTERM      15       Term    Termination signal
       SIGUSR1   30,10,16    Term    User-defined signal 1
       SIGUSR2   31,12,17    Term    User-defined signal 2
       SIGCHLD   20,17,18    Ign     Child stopped or terminated
       SIGCONT   19,18,25    Cont    Continue if stopped
       SIGSTOP   17,19,23    Stop    Stop process
       SIGTSTP   18,20,24    Stop    Stop typed at terminal
       SIGTTIN   21,21,26    Stop    Terminal input for background process
       SIGTTOU   22,22,27    Stop    Terminal output for background process
    
 !!!   The signals SIGKILL and SIGSTOP cannot be caught, blocked, or ignored.  !!!
...
<truncate>
```

end

**\#/bin/bash**

0.0这个开头很关键，记得要在Script 中开头一行一定要指定。

虽然，在Linux Terminal下`run.sh`只要有`x`权限哪怕没有`#!/bin/bash` 也可以运行，这是因为linux的终端本身就有一个`login_bash`了。但是放到容器中直接去执行`run.sh`就会报错，因为，容器执行时，没指定`bash`,它根本不知道怎么去解释`run.sh`

```text
$ cat run.sh
java ... *.jar
### Dockerfile ### 
...
## 或CMD
ENTRYPOINT ["/run.sh"]
## docker run ###
ERROR：exec format error

----
因为： docker run Container 时没有指定解析器，容器不知道怎么执行run.sh。 所以是缺个解释器。
```

end

**unary operator expected（空值判断）**

0.0 if 判断时，报错，但不会终止运行、

因為 == 左邊沒有抓到值，或是值為 null

比较前，应增加空值的判断： `if [ "$var" == "" ];`

**curl**

强

* 仅获取url的状态码

  `curl -I -m 10 -k -o /dev/null -s -w %{http_code} https://URL`

* end

dmesg

[https://mp.weixin.qq.com/s?\_\_biz=MzI1OTY2MzMxOQ==&mid=2247487147&idx=2&sn=b061015e665f55b8780790d80c91ad05&chksm=ea743a13dd03b3058e71781b08391dfd910ca72533b06e343761274dee05a257eff8dfe33eab&mpshare=1&scene=1&srcid=&sharer\_sharetime=1575292133418&sharer\_shareid=ce0ca863d3e16eb115e2a7f2b8e57758&key=8803040f3d69bcd6b5df9d5c92c4cdd1068eaac6a1ecceb04983966ff9b08c182a4cdc450ae67a6e378ab04dac0bba951ca8a59d1ff2c652c5e0bd3c65bd0964ebc966d09d82d1a471ca43a5f69d428d&ascene=1&uin=Mjc1NjYwMDY0Mw%3D%3D&devicetype=Windows+10&version=62070158&lang=zh\_CN&pass\_ticket=qcDvTd2c6m7za0ZIS%2FZ3Ak5M0%2FqMfdsGTB4n%2FQDbdRccgj1Xm0lBn5c%2F1HmpchDS](https://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247487147&idx=2&sn=b061015e665f55b8780790d80c91ad05&chksm=ea743a13dd03b3058e71781b08391dfd910ca72533b06e343761274dee05a257eff8dfe33eab&mpshare=1&scene=1&srcid=&sharer_sharetime=1575292133418&sharer_shareid=ce0ca863d3e16eb115e2a7f2b8e57758&key=8803040f3d69bcd6b5df9d5c92c4cdd1068eaac6a1ecceb04983966ff9b08c182a4cdc450ae67a6e378ab04dac0bba951ca8a59d1ff2c652c5e0bd3c65bd0964ebc966d09d82d1a471ca43a5f69d428d&ascene=1&uin=Mjc1NjYwMDY0Mw%3D%3D&devicetype=Windows+10&version=62070158&lang=zh_CN&pass_ticket=qcDvTd2c6m7za0ZIS%2FZ3Ak5M0%2FqMfdsGTB4n%2FQDbdRccgj1Xm0lBn5c%2F1HmpchDS)

**strace**

待补充

**查看内核已加载模块参数**

`cat /sys/module/module_name/parameters/parameter_name`

eg:

```text
#查看当前连接跟踪表大小HASHSIZE
cat /sys/module/nf_conntrack/parameters/hashsize
#400384  

## overlay 参数 
## 这个256 不知道和 overlay2支持的最大层数128 有什么关系--
cat /sys/module/overlay/parameters/redirect_max
256

```

**yum localinstall**

eg： 安装openvswitch

官方rpm地址： [https://cbs.centos.org/koji/buildinfo?buildID=22569](https://cbs.centos.org/koji/buildinfo?buildID=22569)

下载并安装`yum localinstall openvswitch-2.9.0-4.el7.x86_64.rpm`

记得使用，`yum`安装这样会生成对应的unit文件。

如果，用rpm文件，在你没有熟悉openvswitch时，你可能都不会启动它。

**shift**

shift命令用于对参数的移动\(左移\)，通常用于在不知道传入参数个数的情况下依次遍历每个参数然后进行相应处理（常见于Linux中各种程序的启动脚本）。

示例：依次读取输入的参数并打印参数个数：

```text
$ cat shift_test.sh
#!/bin/bash
while [ $# != 0 ]
do
echo "prama is $1,prama size is $#"
shift
done

---- output ---
$ ./shift_test.sh a b c
prama is a,prama size is 3
prama is b,prama size is 2
prama is c,prama size is 1


====
$ cat shift_3.sh
#!/bin/bash
while [ $# != 0 ]
do
echo "prama is $1,prama size is $#"
shift 3
done

$ bash /tmp/a.sh 0 1 2 3 4 5
prama is 0,prama size is 6
prama is 3,prama size is 3

=== 下面这样则会一直无限循环 ===
$ bash /tmp/a.sh 0 1 2 3 4 5 6|head
prama is 0,prama size is 7
prama is 3,prama size is 4
prama is 6,prama size is 1
...
prama is 6,prama size is 1
```

每次运行shift\(不带参数的\),销毁一个参数，后面的参数前移

Shift命令一次移动参数的个数由其所带的参数指定。

**trap\(signal\)**

使用trap最好直接使用signal name 不要使用num.

[https://www.ibm.com/developerworks/cn/aix/library/au-usingtraps/index.html](https://www.ibm.com/developerworks/cn/aix/library/au-usingtraps/index.html)

trap捕捉到shell接收的信号量时，执行对应的操作command。

`trap 'command_list' signals`

> commnad\_list 可以是函数。

常见信号包括：

* `0`: sig是0的话，不会向pid进程发送任何信号，但是仍然会继续检测错误（进程ID或者进程组ID是否存在）
* `SIGHUP`\(1\)：从终端终止或退出正在前台运行的进程
* `SIGINT`\(2\)：从键盘按下 Ctrl-C
* `SIGQUIT`\(3\)：从键盘按下 Ctrl-\
* `SIGTERM`\(15\)：软件终止信号

  0.0 怪不得说 kill -15 没那么"硬"，SIGTERM是通知软件自己终止不是强制

在接收到信号后，可以采取的操作包括：

* 清除文件\(这个很棒，清楚临时文件\)，类似编程语言中的Final 关键字

  假设临时文件的形式为：hold1.$$ 和 hold2.$$

  `trap 'rm /tmp/hold*.$$; exit' SIGNHUP SIGINT SIGQUIT SIGTERM`

* 提示用户是否应当终止脚本
* 忽略该信号
* 进行处理

查看所有信号量

```text
 $ trap -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX

## IBM 文章
这些信号是无法捕捉的，并且始终采取默认操作。SIGKILL 总是会终止进程。
SIGKILL（信号 9）
SIGSTOP （信号 17）
SIGCONT（信号 19）
```

简单的栗子：

```text
$  cat /tmp/b.sh
#!/bin/bash
# trap1
trap 'echo you hit Ctrl-C/Ctrl-\, now exiting..; sleep 3;exit' SIGINT SIGQUIT
count=0

while :
 do
   sleep 1
   count=$(expr $count + 1)
   echo $count
 done
--- 输入 ctrl + c 终止 ---
$ bash /tmp/b.sh
1
2
3
^Cyou hit Ctrl-C/Ctrl-, now exiting..
```

摘选的trap应用：

```text
create_netns_link () {
    mkdir -p /var/run/netns
    if [ ! -e /var/run/netns/"$PID" ]; then
        ln -s /proc/"$PID"/ns/net /var/run/netns/"$PID"
        trap 'delete_netns_link' 0
        for signal in 1 2 3 13 14 15; do
            trap 'delete_netns_link; trap - $signal; kill -$signal $$' $signal
        done
    fi
}
```

end

**kill 0**

从上图DESCRIPTION（man 2 kill）区域的文字可以看出，**kill函数中的形参sig是0的话，那么不会向pid进程发送任何信号，但是仍然会继续检测错误（进程ID或者进程组ID是否存在）**。

**特殊变量**

| $0 | 当前脚本的文件名 |
| :--- | :--- |
| $n | 传递给脚本或函数的参数。n 是一个数字，表示第几个参数。例如，第一个参数是，第二个参数是​2。 |
| $\# | 传递给脚本或函数的参数个数。 |
| $\* | 传递给脚本或函数的所有参数。 |
| $@ | 传递给脚本或函数的所有参数。被双引号\(" "\)包含时，与 $\* 稍有不同，下面将会讲到。 |
| $? | 上个命令的退出状态，或函数的返回值。 |
| $$ | 当前Shell进程ID。对于 Shell 脚本，就是这些脚本所在的进程ID。 |

**利用Harbor API查询image是否存在**

```text
#!/bin/bash
if [ $# -ne 1 ]
then
   echo "args number is : $#"
   echo "please input image name"
   exit 1
fi
token=$(curl  -ksi -L -u admin:Harbor12345 https://21.49.22.250/service/token?account=admin\&service=harbor-registry\&scop
e=registry:catalog:\*|grep token|gawk -F "\"" '{print $4}')curl  -sk -H "Content-Type: application/json" -H "Authorization: Bearer $token"  -X GET https://21.49.22.250/v2/_catalog|p
ython -m "json.tool"|grep $1
```

end

**shopt 真的强**

[https://www.soosmart.com/topic/681.html](https://www.soosmart.com/topic/681.html)

```text
$ man bash
shopt_option  is one of the shell options accepted by the shopt builtin (see SHELL BUILTIN COMMANDS below). 

shopt 选项 参数

选项 
-s：激活指定的shell行为选项；
-u：关闭指定的shell行为选项。

## login_shell
$ shopt |grep on 
checkwinsize   	on
cmdhist        	on
complete_fullquote	on
expand_aliases 	on
extglob        	on
extquote       	on
force_fignore  	on
huponexit      	off
interactive_comments	on
login_shell    	on
no_empty_cmd_completion	off
progcomp       	on
promptvars     	on
sourcepath     	on

## sub-bash
$ bash -c 'shopt |grep login_shell'
login_shell    	off
$ bash -c 'shopt |grep expand_aliase'
expand_aliases 	off

## 所以脚本中默认不支持别名
$ cat /tmp/a.sh 
alias a='ls -l /tmp/test.py'
a
$ bash /tmp/a.sh
/tmp/a.sh: line 3: a: command not found

## 通过shopt开启expand_aliases
$ cat /tmp/a.sh 
#!/bin/bash
shopt -s expand_aliases
alias a='ls -l /tmp/test.py'
a

$ bash /tmp/a.sh
-rw-r--r-- 1 root root 62 Dec 27 16:35 /tmp/test.py


Bash Shell有个extglob选项，开启之后Shell可以另外识别出5个模式匹配操作符，能使文件匹配更加方便。 

开启方法很简单，使用shopt命令：shopt -s extglob 
关闭，使用shopt命令：shopt -u extglob 

开启之后，以下5个模式匹配操作符将被识别： 

?(pattern-list) - 所给模式匹配0次或1次；
*(pattern-list) - 所给模式匹配0次以上包括0次；
+(pattern-list) - 所给模式匹配1次以上包括1次；
@(pattern-list) - 所给模式仅仅匹配1次；
!(pattern-list) - 不匹配括号内的所给模式。 
```

end

**lol**

**小脚本**

```text
#!/bin/bash

function reserved_mem()
{
   r_mem=`ssh $1 free -m | awk '/Mem/{sum=$4+$6}END{printf "%d\n",sum/1024}'`
   if [ ${r_mem} -lt 8 ] 
   then 
      echo "[WARN] on $2 free memory only left ${r_mem} GB."
   else
      echo "[INFO] on $2 free memory left ${r_mem} GB."
   fi
}


function used_disk()
{
   used_disk=`ssh $1 df -h | awk '/dev.*vda1/{print $5}' | egrep -o "[[:digit:]]{1,3}"`
   if [ ${used_disk} -gt 80 ] 
   then 
      echo "[WARN] on $2, root partition ratio is grater than 80%, please expend disk."
   else
      echo "[INFO] on $2, root partition is ok"
   fi
}



function user_cpu()
{

   max_core_used=`ssh $1 sar -P ALL 1 2|grep Average|grep -v CPU|awk '{printf "%d\n",$3}'|sort -rn|head -n 1`
   if [ ${max_core_used} -gt 95 ]
   then
      echo "[WARN] on $2, some cpu core  ratio is grater than 95%, please check."
   else
      echo "[INFO] on $2, each of cpu cores is ok"
   fi


}

function user_cpu()
{

   max_core_used=`ssh $1 sar -P ALL 1 2|grep Average|grep -v CPU|awk '{printf "%d\n",$3}'|sort -rn|head -n 1`
   if [ ${max_core_used} -gt 95 ]
   then
      echo "[WARN] on $2, some cpu core  ratio is grater than 95%, please check."
   else
      echo "[INFO] on $2, each of cpu cores is ok"
   fi


}


function up_time()
{

   up_time=`ssh $1 uptime | awk -F "days" '{print $1}' | awk -F " " '{print $3}'`
   if [ ${up_time} -le 0 ]
   then
      echo "[WARN] $2 uptime days ${up_time} This node has recently been restarted"
   else
      echo "[INFO] $2 this node uptime days is ${up_time}"
   fi

}

function k8s_stat()
{
   bad_pod=`kubectl get pod --all-namespaces -owide |grep -v Running |grep -v NAME|wc -l`
   bad_node=`kubectl get node|grep -v Ready|grep -v NAME|wc -l`
   if [ $bad_pod -gt 0 ]
   then
      echo "[WARN] on k8s-prod, some pod is unhealthy, please check."
   else
      echo "[INFO] on k8s-prod, each of pods is ok"
   fi
   if [ $bad_node -gt 0 ]
   then
      echo "[WARN] on k8s-prod, some node is unhealthy, please check."
   else
      echo "[INFO] on k8s-prod, each of nodes is ok"
   fi
}


function tce_cpu()
{

   tce_1_used=`ssh 21.48.0.15 sar -P ALL 1 5|grep Average|grep -v CPU|awk '{printf "%d\n",$3}'|sort -rn|head -n 1`
   if [ ${tce_1_used} -gt 95 ]
   then
      echo "[WARN] on tce-1, some cpu core  ratio is grater than 95%, please check."
   else
      echo "[INFO] on tce-1, each of cpu cores is ok"
   fi
   tce_2_used=`ssh 21.48.0.16 sar -P ALL 1 5|grep Average|grep -v CPU|awk '{printf "%d\n",$3}'|sort -rn|head -n 1`
   if [ ${tce_2_used} -gt 95 ]
   then
      echo "[WARN] on tce-2, some cpu core  ratio is grater than 95%, please check."
   else
      echo "[INFO] on tce-2, each of cpu cores is ok"
   fi


}

function unexpected()
{
   used_disk=`ssh k8s-node-1 df -h | awk '/dev.*vdb/{print $5}' | egrep -o "[[:digit:]]{1,3}"`
   if [ ${used_disk} -gt 85 ] 
   then 
      echo "[WARN] on k8s-node-1, root partition ratio is grater than 80%, please expend disk."
   else
      echo "[INFO] on k8s-node-1, root partition is ok"
   fi
  
}


echo "-----------------------start inspection----------------------"
for i in {41..74}
do 
  if [ $i -ne 59 ]
  then 
    ip=21.48.0.$i
    hostname=`ssh 21.48.0.${i} hostname`
    reserved_mem ${ip} ${hostname}
    used_disk ${ip} ${hostname}
    user_cpu ${ip} ${hostname}
    up_time ${ip} ${hostname}
  fi
done

unexpected
tce_cpu
k8s_stat
```

end

**ovs-docker脚本**

```text
#!/bin/bash
# Copyright (C) 2014 Nicira, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at:
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Check for programs we'll need.
# IFS是internal field separator的缩写 Default IFS is 空格、等空白字符
search_path () {
    save_IFS=$IFS
    IFS=:
    for dir in $PATH; do
        IFS=$save_IFS
        if test -x "$dir/$1"; then
            return 0
        fi
    done
    IFS=$save_IFS
    echo >&2 "$0: $1 not found in \$PATH, please install and try again"
    exit 1
}

## 类似alias 
## 这里可以用shopt来开启alias 即
## shopt -s expand_aliases
## alias ovs-vsctl="ovs-vsctl --timeout=60" 
ovs_vsctl () {
    ovs-vsctl --timeout=60 "$@"
}

## ln -s /var/run/docker/netns /var/run/netns
## man ip netns
## ip-netns - process network namespace management
## By convention a named network namespace is an object at /var/run/netns/NAME that can be opened. 
## 呐，默认ip netns操作的目录是/var/run/netns
create_netns_link () {
    mkdir -p /var/run/netns
    if [ ! -e /var/run/netns/"$PID" ]; then
        ln -s /proc/"$PID"/ns/net /var/run/netns/"$PID"
        ## 下面这些全部都是 防止意外出错时记得删掉 ns link.
        trap 'delete_netns_link' 0
        for signal in 1 2 3 13 14 15; do
            ## 当前Shell进程ID。对于 Shell 脚本，就是这些脚本所在的进程ID。
            trap 'delete_netns_link; trap - $signal; kill -$signal $$' $signal
        done
    fi
}

delete_netns_link () {
    rm -f /var/run/netns/"$PID"
}

get_port_for_container_interface () {
    CONTAINER="$1"
    INTERFACE="$2"

    PORT=`ovs_vsctl --data=bare --no-heading --columns=name find interface \
             external_ids:container_id="$CONTAINER"  \
             external_ids:container_iface="$INTERFACE"`
    if [ -z "$PORT" ]; then
        echo >&2 "$UTIL: Failed to find any attached port" \
                 "for CONTAINER=$CONTAINER and INTERFACE=$INTERFACE"
    fi
    echo "$PORT"
}

# eg: ovs-docker add-port vswitch0 eth0 vm01 --ipaddress=1.1.1.2/24
# 核心思想： container network with  none mode，
# 创建veth pair ，一段接入ovs，另一端放到容器中并改名为eth0
add_port () {
    BRIDGE="$1"
    INTERFACE="$2"
    CONTAINER="$3"

    if [ -z "$BRIDGE" ] || [ -z "$INTERFACE" ] || [ -z "$CONTAINER" ]; then
        echo >&2 "$UTIL add-port: not enough arguments (use --help for help)"
        exit 1
    fi

    # shift 3 后第一个参数由add-port 跳到 --ipaddress
    shift 3
    # eg: ovs-docker add-port vswitch0 eth0 vm01 --ipaddress=1.1.1.2/24
    while [ $# -ne 0 ]; do
        case $1 in
        # expr  "--ipaddress=1.1.1.2/24" : '[^=]*=\(.*\)'
        # 取等号后面的pattern 1 
            --ipaddress=*)
                ADDRESS=`expr X"$1" : 'X[^=]*=\(.*\)'`
                shift
                ;;
            --macaddress=*)
                MACADDRESS=`expr X"$1" : 'X[^=]*=\(.*\)'`
                shift
                ;;
            --gateway=*)
                GATEWAY=`expr X"$1" : 'X[^=]*=\(.*\)'`
                shift
                ;;
            --mtu=*)
                MTU=`expr X"$1" : 'X[^=]*=\(.*\)'`
                shift
                ;;
            *)
                echo >&2 "$UTIL add-port: unknown option \"$1\""
                exit 1
                ;;
        esac
    done

    # Check if a port is already attached for the given container and interface
    PORT=`get_port_for_container_interface "$CONTAINER" "$INTERFACE" \
            2>/dev/null`
    if [ -n "$PORT" ]; then
        echo >&2 "$UTIL: Port already attached" \
                 "for CONTAINER=$CONTAINER and INTERFACE=$INTERFACE"
        exit 1
    fi

    if ovs_vsctl br-exists "$BRIDGE" || \
        ovs_vsctl add-br "$BRIDGE"; then :; else
        echo >&2 "$UTIL: Failed to create bridge $BRIDGE"
        exit 1
    fi

    if PID=`docker inspect -f '{{.State.Pid}}' "$CONTAINER"`; then :; else
        echo >&2 "$UTIL: Failed to get the PID of the container"
        exit 1
    fi

    create_netns_link

    # Create a veth pair.
    ID=`uuidgen | sed 's/-//g'`
    PORTNAME="${ID:0:13}"
    ip link add "${PORTNAME}_l" type veth peer name "${PORTNAME}_c"

    # Add one end of veth to OVS bridge.
    # Integration centers around the Open vSwitch database and mostly involves
	#the 'external_ids' columns in several of the tables.  These columns are
	#not interpreted by Open vSwitch itself.  Instead, they provide
	#information to a controller that permits it to associate a database
	#record with a more meaningful entity.  In contrast, the 'other_config'
	#column is used to configure behavior of the switch.  The main job of the
	#integrator, then, is to ensure that these values are correctly populated
	#and maintained.
	# 0.0 骚操作
	# ovsdb-client dump 可以看到openvswitch tables的describe 里面有 external_ids column
    if ovs_vsctl --may-exist add-port "$BRIDGE" "${PORTNAME}_l" \
       -- set interface "${PORTNAME}_l" \
       external_ids:container_id="$CONTAINER" \
       external_ids:container_iface="$INTERFACE"; then :; else
        echo >&2 "$UTIL: Failed to add "${PORTNAME}_l" port to bridge $BRIDGE"
        ip link delete "${PORTNAME}_l"
        exit 1
    fi

    ip link set "${PORTNAME}_l" up

    # Move "${PORTNAME}_c" inside the container and changes its name.
    ip link set "${PORTNAME}_c" netns "$PID"
    ip netns exec "$PID" ip link set dev "${PORTNAME}_c" name "$INTERFACE"
    ip netns exec "$PID" ip link set "$INTERFACE" up

    if [ -n "$MTU" ]; then
        ip netns exec "$PID" ip link set dev "$INTERFACE" mtu "$MTU"
    fi

    if [ -n "$ADDRESS" ]; then
        ip netns exec "$PID" ip addr add "$ADDRESS" dev "$INTERFACE"
    fi

    if [ -n "$MACADDRESS" ]; then
        ip netns exec "$PID" ip link set dev "$INTERFACE" address "$MACADDRESS"
    fi

    if [ -n "$GATEWAY" ]; then
        ip netns exec "$PID" ip route add default via "$GATEWAY"
    fi
}

del_port () {
    BRIDGE="$1"
    INTERFACE="$2"
    CONTAINER="$3"

    if [ "$#" -lt 3 ]; then
        usage
        exit 1
    fi

    PORT=`get_port_for_container_interface "$CONTAINER" "$INTERFACE"`
    if [ -z "$PORT" ]; then
        exit 1
    fi

    ovs_vsctl --if-exists del-port "$PORT"

    ip link delete "$PORT"
}

del_ports () {
    BRIDGE="$1"
    CONTAINER="$2"
    if [ "$#" -lt 2 ]; then
        usage
        exit 1
    fi

    PORTS=`ovs_vsctl --data=bare --no-heading --columns=name find interface \
             external_ids:container_id="$CONTAINER"`
    if [ -z "$PORTS" ]; then
        exit 0
    fi

    for PORT in $PORTS; do
        ovs_vsctl --if-exists del-port "$PORT"
        ip link delete "$PORT"
    done
}

set_vlan () {
    BRIDGE="$1"
    INTERFACE="$2"
    CONTAINER_ID="$3"
    VLAN="$4"

    if [ "$#" -lt 4 ]; then
        usage
        exit 1
    fi

    PORT=`get_port_for_container_interface "$CONTAINER_ID" "$INTERFACE"`
    if [ -z "$PORT" ]; then
        exit 1
    fi
    ovs_vsctl set port "$PORT" tag="$VLAN"
}

usage() {
    cat << EOF
${UTIL}: Performs integration of Open vSwitch with Docker.
usage: ${UTIL} COMMAND

Commands:
  add-port BRIDGE INTERFACE CONTAINER [--ipaddress="ADDRESS"]
                    [--gateway=GATEWAY] [--macaddress="MACADDRESS"]
                    [--mtu=MTU]
                    Adds INTERFACE inside CONTAINER and connects it as a port
                    in Open vSwitch BRIDGE. Optionally, sets ADDRESS on
                    INTERFACE. ADDRESS can include a '/' to represent network
                    prefix length. Optionally, sets a GATEWAY, MACADDRESS
                    and MTU.  e.g.:
                    ${UTIL} add-port br-int eth1 c474a0e2830e
                    --ipaddress=192.168.1.2/24 --gateway=192.168.1.1
                    --macaddress="a2:c3:0d:49:7f:f8" --mtu=1450
  del-port BRIDGE INTERFACE CONTAINER
                    Deletes INTERFACE inside CONTAINER and removes its
                    connection to Open vSwitch BRIDGE. e.g.:
                    ${UTIL} del-port br-int eth1 c474a0e2830e
  del-ports BRIDGE CONTAINER
                    Removes all Open vSwitch interfaces from CONTAINER. e.g.:
                    ${UTIL} del-ports br-int c474a0e2830e
  set-vlan BRIDGE INTERFACE CONTAINER VLAN
                    Configures the INTERFACE of CONTAINER attached to BRIDGE
                    to become an access port of VLAN. e.g.:
                    ${UTIL} set-vlan br-int eth1 c474a0e2830e 5
Options:
  -h, --help        display this help message.
EOF
}

UTIL=$(basename $0)
search_path ovs-vsctl
search_path docker
search_path uuidgen

if (ip netns) > /dev/null 2>&1; then :; else
    echo >&2 "$UTIL: ip utility not found (or it does not support netns),"\
             "cannot proceed"
    exit 1
fi

if [ $# -eq 0 ]; then
    usage
    exit 0
fi

case $1 in
    "add-port")
        shift
        add_port "$@"
        exit 0
        ;;
    "del-port")
        shift
        del_port "$@"
        exit 0
        ;;
    "del-ports")
        shift
        del_ports "$@"
        exit 0
        ;;
    "set-vlan")
        shift
        set_vlan "$@"
        exit 0
        ;;
    -h | --help)
        usage
        exit 0
        ;;
    *)
        echo >&2 "$UTIL: unknown command \"$1\" (use --help for help)"
        exit 1
        ;;
esac
```

end  


