# 信号

## 概念

1. 简单
2. 携带相对简单的信息
3. 满足**某个特定条件**才能发动

 信号的特质：

软件层面上的“中断”，一旦信号产生，程序必须立刻停止运行，处理信号，处理结束再执行后续指令，每个进程收到的所有信号，都是由内核负责发送的，内核处理。

## 与信号相关的事件和状态

- 产生信号:
  1. 按键产生，如:Ctrl+c、Ctrl+z、Ctrl+\
  2. 系统调用产生，如:kill、raise、abort
  3. 软件条件产生，如:定时器 alarm，如sleep函数就是借助这个来实现的
  4. 硬件异常产生，如:非法访问内存(段错误)、除 0(浮点数例外)、内存对齐出错(总线错误) 
  5. 命令产生，如:kill 命令

- 递达：内核产生信号并送达进程
- 未决：产生和递达之间的状态。主要由于阻塞(屏蔽)导致该状态
- 处理信号：
  1. 执行默认动作
  2. 忽略处理，处于已经递达状态
  3. 捕捉（调用用户处理函数） 
- 阻塞信号集(信号屏蔽字): 
  - 本质为位图，将某些信号加入集合，对他们设置屏蔽，当屏蔽某个信号后，再收到该信号，该信号的处理将推后，描述该信号的位在设置屏蔽后，会翻转为 1，表示信号处于阻塞状态。
- 未决信号集
  1. 信号产生，未决信号集中描述该信号的位立刻翻转为 1，表信号处于未决状态。当信号被处理对应位翻转回为0。这一时刻往往非常短暂。
  2. 信号产生后由于某些原因(主要是阻塞)不能抵达。这类信号的集合称之为未决信号集。在屏蔽解除前， 信号一直处于未决状态。

Linux 内核的进程控制块 PCB 是一个结构体，`task_struct`, 除了包含进程 id，状态，工作目录，用户 id，组 id， 文件描述符表，还包含了信号相关的信息，主要指**阻塞信号集**和**未决信号集**，两个的本质都是位图。

## 常规信号

```bash
kill -l # 查看信号
```

不存在编号为 0 的信号。其中 1-31 号信号称之为常规信号(也叫普通信号或标准信号)，34-64 称之为实时信 号，驱动编程与硬件相关。名字上区别不大。而前 32 个名字各不相同。

## 信号4要素

信号编号，信号名称，信号事件，默认处理动作

可通过 man 7 signal 查看帮助文档获取，重点掌握前20个

## 信号的产生

### kill函数

**kill 命令产生信号:kill -SIGKILL pid**

kill函数可以实现kill命令：

```c
int kill(pid_t pid, int sig); 
```

成功:0;失败:-1 (ID 非法，信号非法，普通用户杀 init 进程等权级问题)，设置 errno sig:不推荐直接使用数字，应使用宏名，因为不同操作系统信号编号可能不同，但名称一致。

- pid > 0: 发送信号给指定的进程。
- pid = 0: 发送信号给 与调用 kill 函数进程属于同一进程组的所有进程。
- pid < 0:  取|pid|发给对应进程组。
- pid = -1:发送给进程有权限发送的系统中所有进程。（比较危险）‘
  - 什么是有权限？**权限保护**:super 用户(root)可以发送信号给任意用户，普通用户是不能向系统用户发送信号的。 kill -9 (root 用 户的 pid) 是不可以的。同样，普通用户也不能向其他普通用户发送信号，终止其进程。 只能向自己创建的进程发 送信号。普通用户基本规则是:发送者实际或有效用户 ID == 接收者实际或有效用户 ID。

进程组:每个进程都属于一个进程组，进程组是一个或多个进程集合，他们相互关联，共同完成一个实体任务，每个进程组都有一个进程组长，默认进程组 ID 与进程组长 ID 相同。fork之后组长id等于父进程id。

**ps ajx**,查看进程组id，如果要杀死同一个进程组，在父进程前面加负号：

