+++
title= "Grails框架介绍"
description= "grails introduce"
date = "2014-12-25"
tags = [
    "java"
]
menu = "main"
+++

Grails是一套用于快速Web应用开发的开源框架，它基于Groovy编程语言，并构建于Spring、Hibernate等开源框架之上，是一个高生产力一站式框架。其官方网站上的煽情介绍为：The search is over!, 即为不要再苦苦寻找web开发框架了，Grails是终结者。

如果web项目组大部分懂java，又没有旧的web项目负担。Grails是明智的选择。
缺点: groovy是类java的脚本解释动态语言，尽管其兼容java所有语法，最终也运行于JVM,但是速度有所影响.总体grails
比rails快，慢于传统ssh(struts/spring/hiberate)框架.
除了稍慢的运行速度，grails相对SSH全面胜出，以下是论点:

#### 一、学习成本低:
1. groovy语法不用学习，如果懂得java,groovy就是秒懂，实在不行，完全用java的语法也没有任何问题。其实注重运行速度的功能，比如加密解密，数据排序等算法推荐用java编写，groovy可以无缝引用。
2. grails的springframework\hibernat极易使用，基本不用学习，完全不用理会那些迷宫式xml配置。grails会自动帮你处理好。

#### 二、开发速度快:
1. grails 不是粗糙的ssh,它甚至有命令控制台，比如 grails create-control abc.login,就自动创建好了control及相关的测试文件。不用手动建立目录，建立文件。
2. 调试便捷，run-app命令就可以运行一个web服务器，不需要启动额外的tomcat或者其他web容器, 同时它还是热编译的，也就是说，运行期间，你改动代码，grail自动就编译好了，不需要重新启动，哪怕你改动control都可以。
3.groovy语言大大减少了java的繁杂，同时集成很多通用库，可以大大减少代码.

#### 三、功能完善:
ssh是个粗糙的框架，不怀疑可以很快写出一个CURD的功能块。但是这远远不能应付实际项目需求，还得做大量的工作。 
1. view层： grails 集成了sitmesh模版，而且自动加载缓存，模块化view。GSP相对structs的标签更好用。

2. control层: view数据绑定支持list,可以实现多维数据结构。spring mvc只支持map/bean 键值绑定方式。数据渲染很容易实现json/xml输出。比如 render xxxxbean as JSON 

3. model层: 易于处理多对多，一对多，级联删除，完全不用去配置hibernat XML. 脏数据处理，数据缓存默认集成。

4. 测试功能完备，易于使用，m/v/c各层都容易实现测试.

5. 其他小功能完备，比如内置一个内存数据库，项目发布时候无缝切换实际数据库。

#### 四、项目易于管理:
1. Grails 的原则是约定大于配置，也就是说配置文件，文件目录，命名方式都是约定好的。开放人员就是做填空题。项目自然就规范了。
2. Grails大大减少了配置文件，groovy语法大大缩减了代码量，越少的编码意味更好维护。
3. Grails开发模式下的热部署功能给前端开发提供很大便利，可以不重启web容器就可以即使得到改动的效果。这样利于前端后端开发分工。 

