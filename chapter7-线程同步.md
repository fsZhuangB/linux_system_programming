# chapter7-线程同步

## 互斥锁（mutex）

互斥锁实质上是操作系统提供的一把“建议锁”(又称“协同锁”)，建议程序中有多线程访问共享资源的时候使用该机制。但，并没有强制限定。**因此，即使有了 mutex，如果有线程不按规则来访问数据，依然会造成数据混乱。**

下面一个小例子，数据共享导致的混乱：

```c
#include <stdio.h> 
#include <pthread.h> 
#include <unistd.h>
void *tfn(void *arg)
{
    srand(time(NULL));
    while (1)
    {
        printf("hello ");
        sleep(rand() % 3);
        printf("world\n");
        sleep(rand() % 3);
    }
    return NULL;
}
int main(void)
{
    /*模拟长时间操作共享资源，导致 cpu 易主，产生与时间有关的错误*/
    pthread_t tid;
    srand(time(NULL));
    pthread_create(&tid, NULL, tfn, NULL);
    while (1)
    {
        printf("HELLO ");
        sleep(rand() % 3);
        printf("WORLD\n");
        sleep(rand() % 3);
    }
    pthread_join(tid, NULL);
    return 0;
}
// 程序结果

```



## 使用锁

**注意点，初始化时，锁初值为1，加锁后，锁值--，解锁后，锁值++**

类型：

```c
pthread_mutex_t mutex;
```

初始化函数：

```c
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
 const pthread_mutexattr_t *restrict attr)
```

这里的 restrict 关键字，表示指针指向的内容只能通过这个指针进行修改 restrict 关键字:

用来限定指针变量。被该关键字限定的指针变量所指向的内存操作，必须由本指针完成。 初始化互斥量:

初始化该类型：

```c
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL); // 动态初始化。
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER; // 静态初始化。
```

加锁函数：

```c
pthread_mutex_lock;加锁 1 ----> 0
```

解锁函数：

```c
pthrad_mutext_unlock();解锁 0++ --> 1 6. pthead_mutex_destroy;销毁锁
```

destory函数：

```c
pthead_mutex_destroy;销毁锁
```

![Screen Shot 2022-01-08 at 17.04.23](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151510990.png)

对上面的例子进行加锁：

```c
// 全局锁
pthread_mutex_t mutex;

void *tfn(void *arg)
{
    srand(time(NULL));
    while (1)
    {
        pthread_mutex_lock(&mutex);
        printf("hello ");
        // sleep足够长时间使得CPU介入调度，挂起该进程
        sleep(rand() % 3);
        printf("world\n");
        pthread_mutex_unlock(&mutex);

        sleep(rand() % 3);
    }
    return NULL;
}
int main(void)
{
    /*模拟长时间操作共享资源，导致 cpu 易主，产生与时间有关的错误*/
    pthread_t tid;
    //pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    pthread_mutexattr_t mutex_attr;
    // 第二个参数也可以传入NULL
    pthread_mutex_init(&mutex, NULL);
    srand(time(NULL));
    pthread_create(&tid, NULL, tfn, NULL);
    while (1)
    {
        // 主线程加锁
        pthread_mutex_lock(&mutex);
        printf("HELLO ");
        sleep(rand() % 3);
        printf("WORLD\n");
        pthread_mutex_unlock(&mutex);
        sleep(rand() % 3);
    }
    pthread_join(tid, NULL);
    return 0;
}
```

如果将锁移动到sleep之后，会出现下面的情况：

```c
    while (1)
    {
        // 主线程加锁
        pthread_mutex_lock(&mutex);
        printf("HELLO ");
        sleep(rand() % 2);
        printf("WORLD\n");
        sleep(rand() % 2);
        pthread_mutex_unlock(&mutex);
    }

```

将 unlock 挪至第二个 sleep 后，发现交替现象很难出现。因为线程在操作完共享资源后本应该立即解锁，但修改后，线程抱着锁睡眠。睡醒解锁后又立即加锁，这两个 库函数本身不会阻塞。

所以在这两行代码之间失去 cpu 的概率很小。因此，另外一个线程很难得到加锁的机会。

如果在main 中加 flag = 5 将 flag 在 while 中-- 这时，主线程输出 5 次后试图销毁锁，但子线程未将锁释放，将会无法完成：

```c
    while (flag-- > 0)
    {
        // 主线程加锁
        pthread_mutex_lock(&mutex);
        printf("HELLO ");
        sleep(rand() % 2);
        printf("WORLD\n");
        pthread_mutex_unlock(&mutex);
        sleep(rand() % 2);
    }
➜  chapter6-thread-lock git:(master) ✗ ./simple_lock 
➜  chapter6-thread-lock git:(master) ✗ ./simple_lock 
HELLO WORLD
HELLO WORLD
HELLO WORLD
hello world
HELLO WORLD
hello world
HELLO WORLD
hello destory error:
Device or resource busy% 
```

