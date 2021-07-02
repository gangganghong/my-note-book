# 一些知识收集

## linux命令

### 查找Linux下的大目录

```shell
du -h --max-depth=1
# 搜索当前目录下，超过800M大小的文件
find . -type f -size +800M 
```



## vim

### 改变大小写

```shell
n~ #将光标位置开始的n个字母改变其大小写
g~~ #将光标所在行，全部改变大小写
gUU #将光标所在行，全部改成大写字母
ngUU #将从光标所在行往下n行，小写字母改成大写
nguu #将从光标所在行往下n行，大写字母改成小写
gUw #将光标下的单词改成大写
guw #将光标下的单词改成小写
```



## git

### 设置代理

```shell
# http.https://github.com.proxy 目标网站
# 代理地址 http://127.0.0.1:7890
git config --global http.https://github.com.proxy http://127.0.0.1:7890
```



**在这里，我粗浅的把计算机编程领域的知识分为三个部分：**

- **基础知识**
- **特定领域知识**
- **框架和开发技能**



PingCap 这样的公司做分布式存储



## 计算机课程实验

### 数据库

MIT 6.830

课程地址：http://db.lcs.mit.edu/6.830/sched.php

代码：https://github.com/MIT-DB-Class/simple-db-hw

讲义：https://github.com/MIT-DB-Class/course-info-2018/

mit 的课程 最好的地方在于 lab 自带 test 和打分工具. 做完了心里感觉靠谱一点.

伯克利大学数据库作业实现SimpleDB

https://blog.csdn.net/xpy870663266/article/details/78163423

加州大学伯克利分校 CS186 Introduction to Database System

https://www.bilibili.com/video/av13211143/

伯克利大学课程图谱

https://hkn.eecs.berkeley.edu/courseguides

英文电子书网站

https://1lib.limited/book/841338/456d66?dsource=mostpopular



网络课

https://web.stanford.edu/class/cs244/