# Linux_system_programming
Learning linux system programming notes

Mention:

>
>
>In Linux, Everything is a File.

## 1. 系统目录

一些文件的内容回顾：

bin：存放二进制可执行文件

boot：开机启动程序

dev：存放设备文件

home：存放用户

etc：用户信息和系统配置文件

lib：库文件

root：管理员宿主目录

usr：用户资源管理目录

## 2. 文件和目录操作

```bash
cd - # 返回上一个目录
cd .. # 
cd ~
```

## 3. 软链接和硬链接

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

## 4. find命令

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

注意，`-exec`不会询问`rm -rf`操作，可以将其换成`-ok`