```
kill -9 -10698
```

pid ==0

kill示例：

```c
// 父进程用 kill 函数终止一子进程
int main(void)
{
    pid_t pid;
    pid_t cpid;
    // for (int i= 0; i < 5; i++)
    // {
        
        pid = fork();
        // 父进程
        if (pid > 0)
        {
            printf("I am parent with pid %d, go to sleep\n", getpid());
            sleep(5);
            printf("I am parent with pid %d, go to kill child %d\n", getpid(), pid);
            int ret = kill(pid, SIGKILL);
            if (ret == -1)
            {
                perror("wrong kill");
                exit(1);
            }
        }
        else if (pid == 0) {
            printf("I am child with pid %d, my parent's id is %d\n", getpid(), getppid());
            cpid = getpid();
            while(1);
        }
        else {
            perror("wrong fork\n");
            exit(1);
        }
    return 0;
}
```

练习:循环创建 5 个子进程，父进程用 kill 函数终止任一子进程：

```c
// // 循环创建 5 个子进程，父进程用 kill 函数终止任一子进程
// // 要记得检查所有系统调用的返回值

int main(void)
{
    pid_t pid;
    int i;
    for (i= 0; i < 5; i++)
    {
        pid = fork();
        if (pid == -1)
        {
            perror("wrong fork");
            exit(1);
        }
        if (pid == 0)
        { break; }
    }
    if (i == 5)
    {
        // 父进程
        printf("I am parent with pid %d, go to sleep\n", getpid());
        sleep(5);
        printf("I am parent with pid %d\n", getpid());
        int ret = kill(pid, SIGKILL);
        if (ret == -1)
        {
            perror("wrong kill");
            exit(1);
        }
        printf("killed child %d", pid);
    }
    else {
        printf("I am child with pid %d, my parent's id is %d\n", getpid(), getppid());
        while(1);
    }
    return 0;
}
```

### alarm函数

使用自然计时法

定时发送 **SIGALRM** 给当前进程。
unsigned int alarm(unsigned int seconds);

seconds:定时秒数 返回值:上次定时剩余时间。

无错误现象。 

alarm(0); 取消闹钟。

一个例子：

alarm(5) → 3sec → alarm(4) → 5sec → alarm(5) → alarm(0)

time 命令 : 查看程序执行时间。 实际时间 = 用户时间 + 内核时间 + 等待时间。 --》 优化瓶颈 IO

### setitimer 函数

设置定时器(闹钟)。 可代替 alarm 函数。精度微秒 us，可以**实现周期定时**。

old_value：上次剩余时间

new_value：定时秒数

返回：

- 0：

- -1

练习: 使用 setitimer 函数实现 alarm 函数，重复计算机 1 秒数数程序。

拓展练习，结合 man page 编写程序，测试 it_interval、it_value 这两个参数的作用。

## 信号集操作函数

![信号集示意图](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2022-01-02 at 15.51.18.png)

内核通过读取未决信号集来判断信号是否应被处理。信号屏蔽字 mask 可以影响未决信号集。而我们可以在应用程序中自定义 set 来改变 mask，达到屏蔽指定信号的目的。

### 设置自定义信号集函数

int sigemptyset(sigset_t *set)：信号集清零，清空

int sigfillset(sigset_t *set)：全部都置为1

int sigaddset(sigset_t *set, int signum)：将某个信号加入信号集，置为1

int sigdelset(sigset_t *set, int signum)：将某个信号清出信号集，置为0

int sigismember(const sigset_t *set, int signum)：判断某个信号是否在信号集中

返回值：

- 成功:0;失败:-1

### **sigprocmask** 函数

用来**屏蔽信号、解除屏蔽**也使用该函数。其本质，读取或修改进程的信号屏蔽字(PCB 中)，严格注意，屏蔽信号:只是将信号处理延后执行**(**延至解除屏蔽**)**;而忽略表示将信号丢处理。

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset); 成功:0;失败:-1，设置 errno

