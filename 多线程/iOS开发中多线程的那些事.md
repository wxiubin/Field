> iOS开发入门之后终究是要接触多线程和runloop的，写篇博客讲解、记录下iOS开发中的多线程技术。
> 博客文章地址：[https://sanyucz.top/2016/05/22/multithreading_iOS/](https://sanyucz.top/2016/05/22/multithreading_iOS/)

## 线程、进程
### 什么是线程、进程
&emsp;&emsp;有的人说进程就像是人的脑袋，线程就是脑袋上的头发~~。其实这么比方不算错，但是更简单的来说，用迅雷下载文件，迅雷这个程序就是一个进程，下载的文件就是一个线程，同时下载三个文件就是多线程。一个进程可以只包含一个线程去处理事务，也可以有多个线程。

### 多线程的优点和缺点
&emsp;&emsp;多线程可以大大提高软件的执行效率和资源（CPU、内存）利用率，因为CPU只可以处理一个线程（多核CPU另说），而多线程可以让CPU同时处理多个任务（其实CPU同一时间还是只处理一个线程，但是如果切换的够快，就可以了认为同时处理多个任务）。但是多线程也有缺点：当线程过多，会消耗大量的CPU资源，而且，每开一条线程也是需要耗费资源的（iOS主线程占用1M内存空间，子线程占用512KB）。
### iOS开发中的多线程
&emsp;&emsp;iOS程序在启动后会自动开启一个线程，称为 **主线程** 或者 **UI线程** ，用来显示、刷新UI界面，处理点击、滚动等事件，所以耗费时间的事件（比如网络、磁盘操作）尽量不要放在主线程，否则会阻塞主线程造成界面卡顿。
iOS开发中的多线程实现方案有四种：


<!--more-->

| 技术方案 | 简介 | 语言 | 生命周期管理 |
| ------------ | ------------ | ------------ | ------------ |
| pthread | 一套通用的多线程API，适用于Unix\Linux\Windows等系统，跨平台\可移植，使用难度大 | C | 程序员管理 |
| NSThread | 使用更加面向对象，简单易用，可直接操作线程对象 | Objective-C | 程序员手动实例化 |
| GCD | 旨在替代NSThread等线程技术，充分利用设备的多核 | C | 自动管理 |
| NSOperation | 基于GCD（底层是GCD），比GCD多了一些更简单实用的功能，使用更加面向对象 | Objective-C | 自动管理 |

多线程中GCD我使用比较多，以GCD为例，多线程有两个核心概念：
1. 任务 （做什么？）
2. 队列 （存放任务，怎么做？）

任务就是你开辟多线程要来做什么？而每个线程都是要加到一个队列中去的，队列决定任务用什么方式来执行。

线程执行任务方式分为：
1. 异步执行
2. 同步执行

同步执行只能在当前线程执行，不能开辟新的线程。而且是必须、立即执行。而异步执行可以开辟新的线程。

队列分为：
1. 并发队列
2. 串行队列

并发队列可以让多个线程同时执行（必须是异步），串行队列则是让任务一个接一个的执行。打个比方说，串行队列就是单车道，再多的车也得一个一个的跑（--：我俩车强行并着跑不行？ --：来人，拖出去砍了！），而串行是多车道，可以几辆车同时并着跑。那么到底是几车道？并发队列有个最大并发数，一般可以手动设置。

那么，线程加入到队列中，到底会怎么执行？

| | 并发队列 | 串行队列（非主队列） | 主队列（只有主线程，串行队列） |
| ----- | ----- | ----- | ----- |
| 同步 | 不开启新的线程，串行 | 不开启新的线程，串行 | 不开启新的线程，串行 |
| 异步 | 开启新的线程，并发 | 开启新的线程，串行 | 不开启新的线程，串行 |

**注意：** 
1. 只用在并发队列异步执行才会开启新的线程并发执行；
2. 在当前串行队列中开启一个同步线程会造成 **线程阻塞** ，因为上文说过，同步线程需要立即马上执行，当在当前串行队列中创建同步线程时需要在串行队列立即执行任务，而此时线程还需要向下继续执行任务，造成阻塞。

上面提到线程会阻塞，那么什么是阻塞？除了阻塞之外线程还有其他什么状态？
一般来说，线程有五个状态：

- 新建状态：线程刚刚被创建，还没有调用 **run** 方法，这个时候的线程就是新建状态；
- 就绪状态：在新建线程被创建之后调用了 **run** 方法，但是CPU并不是真正的同时执行多个任务，所以要等待CPU调用，这个时候线程处于就绪状态，随时可能进入下一个状态；
- 运行状态：在线程执行过 **run**方法之后，CPU已经调度该线程即线程获取了CPU时间；
- 阻塞状态：线程在运行时可能会进入阻塞状态，比如线程睡眠（sleep）；希望得到一个锁，但是该锁正被其他线程拥有。。
- 死亡状态：当线程执行完任务或者因为异常情况提前终止了线程

## iOS开发中的多线程的使用
### pthread的使用

使用下面代码可以创建一个线程：

``` C
int pthread_create(pthread_t * __restrict, const pthread_attr_t * __restrict,void *(*)(void *), void * __restrict)
```

可以看到这个方法有四个参数，主要参数有 **pthread_t * __restrict** ，因为该方法是C语言，所以这个参数不是一个对象，而是一个 **pthread_t** 的地址，还有 **void *(*)(void *)** 是一个无返回值的函数指针。
使用代码：

``` objc
void * run(void *param)
{
    NSLog(@"currentThread--%@", [NSThread currentThread]);
    return NULL;
}

- (void)createThread{
    pthread_t thread;
    pthread_create(&thread, NULL, run, NULL);
}
```

控制台输出：

``` objc
currentThread--<NSThread: 0x7fff38602fb0>{number = 2, name = (null)}
```

**number = 1** 的线程是主线程，不为一的时候都是子线程。

### NSThread的使用
NSThread创建线程一般有三种方式：

``` objc
// equivalent to the first method with kCFRunLoopCommonModes
- (void)performSelectorInBackground:(SEL)aSelector withObject:(nullable id)arg;

+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(nullable id)argument;

- (instancetype)initWithTarget:(id)target selector:(SEL)selector object:(nullable id)argument
```

1. 前两种创建之后会自动执行，第三种方式创建后需要手动执行；
2. 第一种创建方式是创建一个子线程，类似的 **- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array** 方法可以创建并发任务在主线程中执行，**- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array** 可以选择在哪个线程中执行。

示例代码：

``` objc
- (void)createThread{
	// 创建线程
	NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run:) object:@"我是参数"];
	thread.name = @"我是线程名字啊";
	// 启动线程
	[thread start];
	
	// 或者 [NSThread detachNewThreadSelector:@selector(run:) toTarget:self withObject:@"我是参数"];
	// 或者 [self performSelectorInBackground:@selector(run:) withObject:@"我是参数"];
}
- (void)run:(NSString *)param{
    NSLog(@"-----run-----%@--%@", param, [NSThread currentThread]);
}
```

控制台输出：

``` objc
-----run-----我是参数--<NSThread: 0x7ff8a2f0c940>{number = 2, name = 我是线程名字啊}
```


### GCD的使用

苹果官方对GCD说：
> 开发者要做的只是定义执行的任务并追加到适当的 Dispatch Queue 中。

在GCD中我们要做的只是两件事：定义任务；把任务加到队列中。

#### dispatch_queue_create 获取/创建队列

GCD 的队列有两种：

| Dispatch Queue 种类 | 说明 |
| ----- | --------- |
| Serial Dispatch Queue | 等待现在执行中处理结束（串行队列） |
| Concurrent Dispatch Queue | 不等待现在执行中处理结束（并行队列） |

GCD中的队列都是 **dispatch_queue_t** 类型，获取/创建方法：

``` objc 
// 1. 手动创建队列
dispatch_queue_t dispatch_queue_create(const char *label, dispatch_queue_attr_t attr);
// 1.1 创建串行队列
    dispatch_queue_t queue = dispatch_queue_create("com.sanyucz.queue", DISPATCH_QUEUE_SERIAL);
// 1.2 创建并行队列
    dispatch_queue_t queue = dispatch_queue_create("com.sanyucz.queue", DISPATCH_QUEUE_CONCURRENT);
    
// 2. 获取系统标准提供的 Dispatch Queue
// 2.1 获取主队列
dispatch_queue_t queue = dispatch_get_main_queue();
// 2.2 获取全局并发队列
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

需要说明的是，手动创建队列时候的两个关键参数，**const char *label** 指定队列名称，最好起一个有意义的名字，当然如果你想调试的时候刺激一下，也可以设置为 **NULL**，而 **dispatch_queue_attr_t attr** 参数文档有说明：

``` objc
/*!
 * @const DISPATCH_QUEUE_SERIAL
 * @discussion A dispatch queue that invokes blocks serially in FIFO order.
 */
#define DISPATCH_QUEUE_SERIAL NULL

/*!
 * @const DISPATCH_QUEUE_CONCURRENT
 * @discussion A dispatch queue that may invoke blocks concurrently and supports
 * barrier blocks submitted with the dispatch barrier API.
 */
#define DISPATCH_QUEUE_CONCURRENT \
		DISPATCH_GLOBAL_OBJECT(dispatch_queue_attr_t, \
		_dispatch_queue_attr_concurrent)
```

- **DISPATCH_QUEUE_SERIAL** 创建串行队列按顺序FIFO（First-In-First-On）先进先出；
- **DISPATCH_QUEUE_CONCURRENT** 则会创建并发队列

#### dispatch_async/dispatch_sync 创建任务

创建完队列之后就是定义任务了，有两种方式：

``` objc
// 创建一个同步执行任务
void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
// 创建一个异步执行任务
void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```

完整的示例代码：

``` objc
dispatch_queue_t queue = dispatch_queue_create("com.sanyucz.queue.asyncSerial", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{
   NSLog(@"异步 + 串行 - %@",[NSThread currentThread]);
});
```

#### dispatch group 任务组

我们可能在实际开发中会遇到这样的需求：在两个任务完成后再执行某一任务。虽然这种情况可以用串行队列来解决，但是我们有更加高效的方法。

直接上代码，在代码的注释中讲解：

``` objc
// 获取全局并发队列
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
// 创建任务组
// dispatch_group_t ：A group of blocks submitted to queues for asynchronous invocation
dispatch_group_t group = dispatch_group_create();
// 在任务组中添加一个任务
dispatch_group_async(group, queue, ^{
	
});
// 在任务组中添加另一个任务
dispatch_group_async(group, queue, ^{
	
});
// 当任务组中的任务执行完毕之后再执行一下任务
dispatch_group_notify(group, queue, ^{
   
});
```

#### dispatch_barrier_async 

从字面意思就可以看出来这个变量的用处，即阻碍任务执行，它并不是阻碍某一个任务的执行，而是在代码中，在它之前定义的任务会比它先执行，在它之后定义的任务则会在它执行完之后在开始执行。就像一个栏栅。

使用代码：

``` objc
dispatch_queue_t queue = dispatch_queue_create("com.gcd.barrier", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(queue, ^{
   NSLog(@"----1-----%@", [NSThread currentThread]);
});
dispatch_async(queue, ^{
   NSLog(@"----2-----%@", [NSThread currentThread]);
});
dispatch_barrier_async(queue, ^{
   NSLog(@"----barrier-----%@", [NSThread currentThread]);
}); 
dispatch_async(queue, ^{
   NSLog(@"----3-----%@", [NSThread currentThread]);
});
dispatch_async(queue, ^{
   NSLog(@"----4-----%@", [NSThread currentThread]);
});
```

控制台输出：

``` objc
----1-----<NSThread: 0x7fdc60c0fd90>{number = 2, name = (null)}
----2-----<NSThread: 0x7fdc60c11500>{number = 3, name = (null)}
----barrier-----<NSThread: 0x7fdc60c11500>{number = 3, name = (null)}
----3-----<NSThread: 0x7fdc60c11500>{number = 3, name = (null)}
----4-----<NSThread: 0x7fdc60c0fd90>{number = 2, name = (null)}
```

#### dispatch_apply 遍历执行任务

**dispatch_apply** 的用法类似于对数组元素进行 **for循环** 遍历，但是 **dispatch_apply** 的遍历是无序的。

``` objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
NSMutableArray *array = [NSMutableArray array];
for (int i = 0; i < 10; i++) {
   [array addObject:@(i)];
}
// array = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
NSLog(@"--------apply begin--------");
dispatch_apply(array.count, queue, ^(size_t index) {
   NSLog(@"%@---%zu", [NSThread currentThread], index);
});
NSLog(@"--------apply done --------");
```

控制台输出：

``` objc
--------apply begin--------
<NSThread: 0x7ffa7bd05800>{number = 1, name = main}---0
<NSThread: 0x7ffa7bd05800>{number = 1, name = main}---4
<NSThread: 0x7ffa7bd05800>{number = 1, name = main}---5
<NSThread: 0x7ffa7bda77c0>{number = 2, name = (null)}---1
<NSThread: 0x7ffa7be1fd00>{number = 4, name = (null)}---3
<NSThread: 0x7ffa7bd05800>{number = 1, name = main}---6
<NSThread: 0x7ffa7be1a920>{number = 3, name = (null)}---2
<NSThread: 0x7ffa7bd05800>{number = 1, name = main}---8
<NSThread: 0x7ffa7be1fd00>{number = 4, name = (null)}---9
<NSThread: 0x7ffa7bda77c0>{number = 2, name = (null)}---7
--------apply done --------
```

可以看到，遍历的时候自动开启多线程，可以无序并发执行多个任务，但是有一点可以确定，就是 **NSLog(@"--------apply done --------");** 这段代码一定是在所有任务执行完之后才会去执行。

#### GCD 的其他用法

除了上面的那些，GCD还有其他的用法

- dispatch_after 延期执行任务
- dispatch_suspend / dispatch_resume 暂停/恢复某一任务
- dispatch_once 保证代码只执行一次，而且线程安全
- Dispatch I/O 可以以更小的粒度读写文件

### NSOperation的使用

#### NSOperation 及其子类
**NSOperation** 和 **NSOperationQueue** 配合使用也能实现并发多线程，但是需要注意的是 **NSOperation** 是个抽象类，想要封装操作需要使用其子类。
系统为我们提供了两个子类：

- NSInvocationOperation
- NSBlockOperation

当然，我们也可以自定义其子类，只是需要重写 **main()** 方法。

先看下系统提供两个子类的初始化方法：

``` objc
- (nullable instancetype)initWithTarget:(id)target selector:(SEL)sel object:(nullable id)arg;
+ (instancetype)blockOperationWithBlock:(void (^)(void))block;
```

两个子类初始化方法不一样的地方就是一个用 **实例对象** 和 **方法选择器** 来确定执行一个方法，另外一个是用block闭包保存执行一段代码块。
另外 **NSBlockOperation** 还有一个实例方法 **- (void)addExecutionBlock:(void (^)(void))block;** ，只要调用这个方法以至于封装的操作数大于一个就会开启新的线程异步操作。
最后调用**NSOperation**的**start**方法启动任务。

#### NSOperationQueue
**NSOperation** 默认是执行同步任务，但是我们可以把它加入到 **NSOperationQueue** 中编程异步操作。

``` objc
- (void)addOperation:(NSOperation *)op;
- (void)addOperations:(NSArray<NSOperation *> *)ops waitUntilFinished:(BOOL)wait;
- (void)addOperationWithBlock:(void (^)(void))block;
```

之前提到过多线程并发队列可以设置最大并发数，以及队列的取消、暂停、恢复操作：

``` objc
// 创建队列
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    
// 设置最大并发操作数
queue.maxConcurrentOperationCount = 2; // 并发队列
queue.maxConcurrentOperationCount = 1; // 串行队列

 // 恢复队列，继续执行
queue.suspended = NO;
// 暂停（挂起）队列，暂停执行
queue.suspended = YES;

// 取消队列
[queue cancelAllOperations];
```

### 线程安全

多线程使用的时候，可能会多条线程同时访问/赋值某一变量，如不加限制的话多相处同时访问会出问题。具体情况可以搜索一下相关资料，多线程的 **买票问题** 很是经典。
iOS线程安全解决方法一般有以下几种：

- @synchronized 关键字
- NSLock 对象
- NSRecursiveLock 递归锁
- GCD (dispatch_sync 或者 dispatch_barrier_async)

在iOS中线程安全问题一般是关键字 **@synchronized** 用加锁来完成。
示例代码：

``` objc
@synchronized(self) {
      // 这里是安全的，同一时间只有一个线程能到这里哦~~      
}
```
需要注意的是 **synchronized** 后面括号里的 **self** 是个 **token** ，该 **token** 不能使用局部变量，应该是全局变量或者在线程并发期间一直存在的对象。因为线程判断该加锁的代码有没有线程在访问是通过该 **token** 来确定的。

