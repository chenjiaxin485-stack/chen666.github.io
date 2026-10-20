nginx -t
检查语法

nginx -s reload
重启动

curl -i localhost/app/index.html
访问

- ### 配置文件

```
event {}
http{
	include /etc/nginx/mime.types;
	include /etc/nginx/conf.d/*.conf; ##引入conf文件后，server模块移到conf文件里面
	server{
		listen 80;
		server_name loaclhost;
		root /var/www/localhost;
		return 200 'successly' ##指定root后return删除，因为return优先级更高
		index egg.html
		
	}
}
```
/etc/nginx/nginx.conf


- ### 外置配置

```
events{
http{
	include /etc/nginx/mime.types
	include /etc/nginx/conf.d/*.conf
}
}
```
/etc/nginx/nginx.conf

```
server{
	listen 80;
	server_name loaclhost;
	error_page 404 /404.html ##定义404定向的页面
	
	
	rewrite /temp /app/index.html ##重定向
	location = /app/index.html{ ## =完全匹配，～正则表达式，*不匹配大小写
		root /var/www/localhost; 
	}
}
```
/etc/nginx/conf.d/default.conf

- ### 代理

正向代理（通过指定端口来找项目）
反向代理（通过项目路径来通端口）

```
##负载均衡
upsteam backend-servers{
	server localhost:3000 weight = 2;
	server localhost:3001 weight = 6; 
}

server{
	listen 80;
	server_name loaclhost;
	
	error_page 404 /404.html ##定义404定向的页面
	root /var/www/localhost; 
	index index.html
	
	##反向代理
	localtion /nextjsapp1{
		proxy_pass http://localhost:3000
	}
	localtion /nextjsapp2{
		proxy_pass http://localhost:3001
	}
	
	localtion / {
		proxy_pass http://backend-servers; ##如果设置了负载均衡就配置这个即可
	}
	
}
```