参数:

- set:指用户自定义集合，传入参数，是一个位图，set 中哪位置 1，就表示当前进程屏蔽哪个信号。 
- oldset：**旧有的信号集，传出参数，保存旧的信号屏蔽集。**
- how参数取值: 假设当前的信号屏蔽字为mask
  - SIG_BLOCK: 当 how 设置为此值，set 表示需要屏蔽（阻塞）的信号。相当于 mask = mask | set
  - SIG_UNBLOCK: 当 how 设置为此，set 表示需要解除屏蔽（解除阻塞）的信号。相当于 mask = mask & ~set
  - SIG_SETMASK: 当 how 设置为此，set 表示用于替代原始屏蔽及的新屏蔽集。相当于 mask = set 若，调用 sigprocmask 解除了对当前若干个信号的阻塞，则在 sigprocmask 返回前，至少将其中一 个信号递达。

### **sigpending** 函数

读取当前进程的未决信号集
 int sigpending(sigset_t *set); 

参数：

- set： 传出参数。 

返回值:成功:0;失败:-1，设置 errno 

## 信号集操作流程

![Screen Shot 2022-01-02 at 16.37.31](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2022-01-02 at 16.37.31.png)

1. 自定义一个信号集set，用来对PCB中的信号集信号进行改变
2. sigaddset设置信号集
3. sigprocmask设置PCB中的信号集

代码如下：

```c
int main(void)
{
    sigset_t myset,oldset;
    sigemptyset(&myset);
    // 屏蔽2号信号
    sigaddset(&myset, SIGINT);
    sigprocmask(SIG_BLOCK, &myset, &oldset);
    while (1) 
    {
        printf("you have add SIGINT to set!\n");
        sleep(1);
    }    
}
```

练习:编写程序。把所有常规信号的未决状态打印至屏幕：该代码没有检查返回值，要注意

```c
void print_signal(sigset_t* set)
{
    int i;
    for (i = 1; i <= 32; ++i)
    {
        // i用来指代信号
        if (sigismember(set, i))
        {
            printf("%d",1);
        }
        else {
            printf("%d",0);
        }
    }
}
int main()
{
    sigset_t myset,oldset,pending_set;
    sigemptyset(&myset);
    // 屏蔽2号信号
    sigaddset(&myset, SIGINT);
    sigaddset(&myset, SIGQUIT);
    sigprocmask(SIG_BLOCK, &myset, &oldset);   
    int ret = sigpending(&pending_set);
    if (ret == -1)
    {
        perror("wrong\n");
        exit(1);
    }
    while (1)
    {
        sigpending(&pending_set);
        print_signal(&pending_set);
        printf("\n");
        sleep(1);
    }
    return 0;
}
```



## 信号捕捉

### signal函数

用来注册一个信号捕捉函数：

```c
void (*signal(int sig, void (*func)(int)))(int);
```

例子：

```c
// signal函数示例
void catch_signal(int sig_num)
{
    printf("catch you:%d!\n",sig_num);
}
int main(void)
{
    // signal函数用来注册信号捕捉
    signal(SIGQUIT, catch_signal);
    while (1);
}
```

简化版`sigaction()`

### sigaction()函数

该函数的使用流程类似于signal，都是为信号注册捕捉函数：

```c
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

参数列表：

- signum：为注册信号的名称
- act：结构体，代表新的处理方式
- oldact：结构体，代表旧的处理方式

sigaction结构体主要内容：

```c
 struct  sigaction {
             union __sigaction_u __sigaction_u;  /* signal handler */
             sigset_t sa_mask;               /* signal mask to apply */
             int     sa_flags;               /* see signal options below */
     };
     union __sigaction_u {
             void    (*__sa_handler)(int);
             void    (*__sa_sigaction)(int, siginfo_t *,
                            void *);
     };
