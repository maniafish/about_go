Go的协程
---

---

# Go协程和线程的区别

* 资源调度
	* 线程由内核调度，根据cpu时间片执行抢占式调度
	* 协程由程序调度(runtime包)，执行协同式调度(2中会详述)

* 内存占用
	* 执行线程所需的栈内存至少是MB级别
	* 执行协程只需要4KB左右的栈内存

* 上下文切换
	* 线程涉及到用户态和内核态的切换：需要通用寄存器(8个)，程序计数器PC，指令寄存器IR，地址寄存器AR，累加寄存器AC，状态寄存器EFLAGS等
	* 协程上下文切换只涉及到栈指针和三个寄存器(程序计数器PC, 栈指针寄存器SP, 数据寄存器DX）的切换

ps: 单核多线程未必会提高效率，更多的抢占式调度和上下文切换，有时反而会让效率降低；经验之谈：3 thread per core is best(from William Kennedy)

> 参考链接
> 
> * [Golang协程详解](http://www.cnblogs.com/liang1101/p/7285955.html)
> * [通用寄存器](https://blog.csdn.net/sinat_38972110/article/details/72927858)

# Go协程调度

![goroutine](./images/goroutine.jpg)

* M：内核线程
* G：go routine，并发的最小逻辑单元，由程序员创建
* P：处理器，执行G的上下文环境，每个P会维护一个本地的go routine队列

goroutine有三个状态：

* waiting: 协程处于全局的队列等待调度
* runnable: 协程处于本地队列，等待执行
* running: 协程正在运行

## goroutine的创建

1. go调用`runtime.newproc()`方法来创建G
2. 首先，检查当前P的空闲队列中有没有可用的G，如果有，就直接从中取一个；如果没有，则分配一个新的G，挂载到P的本地队列中
3. 获取了G之后，将调用参数保存到G的栈中，将SP, PC等上下文环境保存到G的sched域中
4. 此时的G出于runnable状态，一旦分配到CPU，就可以进入running状态

## G何时被调度

1. 当G被创建时，会立即获得一次运行的机会
2. 如果此时正在运行的P的数量没有达到上限，go会调用`runtime.wakep()`方法唤醒P；然后调度器选择M绑定P来执行G，必要时会新建M
3. 当此时正在运行的P数量到达上限时，G会进入本地队列等待，当队列前面的G处于以下几种状态时，会触发切换，进入wait状态：

	* 加锁
	* io操作
	* 系统调用
	* 运行时间过长(runnable)

## G的消亡

1. 当G执行完毕返回后，go会调用`runtime.exit()`方法回收G(包括栈, SP, PC...)
2. 然后将G放入P的空闲队列中，等待`runtime.newproc()`方法取出

> 参考链接
> 
> * [golang之协程](http://www.cnblogs.com/chenny7/p/4498322.html)
> * [goroutine的生老病死](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html)
> * [谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)