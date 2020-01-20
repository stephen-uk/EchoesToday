---
title: Hello Hexo
date: 2020-01-20 16:37:15
tags: 博客搭建
---
之前也用过Hexo搭建过博客，直接生成静态HTML的方式可以上传到Github，然后利用Github Pages的功能，只需要花个域名的钱就能有个自己的博客，还是挺划算的。但是Github国内实在不稳定，最近看了一下阿里云轻服务（香港🇭🇰）一个月才不到30的价格，就入手了一个，速度感觉还是挺快的，顺便搭建了一个ShowDoc。

## 0x0 搭建Nginx 以及 ShowDoc

### 安装Nginx
系统选用的是CentOS，安装Nginx还是很简单,运行下面👇的命令即可

```
sudo yum install nginx
```

### 搭建ShowDoc
[ShowDoc](https://www.showdoc.cc/) 是一个类似于轻量级Wiki的工具，可以自己建立多个项目，我主要是用它来记录一些杂事，以及一些可以分享给小伙伴的东西。具体可以看一下链接。安装方式采用的docker，命令👇

```
  #下载脚本并赋予权限
   wget https://www.showdoc.cc/script/showdoc;chmod +x showdoc;
  #默认安装中文版。如果想安装英文版，请加上en参数，如 ./showdoc en
  ./showdoc

```

安装完成后，ShowDoc的docker的端口为4999，我们可以通过nginx将ShowDoc的域名解析到4999接口，修改nginx.conf，nginx配置如下
```
    server {
        listen       80;
        server_name  wiki.echoes.today;
        location / {
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:4999;
        }
    }

```
至此ShowDoc搭建完成

## 0x1 Hexo 以及自动化部署
Hexo 对于服务器来说只是需要nginx配置一个server就好了，主要说一下自动化部署实现思路，Hexo可以通过`hexo deploy`命令将生成的静态资源上传到Github，那么我们可以在项目中配置一个WebHook，在服务器中写一个脚本来响应。

### webhook 脚本

``` javascript
var http = require('http')
  , exec = require('exec')
 
const PORT = 9988
  , PATH = 'PATH'
 
var deployServer = http.createServer(function(request, response) {
  console.log(request.url)
  if (request.url.search(/deploy\/?$/i) > 0) {
    var commands = [
      'cd ' + PATH,
      'git pull'
    ].join(' && ')
 
    exec(commands, function(err, out, code) {
      if (err instanceof Error) {
        response.writeHead(500)
        response.end('Server Internal Error.')
        throw err
      }
      process.stderr.write(err)
      process.stdout.write(out)
      response.writeHead(200)
      response.end('Deploy Done.')
    })
 
  } else {
    response.writeHead(404)
    response.end('Not Found.')
  }
})
 
deployServer.listen(PORT)
```

### Github webhook配置
![webhook](webhook.png)
因为脚本功能相对简单 只是做一下git pull的操作，所以没有用到github传过来的一些参数，也没有做一些校验，不是很严谨，需要的话可以在进行博冲

### webhook的Nginx配置
```
    server {
        listen       80;
        server_name  deploy_blog.echoes.today;
        location / {
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:9988;
        }
    }

```

这些搭建完成后就可以通过 `hexo d `命令将写好的文章上传到Github，并完成自动部署。

## 0x3 常用命令集合

### Nginx

```
nginx -s reopen #重启Nginx
nginx -s reload #重新加载Nginx配置文件，然后以优雅的方式重启Nginx
nginx -s stop #强制停止Nginx服务
nginx -s quit #优雅地停止Nginx服务（即处理完所有请求后再停止服务）

```

### Docker 常用命令
[Docker 常用命令](https://www.cnblogs.com/DeepInThought/p/10896790.html)