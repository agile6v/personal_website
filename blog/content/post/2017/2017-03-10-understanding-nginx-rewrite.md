---
categories:
    - "articles"
tags:
    - "Nginx"
date: 2017-03-10
title: " Understanding Nginx Rewrite"
url: "/2017/03/10/understanding-nginx-rewrite"
---



网站在使用Nginx时都会进行个性化配置满足自己的业务需要，而URL重写几乎是每个网站都必做的事情，Nginx的URL重写规则不像Apache那样简单直接，逻辑相对要复杂一些，本文将通过例子的方式帮助大家理解Nginx rewrite原理，希望能对您有些启发。

<!--more-->

Nginx中重定向有多种方式，例如：

**外部重定向** 使用return指令返回301或302（return也可以返回其他状态码），可以放在server块中，也可以放在location块中。例如：

```
return 301 https://www.mi.com;
or
return 302 https://www.mi.com;
```

还可以使用rewrite指令，例如：

```
rewrite ^/(.*)$   http://www.mi.com/$1;
or 
rewrite ^/(.*)$   http://www.mi.com/ redirect;

```

**内部重定向** 使用return + error_page指令的组合，try_files指令和rewrite指令，非常灵活。

本文主要讲解rewrite的工作原理，其他指令的使用方法大家可以自行查阅Nginx官网。在使用Nginx的rewrite指令时，flag可以设置为last和break，这两个flag很容易混淆，后面我们会比较这两个flag的区别，下面通过示例我们来认识一下rewrite指令。

#### rewrite语法

```
Syntax:    rewrite regex replacement [flag];
Context:   server, location, if
```

* regex：			对请求的URI做正则匹配
* replacement：目标uri匹配成功后替换的url
* 可以使用的flag有以下4个（flag可以为空）:
  * redirect：返回302临时重定向，客户端地址栏会显示跳转后的地址
  * permanent：返回301永久重定向，客户端地址栏会显示跳转后的地址
  * last：内部重定向，停止处理后续rewrite模块中的指令（区别于break）
  * break：内部重定向，停止处理后续rewrite模块中的指令（区别于last)

NOTE:

1. regex匹配的是uri，不包含hostname和query string，默认query string是被追加到replacement末尾，如果不希望在replacement末尾追加请求的query string，可以在replacement的末尾加一个"?"。

```
server {
 	rewrite ^/mi_one/(.*)$  /mi/$1;
 	rewrite ^/mi_two/(.*)$  /mi/$1?x=0&y=1?;
 	
 	location /mi/ {
    	echo $uri$is_args$args;
 	}
}

$ curl "http://localhost/mi_one/hello?a=1&b=2"
return: /mi/hello?a=1&b=2

$ curl "http://localhost/mi_two/hello?a=1&b=2"
return: /mi/hello?x=0&y=1
```
2. 如果replacement是以"http://"，"https://" 或 $scheme"开始的字符串，那么rewrite指令停止后面的处理，直接返回给客户端，没有指定flag时，与redirect效果相同
3. 没有flag的rewrite指令根据出现的顺序执行，flag可以控制指令的执行顺序
4. 在配置中开启rewrite_log指令，日志文件中会记录rewrite的匹配过程，有助于调试rewrite问题

#### Nginx请求处理流程

在讲rewrite前我们先来简单了解下Nginx请求处理流程，为什么需要了解请求处理流程呢？因为rewrite操作与其中几个phase关系很密切，熟悉了请求处理流程，理解rewrite执行逻辑就会很容易。在Nginx内部将请求处理划分为11个phase，每个phase会执行对应的handler，这里我们不打算逐个进行讲解。在11个phase中与rewrite指令逻辑有关的只有4个，所以在本文我们主要关注SERVER_REWRITE、FIND_CONFIG、REWRITE和POST_REWRITE这四个phase。

![img](/img/nginx_phase.png)

首先我们要清楚的是：

* server块中的rewrite模块指令在SERVER_REWRITE阶段解析；
* location块中的rewrite模块指令在REWRITE阶段解析；

