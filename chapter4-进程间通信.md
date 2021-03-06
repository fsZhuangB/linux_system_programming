# 进程间通信

Linux 环境下，进程地址空间相互独立，每个进程各自有不同的用户地址空间。任何一个进程的全局变量在另 一个进程中都看不到，所以进程和进程之间不能相互访问，要交换数据必须通过内核，在内核中开辟一块缓冲区， 进程 1 把数据从用户空间拷到内核缓冲区，进程 2 再从内核缓冲区把数据读走，内核提供的这种机制称为进程间通 信(IPC，InterProcess Communication)。

## 进程通信常见方式

1. 管道：使用简单
2. 信号：开销比较小
3. 共享映射区：无血缘关系
4. 本地套接字：最稳定

### 管道

图示：



管道是一种最基本的 IPC 机制，作用于有血缘关系的进程之间，完成数据传递。调用 pipe 系统函数即可创建一 个管道。有如下特质:

1. 其本质是一个**伪文件**(实为内核缓冲区)，不占用磁盘空间。（普通文件，目录，软连接占用磁盘空间）
2. 由两个文件描述符引用，一个表示读端，一个表示写端。
3. 规定数据从管道的写端流入管道，从读端流出。

管道的原理: 管道实为内核使用**环形队列机制**，借助内核缓冲区(4k)实现。

管道的局限性:

1. 数据不能进程自己写，自己读。
2. 管道中数据不可反复读取。一旦读走，管道中不再存在。
3. 采用半双工通信方式，数据只能在单方向上流动。
4. 只有有血缘关系的才能通信

### pipe函数

pipe函数创建的叫做**匿名管道**，fifo函数创建的命名管道 

作用：**创建并打开管道**，函数原型：

```c
int pipe(int fd[2]);
fd[0]: read
fd[1]: write
```

返回值：成功:0;失败:-1，设置 errno

只需要用close关闭，不需要打开。

一个管道通信的示例，父进程往管道里写，子进程从管道读，然后打印读取的内容，程序非常需要注意的点在于要先开管道，再fork()否则就会读取不到！

```c
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid;
    // 管道返回的文件描述符，fd[0]:read fd[1]: write
    int fd[2];
    char buffer[1024];
    char* str = "Hello world!\n";
    int ret;

    // 这里要注意，先开管道，再fork，否则就会出现读取不到的情况！开了两个管道！
    ret = pipe(fd);
    pid = fork();

    if (pid > 0)
    {
        close(fd[0]);
        write(fd[1], str, strlen(str));
        close(fd[1]);
    }
    else if (pid == 0)
    {
        close(fd[1]);
        ret = read(fd[0], buffer, sizeof(buffer));
        printf("The child read size %d\n", ret);
        write(STDOUT_FILENO, buffer, ret);
        close(fd[0]);
    }
    return 0;
}
```



![Screen Shot 2021-12-25 at 20.13.45](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-12-25 at 20.13.45.png)

问题：能否保证父进程写完了子进程才读？

### 管道读写行为

1. 读管道
   1. 有数据：返回实际读到的字节数。
   2. 无数据：
      - 管道写端被关闭，返回0
      - 写端未关闭，阻塞等待
2. 写管道
   1. 读端被关闭（没有进程把持），异常终止，返回SIGPIPE信号
   2. 管道读端没有全部关闭:
      - buffer空间够，即管道未满
      - buffer空间不够，管道已满，阻塞等待

练习:使用管道实现父子进程间通信，完成:`ls | wc -l`。假定父进程实现 ls，子进程实现 wc。

ls 命令正常会将结果集写出到 stdout，但现在会写入管道的写端;wc –l 正常应该从 stdin 读取数据，但此时会从管道的读端读，很简单的代码实现，要注意wc -l是从stdin来读数据，而不是输出数据。

```C
// 使用管道实现父子进程间通信，完成:ls | wc –l。
// 假定父进程实现 ls，子进程实现 wc。
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid;
    int fd[2];
    int ret;
    char buf[1024];
    // 打开管道
  	// 父进程持有读端和写端
    ret = pipe(fd);
  
  	// 子进程同样持有读端和写端
    pid = fork(); 
    // 父进程应该是写管道
    if (pid > 0)
    {
        close(fd[0]); // 关闭读
        // 将fd[1]重定向给标准输出
        dup2(fd[1], STDOUT_FILENO);
        execlp("ls", "ls", NULL);
        close(fd[1]);
    }
    else if (pid == 0)
    {
        printf("I am child, exec wc -l\n");
        close(fd[1]); // 关闭写
        // 将标准输入重定向给fd[0]
        dup2(fd[0], STDIN_FILENO);
        execlp("wc", "wc", "-l", NULL);
        ret = read(fd[0], buf, sizeof(buf));
        printf("The child read size %d\n", ret);
        write(STDOUT_FILENO, buf, ret);
        close(fd[0]);
    }
}
```

