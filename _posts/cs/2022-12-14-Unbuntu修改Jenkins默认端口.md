---
title:      Unbuntu修改Jenkins默认端口
date:     	2022-12-14
categories: [Computer Science]
tags:      	[Ubuntu]
math:      	true
---

Jenkins默认占用了8080端口，需要改成自定义端口。

## 第一步

```sh
vim /etc/init.d/jenkins
```

找到
```
check_tcp_port "http" "${HTTP_PORT}" "8080" "${HTTP_HOST}" "0.0.0.0" || return 2
```

把8080改为8088

## 第二步

```sh
vim /etc/default/jenkins
```

找到：

```
# port for HTTP connector (default 8080; disable with -1)
HTTP_PORT=8080
```

同样也把8080改为8088

## 第三步（可选）

```sh
vim /etc/nginx/conf.d/jenkins.conf
```

由于使用了nginx作为反向代理，所以Nginx也需要做相应修改：

```
location / {
    proxy_set_header X-Rea $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Nginx-Proxy true;
    proxy_pass http://localhost:8080;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

同样也把8080改为8088

## 第四步

```sh
vim /usr/lib/systemd/system/jenkins.service
```

找到

```
# Port to listen on for HTTP requests. Set to -1 to disable.
# To be able to listen on privileged ports (port numbers less than 1024),
# add the CAP_NET_BIND_SERVICE capability to the AmbientCapabilities
# directive below.
Environment="JENKINS_PORT=8080"
```

同样也把8080改为8088

## 第五步

重启Jenkins


```sh
systemctl restart jenkins
```

重启Nginx

```sh
systemctl restart nginx
```



Done~
