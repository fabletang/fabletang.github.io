+++
title= "Android Studio 配置"
description= "Android Studio"
date = "2014-09-30"
tags = [
    "android","tips"
]
categories = [
  "技术"
]
series = [
  "工具"
]
+++

由于长期在linux下做开发，严重依赖linux的各种小方便，然而需要参与一个Android项目。    
项目里有公司开发的.so，涉及了JNI。开始我使用的是eclipse ADT, linux 的usb联机调试也    
搞定了，一切似乎都很顺利，usb联机上传也成功了，app出来了，点击，然后就崩溃了，logcat    
提示找不到 XXX.so。本来就对eclipse这个老破车有成见，利用这个借口，开始用android studio  

环境说明:  
Ubuntu /Windows, android studio 0.8.6+   
以下是要点：

#### 一、studio 可以直接从eclipse导入,gradle build成功，但是出现编码错误（\78XXX)   
  这个是编码错误，用笨办法，右键出错文件，enconding GBK, 然后再 UTF-8.   

#### 二、编译通过，但是联机调试出现上传错误，提示XX.jar重复copy.   
  这个是studio本身bug, 在 app下的build.gradle 的 android {...}内加入

  ```json
  packagingOptions {
        exclude 'META-INF/DEPENDENCIES.txt'
        exclude 'META-INF/LICENSE.txt'
        exclude 'META-INF/NOTICE.txt'
        exclude 'META-INF/NOTICE'
        exclude 'META-INF/LICENSE'
        exclude 'META-INF/DEPENDENCIES'
        exclude 'META-INF/notice.txt'
        exclude 'META-INF/license.txt'
        exclude 'META-INF/dependencies.txt'
        exclude 'META-INF/LGPL2.1'
    }
 ```

#### 三、local.properties 是定义 sdk.dir 和 ndk.dir.需要根据本机andoird jdk情况调整。

#### 四、JNI 相关的问题:
- android studio 的ndk 不完善，没有eclipse支持那么好。但是也是可以用的，需要在build.gradle  
里配置，具体请参看gradle官方文档.


- JNI生成的.so打包apk。.so位于 libs/armeabi 目录,在 app下的build.gradle 的 android {...}内加入

 ```json
 sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
 ```