### 练习2：实现兄弟进程间通信

这里要注意的点在于父进程要关闭读写两端，使得管道形成单向数据流通

```C
// 练习题:兄弟进程间通信 兄:ls
// 弟:wc -l
// 父:等待回收子进程
// 要求，使用循环创建 N 个子进程模型创建兄弟进程，使用循环因子 i 标识，注意管道读写行为
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid;
    int ret, i;
    int fd[2];
    char buf[1024];
    ret = pipe(fd);
    if (ret < 0)
    {
        perror("wrong pipe!\n");
        exit(1);
    }
    for (i = 0; i < 2; ++i)
    {
        pid = fork();
        if (pid == 0)
            break;
    }
    if (i == 0)
    {
        printf("I am child1, exec ls\n");
        close(fd[0]); // 关闭读
        // 将fd[1]重定向给标准输出
        dup2(fd[1], STDOUT_FILENO);
        execlp("ls", "ls", NULL);
        close(fd[1]);
    }
    else if (i == 1)
    {
        printf("I am child2, exec wc -l\n");
        close(fd[1]); // 关闭写
        // 将标准输入重定向给fd[0]
        dup2(fd[0], STDIN_FILENO);
        execlp("wc", "wc", "-l", NULL);
        ret = read(fd[0], buf, sizeof(buf));
        printf("The child read size %d\n", ret);
        write(STDOUT_FILENO, buf, ret);
        close(fd[0]);
    }
    else if (i == 2)
    {
        // 父进程，要记得关闭读写两端，否则会造成无法读取
        close(fd[0]);
        close(fd[1]);
        wait(NULL);
        wait(NULL);
    }
    return 0;
}
```



### 练习3：多个读写端操作

用的比较少

测试:

是否允许，一个 pipe 有一个写端多个读端： 可 

是否允许，一个 pipe 有多个写端一个读端： 可



### 命名管道

管道的优劣: 

- 优点:简单，相比信号，套接字实现进程通信，简单很多 

- 缺点:
  1. 只能单向通信，双向通信需建立两个管道
  2. 只能用于有血缘关系的进程间通信。该问题后来使用 fifo 命名管道解决。

**解决没有关系的进程进行通信，借助内核空间实现**

创建方式：

1. 命令：mkfifo myfifo

   ```c
   mkfifo myfifo
   ```

2. 利用函数创建，fifo 操作起来像文件

创建方法：

mkfifo.c

#### 命名管道通信方式

和文件基本上就是一回事，下面这个例子，一个写 fifo，一个读 fifo，操作起来就像文件一样的:



### 文件完成进程间通信

不需要会写：

**打开的文件是内核中的一块缓冲区。**多个无血缘关系的进程，可以同时访问该文件。

通过文件通信的做法，有没有血缘关系都行，只是有血缘关系的进程对于同一个文件，使用的同一个文件描述符，没有血缘关系的进程，对同一个文件使用的文件描述符可能不同。这些都不是问题，打开 的是同一个文件就行。

代码展示，两个没有血缘关系的进程之间的通信：

![Screen Shot 2021-12-27 at 20.40.50](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2021-12-27 at 20.40.50.png)

## 内存映射I/O

可以用指针进行操作，可用的函数更多，更加方便。

存储映射 I/O (Memory-mapped I/O) 使一个磁盘文件与存储空间中的一个缓冲区相映射。于是当从缓冲区中取数据，就相当于读文件中的相应字节。于此类似，将数据存入缓冲区，则相应的字节就自动写入文件。这样，就可在不适用 read 和 write 函数的情况下，使用地址(**指针**)完成 I/O 操作。

mmap函数：

参数：

1. addr：指定映射区首地址，通常传NULL（不需要自己分配，表示让系统分配）
2. int len：共享内存映射区大小（<= 文件实际大小）
3. prot：共享内存映射内存区的读写属性PROT_READ | PROT_WRITE
4. flags: 标注共享属性：MAP_SHARED, MAP_PRIVATE
   1. MAP_SHARED：对内存的修改不会反映到磁盘上
   2. MAP_PRIVATE：对内存的修改会反映到磁盘上
5. fd：用于创建共享内存映射区的那个文件的文件描述符
6. offset：偏移位置，必须为4k的整数倍，默认0表示映射文件全部

返回值：

- 成功：void* （泛型指针）
- MAP_FAILED，(void * (-1))

