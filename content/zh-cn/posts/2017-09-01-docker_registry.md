+++
title= "docker私有仓库搭建"
description= "docker registry 2"
date = "2017-09-01"
tags = [
    "docker",
]
categories = [
  "技术"
]
series = [
  "工具"
]
#menu = "main"
+++

环境:centos7,docker 1.12+，registry 2.
#### 一、准备工作:
```bash
mkdir /data/docker-registry
mkdir /data/docker-registry-auth
```
#### 二、安全设置:
生成 http 密码文件
```bash
docker run --entrypoint htpasswd registry:2 -Bbn anycloud '123'> /data/docker-registry-auth/htpasswd
```
```bash
获取 SSL 证书
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
./letsencrypt-auto --help
./letsencrypt-auto certonly --standalone -d <HOST>
```
#### 三、启动:
新增systemd守护服务,内容如下
```bash
cat /etc/systemd/system/multi-user.target.wants/docker-registry.service

[Unit]
Description=registry
Requires=docker.service
After=docker.target

[Service]
Restart=always
RestartSec=20
TimeoutStartSec=5min
ExecStartPre=-/usr/bin/docker kill registry
ExecStart=/usr/bin/docker run -p 5000:5000 --name registry \
-v /data/docker-registry-auth/:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain.pem \
-e REGISTRY_HTTP_TLS_KEY=/certs/privkey.pem \
-e REGISTRY_STORAGE_DELETE_ENABLED=true \
-v /data/docker-registry-auth/:/auth \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-e REGISTRY_STORAGE_DELETE_ENABLED=true \
-v /data/docker-registry/:/var/lib/registry \
registry:2

ExecStartPost=-/usr/bin/docker rm registry
ExecStop=/usr/bin/docker stop registry
ExecStopPost=-/usr/bin/docker rm registry

[Install]
WantedBy=multi-user.target
```
启动registry

```bash
systemctl daemon-reload
systemctl start docker-registry
```
#### 四、检验:
浏览器 https://localhost:5000/v2/
密码登入,假如 显示 {} 之类的json文本,则说明成功。
