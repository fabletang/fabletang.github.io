+++
title= "Ubuntu下Android USB联机调试"
description= "Ubuntu usb conf for ADT"
date = "2014-09-29"
tags = [
    "android",
    "toolbox"
]
menu = "main"
+++

环境说明:
Ubuntu 14.04, android ADT 4.2.2  
Linux 下 ADT usb联机调试出现 device unknown 的解决办法:  
#### 一、使用lsusb命令查看设备的vendorId和productId。
插入usb线前后用lsusb查看，找出多出的信息。

```bash
$ lsusb
Bus 003 Device 005: ID 0bb4:0c03 HTC (High Tech Computer Corp.)
```

#### 二、在/etc/udev/rules.d/目录下面新建一个规则文件 51-android.rules 添加一行 

```bash
SUBSYSTEM=="usb",ATTRS {idVendor}=="0bb4",ATTRS {idProduct}=="0c03",MODE="0666"
```

0666 表示对所有用户，开放adb所有权限

#### 三、 重启usb服务，命令如下：

```bash
sudo service udev restart
```

#### 四、 重置ADB <br> 
拔下USB与PC连接线，然后再次插上，进入Android-SDK根目录Platform-tools，运行命令:

```bash
$which adb
$sudo ./adb kill-server
$sudo ./adb devices
```
<br> 
出现以下信息表示成功：
````
\* daemon not running. starting it now on port 5037 \*
\* daemon started successfully \*
List of devices attached
0123456789ABCDEF	device
````


#### 五、 把adb命令加入环境变量
查找adb所在路径

```bash
$which adb
```
编辑~/.bashrc文件,增加 

```bash
export PATH=.:$PATH:adb路径
```
这样就不用每次进入adb目录了。


