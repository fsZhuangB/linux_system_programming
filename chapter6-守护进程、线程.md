# chapter6-守护进程、线程

## 会话

会话:多个进程组的集合

创建会话的 6 点注意事项:

1. 调用进程不能是进程组组长，该进程变成新会话首进程
2. 该进程成为一个新进程组的组长进程
3. 需要 root 权限(ubuntu 不需要)
4. 新会话丢弃原有的控制终端，该会话没有控制终端（意思是适合在后台中运行）
5. 该调用进程是组长进程，则出错返回
6.  建立新会话时，先调用 fork，父进程终止，子进程调用 setsid

相关函数：

**getsid(),setsid()函数**

注意事项：进程id，进程组id，会话id三者合一，是一样的

getsid 函数:
 pid_t getsid(pid_t pid) 获取当前进程的会话 id 成功返回调用进程会话 ID，失败返回-1，设置 error

setsid 函数:
 pid_t setsid(void) 创建一个会话，并以自己的 ID 设置进程组 ID，同时也是新会话的 ID 成功返回调用进程的会话 ID，失败返回-1，设置 error

示例代码：scesion.c:

```c
```

## 守护进程

Daemon(精灵)进程，是 Linux 中的后台服务进程，通常独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。一般采用以 d 结尾的名字。

Linux 后台的一些系统服务进程，没有控制终端，**不能直接和用户交互**。不受用户登录、注销的影响，一直在 运行着，他们都是守护进程。如:预读入缓输出机制的实现;ftp 服务器;nfs 服务器等。

创建守护进程，最关键的一步是调用 setsid 函数创建一个新的 Session，并成为 Session Leader。

示例代码：

```c
if (pid > 0)
{exit(0); }
pid_t spid = setsid(0);
int ret = chdir("/"); // 改变工作目录位置
umask(0022); // 改变文件权限掩码
// 关闭文件描述符0
```

## 线程

### 线程概念

（gdb调试不支持线程调试）

LWP:light weight process 轻量级的进程，本质仍是进程(在 Linux 环境下)

线程的特点：

1. 没有独立的地址空间，但是有PCB（TCB）
2. 区别（linux下）：
   - 在于是否共享地址空间
   - 线程是最小的执行单位
   - 进程是资源分配的最小单位

查看线程号：

```c
ps -Lf 16343(pid)
```



![Screen Shot 2022-01-05 at 14.57.56](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2022-01-05 at 14.57.56.png)CPU对线程的认知是将其看作独立的进程：所以开多线程在争夺CPU时会有一定的优势：

![Screen Shot 2022-01-05 at 14.55.15](/Users/fszhuangb/Library/Application Support/typora-user-images/Screen Shot 2022-01-05 at 14.55.15.png)

### 三级映射

### 线程共享资源

1. 文件描述符表
2. 每种信号的处理方式
3. 当前工作目录
4. 用户 ID 和组 ID
5. 内存地址空间 (.text/.data/.bss/heap/共享库) -> 唯独不共享栈空间

### 线程非共享资源

1. 线程 id 
2. 处理器现场和栈指针(内核栈，用户栈)
3. 独立的栈空间(用户空间栈) 
4. errno 变量（errno变量是个全局变量）
5. 信号屏蔽字
6. 调度优先级

### 线程优、缺点

优点: 

1. 提高程序并发性
2. 开销小
3. 数据通信，共享比较方便

缺点: 

1. 库函数，不稳定
2. gdb不支持，编写调试困难
3. 对信号支持不好

 优点相对突出，缺点均不是硬伤。Linux 下由于实现方法导致进程、线程差别不是很大。

## 线程控制原语

可以和进程一一对应

### **pthread_self** 函数

获取线程id，要注意，其和LWP不同

pthread_t pthread_self(void); 返回值:成功:0; 失败:无!

线程 ID:pthread_t 类型，本质:在 Linux 下为无符号整数(%lu)，其他系统中可能是结构体实现

线程 ID 是进程内部，识别标志。(两个进程间，线程 ID 允许相同)

**注意:不应使用全局变量 pthread_t tid，在子线程中通过 pthread_create 传出参数来获取线程 ID，而应使用 pthread_self。**

### **pthread_create** 函数

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);
```

返回值:成功:0; 失败:错误号 -----Linux 环境下，所有线程特点，失败均直接返回错误号。

pthread_t:当前 Linux 中可理解为:typedef unsigned long int pthread_t;

参数 1:传出参数，保存系统为我们分配好的线程 ID

参数 2:通常传 NULL，表示使用线程默认属性。若想使用具体属性也可以修改该参数。 

参数 3:函数指针，指向线程主函数(线程体)，该函数运行结束，则线程结束。

参数 4:线程主函数执行期间所使用的参数，也就是参数3的参数，没有传入NULL

示例代码：

```c

