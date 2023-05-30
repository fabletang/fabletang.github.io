+++
title= "为Go开发配置vscode"
description= "the golang configuration for vscode"
date = "2023-05-30"
tags = [
    "vscode"
]
categories = [
  "技术"
]
series = [
  "工具"
]
+++

### 1. 安装 Go
按照以下步骤安装 Go：
  + 在 Web 浏览器中，转到 “go.dev/doc/install”。
  + 下载操作系统的版本。
  + 下载后，运行安装程序。
  + 打开命令提示符，然后运行 go version 以确认已安装 Go。

### 2. 安装Visual Studio Code
按照以下步骤安装Visual Studio Code：
  + 在 Web 浏览器中，转到 “code.visualstudio.com”。

### 3. 安装 Go 扩展
  + 在“Visual Studio Code”中，单击活动栏中的“扩展”图标，打开“扩展”视图。 或者 (Ctrl+Shift+X) 使用键盘快捷方式。

&emsp; ![image](images/post/vscode/search-extensions-240px.webp)
  + 搜索 Go 扩展，然后选择“安装”

&emsp; ![image](images/post/vscode/install-go-extension.webp)

### 4. 更新 Go 工具
  + 在Visual Studio Code中，打开命令面板的“帮助>显示所有命令”。 或者使用键盘快捷方式 (Ctrl+Shift+P)

&emsp; ![image](images/post/vscode/search-extensions-240px-2.webp)
  + Go: Install/Update tools搜索，然后从托盘运行命令

&emsp; ![image](images/post/vscode/install-go-tools-240px.webp)

  + 出现提示时，选择所有可用的 Go 工具，然后单击“确定”。

&emsp; ![image](images/post/vscode/select-all-go-tools-240px.webp)
  + 等待 Go 工具完成更新。

&emsp; ![image](images/post/vscode/go-tools-install-240x.webp)

### 5. 编写示例 Go 程序
  + 在Visual Studio Code中，打开将在其中创建 Go 应用程序的根目录的文件夹。 若要打开文件夹，请单击活动栏中的“资源管理器”图标，然后单击“ 打开文件夹”。
  + 单击“资源管理器”面板中的“ 新建文件夹 ”，然后为名为“Go”的示例应用程序创建根控制器 sample-app
  + 单击资源管理器面板中的“ 新建文件 ”，然后为文件命名 main.go
  + 打开终端 “终端 > ”“新建终端”，然后运行 命令 go mod init sample-app 以初始化示例 Go 应用。
  + 将以下代码复制到 文件中 main.go 。
  ```golang 
package main

import "fmt"

func main() {
    name := "Go Developers"
    fmt.Println("Azure for", name)
}
  ```

### 6.运行调试器
  + 通过在编号行左侧单击左侧，在行 7 上创建断点。 或者将光标放在第 7 行并点击 F9。

&emsp; ![image](images/post/vscode/create-breakpoint-240px.webp)
  + 单击Visual Studio Code一侧的活动栏中的调试图标，打开“调试”视图。 或者 (Ctrl+Shift+D) 使用键盘快捷方式。

&emsp; ![image](images/post/vscode/run-debugger-240px.webp)
  + 单击“ 运行并调试”，或按 F5 运行调试器。 然后将鼠标悬停在第 7 行上的变量 name 上以查看其值。 单击调试器栏上的“ 继续 ”或按 F5 退出调试器。

&emsp; ![image](images/post/vscode/debug-variable-240px.webp)
 

