+++
title= "Gradle 介绍"
description= "introduce gradle"
date = "2014-10-13"
tags = [
    "toolbox"
]
categories = [
  "技术"
]
series = [
  "工具"
]
+++

gradle 现在已经2.1版本了。从1.0版本就一直关注这个groovy项目，但是没有应用到  
公司项目的打算，但是现在时机到了。gradle已经是最好的java自动构建工具，没有之一。

#### 一、gradle基于动态语言groovy   
  groovy是java的动态版，闭包的特性让其编译脚本更加自由。  
  比如你可以指定项目的某个目录用特定jdk版本编译。  

#### 二、gradle 可以完全利用已有的maven库资源   
  maven库资源已经发展非常成熟，几乎没有找不到的开源库。

#### 三、gradle插件非常丰富，已经支持android项目构建。
  没错，你现在可以用gradle一个命令实现android项目的编译，打包，上传，运行。  
  插件: com.android.tools.build:gradle:0.13,已经可以胜任任何android项目构建，包括NDK.  
  Android Studio 0.8.12 已经非常成熟，可以自动生成gradle.build配置文件。  
  至于普通的java构建，就没有gradle不能做的。

#### 四、gradle学习成本低
  gradle学习成本很低，比ant低，比maven低。但是更加强大。