// 创建的线程会默认执行该函数
void * tfn(void *arg)
{
    printf("tfn: pid %d, tid %lu\n", getpid(), pthread_self());

    return NULL;
}
int main(void)
{
    pthread_t tid;
    printf("main: pid %d, tid %lu\n", getpid(), pthread_self());
    int ret = pthread_create(&tid, NULL, tfn, NULL);
    if (ret != 0)
    { perror("wrong create!\n"); exit(1); }
    // 如果不sleep，主线程退出之后，子线程也就被销毁掉
    sleep(1);

    return 0;
}

➜  chapter5-thread git:(master) ✗ ./thread_creat 
main: pid 1179, tid 139785395701568
tfn: pid 1179, tid 139785387124480
```

### 循环创建多个子线程

要注意传入：

```C

// 创建的线程会默认执行该函数
void * tfn(void *arg)
{
    int i = (int) arg;
    sleep(i);
    printf("tfn: pid %d, i = %d, tid %lu\n", getpid(), i+1, pthread_self());

    return NULL;
}
int main(void)
{
    pthread_t tid;
    int i;
    for (i = 0; i < 5; ++i)
    {
        int ret = pthread_create(&tid, NULL, tfn, (void *) i);
        if (ret != 0)
        { perror("wrong create!\n"); exit(1); }
    }

    // 如果不sleep，主线程退出之后，子线程也就被销毁掉
    sleep(i);
    printf("main: pid %d, tid %lu\n", getpid(), pthread_self());


    return 0;
}
```

### 重要！错误分析

如果传入地址，会出错：

即如果将 i 取地址后再传入线程创建函数里，就是说 当前传的是:

`(void *)i`
 改成: `(void *)&i`
 相应的，修改回调函数:`int i = *((int *)arg)` 运行代码，会出现如下结果:

```c
➜  chapter5-thread git:(master) ✗ ./thread_join_loops 
i = 3
i = 5
child 1th exit with 1501672348!
i = 5
i = 4
i = 3
child 2th exit with 1501672348!
child 3th exit with 1501672348!
child 4th exit with 1501672348!
child 5th exit with 1501672348!
```

**从现象中，可以看出来i传入的是不固定的。**

原因：

1. i是不断变化的
2. for循环不会进行内核空间和用户空间的切换，所以很快，导致i不停变化

修改：要进行值传递，而不是引用传递

### 线程间全局变量共享

### pthread_exit 退出

void pthread_exit(void *retval); 指退出当前线程。

参数：

retval:退出值。 无退出值时，NULL

如果在回调函数里加一段代码: 

```c
if(i == 2)
	exit(0);

```

 看起来好像是退出了第三个子线程，然而运行时，发现后续的 4,5 也没了。这是因为，**exit 是退出进程。**

#### 修改一

修改一下，换成:` if(i == 2) return NULL;`
 这样运行一下，发现后续线程不会凉凉，说明 return 是可以达到退出线程的目的。然而真正意义上， return 是返回到函数调用者那里去，线程并没有退出。

#### 修改二

再修改一下，再定义一个函数 func，直接返回那种（这里不可能退出，只是返回到调用者那里）

```c
void *func(void){
	return NULL;
}
if(i == 2) func();
```

运行，发现 1,2,3,4,5 线程都还在，说明没有达到退出目的。

#### 修改三

改为：

```c
void *func(void){
  pthread_exit(NULL); // 将当前线程退出
	return NULL;
}
if(i == 2) func();
```



回到之前说的一个问题，由于主线程可能先于子线程结束，所以子线程的输出可能不会打印出来， 当时是用主线程 sleep 等待子线程结束来解决的。现在就可以使用 `pthread_exit` 来解决了。方法就 是将 return 0 替换为 `pthread_exit`，只退出当先线程，不会对其他线程造成影响，这样就可以不用sleep函数去睡眠了。

#### 总结

1. exit(); 退出当前进程。
2. return: 返回到调用者那里去。 
3. pthread_exit(): 退出当前线程。

### pthread_join回收线程

回收线程只需要指定线程id即可，没有父子进程回收的概念。

```c
int pthread_join(pthread_t thread, void **retval);
void **retval原因是因为pthread_exit时retval为void* retval
```

retval如果不是一个空指针，则会保存线程终止时的返回值的拷贝，也是线程调用return或者pthread_exit()时所指定的值。

pthread_join作业：

```c
// 循环创建子线程，并用thread_join循环回收

