---
layout:     post
title:      "Understanding Nginx Rewrite"
subtitle:   " \"nginx rewrite rules.\""
date:       2017-03-10 18:00:00
author:     "Agile6v"
header-img: "img/post-bg-2015.jpg"
tags:
    - Nginx
---


Nginx rewrite rules compared with Apache rewrite rules are much simple. However, new came into the Nginx world friends will also get dizzy with these rules. So, I think it is necessary to introduce Nginx rewrite rules through this article.

##### First, Let's take a look at the rewrite rules of the Nginx.

The rewrite directive systax refers to [rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite):


```
Syntax:    rewrite regex replacement [flag];
Context:   server, location, if
```


From the official document we learned the following points：


```
1. If the specified regular expression matches a request URI, URI is changed
as specified in the replacement string.
2. The rewrite directives are executed sequentially in order of their
appearance in the configuration file.
3. It is possible to terminate further processing of the directives using flags.
4. If a replacement string starts with “http://”, “https://”, or “$scheme”,
the processing stops and the redirect is returned to a client.

1 - very easy to understand
2 - when there is no flag, rewrite directives are executed in the order in which they appear
3 - when there is a flag, it will affect the execution order of the rewrite directives
4 - very easy to understand
```

##### Second, If the optional flag exists, then it will affect the execution order of the rewrite directives.

So, We mainly look at the flag can choose what values.

```
last
    stops processing the current set of ngx_http_rewrite_module directives and
    starts a search for a new location matching the changed URI;
break
    stops processing the current set of ngx_http_rewrite_module directives as with
    the break directive;
redirect
    returns a temporary redirect with the 302 code; used if a replacement string
    does not start with “http://”, “https://”, or “$scheme”;
permanent
    returns a permanent redirect with the 301 code.
```

It is not easy to understand the last and break. However, there is a simple way to quickly understand them.<br/>
The last flag is like "continue" we used when writing C programs.<br/>
The break flag is like "break" we used when writing C programs.<br/>


##### Third, Through a few examples to understand the flag of the last or break.


###### Example 1: rewrite with break inside server block

```shell
server {
    listen 80;
    server_name localhost;

    rewrite ^(.*)$ /test_1/$1 break;
    rewrite ^/test_1/(.)*$ /test_2/$1 break;

    location / {
        echo "This is default location";
    }

    location /test_1/ {
        echo "This is test_1 location";
    }

    location /test_2/ {
        echo "This is test_2 location";
    }
}


# curl http://localhost/test_1/test.txt
This is test_1 location
```


###### Example 2: rewrite with last inside server block
```shell
server {
    listen 80;
    server_name localhost;

    rewrite ^(.*)$ /test_1/$1 last;
    rewrite ^/test_1/(.)*$ /test_2/$1 last;

    location / {
        echo "This is default location";
    }

    location /test_1/ {
        echo "This is test_1 location";
    }

    location /test_2/ {
        echo "This is test_2 location";
    }
}

# curl http://localhost/test_1/test.txt
This is test_1 location

```

###### Example 3: rewrite with last inside location block
```shell
server {
    listen 80;
    server_name localhost;

    rewrite ^/test_1/(.*)$ /test_2/$1;

    location / {
        rewrite ^(.*)$ /test_1/$1 last;
        rewrite ^(.*)$ /test_3/$1 last;
        echo "This is default location";
    }

    location /test_1/ {
        echo "This is test_1 location";
    }

    location /test_2/ {
        echo "This is test_2 location";
    }

    location /test_3/ {
        echo "This is test_3 location";
    }
}

# curl http://localhost/test.txt
This is test_1 location
```


###### Example 4: rewrite with last inside location block
```shell
server {
    listen 80;
    server_name localhost;

    rewrite ^/test_1/(.*)$ /test_2/$1;

    location / {
        rewrite ^(.*)$ /test_1/$1 break;
        rewrite ^(.*)$ /test_3/$1 break;
        echo "This is default location";
    }

    location /test_1/ {
        echo "This is test_1 location";
    }

    location /test_2/ {
        echo "This is test_2 location";
    }

    location /test_3/ {
        echo "This is test_3 location";
    }
}

# curl http://localhost/test.txt
This is default location
```

###### Example 5: rewrite with last inside location block

```shell
server {
    listen 80;
    server_name localhost;

    rewrite ^/test_1/(.*)$ /test_2/$1;

    location / {
        rewrite ^(.*)$ /test_1/$1 last;
        echo "This is default location";
    }

    location /test_1/ {
        rewrite ^/test_1/(.*)$ /test_2/$1 last;
        echo "This is test_1 location";
    }

    location /test_2/ {
        echo "This is test_2 location";
    }
}

# curl http://localhost/test.txt
This is default location
```

##### Finally, through a picture to understand both last and break.

![img](/img/ngx_rewrite.png)