+++
title= "Golang介绍"
description= "Introduce of golang "
date = "2021-11-18"
tags = [
    "golang"
]
categories = [
  "技术"
]
series = [
  "编程"
]
#menu = "main"
+++

   目前golang最新版本为1.17.3，经过12年的发展，生态已经成熟。

### Go是一门工程语言
  go诞生的初衷就是解决c/c++的各种弊端，提高生产力。

  + go编译速度极快，相对于c++/rust,10倍以上编译速度。
  + go fmt 代码格式化,代码风格统一。
  + 自动垃圾搜集。简化c/c++的内存管理。
  + 单一执行文件。

### Go与其他语言的区别
  #### Go vs Java:

   + Go有内存垃圾自动处理，无虚拟机。
   + Go无对象编程的类class的概念，更强调struct（类似C）.
   + Go的对象继承是“鸭子原理",Java是“血缘关系".

  #### Go vs C:

   + Go无宏定义.兼容C。基本上可以用C的风格来写Go代码。

  #### Go vs C++:

   + C++ 20是一门现代语言，有非常多的特性，Go语言是以C为参考的，所以大多数特性Go都不具备，比如虚函数等。

  #### Go vs Rust:

   + Rust是在Go诞生后出现的，Go语法简单，Rust语法复杂。Rust强调严格的内存安全，Go的自动垃圾搜集是宽松的内存管理。Rust具备大多数现代语言特性，其复杂性与C++相当。

  #### Go vs javascript/typescript/php:

   + Go 是强类型语言，性能高很多。

### Go的优势
  #### 语法简洁
    简洁不意味简单，但是编码效率高,清晰易懂也减少维护成本。

  #### 性能优异
    性能介于c++/rust 与 Java之间。2倍以上java性能。

  #### 生态丰富，开发效率高

   + 基础类库多,经过12年的发展，驱动/扩展库也多
   + 开发工具成熟,包管理go mod,编程IDE:goland/vscode/vim-go
   + 开发资料丰富，常见问题大部分已经可以baidu/google.

### 建议掌握的编程要点

  + 多值返回
  + 函数式编程
  + 单元测试 
  + goroutine
  + sync
  + channel 
  + reflect
  + context库
  + web框架gin
  + Generic泛型(go 1.18)
  + ORM框架 gomybatis
  + log组件 zap
  + 配置组件 viper
