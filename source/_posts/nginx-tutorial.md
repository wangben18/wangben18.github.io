---
title: Nginx 站点设置入门教程
date: 2019-02-25 16:24:45
updated: 2019-02-26 16:54:23
tags:
- nginx
- tutorial
categories:
- tutorial
- nginx
comments: true
---


Nginx是一个快速且强大的http和反向代理服务器，能够快捷方便地提供服务

### 安装

假设运行的系统是Ubuntu，在terminal输入如下命令安装nginx

```bash
apt-get install nginx
```

此时用浏览器访问你服务器的IP地址，你将会看见“welcome to nginx”页面

### nginx的位置

所有nginx的配置文件都在**/etc/nginx**目录下，cd到这个目录。本次教程要添加的配置文件会放在其中一个名为**sites-enabled**目录下，cd到该目录里，你会发现有个default文件在里面，就是这个文件使你看见“welcome to nginx”。touch一个**test**在该目录下并用你喜欢的编辑器打开

### 设置静态服务器

Nginx的config文件有它自己的一套与css的类似的语法规则。如下是一个名为**server**的一级命名空间

```
server {

}
```

在这个命名空间内部，可以像css那样添加以分号结尾的键值对，也可以再添加一块子命名空间

键值对（key-value pairs）也称为指令（directives），命令有很多很多，本教程只用到其中一小部分，下面每个命令都加了一个指向该命令文档的链接，想要了解更多就看文档吧，文档是唯一的官方了解途径。如果你想要更加复杂高级的设置，也免不了要看文档。

[listen](https://wiki.nginx.org/HttpCoreModule#listen)指令用来设置服务器监听的端口号，默认为80

```
server {
    listen 80;
}
```

由于80是默认值，可以不用写该指令，但为了表示清楚，写上也是极好的

[server_name](https://wiki.nginx.org/HttpCoreModule#server_name)指令用来匹配url。如果你的站点在`https://wongben.com`，那根server_name就是wongben.com。如果还有一个在`https://abc.com`，可以再添加一个abc.com的server_name，各自的请求就会匹配到各自的站点里。

```
server {
    listen 80;
    server_name wongben.com;
}
```

[root](https://wiki.nginx.org/HttpCoreModule#root)指令是设置静态网页的关键组成部分。如果你只是想要发布一些html和css文件，root指令描述了这些文件的存放位置。现在，创建目录 **/var/www/example**（也可以是其他任意位置），在此目录下，touch index.html，编辑该文件，存入一些类似hello world的文字，保存。回到刚才的配置文件，添加root指令

```
server {
    listen 80;
    server_name wongben.com;
    root /var/www/example;
}
```

[location](https://wiki.nginx.org/HttpCoreModule#location)指令有两个参数：1.字符串或正则 2.一个block（即命名空间namespace）。如果你想要给 **wongben.com/whatever** 指定页面，就用'whatever'作为uri。在这里，我们要匹配的是根目录，所以使用 **/** 作为uri

```
server {
    listen 80;
    server_name wongben.com;
    root /var/www/example;

    location / {

    }
}
```

“**/**” 会匹配所有url，因为它被看成是一个正则。如果你想要单独匹配字符串，就加个等号

```
location = / { ... }
```

[try_files](https://wiki.nginx.org/HttpCoreModule#try_files)指令根据文件名列表或模式在root指定的目录下寻找匹配的文件，在这里我们要做的很简单，匹配任何斜杠之后的字符，例如'whatever.html'，要是斜杠后没有任何字符，则匹配**index.html**。

```
server {
    listen 80;
    server_name wongben.com;
    root /var/www/example;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 发布

运行如下命令让nginx重新载入

```bash
service nginx reload
```

然后就可以在浏览器输入地址访问服务器啦

[English](https://carrot.is/coding/nginx_introduction)
