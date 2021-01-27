+++
title= "Golang代码结构"
description= "The Struct of golang code"
date = "2017-05-01"
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

   Go的设计思想是代码至上，依赖于代码而不是象其它语言依赖于编译好的库。尽管go的版本号已经到1.8.1了,但是还没没有好的package版本管理策略。1.6推出的vender只是稍微缓解了困境,只有寄希望与将来的1.9版本了。Go的代码结构不同于其它语言的地方,以java为对照:

#### 代码文件位于src目录
  假如GOPATH对应项目goprojectstruct,代码文件应该处于 goprojectstruct/src目录内.

#### Go无class的概念 
  Go无对象编程的类class的概念。所以代码文件名称可以随意定义，但是一般位于package目录,一般与package同名，但是拆分成几个文件也是可以的。

#### 代码文件/package名为小写 
  Java文件名一般为大写。Go以简单为宗旨，显然小写字母要简单,也符合go的定义，小写代表私有private，大写代表公开public。

#### 单元测试文件 
  单元测试文件与被测试文件处于同一目录，文件名追加"_test",比如 hello.go的测试文件为 hello_test.go.

#### 引用库位于vender
  所有外来引入的package都应该处于vender目录下。.gitignore也应该忽略vender目录。以此保持代码结构的精简。
  
#### package版本管理 
  在go1.9之前，推荐采用glide来管理。
   