从上面的结果中可以看到，打印了5次后，锁并未被销毁，而是由子线程一直占用着。同时，destroy会抱错

### 锁的粒度

要注意的是，解锁位置放在sleep前和sleep后是有区别的，保证锁的**粒度**越小越好，访问共享数据前加锁，访问完【立刻】解锁。

### pthread_mutex_trylock函数

pthread_mutex_trylock 函数 尝试加锁

```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
```

try 锁:尝试加锁，成功--，加锁失败直接返回错误号(如 **EBUSY**)，不会阻塞等待。

## 死锁问题

出现死锁的原因：（自己整理）

1. 反复lock()同一把锁。
2. 循环等待

作业：针对以上两种情况，写出相应的死锁代码

```C
// 全局锁
pthread_mutex_t mutex1;
pthread_mutex_t mutex2;

void *tfn(void *arg)
{
    srand(time(NULL));
    // 每个线程都成功的锁住了一个mutex
    // 同时试图为另外一个线程已经锁住的mutex加锁
    pthread_mutex_lock(&mutex1);
    pthread_mutex_lock(&mutex2);
    printf("hello ");
    // sleep足够长时间使得CPU介入调度，挂起该进程
    sleep(rand() % 2);
    printf("world\n");
    pthread_mutex_unlock(&mutex1);
    pthread_mutex_unlock(&mutex2);
    sleep(rand() % 2);
    return NULL;
}
int main(void)
{
    /*模拟长时间操作共享资源，导致 cpu 易主，产生与时间有关的错误*/
    pthread_t tid;
    // pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    pthread_mutexattr_t mutex_attr;
    // 第二个参数也可以传入NULL
    pthread_mutex_init(&mutex1, NULL);
    pthread_mutex_init(&mutex2, NULL);
    srand(time(NULL));
    pthread_create(&tid, NULL, tfn, NULL);
    // 主线程加锁
    pthread_mutex_lock(&mutex2);
    sleep(1);
    pthread_mutex_lock(&mutex1);
    printf("HELLO ");
    sleep(rand() % 2);
    printf("WORLD\n");
    pthread_mutex_unlock(&mutex1);
    pthread_mutex_unlock(&mutex2);

    sleep(rand() % 2);
    int ret = pthread_mutex_destroy(&mutex1);
    pthread_mutex_destroy(&mutex2);
    if (ret != 0)
    {
        fprintf(stdout, "destory error:\n%s", strerror(ret));
        exit(1);
    }
    ret = pthread_join(tid, NULL);
    if (ret != 0)
    {
        fprintf(stdout, "join error:%s", strerror(ret));
    }

    return 0;
}
```

上面的代码主要是在循环等待上出了问题：

**每个线程都成功的锁住了一个mutex，同时试图为另外一个线程已经锁住的mutex加锁**

要解决这个问题很简单：

1. 可以定义互斥量的层级关系，当多个线程对一组互斥量进行操作时，总是应该以相同顺序加锁：即对于上面的场景，应该先锁定mutex1，再锁定mutex2，死锁就不会出现：

   ```c
       // 主线程加锁
       pthread_mutex_lock(&mutex1);
       sleep(1);
       pthread_mutex_lock(&mutex2);
       printf("HELLO ");
       sleep(rand() % 2);
       printf("WORLD\n");
       pthread_mutex_unlock(&mutex1);
       pthread_mutex_unlock(&mutex2);
       sleep(rand() % 2);
   // 子线程加锁
       pthread_mutex_lock(&mutex1);
       pthread_mutex_lock(&mutex2);
       printf("hello ");
       // sleep足够长时间使得CPU介入调度，挂起该进程
       sleep(rand() % 2);
       printf("world\n");
       pthread_mutex_unlock(&mutex1);
       pthread_mutex_unlock(&mutex2);
   ```

2. 使用try锁



## 读写锁

特别强调:读写锁只有一把，但其具备两种状态:

1. 读模式下加锁状态 (读锁)
2. 写模式下加锁状态 (写锁)

### 特性

1. 读写锁是“**写模式加锁**”时， 解锁前，所有对该锁加锁的线程都会被阻塞。
2. 读写锁是“**读模式加锁**”时， 如果线程以读模式对其加锁会成功;如果线程以写模式加锁会阻塞。
3. 读写锁是“**读模式加锁**”时， 既有试图以写模式加锁的线程，也有试图以读模式加锁的线程。那么读写锁会阻塞随后的读模式锁请求。优先满足写模式锁。**读锁、写锁并行阻塞，写锁优先级高**。

