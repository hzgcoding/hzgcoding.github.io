---
title: Linux shell语言——dash和bash
date: 2020-01-23 15:13:45
tags:
---

什么是bash ？

Bash(GNU Bourne-Again Shell)是许多Linux平台的内定Shell，事实上，还有许多传统UNIX上用的Shell，像tcsh、csh、ash、bsh、ksh等等。

GNU/Linux 操作系统中的 /bin/sh 本是 bash (Bourne-Again Shell) 的符号链接，但鉴于 bash 过于复杂，有人把 bash 从 NetBSD 移植到 Linux 并更名为 dash (Debian Almquist Shell)，并建议将 /bin/sh 指向它，以获得更快的脚本执行速度。Dash Shell 比 Bash Shell 小的多，符合POSIX标准。

Debian和Ubuntu中，/bin/sh默认已经指向dash，这是一个不同于bash的shell，它主要是为了执行脚本而出现，而不是交互，它速度更快，但功能相比bash要少很多，语法严格遵守POSIX标准。

就是这个倒霉的dash解释器使得我按照bash语法写的shell 脚本不能运行。

要知道自己的/bin/sh指向何种解释器，可以用 ls /bin/sh -al 命令查看：
```
$ ls /bin/sh -al
lrwxrwxrwx 1 root root 4 11月 16 15:33 /bin/sh -> bash	
```

以上结果就表示当前系统用的是dash解释器。

切换到bash的方式其实挺简单的，关键是一直没找出这个原因……

修改默认的sh，可以采用命令`sudo dpkg-reconfigure dash`

会出现一个图片状的配置菜单，选no就可以了

再次检查一下， ls /bin/sh -al 发现软链接指向/bin/bash

         lrwxrwxrwx 1 root root 4 11月 16 15:33 /bin/sh -> bash



注：dash 和 bash 语法上的主要的区别有:

1. 定义函数

bash: function在bash中为关键字

dash: dash中没有function这个关键字

2. select var in list; do command; done

bash:支持

dash:不支持, 替代方法:采用while+read+case来实现

3. echo {0..10}

bash:支持{n..m}展开

dash:不支持，替代方法, 采用seq外部命令

4. here string

bash:支持here string

dash:不支持, 替代方法:可采用here documents

5. >&word重定向标准输出和标准错误

bash: 当word为非数字时，>&word变成重定向标准错误和标准输出到文件word

dash: >&word, word不支持非数字, 替代方法: >word 2>&1; 常见用法 >/dev/null 2>&1

6. 数组

bash: 支持数组, bash4支持关联数组

dash: 不支持数组，替代方法, 采用变量名+序号来实现类似的效果

7. 子字符串扩展

bash: 支持${parameter:offset:length},${parameter:offset}

dash: 不支持， 替代方法:采用expr或cut外部命令代替

8. 大小写转换

bash: 支持${parameter^pattern},${parameter^^pattern},${parameter,pattern},${parameter,,pattern}

dash: 不支持，替代方法:采用tr/sed/awk等外部命令转换

9. 进程替换<(command), >(command)

bash: 支持进程替换

dash: 不支持, 替代方法, 通过临时文件中转

10. [ string1 = string2 ] 和 [ string1 == string2 ]

bash: 支持两者

dash: 只支持=

11. [[ 加强版test

bash: 支持[[ ]], 可实现正则匹配等强大功能

dash: 不支持[[ ]], 替代方法，采用外部命令

12. for (( expr1 ; expr2 ; expr3 )) ; do list ; done

bash: 支持C语言格式的for循环

dash: 不支持该格式的for, 替代方法，用while+$((expression))实现

13. let命令和((expression))

bash: 有内置命令let, 也支持((expression))方式

dash: 不支持，替代方法，采用$((expression))或者外部命令做计算

14. $((expression))

bash: 支持id++,id--,++id,--id这样到表达式

dash: 不支持++,--, 替代方法:id+=1,id-=1, id=id+1,id=id-1
