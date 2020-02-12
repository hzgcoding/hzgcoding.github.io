---
title: 批量转换换行符CRLF到LF
date: 2020-02-12 13:25:41
tags: git
---

## 关闭Git自动转换功能

```
git config --global core.autocrlf false
```

## CRLF转换成LF

**vscode或者visual studio等一些代码编辑器仅仅支持单个文件的格式转换，以下三种方法均是支持批量转换的方法**

### 第一种方法 (亲测))

已经上传到`/y-server/doc/format/` 目录下 或者点击下面链接下载最新

[下载dos2unix工具包](https://waterlan.home.xs4all.nl/dos2unix/dos2unix-7.4.1-win64.zip)

[dos2unix文档说明](https://waterlan.home.xs4all.nl/dos2unix/zh_CN/man1/dos2unix.htm#9)

结合 find(1) 和 xargs(1) 使用 dos2unix 可以递归地转换目录树中的文本文件。例如，转换当前目录的目录树中所有的 .txt 文件：
```
dos2unix < a.txt
cat a.txt | dos2unix
```
若文件名中有空格或引号，则需要使用 find(1) 选项 -print0 及相应的 xargs(1) 选项 -0；其他情况下则可以省略它们。也可以结合 -exec 选项来使用 find(1)：
```
find . -name '*.txt' -exec dos2unix {} \;
```
在Windows命令提示符中，可以使用下列命令：
```
for /R %G in (*.txt) do dos2unix "%G"
```
PowerShell用户可以在Windows PowerShell中使用如下命令：
```
get-childitem -path . -filter '*.txt' -recurse | foreach-object {dos2unix $_.Fullname}
```

### 第二种方法 (没有尝试)

采用EditPlus批量转换文件格式

![1](https://img-blog.csdnimg.cn/20190414011144786.png)
![2](https://img-blog.csdnimg.cn/20190414011207506.png)
![2](https://img-blog.csdnimg.cn/20190414011249745.png)

### 第三种方法 (没有尝试)

[巧妙的借助git快速批量转换crlf到lf](https://rizon.top/tech/%E8%BD%AC%E6%8D%A2crlf%E5%88%B0lf/)