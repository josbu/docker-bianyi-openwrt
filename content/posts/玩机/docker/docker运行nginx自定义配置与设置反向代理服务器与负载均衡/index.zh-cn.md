---
date: 2026-02-23T13:11:24+08:00
lastmod: 2026-02-23T13:17:06+08:00
categories:
  - 玩机
  - docker
title: docker运行nginx自定义配置与设置反向代理服务器与负载均衡
draft: "false"
tags:
  - MySQL
  - docker
  - nginx
  - 反向代理
  - 负载均衡
series: []
---
## 查看docker默认配置
查看默认配置
```bash
docker --rm --entrypoint=cat nginx /etc/nginx/nginx.conf
```
- `--rm` 表示运行完就删掉容器
- `--entrypoint=cat` 表示替换掉容器的默认命令，使用cat命令

导出默认配置到宿主机当前运行目录
```bash
docker --rm --entrypoint=cat nginx /etc/nginx/nginx.conf > myconf.nginx
```

## 设置反向代理

服务端真实地址是 `http://localhost:8080/admin/` 我们希望隐藏掉服务器地址和/admin这个路径，使得访问地址变成 `http://localhost/api`，那我们就可以匹配这个路径，设置代理服务器。
```nginx
# 反向代理,处理管理端发送的请求。http://localhost/api --> http://localhost:8080/admin
location /api/ {
	proxy_pass   http://localhost:8080/admin/;
}	
```


在本地调试的时候，如果想访问宿主机springboot运行的后端8080端口，无法在docker上使用localhost访问，查阅文档知，容器可以通过`host.docker.internal` 访问宿主机ip。所以我们可以写成
```nginx
# 反向代理,处理前端发送的请求到宿主机
# http://localhost/api --> http://host.docker.internal:8080/admin
location /api/ {
	# proxy_pass   http://localhost:8080/admin/;
	proxy_pass http://host.docker.internal:8080/admin/;
}
```

## 设置负载均衡
例如有两台后端服务器，分别是 ip1:8080 与ip2:8080，配置如下，weight是权重配置
```nginx
upstream webservers{
  server ip1:8080 weight=90;
  server ip2:8088 weight=10;
}
```
设置完之后别忘了修改反向代理服务器为负载均衡`upstream` 指定的名称`webservers` 
```nginx
# 反向代理,处理管理端发送的请求
location /api/ {
	# proxy_pass   http://localhost:8080/admin/;
	proxy_pass   http://webservers/admin/;
}
```


## docker运行nginx指定html和配置文件
使用-v指令挂载宿主机html目录与nginx.conf文件
```bash
docker run --rm -it -p 80:80 -v C:\\Users\\Administrator\\docker\\cangqiongwaimai\\nginx1.20.0\\html:/etc/nginx/html -v C:\\Users\\Administrator\\docker\\cangqiongwaimai\\nginx1.20.0\\conf\\nginx.conf:/etc/nginx/nginx.conf nginx:1.20.2
```



## docker compose写法
由于反斜杠在yml中会被识别为转义字符，所以只能用`/` 替代

- docker-compose.yml
```yml
version: '3.8'
services:
  nginx:
    image: nginx:1.20.2
    ports:
      - "80:80"
    volumes:
      # 修正路径分隔符 + 确保文件存在
      - "C:/Users/Administrator/docker/dockercompose/cangqiongwaimai/nginx1.20.0/conf/nginx.conf:/etc/nginx/nginx.conf"
      # 修正HTML目录挂载位置
      - "C:/Users/Administrator/docker/dockercompose/cangqiongwaimai/nginx1.20.0/html:/usr/share/nginx/html"
```
也可以用相对目录

```yml
services:
  nginx:
    image: nginx:1.20.0-alpine  # 推荐使用alpine版本
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./conf/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/etc/nginx/html:ro
      - ./logs:/var/log/nginx
    networks:
      - cangqiongwaimai


networks:
  cangqiongwaimai:

```

dockercompose 运行命令
- `-f docker-compose.yml` 指定配置文件为docker-compose.yml，如果配置文件名称就是`docker-compose.yml` 可以省略
- `up` 启动服务
- `-d` 和`docker run -d` 命令一样，是后台运行
```bash
docker compose -f docker-compose.yml up -d
```

停止服务
```bash
docker compose -f docker-compose.yml down
```


## 完整nginx配置
```nginx

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
	
	map $http_upgrade $connection_upgrade{
		default upgrade;
		'' close;
	}

	upstream webservers{
	  server host.docker.internal:8080 weight=90 ;
	  #server 127.0.0.1:8088 weight=10 ;
	}

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html/sky;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # 反向代理,处理管理端发送的请求
        location /api/ {
			# proxy_pass   http://localhost:8080/admin/;
			proxy_pass http://host.docker.internal:8080/admin/;
            #proxy_pass   http://webservers/admin/;
        }
		
		# 反向代理,处理用户端发送的请求
        location /user/ {
            proxy_pass   http://webservers/user/;
        }
		
		# WebSocket
		location /ws/ {
            proxy_pass   http://webservers/ws/;
			proxy_http_version 1.1;
			proxy_read_timeout 3600s;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection "$connection_upgrade";
        }

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

```