读写锁也叫共享-独占锁。当读写锁以读模式锁住时，它是以共享模式锁住的;当它以写模式锁住时，它是以独占模式锁住的。**写独占、读共享。**

读写锁非常适合于对数据结构读的次数远大于写的情况。

### 代码实现

## 条件变量

不是锁，但是通常结合锁来使用，包括：

pthread_cond_init 函数

pthread_cond_destroy 函数

pthread_cond_wait 函数

pthread_cond_timedwait 函数

pthread_cond_signal 函数

pthread_cond_broadcast 函数

以上 6 个函数的返回值都是:成功返回 0， 失败直接返回错误号。

pthread_cond_t类型 用于定义条件变量，pthread_cond_t cond;

### pthread_cond_init 函数

```c
1. pthread_cond_init(&cond, NULL);
2. pthread_cond_t cond = PTHREAD_COND_INITIALIZER; 静态初始化。
```

### 条件变量和相关函数 wait

阻塞等待条件变量满足，并加锁互斥量:
 pthread_cond_wait(&cond, &mutex);

结合pthread_cond_signal 函数和pthread_cond_broadcast 函数使用

![Screen Shot 2022-01-10 at 20.35.26](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151511182.png)

## 生产者消费者模型

![Screen Shot 2021-12-19 at 14.47.54](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151601991.png)



![Screen Shot 2022-01-10 at 20.43.23](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151511667.png)

### 条件变量生产者消费者代码

```c
// 实现生产者消费者模型
struct msg
{
    int num;
    struct msg *next;
};
struct msg *header;

pthread_mutex_t mutex;
pthread_cond_t hasData;

void pthread_err(int ret, char *str)
{
    if (ret != 0)
    {
        fprintf(stdout, "%s:%s", str, strerror(ret));
        pthread_exit(NULL);
    }
}

void* consumer(void *arg)
{
    while (1)
    {
    struct msg *mp;

    // 上来就要加锁
    int ret = pthread_mutex_lock(&mutex);
    pthread_err(ret, "wrong lock!\n");
    if (header == NULL)
    {
        // 如果条件变量不满足，则阻塞在此处，并将mutex解锁
        // 满足后，加锁mutex
        pthread_cond_wait(&hasData, &mutex);
    }
    // 当阻塞结束，去读数据
    mp = header;
    header = mp->next;

    pthread_mutex_unlock(&mutex);
    printf("---------consumer:%d\n", mp->num);

    free(mp);
    sleep(rand() % 3);
    }

}

void* producer(void *arg)
{
    while (1)
    {
    struct msg *mp = malloc(sizeof(struct msg));
    // 模拟生产一个数据
    mp->num = rand() % 1000 + 1;
    printf("--produce %d\n", mp->num);

    // 并挂上去
    int ret = pthread_mutex_lock(&mutex);
    mp->next = header;
    header = mp;  // 写公共区域
    pthread_mutex_unlock(&mutex);

    // 通知条件变量
    pthread_cond_signal(&hasData);
    sleep(rand() % 3);
    }
}
int main(void)
{
    int ret;
    pthread_t cid, pid;
    srand(time(NULL));
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&hasData, NULL);

    ret = pthread_create(&cid, NULL, &consumer, NULL);
    pthread_err(ret, "wrong create!\n");
    ret = pthread_create(&pid, NULL, &producer, NULL);
    pthread_err(ret, "wrong create!\n");
    pthread_join(cid, NULL);
    pthread_join(pid, NULL);

    pthread_exit(NULL);
}
```

### 多个消费者

直接多创建几个消费者就可以，逻辑来分析一下：

![Screen Shot 2022-01-15 at 16.13.42](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151613167.png)

比如，两个消费者都阻塞在条件变量上，就是说没有数据可以消费。于是都把锁还回去了，而生产者此时生产了一个数据，会同时唤醒两个因条件变量阻塞的消费者，两个消费者会同时去抢锁。此时假定就是 A 消费者拿到锁，开始消费数据，B 消费者阻塞在锁上。之后 A 消费完数据，把锁归还，B 被唤醒，**然而此时已经没有数据供 B 消费了。**所以这里有个逻辑错误，消费者阻塞在条件变量那里应该使用 while 循环。这样 A 消费完数据后，B 做的第一件事不是去拿锁，而是 判定是否有数据存在，否则

```c
// 此处应该改为while循环    
	while (header == NULL)
    {
        // 如果条件变量不满足，则阻塞在此处，并将mutex解锁
        // 满足后，加锁mutex
        pthread_cond_wait(&hasData, &mutex);
    }
```