示例代码：

```c
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/mman.h>
#include <fcntl.h>
int main(void)
{
    int fd;
    fd = open("file_map.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0)
    {
        perror("wrong open");
        exit(1);
    }
    // 下面两行代码等同于ftruncate
    // lseek(fd, 19, SEEK_END);  
    // write(fd, "\0", 1);
    // 文件大小置为20
    ftruncate(fd, 20);  // 该函数需要写权限才能改变文件大小
    int len = lseek(fd, 0, SEEK_END);
    char* p = NULL;
    p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED)
    {
        perror("wrong map!\n");
        exit(1);
    }
    strcpy(p, "Hello mmap--------\n");
    printf("%s", p);
    int ret = munmap(NULL, len);
    if (ret == -1)
    {
        perror("worong!");
        exit(1);
    }
    return 0;
}
```



1. 打开文件
2. 获取文件大小（lseek）
3. 使用mmap函数
4. 使用p来对文件进行操作
   1. printf
   2. strcpy

5. 使用munmap释放映射区

```c
ftruncate()
```

munmap函数：释放映射区

参数：

1. addr：mmap返回值
2. length：大小

返回值：

1. 成功：返回
2. 失败

### mmap注意事项

1. 可以 open 的时候 O_CREAT 一个新文件来创建映射区吗?

   可以，用于但是不能创建比文件大小大的映射区，会出总线错误7 SIGBUS：

   ```c
   ➜  chapter3-IPC git:(master) ✗ ./mmap 
   [1]    1263 bus error (core dumped)  ./mmap
   ```

2. 用于创建映射区的文件大小为 0，实际指定 0 大小创建映射区， 出 “无效参数”。

3. 用于创建映射区的文件读写属性为，只读（READ_ONLY）。映射区属性为 读、写。 出 “无效参数”。

4. 对文件的只读权限打开，并且映射是可以的，但是不能有写操作。

5. 创建映射区的权限需要最基本的：READ，同时，mmap的读写权限应该小于等于文件的open权限，指写不下

6. 文件描述符 fd，在 mmap 创建映射区完成即可关闭。后续访问文件，用指针访问。

7. offset 必须是 4096 的整数倍。(MMU 映射的最小单位 4k )

8. 不能越界访问，可能越界一点点不会有问题，但是太多会core dumped，最好还是不要越界

   ```c
   // 越界访问报错
   strcpy(p+len+4096, "Hello mmap--------\n");
   
   ➜  chapter3-IPC git:(master) ✗ ./mmap 
   [1]    1351 segmentation fault (core dumped)  ./mmap
   ```

9. munmap 用于释放的 地址，必须是 mmap 申请返回的地址：如果将p指针进行++操作，munmap就会出现参数错误

10. mmap所有的参数都有可能出错，所以务必要对返回值进行检查

mmap 函数的保险调用方式:

```c
fd = open("文件名"， O_RDWR);
mmap(NULL, 有效文件大小， PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
```

### 父子进程通信

注意事项：

1. 映射区为shared
2. 先mmap，再fork

```C
// 父子进程通信
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/mman.h>
#include <fcntl.h>

// 注意点在于，全局变量：读时共享，写时复制
int var = 100;
int main(void)
{
    int fd;
    pid_t pid;
    fd = open("file_map.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0)
    {
        perror("wrong open");
        exit(1);
    }

    // 文件大小置为20
    ftruncate(fd, 20);
    int len = lseek(fd, 0, SEEK_END);
    int* p = NULL;
    p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED)
    {
        perror("wrong map!\n");
        exit(1);
    }
    pid = fork();
    if (pid == 0)
    {
        // 子进程修改
        *p = 1000;
        var = 1000;
        printf("I am child, and p=%d, var = %d\n", *p, var);
    }
    else if (pid > 0)
    {
        // 父进程读
        sleep(1);
        printf("I am parent, and p = %d, var=%d\n", *p, var);
        wait(NULL);

        int ret = munmap(p, len);
        if (ret == -1)
        {
            perror("wrong unmap");
            exit(1);
        }
    }
}
```

### 无血缘关系通信

两个进程打开同一个文件，创建映射区，指定flags为**MAP_SHARED**，一个进程读入，另一个进程写出。

【注意】:无血缘关系进程间通信。mmap:数据可以重复读取。

而fifo属于管道，数据不能重复读，读完了就没了

### mmap匿名映射区

匿名映射:只能用于 **血缘关系进程间**通信：

要注意fd位置传-1；

 p = (int *)mmap(NULL, 40, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_ANONYMOUS, -1, 0);

/dev/null：文件空洞

/dev/zero：