1. SERVER_REWRITE - 请求到达后首先处理这个阶段的rewrite指令操作
2. FIND_CONFIG - 根据上一个阶段的得到的uri查找location
3. REWRITE - 确定location后执行locaton中rewrite操作
4. POST_REWRITE - 根据上一阶段的uri重写结果选择phase，可以跳回FIND_CONFIG阶段重新查找location，也可以继续执行后边的phase。例如：在location中配置了rewrite指令并且指定flag=break，执行完本条rewrite终止后边的rewrite匹配，然后执行PREACCESS阶段中的handler。同样的场景下flag=last，执行完本条rewrite终止后边的rewrite匹配，然后跳到FIND_CONFIG阶段再次查找location。未指定flag的情况与flag=last类似，唯一区别是在同一层级中未指定flag的rewrite语句不会终止后续的rewrite匹配。

#### 通过例子理解rewrite指令

##### 1. 未指定flag

未指定flag的rewrite会按照出现顺序进行匹配，server块中rewrite匹配完根据改写的uri查找location，然后再匹配location中的rewrite，location中的rewrite指令匹配成功后会再次查找location。

```
server {
    rewrite ^/(.*)$         /mi_one/$1;
    rewrite ^/mi_one/(.*)$  /mi_two/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }
}

$ curl http://localhost/mi_zero/hello
This is mi_two location

```

说明：

1. 匹配第一条rewrite成功，uri被改写为/mi_one/mi_zero/hello，没有指定flag的rewrite继续匹配后面的rewrite
2. 匹配第二条rewrite成功，此时uri被改写为/mi_two/mi_zero/hello
3. 查找location，mi_two被确定为最终的location


```
server {
    rewrite ^/(.*)$         /mi_one/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        rewrite ^/mi_one/(.*)$  /mi_two/$1;
        
        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }
}

$ curl http://localhost/hello
This is mi_two location

```

说明：

1. 匹配server块中的rewrite成功，uri被改写为/mi_one/hello
2. server块中只有一条rewrite指令，开始查找location
3. location mi_one被找到，开始匹配location中rewrite
4. location中的rewrite匹配成功，uri被改写为/mi_two/hello
5. 再次查找location，mi_two被确定为最终使用的location

再来看一个例子：

```
server {
    rewrite ^/(.*)$         http://www.mi.com/$1;
    rewrite ^/mi_one/(.*)$  /mi_two/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }
}

$ curl -I http://localhost/mi_one/hello
http code: 302
Location: http://www.mi.com/mi_one/hello

```

说明：

1. 匹配第一条rewrite成功，由于replacement是以http://开始的字符串，所以rewrite指令直接返回给客户端302，并且停止匹配后续的rewrite

##### 2. flag指定为redirect

指定flag为redirect时，第一条rewrite匹配成功后直接返回给客户端302，不会继续匹配后续的rewrite。

```
server {
    rewrite ^/(.*)$         /mi_one/$1 redirect;
    rewrite ^/mi_one/(.*)$  /mi_two/$1;
    rewrite ^/mi_two/(.*)$  /mi_three/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }

    location /mi_three/ {
        echo "This is mi_three location";
    }
}

$ curl -I http://localhost/mi_zero/hello
http code: 302
Location: http://localhost/mi_one/mi_zero/hello

$ curl -I http://localhost/mi_one/hello
http code: 302
Location: http://localhost/mi_one/mi_one/hello

$ curl -I http://localhost/mi_two/hello
http code: 302
Location: http://localhost/mi_one/mi_two/hello

```

再来看一个例子：

```
server {
    rewrite ^/mi_one/(.*)$  /mi_two/$1;
    rewrite ^/mi_two/(.*)$  /mi_three/$1 redirect;
    rewrite ^/(.*)$         /mi_one/$1;
    
    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }

    location /mi_three/ {
        echo "This is mi_three location";
    }
}

$ curl -I http://localhost/mi_one/hello
http code: 302
Location: http://localhost/mi_three/hello

$ curl -I http://localhost/mi_two/hello
http code: 302
Location: http://localhost/mi_three/hello

$ curl http://localhost/mi_zero/hello
This is mi_one location

```

##### 3. flag指定为permanent
指定flag=permanent时，与redirect效果相同，唯一的区别http code返回301。

