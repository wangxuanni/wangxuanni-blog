---
title: Linux查看日志利器
date: 2019-11-25 20:09:50
categories: 工具
description: 实习期间所学的一些开发技巧。
---

# Linux查看日志利器理

常用命令

[菜鸟教程常用命令](https://www.runoob.com/w3cnote/linux-common-command-2.html)

```
切换目录
cd /usr查看目录下的文件lsls -lh 遍历详细信息，如权限，所属用户，创建日期，大小等等信息
查看当前所处目录pwd
创建目录文件mkdir -p /test/abctouch test.java
删除文件
rm -f test.java
vi abc.txt要进行编辑，要输入 a 或者 i ，才可以进行编辑哦
要退出，首先要离开当前的编辑模式，点击左上角的ESC键，退出编辑模式然后输入冒号 (shift+分号) 打开控制命令接着输入wq，然后敲回车，即保存退出wq 是quit+write的缩写
```



### grep搜索文本，支持正则

#### 参数

```
grep [-acinv] [--color=auto] '搜寻字符串' filename
-a：将binary文件以text文件的方式搜寻数据
-c：计算找到'搜寻字符串'的次数
-i：忽略大小写
-n：顺便输出行号
-v：反向选择，即输出没有'搜索字符串'的内容
--color=auto：可以将找到的关键词部分加上颜色显示
-A 5：可以显示前面5行加上查找行的信息
-B 5：可以显示后面5行加上查找行的信息
-C 5：可以显示前后5行加上查找行的信息
```

#### 例子

 `grep 2017010500345878 --color info.log`

这行命令在info.log中搜索含有"2017010500345878"关键词的段落并且使用其他颜色标记关键词。
复制代码

`grep -v 'ERROR' error.log` 查找不含”ERROR”的行

`grep -C 50 "关键字" info.log`





### less分页查看，比more更强

`ps -ef |less`

ps查看进程信息并通过less分页显示

搭配grep使用

`less -N tomcat_stdout.log|grep job`
`less xxx.log | grep -i -n -C10 --color=auto xx`

有用的命令

- 按F,监控更新.如果要暂停监控，可以CTRL+C.
- /字符串：搜索“字符串”的功能
- G:调到文本末尾
- 空格键 滚动一页
- 回车键 滚动一行
- Q: 退出
- gg: 调到文本最前面
- 按v进入编辑模式，可以类似于vim的方式来编辑保存文件了

其他有用的命令

1.全屏导航

- ctrl + F - 向前移动一屏forward
- ctrl + B - 向后移动一屏backward
- ctrl + D - 向前移动半屏
- ctrl + U - 向后移动半屏

2.单行导航

- j - 向前移动一行
- k - 向后移动一行

3.其它导航

- G - 移动到最后一行
- g - 移动到第一行
- q / ZZ - 退出 less 命令

4.标记导航

当使用 less 查看大文件时，可以在任何一个位置作标记，可以通过命令导航到标有特定标记的文本位置：

- ma - 使用 a 标记文本的当前位置
- 'a - 导航到标记 a 处

### vi编辑文本

> 使用找一个字符串，在vi命令模式下键入“/”，后面跟要查找的字符串，再按回车。vi将光标定位在该串下一次出现的地方上。键入n跳到该串的下一个出现处，键入N跳到该串的上一个出现处。



### tail查看文件后面几行，默认尾10行，可实时更新

**-f 关键字，tail 会自动实时更新文件内容。**

`tail -f [logfile]`

`tail -n +100 [logfile]` 

查询从日志文件的第100行开始

`tail -n 100 [logfile] | grep '结果' --color`   

–color是查询结果带颜色



### head查看文件前面几行,默认前10行;



 `head -50 info.log` 

查看info.log文件的前50行。

`head -n -100 [logfile]`

查询日志文件除了最后10行的其他所有日志



### cat查看文件

常用有三大功能:

- 1.一次显示整个文件;
- 2.从键盘创建一个文件。
- 3.将几个文件合并为一个文件。



**-n 可以显示行号**

```
cat -n test.log |tail -n +63820|head -n 20
```

tail -n +63820表示查询63820行之后的日志
head -n 20 则表示在前面的查询结果里再查前20条记录

```
cat -n info.log |grep "关键字" |more
```

这样就分页打印了,通过点击空格键翻页





### sed编辑文件

- 常见使用方法之: sed -n '800,900' info.log

```
查看info.log文件800到900行之间的内容
复制代码
```



### 常见组合使用

> 使用[grep -n 异常 --color info.log ]查询到异常在文件中发生的行数,然后再看前后几十行日志的内容[sed -n '800,900' info.log].





[开发中常用日志搜索技巧]: https://juejin.im/post/5a992bbf6fb9a028bf04c429

[ less命令](https://www.runoob.com/linux/linux-comm-less.html)