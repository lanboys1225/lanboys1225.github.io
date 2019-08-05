---
title: Linux命令行样式定制
date: 2019-08-05 21:53:02
category: linux
tags: [linux, xshell]
---

在使用linux命令行时候，总是难以区分命令和结果的具体界线，不知道当前的路径在哪里，就像下面这样，无形之中降低了我们的效率。但是我们在用git bash的时候不会有这个烦恼，那我们能不能将linux中的界面样式定制一下，变成git bash这样呢？答案是肯定的。

![](https://pic.superbed.cn/item/5d482648451253d1787e9953.png)

linux命令行界面

![](https://pic3.superbed.cn/item/5d482765451253d1787ec84e.png)

git bash 界面

通过查资料发现，要想修改命令行头部显示样式，实际的操作就是覆盖系统本身的 PS1 变量，如下代码所示，在当前用户(这里是root用户)的./bashrc中添加 PS1 的值就可以了

```
[root@VM_72_235_centos ~]# vi .bashrc
[root@VM_72_235_centos ~]# cat .bashrc
# .bashrc
# User specific aliases and functions
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
# Source global definitions
if [ -f /etc/bashrc ]; then
	. /etc/bashrc
fi
# 添加这行就可以了
PS1='[\[\e[00;35m\]\u@dev \t\[\e[0m\]]\[\e[0;33m\](\w)\$\[\e[0m\] '

[root@VM_72_235_centos ~]# source .bashrc (执行这句设置生效，只对当前用户生效)
[root@dev 22:36:51](~)# ls (发现生效了)
```

先将这行代码拆分如下，再来分析具体意思
```
                    [               //表示【 [ 】 颜色为 默认颜色
\[\e[00;35m\]       \u@dev \t       //表示【 \u@dev \t 】 颜色为 【 \[\e[00;35m\] 】
\[\e[0m\]           ]               //表示【 ] 】 颜色为 【 \[\e[0m\] 】
\[\e[0;33m\]        (\w)\$          //表示【 (\w)\$ 】 颜色为 【 \[\e[0;33m\] 】
\[\e[0m\]                           //取消设置(设置回默认颜色)
```
> \[\e[00;35m\] 该设置是应用到后面字符上，直到有其他设置，所以最后要设置回默认颜色 

> 设置字符序列颜色的格式为：\[\e[F;Bm\] 其中“F”为字体颜色，编号30 ~ 37；“B”为背景色，编号40 ~ 47

颜色表
前景 | 背景 | 颜色
---|---|---
30 | 40 | 黑色
31 | 41 | 红色  
32 | 42 | 绿色  
33 | 43 | 黄色  
34 | 44 | 蓝色  
35 | 45 | 紫红色  
36 | 46 | 青蓝色  
37 | 47 | 白色

变量值：
- \d ：可显示出『星期 月 日』的日期格式，如："Mon Feb 2"
- \H ：完整的主机名称。举例来说，鸟哥的练习机为『www.vbird.tsai』
- \h ：仅取主机名称在第一个小数点之前的名字，如鸟哥主机则为『www』后面省略
- \t ：显示时间，为 24 小时格式的『HH:MM:SS』
- \T ：显示时间，为 12 小时格式的『HH:MM:SS』
- \A ：显示时间，为 24 小时格式的『HH:MM』
- \@ ：显示时间，为 12 小时格式的『am/pm』样式
- \u ：目前使用者的帐号名称，如『root』；
- \v ：BASH 的版本资讯，如鸟哥的测试主机版本为 3.2.25(1)，仅取『3.2』显示
- \w ：完整的工作目录名称，由根目录写起的目录名称。但家目录会以 ~ 取代；
- \W ：利用 basename 函数取得工作目录名称，所以仅会列出最后一个目录名。
- \# ：下达的第几个指令。
- \$ ：提示字元，如果是 root 时，提示字元为 # ，否则就是 $ 
- \n : new line,表示另起一行显示指令符

通过上面的颜色及变量值对照表，就可以轻松的定制出适合自己的样式啦。

我的配置如下，跟git bash相近
```
PS1='\[\e[00;35m\]\u@\h \t \[\e[0;33m\]\w (12.22.34.179)\n\$\[\e[0m\] ' 

效果如下(颜色在这里没显示出来)：
root@VM_72_235_centos 21:22:43 /usr/local/share (12.22.34.179)
$ less log.log

```

参考：[在xshell中将命令行移至下一行:  http://blog.sina.com.cn/s/blog_96a11ddf0102vbb7.html](http://blog.sina.com.cn/s/blog_96a11ddf0102vbb7.html) 