##### 4. flag指定为last
rewrite的last和break这两个flag使用场景很多并且也很容易混淆，他们的共同点都会停止当前层级后续的rewrite匹配，区别需要分两种情况：第一种使用在server block中，last和break没有区别。第二种使用在location block中，last会根据改写的uri重新查找location，break不会重新查找location，而是在当前location中执行后续的指令。（注：last和break不仅停止rewrite的匹配，同时还会停止Nginx rewrite模块中其他指令的执行，例如：set、return指令)

```
server {
    rewrite ^/mi_one/(.*)$  /mi_two/$1 last;
    rewrite ^/mi_two/(.*)$  /mi_three/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }

    location /mi_three/ {
        echo "This is mi_three location";
    }
}

$ curl http://localhost/mi_one/hello
This is mi_two location

```

说明：

1. 匹配第一条rewrite成功，uri被改写为/mi_two/hello，由于flag指定为last会停止后续的rewrite的匹配（仅停止server block中的rewrite匹配），所以会根据改写的uri查找location

再来看一个last在location block中使用的例子：

```
server {
    rewrite ^/(.*)$  /mi_one/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        rewrite ^/(.*)$             /mi_three/$1 last;
        rewrite ^/mi_three/(.*)$    /;

        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }

    location /mi_three/ {
        echo "This is mi_three location";
    }
}

$ curl http://localhost/hello
This is mi_three location

```

说明：

1. 匹配server block中第一条rewrite成功，uri被改写为/mi_one/hello
2. 查找location，mi_one被确定为使用的location
3. 匹配location block中的rewrite，location中的第一条rewrite匹配成功，uri被改写为/mi_three/mi_one/hello，由于flag指定为last，所以停止location中后续的rewrite匹配，此时再根据uri=/mi_three/mi_one/hello查找location，最终mi_three被确定为使用的location

##### 5. flag指定为break

有网友把break比喻为编程语言switch中的break，大家可以感受下。

```
server {
    rewrite ^/mi_one/(.*)$  /mi_two/$1 break;
    rewrite ^/mi_two/(.*)$  /mi_three/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        echo "This is mi_one location";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }

    location /mi_three/ {
        echo "This is mi_three location";
    }
}

$ curl http://localhost/mi_one/hello
This is mi_two location

```

说明：

1. 匹配server block中第一条rewrite成功，uri被改写为/mi_two/hello，由于flag指定为break所以会停止server block中后续的rewrite匹配，根据uri=/mi_two/hello查找location，最终mi_two被确定为使用的location


```
server {
    rewrite ^/(.*)$  /mi_one/$1;

    location / {
        echo "This is default location";
    }

    location /mi_one/ {
        rewrite ^/(.*)$         /mi_three/$1 break;
        rewrite ^/mi_two/(.*)$  /;

        echo "This is mi_one location";
        echo "uri: ${uri}";
    }

    location /mi_two/ {
        echo "This is mi_two location";
    }

    location /mi_three/ {
        echo "This is mi_three location";
    }
}

$ curl http://localhost/hello
This is mi_one location
uri: /mi_three/mi_one/hello

```

说明：

1. 匹配server block中第一条rewrite成功，uri被改写为/mi_one/hello
2. 根据uri=/mi_one/hello查找location，mi_one被确定为使用的location
3. 匹配location block中的rewrite，location中的第一条rewrite匹配成功，uri被改写为/mi_three/mi_one/hello，由于flag指定为break，所以停止location中后续的rewrite匹配，并且把当前location作为最终使用的location，不会重新查找location（last会继续查找location）


#### 总结：

在Nginx的配置中可以实现简单的编程，理解起来相对有点难度，通过阅读此文希望能对你有些启发，能够根据项目需求可以配置更复杂的rewrite规则。想要更深入的理解rewrite，还需要大家去自己去实践。

FYI:

* http://nginx.org/en/docs/http/ngx_http_rewrite_module.html
* https://www.nginx.com/blog/creating-nginx-rewrite-rules/
* https://www.nginx.com/blog/converting-apache-to-nginx-rewrite-rules/
* http://www.thegeekstuff.com/2017/08/nginx-rewrite-examples/
* http://winginx.com/en/htaccess
* https://w3techs.com/technologies/overview/web_server/all
* http://nginx.org/en/docs/dev/development_guide.html#http_phases