```

结构体内容说明：

1. sa_handler:指定信号捕捉后的处理函数名(即注册函数)。也可赋值为 SIG_IGN 表示忽略 或 SIG_DFL 表执行默认动作
2. sa_mask: 调用信号处理函数时，所要屏蔽的信号集合(信号屏蔽字)。注意:仅在处理函数被调用期间屏蔽生效，是临时性设置。
3.  sa_flags:通常设置为 0，表使用默认属性。
4. sa_sigaction参数用的很少:当 sa_flags 被指定为 SA_SIGINFO 标志时，使用该信号处理程序。(很少使用)

下面的小例子，使用 sigaction 捕捉两个信号:

```C
void catch_signal(int sig_num)
{
    if (sig_num == SIGQUIT)
    {
        printf("catch %d\n", sig_num);
        sleep(5);
    }
    else if (sig_num == SIGINT)
    {
        printf("catch %d\n", sig_num);
    }
}

int main(void)
{
    struct sigaction act, oact;
    act.sa_handler = catch_signal;
    // set mask when sig_catch working. 清空sa_mask屏蔽字, 只在 sig_catch 工作时有效
    sigemptyset(&(act.sa_mask)); 
    act.sa_flags = 0; // usually use. 默认值
    int ret = sigaction(SIGINT, &act, &oact);
    if (ret == -1)
    {
        perror("wrong\n");
        exit(1);
    }
    ret = sigaction(SIGQUIT, &act, &oact);
    if (ret == -1)
    {
        perror("wrong\n");
        exit(1);
    }
    while (1);

}
```

## 信号捕捉特性

1. （和mask（信号屏蔽字）的区别）：进程正常运行时，默认 PCB 中有一个信号屏蔽字，假定为mask，它决定了进程自动屏蔽哪些信号。当注册了某个信号捕捉函数，捕捉到该信号以后，要调用该函数。而该函数有可能执行很长时间，在这期间所屏蔽的信号不由mask来指定。而是用 sa_mask 来指定。调用完信号处理函数，再恢复为mask。
2. XXX 信号捕捉函数执行期间，XXX 信号自动被屏蔽。
3. 阻塞的常规信号不支持排队，产生多次只记录一次。(后 32 个实时信号支持排队)

## 信号捕捉过程

SIGCHILD信号：只要子进程状态发生变化，就会产生该信号，包括：

- 子进程终止时
- 子进程接收到 SIGSTOP 信号停止时 
- 子进程处在停止态，接受到 SIGCONT 后唤醒时

子进程结束运行，其父进程会收到 SIGCHLD 信号。该信号的默认处理动作是忽略。可以捕捉该信号，在捕捉函数中完成子进程状态的回收。

示例代码：

```c
```

加入一个while(1)循环，用ps ajx会出现僵尸进程，同时多次执行，会发现回收到的子进程数量不是固定的。 原因分析:问题出在，一次回调只回收一个子进程这里。同时出现多个子进程死亡时，其中一个子进程死亡 信号被捕捉，父进程去处理这个信号，此时其他子进程死亡信号发送过来，由于**相同信号的不排队原则**，就只会回收累积信号中的一个子进程。



还有一个问题需要注意，这里有可能父进程还没注册完捕捉函数，子进程就死亡了，解决这个问 题的方法，首先是让子进程 sleep，但这个不太科学。在 fork 之前注册也行，但这个也不是很科学。 最科学的方法是在 int i 之前设置屏蔽，等父进程注册完捕捉函数再解除屏蔽。这样即使子进程先死 亡了，信号也因为被屏蔽而无法到达父进程。解除屏蔽过后，父进程就能处理累积起来的信号了。

```c
```

## 中断系统调用

慢速系统调用: 可能会使进程永久阻塞的一类。如果在阻塞期间收到一个信号，该系统调用就被中断，不再继续执行(早期)，也可以设定系统调用是否重启。如 read, write, pause...

这里需要对sa_flags进行一定的设置，传入SA_RESTART