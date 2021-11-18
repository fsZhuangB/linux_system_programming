# Linux_system_programming



Learning linux system programming notes

Mention:

>
>
>In Linux, Everything is a File.

[TOC]

## 系统目录

一些文件的内容回顾：

bin：存放二进制可执行文件

boot：开机启动程序

dev：存放设备文件

home：存放用户

etc：用户信息和系统配置文件

lib：库文件

root：管理员宿主目录

usr：用户资源管理目录

## 文件和目录操作

```bash
cd - # 返回上一个目录
cd .. # 
cd ~
```

## 软链接和硬链接

### 软链接

```bash
ln -s file file.s # 为file创建其软链接file.s，相当于windows的快捷方式

```

注意点：

1. 注意4个字节的内容是什么：（指访问路径）

   ![](/Users/fszhuangb/Desktop/Screen Shot 2021-11-17 at 20.04.05.png)

2. 为了保证软链接可以任意搬移，注意要用绝对路径创建软链接，相对路径会出问题
3. 软链接的权限是全开放的，rwx全部可以，任意一个人都可以读写软链接，但是是否可以进行读写修改还是要看链接文件的权限。

### 硬链接

```bash
ln file file.hard
ln file file.hard2
```



硬链接计数，修改任意一文件，其余硬链接文件都会发生变化。

硬链接实现：Inode，指针的思想。文件存在于磁盘上，但指针是内存的概念，不能直接用指针。

查看文件Inode：

```bash
stat file.hard
```

读写修改：系统理解为读取相同Inode的文件。

删除具有硬链接的文件：一次只能删一个文件，并将硬链接计数减一。

## find命令

```bash
# -type 按照文件类型搜索：链接文件（七种文件类型）
find ./ -type l

# 按名字寻找
# -maxdepth指定搜索深度
find ./ -maxdepth 1 -name "*.jpg*"

# -size 按文件大小搜索（大于20M小于50M，要注意上限和下限间都需要有size，size默认大小是b，block大小为512字节）
find ./ -size +20M -size -50M

# -atime,-mtime,-ctime
# mtime：按照最后一次属性修改时间
# ctime：按照内容最后被修改时间
# atime：按照最近访问时间

# -exec
# { } 代表结果集
# \;代表语句结束，\是转义字符
find /usr/ -name "*tmp*" -exec ls -l {} \;
```

注意： 

1. `-exec`不会询问`rm -rf`操作，可以将其换成`-ok`

2. 对于`find`命令，无法直接使用管道操作，如：

   ```bash
   find /usr/ -name "*tmp*" | ls -l # 错误的！
   
   ## 可以这样：增加一个xargs 
   find /usr/ -name "*tmp*" | xargs ls -l
   ```

xargs和exec的区别在于性能上，当结果集数量过大时，可以分片映射，故xargs更好一些，具体注意点：

```shell
# 对于文件名中有空格的，xargs会有问题
touch "abc xyz"
# 显示没问题
find ./ -maxdepth 1 -type f -exec ls -l {} \;
-rw-r--r--  1 fszhuangb  staff  0 Nov 17 22:26 .//abc xyz
-rw-r--r--  1 fszhuangb  staff  0 Nov 17 22:26 .//abc
-rw-r--r--  1 fszhuangb  staff  0 Nov 17 22:26 .//xyz
# xargs：将abc拆开了
find ./ -maxdepth 1 -type f | ls -l
-rw-r--r--  1 fszhuangb  staff  0 Nov 17 22:26 .//abc
-rw-r--r--  1 fszhuangb  staff  0 Nov 17 22:26 .//abc
-rw-r--r--  1 fszhuangb  staff  0 Nov 17 22:26 .//xyz
-rw-r--r--  1 fszhuangb  staff  0 Nov 17 22:26 xyz

# 更改：-print0指用null来做拆分依据
find ./ -maxdepth 1 -type f -print0 | -print0 ls -l
```



[Stack overflow解释](https://stackoverflow.com/questions/896808/find-exec-cmd-vs-xargs)

## grep命令

## ps命令

常用命令：

`ps aux | grep` ，检索进程结果集。

如果`ps aux | grep` 命令只返回一条，那么系统中就没有当前该进程。

## 压缩和解压

```shell
tar zcvf test.tar.gz file1 file2 dir
z: gzip方式进行压缩
c: create
f: file
v: 显示压缩过程
j: bzip2方式进行压缩

# 解压，将c替换成x
tar zxvf test.tar.gz
```

真正用来压缩的是`gzip`，`tar`命令是用来打包的，但是`gzip`无法同时压好几个文件。

## 系统编程

C 标准函数和系统函数调用关系。一个 helloworld 如何打印到屏幕。 

![Screen Shot 2021-11-18 at 20.28.30](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-11-18 at 20.28.30.png)

## 文件IO

函数原型：

```shell
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode); 
int close(int fd);
```

常用参数:

O_RDONLY、O_WRONLY、O_RDWR O_APPEND、O_CREAT、O_EXCL、 O_TRUNC、 O_NONBLOCK

创建文件时，指定文件访问权限。权限同时受 umask 影响。结论为:

文件权限 = mode & ~umask 

使用头文件:<fcntl.h>

O_TRUNC参数：截断为0，将文件清空。

open 常见错误 :

1. 打开文件不存在
2. 以写方式打开只读文件(打开文件没有对应权限) 
3. 以只写方式打开目录
