# nginx技能点汇总

## 常用正则
```text
. ： 匹配除换行符以外的任意字符
? ： 重复0次或1次
+ ： 重复1次或更多次
* ： 重复0次或更多次
\d ：匹配数字
^ ： 匹配字符串的开始
$ ： 匹配字符串的结束
{n} ： 重复n次
{n,} ： 重复n次或更多次
[c] ： 匹配单个字符c
[a-z] ： 匹配a-z小写字母的任意一个
```

## 全局变量
```text
$args： #这个变量等于请求行中的参数，同$query_string

$content_length： 请求头中的Content-length字段。

$content_type： 请求头中的Content-Type字段。

$document_root： 当前请求在root指令中指定的值。

$host： 请求主机头字段，否则为服务器名称。

$http_user_agent： 客户端agent信息

$http_cookie： 客户端cookie信息

$limit_rate： 这个变量可以限制连接速率。

$request_method： 客户端请求的动作，通常为GET或POST。

$remote_addr： 客户端的IP地址。

$remote_port： 客户端的端口。

$remote_user： 已经经过Auth Basic Module验证的用户名。

$request_filename： 当前请求的文件路径，由root或alias指令与URI请求生成。

$scheme： HTTP方法（如http，https）。

$server_protocol： 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。

$server_addr： 服务器地址，在完成一次系统调用后可以确定这个值。

$server_name： 服务器名称。

$server_port： 请求到达服务器的端口号。

$request_uri： 包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。

$uri： 不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”。

$document_uri： 与$uri相同。
```
```
  例：http://localhost:88/test1/test2/test.php
  $host：localhost
  $server_port：88
  $request_uri：http://localhost:88/test1/test2/test.php
  $document_uri：/test1/test2/test.php
  $document_root：/var/www/html
  $request_filename：/var/www/html/test1/test2/test.php
```

## if语句块

### if判断指令
```text
语法为if(condition){...}，对给定的条件condition进行判断。如果为真，大括号内的rewrite指令将被执行，if条件(conditon)可以是如下任何内容：

当表达式只是一个变量时，如果值为空或任何以0开头的字符串都会当做false
直接比较变量和内容时，使用=或!=
~正则表达式匹配，~*不区分大小写的匹配，!~区分大小写的不匹配
-f和!-f用来判断是否存在文件
-d和!-d用来判断是否存在目录
-e和!-e用来判断是否存在文件或目录
-x和!-x用来判断文件是否可执行
```

### 例如：
```
if ($http_user_agent ~ MSIE) {
    rewrite ^(.*)$ /msie/$1 break;
} //如果UA包含"MSIE"，rewrite请求到/msid/目录下

if ($http_cookie ~* "id=([^;]+)(?:;|$)") {
    set $id $1;
 } //如果cookie匹配正则，设置变量$id等于正则引用部分

if ($request_method = POST) {
    return 405;
} //如果提交方法为POST，则返回状态405（Method not allowed）。return不能返回301,302

if ($slow) {
    limit_rate 10k;
} //限速，$slow可以通过 set 指令设置

if (!-f $request_filename){
    break;
    proxy_pass  http://127.0.0.1;
} //如果请求的文件名不存在，则反向代理到localhost 。这里的break也是停止rewrite检查

if ($args ~ post=140){
    rewrite ^ http://example.com/ permanent;
} //如果query string中包含"post=140"，永久重定向到example.com

location ~* \.(gif|jpg|png|swf|flv)$ {
    valid_referers none blocked www.jefflei.com www.leizhenfang.com;
    if ($invalid_referer) {
        return 404;
    } //防盗链
}
```

## flag标志位

```
last: 相当于Apache的[L]标记，表示完成rewrite
break: 停止执行当前虚拟主机的后续rewrite指令集
redirect: 返回302临时重定向，地址栏会显示跳转后的地址
permanent: 返回301永久重定向，地址栏会显示跳转后的地址
因为301和302不能简单的只返回状态码，还必须有重定向的URL，这就是return指令无法返回301,302的原因了。这里 last 和 break 区别有点难以理解：

last一般写在server和if中，而break一般使用在location中
last不终止重写后的url匹配，即新的url会再从server走一遍匹配流程，而break终止重写后的匹配
break和last都能组织继续执行后面的rewrite指令
```

## 隐藏ngxin版本号
```
当前使用的nginx可能会有未知的漏洞，如果被黑客使用将会造成无法估量的损失，但是我们可以将nginx的版本隐藏，如下：

server_tokens off; #在http 模块当中配置
```

## Rewrite规则

> rewrite功能就是，使用nginx提供的全局变量或自己设置的变量，结合正则表达式和标志位实现url重写以及重定向。rewrite只能放在server{},location{},if{}中，并且只能对域名后边的除去传递的参数外的字符串起作用

例如http://seanlook.com/a/we/index.php?id=1&u=str
只对/a/we/index.php重写。语法rewrite regex replacement [flag];

如果相对域名或参数字符串起作用，可以使用全局变量匹配，也可以使用proxy_pass反向代理。

表明看rewrite和location功能有点像，都能实现跳转，主要区别在于rewrite是在同一域名内更改获取资源的路径，而location是对一类路径做控制访问或反向代理，可以proxy_pass到其他机器。很多情况下rewrite也会写在location里，它们的执行顺序是：

