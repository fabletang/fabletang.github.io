+++
title= "linux svn server 搭建"
description= "svn server"
date = "2014-09-23"
tags = [
    "linux",
    "toolbox"
]
+++

网上很多svn server的配置文章，要么是windows下的visualSVN,要么配置错误。
以下是关键点:
#### 一、确保仓库目录权限正确:
```bash
$ sudo adduser svnuser              添加SVN账号
$ sudo addgroup subversion       添加SVN组
$sudo usermod -G subversion -a svnuser          将svnuser添加到subversion组
$sudo chown -R svnuser:subversion 仓库目录 
```
#### 二、修改任何svn的文件必须切换svnuser用户:
```bash
$ sudo su svnuser 
```

#### 三、svn实质是对目录的权限控制。
