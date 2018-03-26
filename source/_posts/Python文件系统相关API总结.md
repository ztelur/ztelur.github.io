---
title: Python文件系统相关API总结
date: 2016-07-06 22:50:29
tags: 
    - filesystem
categories:
    - Python
---
&emsp;前段时间毕业，然后希望在上研究生之前实习两个月赚点生活费，可是被公司拒掉了，所以只得回家继续苦练内功。我准备利用暑假的时间学习一下python，主要方便以后写一些自动化脚本。
&emsp;首先，我就开始用python进行文件系统的相关操作，比如文件和文件夹的新添，移动，删除，遍历。在这里总结一下，便于以后查阅。

&emsp;为了节约你的时间，本文内容如下所示：
- os模块与文件相关的API
- shutil模块的相关API
- os.path模块相关API

#### 概述
&emsp;python的模块中 文件相关的有os,os.path,shutil等。其中os是系统服务应用程序接口，os.path实现了一些有用的文件路径相关的接口，shutil则提供一些文件或文件集相关的高级操作。

#### os模块
&emsp;os模块提供很多系统命令，比如文件(目录)的增删改，文件(目录)属性更改，用户相关操作，进程相关操作。
&emsp;下面就列举一下os中常用的API:

- `os.chdir(src)`: 更改当前工作目录。
- `os.getcwd()`: 获得当前工作目录路径的字符串表示。
- `os.chmod(path,mode)`:改变path所指的文件的mode。
- `os.listdir(path)`:返回路径下所有文件的文件名
- `os.mkdir(path[,mode])`:创建一个文件夹，如果路径中有文件夹不存在，就取消创建
- `os.makedirs(path[,mode])`:创建多层级的文件夹，如果路径中有文件夹不存在，直接全部创建出来。
- `os.rename(src,dst)`:文件重命名
- `os.walk(top[,topdown=True,onerror=None,followlinks=False])`:遍历top所指路径，返回(dirpath,dirnames,filenames)的三元组。

#### shutil模块
&emsp;shutil模块主要提供了对文件或文件集的拷贝和删除操作。

- `shutil.copyfileobj(fsrc,fdst[,length])`:把一个文件对象的内容拷贝到另外一个文件对象中。
- `shutil.copyfile(src,dst)`:把src路径所指的文件内容拷贝到dst路径下。不会拷贝文件元信息。
- `shutil.copymode(src,dst)`:拷贝文件的权限
- `shutil.copystat(src,dst)`:拷贝文件的stat，比如权限位，最后访问时间，最后修改时间，和flags等。
- `shutil.copy(src,dst)`:拷贝文件。
- `shutil.move(srcc,dst)`:移动文件。

#### os.path模块
&emsp;os.path模块主要提供文件目录相关的操作。

- `os.path.basename(path)`:返回路径的basename。比如/home/randy/program，返回program。
- `os.path.dirname(path)`:返回当前路径的文件夹名。比如/home/randy/program返回/home/randy。
- `os.path.isfile(path)`:判断path路径是否为标准文件。
- `os.path.isdir(path)`:判断path是否为已经存在的文件夹。
- `os.path.exist(path)`:判断path路径是否存在。