#define N 5
void* work(void* arg)
{
    int i = (int) arg;
    return (void*) (i+1);
}
int main(void)
{
    pthread_t tid[N];
    int ret_status[N];
    int ret;
    //int count = 0;
    for (int i = 0; i < N;++i)
    {
        // 这里一定要注意不要传地址，而是传入值，否则i会一直变化
        int ret = pthread_create(&tid[i], NULL, &work, (void*) i);
        if (ret != 0)
        { perror("wrong create!\n"); exit(1); }
    }
    for (int i = 0; i < N; ++i)
    {
        ret = pthread_join(tid[i], (void**)&ret_status[i]);
        if (ret != 0)
        { 
            perror("worong!\n");
            exit(1);
        }
        printf("child %dth exit with %d!\n", i+1, ret_status[i]);
    }
    pthread_exit(NULL);

}
```

该作业的注意点在于要把tid和pthread_join的传出参数ret_status都用数组，有多少个就用多少个进行存放。

同时，要之前讲到的注意点，不要将i传入地址，而是传入值`(void*) i`：

```c
int ret = pthread_create(&tid[i], NULL, &work, (void*) i);
```

### pthread_cancel函数

杀死(取消)线程 其作用，对应进程中 kill() 函数。

int pthread_cancel(pthread_t thread); 成功:0;失败:错误号

【注意】:线程的取消并不是实时的，而有一定的延时。需要等待线程到达某个取消点(检查点)。

线程终止练习1:

```c
// endof3.c
// 测试线程保存点
void * work(void * arg)
{
    while (1)
    {
        printf("I am child thread, and my tid is %lu\n", pthread_self());
        sleep(1);
    }
}
int main(void)
{
    pthread_t tid;
    int ret;
    ret = pthread_create(&tid, NULL, &work, NULL);
    // 返回0表示成功
    if (ret != 0)
    {
        fprintf(stdout, "create error:%s\n", strerror(ret));
        exit(1);
    }
    printf("main: pid = %d, tid = %lu\n", getpid(), pthread_self());
    sleep(5);
    ret = pthread_cancel(tid);
    if (ret != 0)
    {
        fprintf(stdout, "cancel error:%s\n", strerror(ret));
        exit(1);
    }
    while (1);
    pthread_exit(NULL);
}
```



终止练习2:endof3

```c
// endof3.c
// 测试线程保存点
void * work(void * arg)
{
    while (1)
    {
        printf("I am child thread, and my tid is %lu\n", pthread_self());
       sleep(1);
    }
}
int main(void)
{
    pthread_t tid;
    int ret;
    void* nret = NULL;
    ret = pthread_create(&tid, NULL, &work, NULL);
    // 返回0表示成功
    if (ret != 0)
    {
        fprintf(stdout, "create error:%s\n", strerror(ret));
        exit(1);
    }
    printf("main: pid = %d, tid = %lu\n", getpid(), pthread_self());
    sleep(5);
    ret = pthread_cancel(tid);
    if (ret != 0)
    {
        fprintf(stdout, "cancel error:%s\n", strerror(ret));
        exit(1);
    }
    pthread_join(tid, &nret);
    printf("Thread exit with exit code:%d\n", (int) nret);
    while (1);
    pthread_exit(NULL);
}
```

该练习说明，如果已经被cancel杀死的进程，使用join是无法被回收到的。

```C
➜  chapter5-thread git:(master) ✗ ./thread_cancel2
main: pid = 19527, tid = 140512329911168
I am child thread, and my tid is 140512320292608
I am child thread, and my tid is 140512320292608
I am child thread, and my tid is 140512320292608
I am child thread, and my tid is 140512320292608
I am child thread, and my tid is 140512320292608
Thread exit with exit code:-1
^C
```



等待线程到达某个取消点(检查点)，可粗略认为一个系统调用(**进入内核**)即为一个取消点。如线程中没有取消点，可以通过调用 `pthread_testcancel()` 函数自行设置一个取消点。

被取消的线程，退出值定义在Linux的pthread库中。常数PTHREAD_CANCELED的值是-1。可在头文件pthread.h 中找到它的定义:**#define PTHREAD_CANCELED ((void \*) -1)**。因此当我们对一个已经被取消的线程使用 pthread_join 回收时，得到的返回值为-1。

### **pthread_detach** 函数

注意点，在线程中，检查错误不能再用perror了，因为会直接返回错误号，可以用strerror来查询错误号：

```c
fprintf(stderr, "xxx error: %s\n", strerror(ret));
```

int pthread_detach(pthread_t thread); 设置线程分离 thread: 待分离的线程 id

返回值:成功:0 失败:errno

下面这个例子，使用 detach 分离线程，照理来说，分离后的线程会自动回收:

```c
// 测试detach后是否会被自动回收
void * work(void * arg)
{
    printf("thread: pid = %d, tid = %lu\n", getpid(), pthread_self());
    return NULL;
}
int main(void)
{
    pthread_t tid;
    int ret;
    ret = pthread_create(&tid, NULL, &work, NULL);
    // 返回0表示成功
    if (ret != 0)
    {
        fprintf(stdout, "create error:%s\n", strerror(ret));
        exit(1);
    }
    printf("main: pid = %d, tid = %lu\n", getpid(), pthread_self());
    sleep(5);
    ret = pthread_detach(tid);
    if (ret != 0)
    {
        fprintf(stdout, "detach error:%s\n", strerror(ret));
        exit(1);
    }
    ret = pthread_join(tid, NULL);
    if (ret != 0)
    {
        fprintf(stdout, "join error:%s\n", strerror(ret));
        exit(1);
    }
    pthread_exit(NULL);

}
➜  chapter5-thread git:(master) ✗ ./pthread_detach 
main: pid = 22143, tid = 140454001974144
thread: pid = 22143, tid = 140453992355584
join error:Invalid argument
```

从运行结果看，detach之后，join会出现join error，因为tid已经被分离了，是个invalid的内容。

### 线程进程控制原语对比

线程控制原语

```
    pthread_create()
    pthread_self()
    pthread_exit()
    pthread_join()
    pthread_cancel()
    pthread_detach()
