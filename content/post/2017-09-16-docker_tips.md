+++
title= "docker tips"
description= "docker 小技巧"
date = "2017-09-16"
tags = [
    "docker",
]
menu = "main"
+++

#### 一、docker rmi:

1. 使用多个images id删除,前四位、空格区分

```bash
docker rmi 861b 7d51
```

2. 过滤批量删除镜像, 对docker images 显示的行进行过滤。

 *  根据tag名删除

```bash
docker rmi -f $(docker images | grep "fabletang/test-*" | awk "{print \$3}")
docker rmi -f $(docker images | grep "<none>" | awk "{print \$3}")
```
 *  根据版本号删除

```bash
docker rmi -f $(docker images | grep "0.0.1" | awk "{print \$3}")
```

 *  根据Tag和版本号删除

```bash
docker rmi -f $(docker images | grep "fabletang/service-java" | awk -F' 0.' '{if ($2<0.6) print $0}' | awk "{print \$3}")
```
