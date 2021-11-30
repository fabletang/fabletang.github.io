+++
title= "vim 窗口相关(三)"
description= "vim about windows"
date = "2021-11-30"
tags = [
    "vim"
]
categories = [
  "技术"
]
series = [
  "工具"
]
+++

### 一、vim 窗口类别

#### 1.文本窗口:
&emsp; 主窗口，文本区。

#### 2.buffer窗口:
  + vim运行期间的缓冲文件列表。关闭后清零。
  + 打开: ":ls"
  + 关闭: \<ESC\>或者选择一个文件序号自动关闭

#### 3. quickfix窗口:
  + 运行结果显示等。也有插件利用这个窗口显示文件列表。
  + 打开: "\<leader\>o",此快捷键参照配置文件定义。 \<leader\>为","
  + 关闭: 按提示\<CR\>回车，或者 "\<leader\>oo". 参照配置文件。

&emsp;\<leader\>配置:
```vim
" like <leader>w saves the current file
let mapleader = ","
let g:mapleader = ","
```
#### 4. nerdtree窗口:
&emsp;nerdtree插件非常流行，用来文件管理。
```vim
" NERDTree插件的快捷键
nn <silent> <F5> :NERDTreeToggle<CR>
```
  + 打开/关闭: \<F5\>切换,一般配置为自动打开。
  + 退出: 切换到nerdtree窗口然后"q"

### 二、vim 窗口切换:
&emsp;见配置,ctrl键盘+jkhl:
```vim
" Smart way to move between windows
 map <C-j> <C-W>j
 map <C-k> <C-W>k
 map <C-h> <C-W>h
 map <C-l> <C-W>l
```
