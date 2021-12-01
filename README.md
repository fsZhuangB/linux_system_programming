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

为文件创建硬链接。

## unlink函数

结合link函数，可以实现mv命令

无dentry对应的文件，会被择机释放。

删除文件，只是让文件具备了被释放的条件。**unlink** 函数的特征:清除文件时，如果文件的硬链接数到 0 了，没有 dentry 对应，但该 文件仍不会马上被释放。**要等到所有打开该文件的进程关闭该文件**，系统才会挑时间将该文 件释放掉。 【unlink_exe.c】

## 隐式回收

当进程结束运行时，所有该进程打开的文件会被关闭，申请的内存空间会被释放。系统 的这一特性称之为隐式回收系统资源。 

## 文件目录权限操作

 ## opendir函数

## closedir函数

## readdir函数

 通过以上函数，实现简单的ls命令:

### 实现递归遍历目录

ls -R

```c
```