```

进程控制原语

```
    fork();
    getpid();
    exit();       / return
    wait()/waitpid()
    kill()
```

### 线程分离属性

可以一开始创建线程时就为线程设置分离属性，否则创建1000个线程就要为1000个线程分别进行detach，很麻烦，此时需要一个结构体，先进行初始化

```c
pthread_attr_t
注意:应先初始化线程属性，再 pthread_create 创建线程 初始化线程属性
int pthread_attr_init(pthread_attr_t *attr); 成功:0;失败:错误号 销毁线程属性所占用的资源
int pthread_attr_destroy(pthread_attr_t *attr); 成功:0;失败:错误号
```

同时，设置分离属性：

```c
设置线程属性，分离 or 非分离
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate); 获取程属性，分离 or 非分离
int pthread_attr_getdetachstate(pthread_attr_t *attr, int *detachstate); 参数: attr:已初始化的线程属性
detachstate: 
PTHREAD_CREATE_DETACHED(分离线程) 
PTHREAD _CREATE_JOINABLE(非分离线程)

```

课堂回收例子：

```c
void * work(void * arg)
{
    printf("Work:thread: pid = %d, tid = %lu\n", getpid(), pthread_self());
    return NULL;
}

int main(void)
{
    pthread_t tid;
    int ret;
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    ret = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    if (ret != 0)
    {
        fprintf(stdout, "set attr error:%s\n", strerror(ret));
        exit(1);
    }
    ret = pthread_create(&tid, &attr, &work, NULL);
    if (ret != 0)
    {
        fprintf(stdout, "create error:%s\n", strerror(ret));
        exit(1);
    }
    printf("Main:thread: pid = %d, tid = %lu\n", getpid(), pthread_self());
    sleep(1);
    ret = pthread_join(tid, NULL);
    if (ret != 0)
    {
        fprintf(stdout, "join error:%s\n", strerror(ret));
        exit(1);
    }

    pthread_attr_destroy(&attr);

    pthread_exit(NULL);
}

➜  chapter5-thread git:(master) ✗ ./set_detach_attr 
Main:thread: pid = 10921, tid = 139861450812288
Work:thread: pid = 10921, tid = 139861441193728
join error:Invalid argument
```

### 线程使用注意事项

1. 主线程退出其他线程不退出，主线程应调用 pthread_exit

2. 避免僵尸线程

   pthread_join
    pthread_detach
    pthread_create 指定分离属性
    被 join 线程可能在 join 函数返回前就释放完自己的所有内存资源，所以不应当返回被回收线程栈中的值;

3. malloc 和 mmap 申请的内存可以被其他线程释放

4. 应避免在多线程模型中调用 fork 除非，马上 exec，子进程中只有调用 fork 的线程存在，其他线程在子进程

   中均 pthread_exit

5. 信号的复杂语义很难和多线程共存，应避免在多线程引入信号机制