执行server块的rewrite指令
执行location匹配
执行选定的location中的rewrite指令
如果其中某步URI被重写，则重新循环执行1-3，直到找到真实存在的文件；循环超过10次，则返回500 Internal Server Error错误。

### write实例
```
http {
    # 定义image日志格式
    log_format imagelog '[$time_local] ' $image_file ' ' $image_type ' ' $body_bytes_sent ' ' $status;
    # 开启重写日志
    rewrite_log on;

    server {
        root /home/www;

        location / {
                # 重写规则信息
                error_log logs/rewrite.log notice;
                # 注意这里要用‘’单引号引起来，避免{}
                rewrite '^/images/([a-z]{2})/([a-z0-9]{5})/(.*)\.(png|jpg|gif)$' /data?file=$3.$4;
                # 注意不能在上面这条规则后面加上“last”参数，否则下面的set指令不会执行
                set $image_file $3;
                set $image_type $4;
        }

        location /data {
                # 指定针对图片的日志格式，来分析图片类型和大小
                access_log logs/images.log mian;
                root /data/images;
                # 应用前面定义的变量。判断首先文件在不在，不在再判断目录在不在，如果还不在就跳转到最后一个url里
                try_files /$arg_file /image404.html;
        }
        location = /image404.html {
                # 图片不存在返回特定的信息
                return 404 "image not found\n";
        }
}
```

## 错误码原因和解决方案

### 400 bad request
```
错误的原因和解决办法 配置nginx.conf相关设置如下.

client_header_buffer_size 16k;

large_client_header_buffers 4 64k;

根据具体情况调整，一般适当调整值就可以。
```
### Nginx 502 Bad Gateway错误
```
proxy_next_upstream error timeout invalid_header http_500 http_503;

或者尝试设置:

large_client_header_buffers 4 32k;
```

### Nginx出现的413 Request Entity Too Large错误
```
这个错误一般在上传文件的时候会出现，

编辑Nginx主配置文件Nginx.conf，找到http{}段，添加

client_max_body_size 10m; //设置多大根据自己的需求作调整.

如果运行php的话这个大小client_max_body_size要和php.ini中的如下值的最大值一致或者稍大，这样就不会因为提交数据大小不一致出现的错误。

post_max_size = 10M

upload_max_filesize = 2M
```

### 解决504 Gateway Time-out(nginx)
```
遇到这个问题是在升级discuz论坛的时候遇到的

一般看来, 这种情况可能是由于nginx默认的fastcgi进程响应的缓冲区太小造成的, 这将导致fastcgi进程被挂起, 如果你的fastcgi服务对这个挂起处理的不好, 那么最后就极有可能导致504 Gateway Time-out

现在的网站, 尤其某些论坛有大量的回复和很多内容的, 一个页面甚至有几百K。

默认的fastcgi进程响应的缓冲区是8K, 我们可以设置大点

在nginx.conf里, 加入： fastcgi_buffers 8 128k

这表示设置fastcgi缓冲区为8×128k

当然如果您在进行某一项即时的操作, 可能需要nginx的超时参数调大点，例如设置成60秒：send_timeout 60;

只是调整了这两个参数, 结果就是没有再显示那个超时, 可以说效果不错, 但是也可能是由于其他的原因, 目前关于nginx的资料不是很多, 很多事情都需要长期的经验累计才有结果.
```

## 打开目录浏览功能

```
Nginx默认是不允许列出整个目录的。如需此功能，打开nginx.conf文件，在location server 或 http段中加入

autoindex on;  
另外两个参数最好也加上去:

autoindex\_exact\_size off;  
默认为on，显示出文件的确切大小，单位是bytes。
改为off后，显示出文件的大概大小，单位是kB或者MB或者GB

autoindex\_localtime on;  
默认为off，显示的文件时间为GMT时间。
改为on后，显示的文件时间为文件的服务器时间
```

## 显示乱码
```
server {
  listen 80;
  server_name example.com;
  root /var/www/example;

  location / {
    charset utf-8; #一般是在个别的location中加入此项，具体情况具体对待
    rewrite .* /index.html break;
  }
}
```

## 配置nginx worker进程最大打开文件数

```
worker_rlimit_nofile 65535;
```

## 单个工作进程的最大连接数
> 通过worker_connections number；进行设置，numebr为整数，number的值不能大于操作系统能打开的最大的文件句柄数，使用ulimit -n可以查看当前操作系统支持的最大文件句柄数，默认为为1024.
  
```
events {
    worker_connections  102400; #设置单个工作进程最大连接数102400
}
```

## 灰度发布

### 根据ip实现灰度发布
> 在百度查自己公司的公网IP
### 原理
> 同时把两个不同版本的代码拉成两个项目，根据ip来判断用户可以去哪个项目，灰度发布的项目目录指向高版本的项目，其他ip的所有用户仍然访问相对的低版本的项目。
  
### nginx配置
```
server {
    listen 80;
      
    server_name  mb.com;

        gzip on;
    charset utf-8;
  

	set $mulu  /var/www/mb/dist ;
	 
	if ($remote_addr = 1.2.3.4) {
		set $mulu  /var/www/mr/build; 
	 } 

    location / { 
        root $mulu;    
        index  index.html;
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```
