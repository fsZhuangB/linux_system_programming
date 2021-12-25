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

   ![](https://github.com/fsZhuangB/Photos_Of_Blog/blob/master/photos/Screen%20Shot%202021-11-24%20at%2020.00.17.png?raw=true)

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

## 创建修改用户和用户组



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

## Vim使用

### Vim三种工作模式

![Screen Shot 2021-11-20 at 20.18.06](https://github.com/fsZhuangB/Photos_Of_Blog/blob/master/photos/Screen%20Shot%202021-11-24%20at%2020.01.57.png?raw=true)



## 系统编程

C 标准函数和系统函数调用关系。一个 helloworld 如何打印到屏幕。 

![Screen Shot 2021-11-18 at 20.28.30](https://github.com/fsZhuangB/Photos_Of_Blog/blob/master/photos/Screen%20Shot%202021-11-24%20at%2020.04.19.png?raw=true)

## 文件IO

### open() and close()

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

### read() and write()

函数原型：

```c
ssize_t read(int fd, void *buf, size_t count);

ssize_t write(int fd, const void *buf, size_t count);
```

对于read函数：

参数: 

- fd:文件描述符
- buf:存数据的缓冲区
- count:缓冲区大小 

返回值：

- 0:读到文件末尾。
- 成功; > 0 读到的字节数。
- 失败: -1， 设置 errno
- -1: 并且 errno = EAGIN 或 EWOULDBLOCK, 说明不是 read 失败，而是 read 在以非阻塞方 式读一个设备文件(网络文件)，并且文件无数据。

对于write函数：

参数: 

- fd:文件描述符
- buf:待写出数据的缓冲区
- count:数据大小

返回值：

- 成功：写入的字节数。
- 失败： -1， 设置 errno。

read 与 write 函数原型类似。使用时需注意:read/write 函数的第三个参数，一个指读取的最大容量，一个指写入的容量。

read() and write()又被称为无buffer IO

### 实现copy

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>

#define MAX_READ 1024
int main(int argc, char* argv[])
{
    char buffer[MAX_READ + 1];
    ssize_t numRead;

    // get the file name
    char* file1 = *(argv+1); // 或者直接 char* file1 = argv[1];
    char* file2 = *(argv+2);
    int fd1 = open(file1, O_RDWR);
    // O_TRUNC：如果文件已经存在，截断为0
    // O_CREAT后，要记得指定一下权限
    int fd2 = open(file2, O_RDWR | O_CREAT | O_TRUNC, 0644);

    if (fd1 == -1 || fd2 == -1)
    {
        perror("open error");
        exit(1);
    }

    while ((numRead = read(fd1, buffer, MAX_READ)) != 0)
    {
        if (numRead == -1)
        {
            perror("read error");
            exit(1);
        }
        write(fd2, buffer, numRead);
    }

    close(fd1);
    close(fd2);

    return 0;
}
```

用 read/write 实现的 copy 和 fgetc/fputc 实现的 copy 对比:

提出问题：对于write()系统调用和标准库函数`fgetc/fputc`来说，fgetc/fputc调用了系统函数，可能会更慢一点？

![](https://github.com/fsZhuangB/Photos_Of_Blog/blob/master/photos/Screen%20Shot%202021-11-22%20at%2019.54.48.png?raw=true)

**事实上，fget相对系统调用要快。**

原因分析:
 read/write 这块，每次写一个字节，会疯狂进行内核态和用户态的切换，所以非常耗时。 fgetc/fputc，有个自带的缓冲区，大小为4096字节，所以它并不是一个字节一个字节地写，而是内核和用户切换就比较少。

这个被称为**预读入，缓输出机制。** 所以系统函数并不是一定比库函数牛逼，能使用库函数的地方就使用库函数。

标准 IO 函数自带用户缓冲区，系统调用无用户级缓冲。系统缓冲区是都有的。

## 文件描述符

这是一个结构体 PCB 的成员变量 ：file_struct *file 指向的文件描述符表。是一个指向文件描述符的指针，遵循着打开文件描述符表中可用的最小的文件描述符的原则，比如打开一个文件，fd=3，关闭后再打开还是fd =3。

从应用程序使用角度，该指针可理解记忆成一个字符指针数组，下标 0/1/2/3/4...找到文件 结构体。

标准文件描述符：

```c
STDIN_FILENO  0 
STDOUT_FILENO 1 
STDERR_FILENO 2
```

**最大打开文件数目：**

一个进程默认打开文件的个数 1024，命令查看 ulimit -a 查看 open files 对应值。默认为 1024

PCB进程控制块，本质是一个结构体：

![Screen Shot 2021-11-22 at 19.32.33](https://github.com/fsZhuangB/Photos_Of_Blog/blob/master/photos/Screen%20Shot%202021-11-24%20at%2020.05.26.png?raw=true)

## 阻塞与非阻塞

读**常规文件**是不会阻塞的，不管读多少字节，read 一定会在有限的时间内返回。从终 端设备或网络读则不一定，如果从终端输入的数据没有换行符，调用 read 读终端设备就会 阻塞，如果网络上没有接收到数据包，调用 read 从网络读就会阻塞，至于会阻塞多长时间 也是不确定的，如果一直没有数据到达就一直阻塞在那里。同样，写常规文件是不会阻塞的， 而向终端设备或网络写则不一定。

下面的代码演示了读取标准输入的操作：

```C
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

#define MAX_READ 20
int main(void)
{
    int numRead;
    char buffer[MAX_READ + 1];
    if ((numRead = read(STDIN_FILENO, buffer, MAX_READ)) == -1)
        printf("Wrong!");
    buffer[MAX_READ] = '\0';
    write(STDOUT_FILENO, buffer, numRead);

    return 0;
}

(base) ➜  Documents ./a.out 
ds
ds
```



在文件属性中，有一个重要属性，为`O_NONBLOCK`，可以读取终端文件`/dev/tty -- 终端文件`。，将该文件从默认阻塞读取的方式改为非阻塞读取，下面代码演示了如何修改：

通过读取终端文件进行修改：

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>

#define MAX_READ 20
int main(void)
{
    int numRead;
    char buffer[MAX_READ + 1];
    int fd, n;
    fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);

tryagain:
    n = read(fd, buffer , MAX_READ+1);
    if (n < 0)
    {
        if (errno != EAGAIN)
        {
            perror("wrong!");
            exit(1);
        }
        else {
            write(STDOUT_FILENO, "try_again\n", strlen("try_again\n"));
            sleep(2);
            goto tryagain;
        }
    }
    write(STDOUT_FILENO, buffer, n);

    close(fd);
    return 0;
}

```

总结 read 函数返回值:

1. 返回非零值: 实际 read 到的字节数
2. 返回-1: 
   1. errno != EAGAIN (或!= EWOULDBLOCK) read 出错
   2. errno == EAGAIN (或== EWOULDBLOCK) 设置了非阻塞读，并且没有 数据到达。

3. 返回 0:读到文件末尾

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>

#define MAX_READ 20
int main(void)
{
    int numRead;
    char buffer[MAX_READ + 1];
    int fd, n;
    fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);

tryagain:
    n = read(fd, buffer , MAX_READ+1);
    if (n < 0)
    {
        if (errno != EAGAIN)
        {
            perror("wrong!");
            exit(1);
        }
        else {
            write(STDOUT_FILENO, "try_again\n", strlen("try_again\n"));
            sleep(2);
            goto tryagain;
        }
    }
    write(STDOUT_FILENO, buffer, n);

    close(fd);  
    return 0;
}

```

有关EAGAIN`和`EWOULDBLOCK错误码的说明：

## fcntl函数

改变一个**已经打开**的文件的 访问控制属性。 重点掌握两个参数的使用，F_GETFL 和 F_SETFL。

fcntl:
 int (int fd, int cmd, ...)

fd 文件描述符
 cmd 命令，决定了后续参数个数

位或操作：|=

根据二进制位图（节省内存）来说明：存在的flags为1，不存在的为1，O_NONBLOCK利用位或操作将其设置为1。

![Screen Shot 2021-11-24 at 20.49.40](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-11-24 at 20.49.40.png)

```c
// fcntl的主要使用：    
int numRead;
    char buffer[MAX_READ + 1];
    int flags, n;
    // 获取stdin属性信息
    flags = fcntl(STDIN_FILENO, F_GETFL);
    if (flags == -1)
    {
        perror("get error");
        exit(1);
    }
    // 位或操作，为文件增加NON_BLOCK文件属性
    flags |= O_NONBLOCK;

    int ret = fcntl(STDIN_FILENO, F_SETFD, flags);
    if (ret == -1)
    {
        perror("set error");
        exit(1);
    }

```

获取文件状态: F_GETFL 设置文件状态: F_SETFL

## lseek函数

函数原型：

​	off_t lseek(int fd, off_t offset, int whence);

参数:

-  fd:文件描述符
- offset: 偏移量，就是将读写指针从 whence 指定位置向后偏移 offset 个单位
- whence:起始偏移位置: SEEK_SET/SEEK_CUR/SEEK_END

返回值:

- 成功:较起始位置偏移量
- 失败:-1 errno

在Linux 中可使用系统函数 lseek 来修改文件偏移量，也就是文件的读写位置。

每个打开的文件都记录着当前读写位置，打开文件时读写位置是 0，表示文件开头，通常来说，读写多少个字节就会将读写位置往后移多少个字节。但是有一个例外，如果以 `O_APPEND` 方式打开，每次写操作都会在文件末尾追加数据，然后将读写位置移到新的文件末尾。lseek 和标准 I/O 库的 fseek 函数类似，可以移动当前读写位置(或者叫偏移量)。



**注意，文件“读”和“写”使用同一偏移位置。**意思就是如果写一个句子到空白文件，然后去读取刚才写那个文件，如果不调整光标位置，是读取不到内容的，因为读写指针在内容的末尾。

代码验证一下：

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>
#include <sys/stat.h>

int main(int argc, char *argv[])
{
    int fd, n;
    char msg[] = "test for lseek";
    fd = open("lseek.txt", O_RDWR | O_CREAT, 0644);

    if (fd < 0)
    {
        perror("open error");
        exit(1);
    }
    write(fd, msg, strlen(msg));
    //lseek(fd, 0, SEEK_SET);

    char ch;
    while ((n = read(fd, &ch, 1))) 
    {
        if (n < 0)
        {
            perror("read error");
            exit(1);
        }
        write(STDOUT_FILENO, &ch, n);
    }

    close(fd);
    return 0;

}
```

在上面的代码中，如果注释掉lseek，读是读取不到内容的，因为文件的读写指针在文件最后， `lseek(fd, 0, SEEK_SET);`代码的含义是将文件偏移量设为0，即回到文件开头。

应用场景：

1. 获取文件大小，这里会用到SEEK_END参数。

   ```C
   
   int main(int argc, char *argv[])
   {
       int fd, n;
       fd = open(argv[1], O_RDWR);
       if (fd == -1)
       {
           perror("bad read");
           exit(1);
       }
       int len = lseek(fd, 0, SEEK_END);
       if (len < 0)
       {
           perror("bad lseek");
           exit(1);
       }
       printf("The file's size is %d", len);
   
       close(fd);
       return 0;
   
   }
   ➜  RemoteWorking git:(master) ✗ ./testsize test.c
   The file's size is 484#              
   ```

   

2. 拓展文件大小（要想文件大小真正被拓展，必须引起IO操作）

   上面的代码只需要改动一行就可以拓展文件大小：

   ```c
   
   int main(int argc, char *argv[])
   {
       int fd, n;
       fd = open(argv[1], O_RDWR);
       if (fd == -1)
       {
           perror("bad read");
           exit(1);
       }
     // 这里改成88，将其拓展成200
       int len = lseek(fd, 88, SEEK_END);
       if (len < 0)
       {
           perror("bad lseek");
           exit(1);
       }
       printf("The file's size is %d", len);
   
       close(fd);
       return 0;
   
   }
   ```

   但是问题来了：

   ```c
   ➜  RemoteWorking git:(master) ✗ ./testsize main.cpp 
   The file's size is 200#                                                                                                                                                                   
   ➜  RemoteWorking git:(master) ✗ ll
   total 64K
   -rw-r--r--. 1 root root   14 Nov 29 21:34 lseek.txt
   -rw-r--r--. 1 root root  112 Nov 26 22:27 main.cpp
   ```

   虽然程序显示是拓展了，但是ll一下，发现文件的实际大小还是112，但是如果用lseek来扩展文件大小，必须引起 IO 才行，所以至少要用write写入一个字符。这里问题在于没有引起真正的IO操作：

   ```c
   
   int main(int argc, char *argv[])
   {
       int fd, n;
       fd = open(argv[1], O_RDWR);
       if (fd == -1)
       {
           perror("bad read");
           exit(1);
       }
       int len = lseek(fd, 87, SEEK_END);
       if (len < 0)
       {
           perror("bad lseek");
           exit(1);
       }
       write(fd, "&", 1);
       printf("The file's size is %d", len);
   
       close(fd);
       return 0;
   
   }
   ```

   编译运行一下，发现被写入了字符。

   ![Screen Shot 2021-11-30 at 09.23.41](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-11-30 at 09.23.41.png)

   拓展文件大小会造成“文件空洞”，填入一大串`\0`，而不会真的有内容，也可以使用truncate函数直接拓展文件，简单粗暴：

   ```c
   int ret = truncate("dict.cp", 250);
   ```

   

​		



## 传入参数和传出参数

传入参数:

1. 指针作为函数参数。
2. 通常有 const 关键字修饰 
3. 指针指向有效区域， 在函数内部做读操作。

例如：

```c
 char * stpcpy(char * dst, const char * src);
```

第二个参数就是传入参数。

传出参数:

1. 指针作为函数参数。
2. 在函数调用之前，指针指向的空间可以无意义，但必须有效。 
3. 在函数内部，做写操作。
4. 函数调用结束后，充当函数返回值。

例如：

```c
 char * stpcpy(char * dst, const char * src);
```

第一个dst参数就是传出参数。

传入传出参数:

1. 指针作为函数参数。
2.  在函数调用之前，指针指向的空间有实际意义。 
3. 在函数内部，先做读操作，后做写操作。
4. 函数调用结束后，充当函数返回值。

例如：

``` c
char *
 strtok_r(char *restrict str, const char *restrict sep, char **restrict lasts);
```

`char **restrict lasts`就是传入传出参数。

## 文件存储

文件有两部分储存，Inode和dentry

硬链接的本质是为文件创建不同的dentry：

![Screen Shot 2021-11-28 at 09.30.16](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-11-28 at 09.30.16.png)

### Inode

其本质为结构体，存储文件的属性信息。如:权限、类型、大小、时间、用户、**盘块位置**......也叫作文件属性管理结构，大多数的 inode 都存储在磁盘上。

少量常用、近期使用的 inode 会被缓存到内存中。

### dentry

目录项

## stat函数

获取文件属性，(从 inode 结构体中获取) 

函数原型：

int stat(const char *path, struct stat *buf);

成返回 0;失败返回-1 设置 errno 为恰当值。 参数 2:inode 结构体指针 (传出参数)

参数 1:文件名

文件属性将通过传出参数返回给调用者。

**问题：**符号链接穿透问题

通过stat函数查看文件mode时，stat会穿透符号链接，直接显示源文件的类型。 可以改用lstat函数查看

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>
#include <sys/stat.h>

int main(int argc, char *argv[])
{
    struct stat buf;
    int ret = stat(argv[1], &buf);
    if (ret == -1)
    {
        perror("error!");
        exit(1);
    }
    if (S_ISDIR(buf.st_mode))
    {
        printf("It's dir");
    }
    else if (S_ISREG(buf.st_mode))
    {
        printf("It's regular file");
    }
    else
    {
        printf("bad!");
    }

    return 0;

}
```

穿透符号链接:stat:会;lstat:不会

 ## link函数

为文件创建硬链接，硬链接数就是 dentry 数目。

函数原型:
` int link(const char *oldpath, const char *newpath)`

## unlink函数

用这个结合link函数来实现 mv，用 oldpath 来创建 newpath，最后删除 oldpath 就行。



无dentry对应的文件，会被`择机释放`。

删除文件，只是让文件具备了被释放的条件。**unlink** 函数的特征:清除文件时，如果文件的硬链接数到 0 了，没有 dentry 对应，但该 文件仍不会马上被释放。**要等到所有打开该文件的进程关闭该文件**，系统才会挑时间将该文 件释放掉。 下面的程序演示了这一个特性：

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main(void)
{
    int fd, ret;
    char *p = "Writing something";
    char *p1 = "Writing another";
    fd = open("temp.txt", O_CREAT | O_RDWR | O_TRUNC, 0644);
    if (fd < 0)
    {
        perror("Open error");
        exit(1);
    }
    // 只要一打开文件就将其unlink，就可以给予文件被unlink的条件，而后面的write都可以完成。
    unlink("temp.txt");

    ret = write(fd, p, strlen(p));
    if (ret < 0)
    {
        perror("Writing error");
        exit(1);
    }
    printf("Now writing:");
    ret = write(fd, p1, strlen(p1));
    getchar();
    if (ret < 0)
    {
        perror("Writing error");
        exit(1);
    }
    // 如果出错，即使程序崩溃，temp.txt也只能被删除，所以刚刚open文件，就要将文件unlink
    p[3] = "h"; // 这里会发生段错误
    close(fd);

    return 0;
}
```

上面的程序，在程序中加入段错误成分，段错误在 unlink 之前，由于发生段错误，程序后续删除 temp.txt 的 dentry 部分就不会再执行，temp.txt 就保留了下来，这是不科学的。

解决办法是检测 fd 有效性后，立即释放 temp.txt，由于进程未结束，虽然 temp.txt 的硬链接数已 经为 0，但还不会立即释放，仍然存在，要等到程序执行完才会释放。这样就能避免程序出错导致临 时文件保留下来。
 因为文件创建后，硬链接数立马减为 0，即使程序异常退出，这个文件也会被清理掉。这时候的内容 是写在内核空间的缓冲区。

如果没有隐式回收的话，这个文件描述符会保留， 多次出现这种情况会导致系统文件描述符耗尽。所以隐式回收会在程序结束时收回它打开的文件使用 的文件描述符。

## 隐式回收

当进程结束运行时，所有该进程打开的文件会被关闭，申请的内存空间会被释放。系统 的这一特性称之为隐式回收系统资源。 

## 文件目录权限操作

一共三个比较重要的函数：

**opendir函数**

DIR * opendir(char *name);
**closedir函数**

 int closedir(DIR *dp);

**readdir函数**

 struct dirent *readdir(DIR * dp);

```c
   struct dirent {
       inode;
       char dname[256];
   }
```

 通过以上函数，实现简单的ls命令:

```c
int main(int argc, char* argv[])
{
    DIR * dp;
    struct dirent* dirp;
    dp = opendir(argv[1]);
    if (dp == NULL)
    {
        perror("Open dir error!");
        exit(1);
    }

    while (dirp = readdir(dp))
    {
        printf("%s\t", dirp->d_name);
    }
    printf("\n");
    closedir(dp);
}
```



### 实现递归遍历目录

ls -R

主要思路：

1. 判断命令行参数：
   - 什么都不输入，`argc == 1`，那么直接读取默认当前目录。
   - 使用isFile函数，判断是否为文件，或者为目录。
2. isFile函数：
   - 首先读取文件status，判断是文件还是目录
   - 如果是文件，那么将文件的s_size读取出来
   - 如果是目录，调用read_dir函数读取目录
3. read_dir函数：
   - 使用opendir，readdir和closedir对目录进行读取
   - 需要注意的是对于`.`目录和`..`目录的特殊处理，防止造成无限递归
   - 其次要注意读取到的文件/目录不能直接传入，而是要利用路径进行字符串拼接，传入绝对路径。

```c
// 完整示例代码
void isFile(char *name);

void read_dir(char* dir)
{
    char buffer[256];
    DIR *dp = opendir(dir);
    struct dirent* dren;
    if (dp == NULL)
    { 
        perror("open dir error");
        exit(1);
    }
    while ((dren = readdir(dp)) != NULL)
    {
        if (strcmp(dren->d_name, ".") == 0 || strcmp(dren->d_name, "..") == 0){
            continue;
        }
        sprintf(buffer, "%s/%s", dir, dren->d_name);
        isFile(buffer);
    }
    closedir(dp);

}
// 查看是否为文件
 void isFile(char * name)
 {
     int ret = 0;
     struct stat sb;
     int fd = stat(name, &sb);
     if (fd < 0)
     {
         perror("Wrong fd!");
         exit(1);
     }
     if (S_ISDIR(sb.st_mode))
     {
        read_dir(name);
     }
     printf("%s\t%ld\n", name, sb.st_size);

 }
int main(int argc, char* argv[])
{
    if (argc == 1)
    {
        isFile(".");
    }
    else 
    {
        isFile(argv[1]);
    }
    return 0;

}
```

## 重定向：dup，dup2

## dup

功能:文件描述符拷贝。 使用现有的文件描述符，拷贝生成一个新的文件描述符，且函数调用前后这个两个文件

描述符指向同一文件。
 int dup(int oldfd); 成功:返回一个新文件描述符;失败:-1 设置 errno 为相应值

当作一个保存文件副本的作用，

## dup2

浅拷贝的感觉，拷贝一个旧文件描述符到新的文件描述符中。用来做重定向，将写入新文件描述符的文字写入旧文件描述符。

用 dup2 将 fd1 复制给了 fd2，于是在对 fd2 指向的文件进行写操作时，实际上就是对 fd1 指向的 hello.c 进行写操作。
 这里需要注意一个问题，由于 hello.c 和 hello2.c 都是空文件，所以直接写进去没关系。但如果 hello.c 是非空的，写进去的内容默认从文件头部开始写，会覆盖原有内容。

dup2 也可以用于标准输入输出的重定向。

![Screen Shot 2021-12-05 at 19.56.21](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-12-05 at 19.56.21.png)



![Screen Shot 2021-12-05 at 20.11.42](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-12-05 at 20.11.42.png)



## fcntl实现dup2功能

F_DUPFD参数，用第三个参数来重定向，第三个参数可以指定文件描述符。



## gcc的编译步骤

预处理->编译->汇编->链接

## gcc参数说明



## CPU管理

## 第二节：进程管理

### 进程与线程

进程相关概念

进程，一个具有独立功能的程序在一定数据集合上的一次动态执行的过程。

![进程定义](https://img-blog.csdnimg.cn/20200311100415775.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2l3YW5kZXJ1,size_16,color_FFFFFF,t_70)

一个进程应该包括：

程序的代码
-程序处理的数据
-程序计数器的值，指示下一条将运行的指令
-一组通用的寄存器的当前值，堆，栈
-一组系统资源(如打开的文件)
总之，进程包含了正在运行的一个程序的所有状态信息。



程序是产生进程的基础
-程序的每次运行构成不同的进程
-进程是程序功能的体现
-通过多次执行，一个程序可对应多个进程；通过调用关系，一个进程可包括多个程序。

## 进程的结构

使用PCB（Process Control Block）去维护，保存各种状态信息。



**程序和进程的区别？**程序，是指编译好的二进制文件，在磁盘上，**不占用系统资源**(cpu、内存、打开的文件、 设备、锁....)

进程，是一个抽象的概念，与操作系统原理联系紧密。进程是活跃的程序，占用系统资 源。在内存中执行。(程序运行起来，产生一个进程)

程序 → 剧本(纸) 进程 → 戏(舞台、演员、灯光、道具...) 同一个剧本可以在多个舞台同时上演。同样，同一个程序也可以加载为不同的进程(彼

此之间互不影响)
 如:同时开两个终端。各自都有一个 bash 但彼此 ID 不同。

### 并发

并发，在操作系统中，一个时间段中有多个进程都处于已启动运行到运行完毕之间的状 态。但，任一个时刻点上仍只有一个进程在运行。

unix唯一的分时复用操作系统

多道程序设计

在计算机内存中同时存放几道相互独立的程序，它们在管理程序控制之下，相互穿插的 运行。多道程序设计必须有硬件基础作为保证。

**时钟中断**即为多道程序设计模型的理论基础。 并发时，任意进程在执行期间都不希望 放弃 cpu。因此系统需要一种强制让进程让出 cpu 资源的手段。时钟中断有硬件基础作为保 障，对进程而言不可抗拒。 操作系统中的中断处理函数，来负责调度程序执行。

并行

单道程序设计：

串行，所有进程一个一个排对执行。若 A 阻塞，B 只能等待，即使 CPU 处于空闲状态。

### CPU和MMU

MMU虚拟内存映射单元的图



### 虚拟内存和物理内存的映射关系

程序运行起来之后，产生0-4G地址空间，在虚拟内存中（32位系统）

地址空间为虚拟地址和可用范围（4G空间为可用的，而不是实际使用的）。



MMU就是虚拟内存的映射，将虚拟翻译成物理内存，大小为4k

MMU可以把内存空间不连续的映射为连续的

### PCB内容

\* 进程 id。系统中每个进程有唯一的 id，在 C 语言中用 pid_t 类型表示，其实就是一个 非负整数。

\* 进程的状态，有就绪、运行、挂起、停止等状态。 

\*进程切换时需要保存和恢复的一些 CPU 寄存器。 

\* 描述虚拟地址空间的信息。
\* 描述控制终端的信息。

\* 当前工作目录(Current Working Directory)。
 \* umask 掩码。
 \* 文件描述符表，包含很多指向 file 结构体的指针。 * 和信号相关的信息。
 \* 用户id和组id。

### 环境变量

```bash
 echo $PATH
 echo $SHELL
 echo $TERM
 echo $HOME
 echo $LANG
 # 查看 所有环境变量
 env
```

## 进程控制

### fork()函数

父进程返回：子进程pid

子进程返回：0

子进程将父进程的代码复制一遍

![Screen Shot 2021-12-22 at 13.21.53](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-12-22 at 13.21.53.png)

下面的代码会打印两遍：

fork 之前的代码，父子进程都有，但是只有父进程执行了，子进程没有执行，fork 之后的代码，父子进程都有机会执行。

```c
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(void)
{
    printf("before fork()\n");
    printf("before fork()\n");
    printf("before fork()\n");
    printf("before fork()\n");

    pid_t pid = fork();
    if (pid == -1)
    { perror("wrong fork!"); exit(1); }
    else if (pid == 0)
    {
        printf("I am child----");
        printf("my id is %d\n", pid);
    }
    else 
    {
        printf("I am parent----");
        printf("my id is %d\n", pid);

    }
    printf("end of file------\n");
}
// 结果：
before fork()
before fork()
before fork()
before fork()
I am parent----my id is 1080
end of file------
I am child----my id is 0
end of file------
```

### getpid和getppid

加入两个函数

```c
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(void)
{
    printf("before fork()\n");
    printf("before fork()\n");
    printf("before fork()\n");
    printf("before fork()\n");

    pid_t pid = fork();
    if (pid == -1)
    { perror("wrong fork!"); exit(1); }
    else if (pid == 0)
    {
        printf("I am child----my id is %d, \
                my id from getpid is %d, \
                my parent id from getppid is %d\n"\
                , pid, getpid(), getppid());
    }
    else 
    {
        printf("I am parent----child id is %d, \
                my id from getpid is %d, \
                my parent id from getppid is %d\n"\
                , pid, getpid(), getppid());
    }
    printf("end of file------\n");
}

➜  chapter2-process git:(master) ✗ ./a.out 
before fork()
before fork()
before fork()
before fork()
I am parent----child id is 1585,                 my id from getpid is 1584,                 my parent id from getppid is 125
end of file------
I am child----my id is 0,                 my id from getpid is 1585,                 my parent id from getppid is 1584
end of file------
```



### 问题：循环创建n个子进程

![Screen Shot 2021-12-22 at 15.41.58](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-12-22 at 15.41.58.png)

从上图我们可以很清晰的看到，当 n 为 3 时候，循环创建了**(2^n)-1** 个子进程，而不是 N 的子进程。需要在循环的过程，保证子进程不再执行 fork ，因此当(fork() == 0)时，子进程 应该立即 break;才正确。

正确代码：

```C

int main(void)
{
    for (int i = 0; i < 5; ++i)
    {
        pid_t pid = fork();
        if (pid == 0)
        { break; }
        else
        {
            printf("I am parent:%d\n", pid);
        }
    }
}
```

可以在循环外打印次序，但是不是顺序执行：

```C
// 循环创建子进程
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>
int main(void)
{
    int i;
    pid_t pid;
    for (i = 0; i < 5; ++i)
    {
        pid = fork();
        if (pid == 0)
        { break; }
    }
    if (i == 5)
    { printf("I am parent\n"); }
    else {
        printf("I am %dth child\n", i+1);
    }


➜  chapter2-process git:(master) ✗ ./nchild 
I am 1th child
I am 2th child
I am 3th child
I am parent
I am 4th child
I am 5th child
```

现在还有两个问题：

**一个就是包括父进程在内的所有进程不是按顺序出现**，多运行几次，发现是随机序列出现的。这主要因为，对操作系统而言，这几个子进程几乎是同时出现的，它们和父进程一起争夺 cpu，谁抢到， 谁打印，所以出现顺序是随机的。

**第二问题就是终端提示符混在了输出里**，这个是因为，loop_fork 是终端的子进程，**一旦 loop_fork 执行完，也就是父进程执行完毕，**终端就会打印提示符。就像之前没有子进程的程序，一旦执行完，就出现了终端 提示符。这里也就是这个道理，loop_fork 执行完了，终端提示符出现，然而 loop_fork 的子进程还 没执行完，所以输出就混在一起了。



解决方法，让父进程最后结束即可让他们顺序执行，**加一个wait(0)**，让所有父进程等待其后面的子进程先执行：

```c
// 循环创建子进程
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>
int main(void)
{
    int i;
    pid_t pid;
    for (i = 0; i < 5; ++i)
    {
        pid = fork();
        if (pid == 0)
        { break; }
        else {
            wait(0);
        }
    }
    if (i == 5)
    { printf("I am parent\n"); }
    else {
        printf("I am %dth child\n", i+1);
    }
}

➜  chapter2-process git:(master) ✗ ./nchild
I am 1th child
I am 2th child
I am 3th child
I am 4th child
I am 5th child
I am parent
```

通过视频就可以看出来，最先允许的loop_fork进程最后打印了“I am parent”。

### 进程共享内容

父子进程之间在 fork 后。有哪些相同，那些相异之处呢?

**刚 fork 之后:**

父子相同处: 全局变量、.data（数据段）、.text（代码段）、栈、堆、环境变量、用户 ID、**宿主目录、进程工作目录、信号处理方式...**后面三个不常用

父子不同处: 1.进程 ID 2.fork 返回值 3.父进程 ID 4.进程运行时间 5.闹钟(定时器) 6.未决信号集

似乎，子进程复制了父进程 0-3G 用户空间内容，以及父进程的 PCB，但 pid 不同。真 的每 fork 一个子进程都要将父进程的 0-3G 地址空间完全拷贝一份，然后在映射至物理内存 吗?

当然不是!父子进程间遵循读时共享写时复制的原则。这样设计，无论子进程执行父进 程的逻辑还是执行自己的逻辑都能节省内存开销。

**父子进程真正共享：**

1. 文件描述符
2. mmap映射区

#### 父子进程是否共享全局变量？

注意一点：

父子进程是不共享全局变量！原因是**读时共享写时复制**原则！

### exec 函数族

exec函数不能够返回

fork 创建子进程后执行的是和父进程相同的程序(但有可能执行不同的代码分支)，子 进程往往要调用一种 exec 函数以执行另一个程序。当进程调用一种 exec 函数时，该进程的 用户空间代码和数据完全被新程序替换，从新程序的启动例程开始执行。调用 exec 并不创 建新进程，所以调用 exec 前后该进程的 id 并未改变。

**将当前进程的.text、.data 替换为所要加载的程序的.text、.data，然后让进程从新的.text 第一条指令开始执行，但进程 ID 不变，换核不换壳。**

重点看两个：

#### execlp函数

注意最后参数要加上哨兵`NULL`，因为是一个变参：

同时，可变参数那里，是从 argv[0]开始计算的。

```C
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>
int main(void)
{
    pid_t pid = fork();
    if (pid == -1)
    { perror("wrong fork!"); exit(1); }
    else if (pid == 0)
    {
        // 注意execlp的第二个参数才是argv[0]，所以要多传一个ls进去
        // 错误写法：execlp("ls", "-l","-h" NULL);
        execlp("ls", "ls", "-l","-h" NULL);
        // 如果出错才会返回
        perror("exec error!");
        exit(1);
    }
    else 
    {
        wait(0);
        printf("I am parents: %d\n", pid);
    }
}

➜  chapter2-process git:(master) ✗ ./execfork 
total 48
-rwxr-xr-x. 1 root root 8608 Dec 22 15:27 a.out
-rw-r--r--. 1 root root  450 Dec 22 15:57 create_n_child.c
-rwxr-xr-x. 1 root root 8520 Dec 22 16:58 execfork
-rw-r--r--. 1 root root  561 Dec 22 16:57 fork_exec.c
-rw-r--r--. 1 root root  841 Dec 22 15:27 myfork.c
-rwxr-xr-x. 1 root root 8440 Dec 22 15:57 nchild
I am parents: 3545
```

下面使用 execl 来让子程序调用自定义的程序。
 int execl(const char *path, const char *arg, ...) 这里要注意，和 execlp 不同的是，第一个参数是路径，不是文件名。 这个路径用相对路径和绝对路径都行。

使用execl函数：

只是第一个参数不一样，是path：

```c
int main(void)
{
    pid_t pid = fork();
    if (pid == -1)
    { perror("wrong fork!"); exit(1); }
    else if (pid == 0)
    {
        // 注意execlp的第二个参数才是argv[0]，所以要多传一个ls进去
        //execlp("ls", "ls", "-l", NULL);
        execl("./nchild", "./nchild", NULL);
        // 如果出错才会返回
        perror("exec error!");
        exit(1);
    }
    else 
    {
        wait(0);
        printf("I am parents: %d\n", pid);
        
    }
}
```



### exec 函数族特性

写一个程序，使用 execlp 执行进程查看，并将结果输出到文件里。 要用到 open, execlp, dup2

代码如下：

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>

int main(void)
{
    int fd = open("ps.out", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0)
    {
        perror("wrong open!");
        exit(1);
    }
    dup2(fd, STDOUT_FILENO);
    execlp("ps", "ps", "aux", NULL);
    perror("exec wrong");
    
    return 0;
}
```



#### execvp用法

只是把后面变参换成了

`char* argv[] = {"ls", "-l", "-h", NULL}`

### exec 函数族一般规律

exec 函数一旦调用成功，即执行新的程序，不返回。只有失败才返回，错误值-1，所以通常我们直接在 exec 函数调用后直接调用 perror()，和 exit()，无需 if 判断。

- l(list) 命令行参数列表
- p(path) 搜索 file 时使用 path 变量
- v(vector) 使用命令行参数数组
- e(environment) 使用环境变量数组，不适用进程原有的环境变量，设置新加载程序运行的环境变量

事实上，只有 execve 是真正的系统调用，其他 5 个函数最终都调用 execve，是库函数，所以 execve 在 man 手册第二节，其它函数在 man 手册第 3 节。

### 孤儿进程和僵尸进程

**孤儿进程:**
父进程先于子进终止，子进程沦为“孤儿进程”，会被 init 进程领养。

demo演示：



**僵尸进程:**

死亡以后没来得及回收的为**僵尸态**

子进程终止，父进程尚未对子进程进行回收，在此期间，子进程为“僵尸进程”。 kill 对其 无效。这里要注意，每个进程结束后都必然会经历僵尸态，时间长短的差别而已。

子进程终止时，子进程残留资源 PCB 存放于内核中，PCB 记录了进程结束原因，进程回收就是回收 PCB。

回收子进程方式：

​	回收僵尸进程，得 kill 它的父进程，让孤儿院去回收它。

​	kill命令无法杀死僵尸进程

demo演示：

### wait函数回收子进程

父进程调用 wait 函数可以回收子进程终止信息。该函数有三个功能: 

1. 阻塞等待子进程退出
2. 回收子进程残留资源
3. 获取子进程结束状态(退出原因)。

wait 函数: 回收子进程退出资源， 阻塞回收任意一个。 

`pid_t wait(int *status)`
参数:(传出) status用来回收进程的状态。
返回值:成功: 回收进程的pid

失败: -1， errno



当进程终止时，操作系统的隐式回收机制会:

1. 关闭所有文件描述符
2. 释放用户空间分配的内存。内核的 PCB 仍存在。其中保存该进程的退出状态。(正常终止→退出值;异常 终止→终止信号)

可使用 wait 函数传出参数 status 来保存进程的退出状态。借助宏函数来进一步判断进程终止的具体原因。

wait函数与宏的使用：*很重要！*

```c
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>
int main(void)
{
    pid_t pid, wpid;
    int status;
    pid = fork();
    if (pid == 0)
    {
        printf("I am child %d, sleep 5s,\n", getpid());
        sleep(10);
        printf("Child die-----\n");
        printf("The status is %d\n", status);
        return 73;
    }
    else if (pid > 0)
    {
      // wpid = wait(NULL) , 指不关心子进程结束原因
        wpid = wait(&status);
        if (wpid == -1)
        {
            perror("wait error\n");
            exit(1);
        }
        printf("parent wait finished\n");
        // 为真，说明子进程正常终止
        if (WIFEXITED(status))
        {
            printf("Child exit with %d\n ", WEXITSTATUS(status));
        }
        // 信号终止，所有程序异常终止的情况都是信号
        if(WIFSIGNALED(status))
        {
            printf("Child exit with %d\n", WTERMSIG(status));
        }
    }
    else 
    {
        perror("fork error\n");
        exit(1);
    }
    return 0;
}

```

- WIFEXITED(status) 为非 0 → 进程正常结束
  -  WEXITSTATUS(status) 如上宏为真，使用此宏 → 获取进程退出状态 (exit 的参数)
-  WIFSIGNALED(status) 为非 0 → 进程异常终止
  - WTERMSIG(status) 如上宏为真，使用此宏 → 取得使进程终止的那个信号的编号。 
- WIFSTOPPED(status) 为非 0 → 进程处于暂停状态
  - WSTOPSIG(status) 如上宏为真，使用此宏 → 取得使进程暂停的那个信号的编号。
  -  WIFCONTINUED(status) 为真 → 进程暂停后已经继续运行



### waitpid函数，非常重要！

waitpid函数可以指定一个进程进行回收，其函数原型如下：

```c
pid_t waitpid(pid_t pid, int *stat_loc, int options);
```

第一个参数的具体内容如下：

1. \>0指某一个子进程的pid

   ```c
   waitpid(pid, &status, options)
   ```

   

2. 无差别回收，**-1** 回收任意子进程(相当于 **wait**)

   ```c
   waitpid(-1, &status, 0) == wait(&status);
   ```

3. 0:指同组的子进程

返回值参数如下：

1. \>0指成功回收的子进程pid
2. 0 : 函数调用时， 参 3 指定了 WNOHANG， 并且，没有子进程结束，没有导致没有回收到子进程
3. -1: 失败。errno

第三个参数可以通过WNOHANG来指定非阻塞回收。

在回收子进程的过程中，要注意子进程组的概念，对于父进程fork出来的子进程，都属于同一个组。

一次 wait/waitpid 函数调用，只能回收一个子进程。上一个例子，父进程产生了 5 个子进程， wait 会随机回收一个，捡到哪个算哪个。

老师举例的错误代码如下：

```c
// 循环创建子进程
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>
int main(void)
{
    int i;
    pid_t pid, wpid;
    for (i = 0; i < 5; ++i)
    {
        pid = fork();
        if (pid == 0)
        { 
            if (i == 2)
            {
                pid = getpid();
                //printf("------pid = %d\n", pid);
            }
            break;
        }
        // else {
        //     wait(0);
        // }
    }
    if (5 == i)
    { 
        //sleep(10);
        //waitpid(-1, NULL, WNOHANG);
        wait(NULL);
        //printf("------in parent , before waitpid, pid= %d\n", pid);
       // wpid = waitpid(pid, NULL, 0); //指定一个进程回收
        if (wpid == -1)
        {
            perror("wrong wait pid!\n");
            exit(1);
        }
        // wpid如果返回0，说明没有回收到
        printf("I am parent, wait a child finished:%d\n", wpid); 
    }
    else {
        sleep(i);
        printf("I am %dth child, my pid is %d\n", i+1, getpid());
    }
}
```

老师的判断语句是在if(pid == 0)时，再用i==2获取第三个子进程，但是此时的pid的值属于子进程空间的变量，父进程并不能读取到，所以应该在父进程空间内对于pid进行赋值，使得父进程获取到子进程pid：

```c
    for (i = 0; i < 5; ++i)
    {
        pid = fork();
        if (pid == 0)
        { 
            break;
        }
        if (i == 2)
        {
            tmppid = pid;
        }
    }
```

![Screen Shot 2021-12-25 at 15.31.54](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-12-25 at 15.31.54.png)

改动之后，可以看出来没问题了，获取到了指定的pid

waitpid的回收实现有两种，一个是阻塞等待回收指定进程，一个是非阻塞，但是用 sleep 延时父进程，以保证待回收的指定子进程已经执行结束。上面这个代码使用的阻塞回收：

```c
wpid = waitpid(tmpid, NULL, 0); //指定一个进程回收, 阻塞回收
```

这个方案的问题在于终端提示符会和输出混杂在一起。下面使用非阻塞回收 +延时的方法，这样终端提示符就不会混在输出里：

```C
sleep(10);
printf("------in parent , before waitpid, pid= %d\n", tmppid);
wpid = waitpid(tmppid, NULL, WNOHANG); // 非阻塞 + 延时的方法

(base) ➜  leetcode-revise (master) ✗ ./test 
I am 1th child, my pid is 78964
I am 2th child, my pid is 78965
I am 3th child, my pid is 78966
I am 4th child, my pid is 78967
I am 5th child, my pid is 78968
------in parent , before waitpid, pid= 78966
I am parent, wait a child finished:78966
```

### waitpid回收多个子进程



为了让waitpid回收多个子进程，可以使用while循环回收，

阻塞方式回收：

```C
       // 阻塞方式回收
        while ((wpid = waitpid(-1, NULL, 0)))
        {
            printf("wait child %d \n", wpid);
        }
```

结果如下：

```c
(base) ➜  leetcode-revise (master) ✗ ./test 
I am 1th child, my pid is 79204
wait child 79204 
I am 2th child, my pid is 79205
wait child 79205 
I am 3th child, my pid is 79206
wait child 79206 
I am 4th child, my pid is 79207
wait child 79207 
I am 5th child, my pid is 79208
wait child 79208 
wait child -1 
```

可以看到，当子进程全部被回收之后，会返回-1。

非阻塞方式回收：

在这里要多加一个判断，因为wpid多数都是返回的0，意思是未回收到，所以当判断到返回为0时，可以sleep(1)再继续回收。

```C
        while ((wpid = waitpid(-1, NULL, WNOHANG)) != -1)
        {
            // 做个判断
            if (wpid > 0)
                printf("wait child %d \n", wpid);
            else if (wpid == 0)
            {
                sleep(1);
                continue;
            }
        }
```



waitpid作业：

作业:父进程 fork 3 个子进程，三个子进程一个调用 ps 命令， 一个调用自定义程序

1(正常)，一个调用自定义程序 2(会出段错误)。父进程使用 waitpid 对其子进程进行回收。

```c
// homework of waitpid
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>
int main(void)
{
    pid_t pid,wpid, tmppid;
    int status;
    int i;
    for (i = 0; i < 3; ++i)
    {
        pid = fork();
        // 如果是子进程
        if (pid == 0)
            { break; }
    }
    if (i == 0)
    {
        printf("I am child process 1, mypid is %d, exec ps command\n", getpid());
        execlp("ps", "ps", "a", NULL);
    }
    else if (i == 1)
    {
        printf("I am child process 2, mypid is %d, exec print command\n", getpid());
        execl("./child2", "./child2", NULL);
    }
    else if (i == 2)
    {
        printf("I am child process 3, mypid is %d, exec bad command\n", getpid());
        execl("./child3", "./child3", NULL);
    }
    else if (i == 3)
    {
        // 阻塞方式回收
        while ((wpid = waitpid(-1, &status, WNOHANG)) != -1)
        {
            if (wpid == 0)
            {
                sleep(1);
                continue;
            }
            else {
                printf("wait child %d \n", wpid);
                        // 正常终止
                if (WIFEXITED(status))
                {
                    printf("Child exit normally:");
                    printf("Child exit with %d\n ", WEXITSTATUS(status));
                }
                // 信号终止，所有程序异常终止的情况都是信号
                if(WIFSIGNALED(status))
                {
                    printf("Child exit abnormally, with:");
                    printf("Child exit with %d\n", WTERMSIG(status));
                }
            }
        }
        if (wpid == -1)
        {
            perror("wrong wait pid!\n");
            exit(1);
        }
        // wpid如果返回0，说明没有回收到
    }
    else
    {
        sleep(i);
        printf("I am %dth child, my pid is %d\n", i+1, getpid());
    }
    return 0;
}

// 结果：
➜  chapter2-process git:(master) ✗ ./waitpid_hw
I am child process 1, mypid is 1628, exec ps command
I am child process 2, mypid is 1629, exec print command
I am child process 3, mypid is 1630, exec bad command
hello!--------------
    PID TTY      STAT   TIME COMMAND
    125 pts/0    Ss     0:01 /bin/zsh
   1627 pts/0    S+     0:00 ./waitpid_hw
   1628 pts/0    R+     0:00 ps a
   1629 pts/0    Z+     0:00 [child2] <defunct>
   1630 pts/0    S+     0:00 ./child3
wait child 1628 
Child exit normally:Child exit with 0
 wait child 1629 
Child exit normally:Child exit with 0
 wait child 1630 
Child exit abnormally, with:Child exit with 11
wrong wait pid!
: No child processes
```

