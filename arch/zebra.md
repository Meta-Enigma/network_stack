

[toc]

# Overview

Quagga是Zebra的继承版



> Zebra is a multi–server routing software which provides TCP/IP based routing protocols. Zebra turns your machine into a full powered router.
>
> [Zebra – multi-server routing software](http://www.zebra.org/)

> [Quagga](http://www.quagga.net/) is a routing software suite, providing implementations of OSPFv2, OSPFv3, RIP v1 and v2, RIPng and BGP-4 for Unix platforms, particularly FreeBSD, Linux, Solaris and NetBSD. Quagga is a fork of [GNU Zebra](http://www.zebra.org/) which was developed by Kunihiro Ishiguro.



- 提供类Cisco命令行的分级多用户命令解析引擎，负责对访问的安全验证、数据缓冲、命令解析、模式切换和命令调用

- 提供整套基于tcp/ip网络的路由协议的支持(ripv1,ripv2,bgp等)

- 对程序架构组织，便于剥离实现专用cli程序，或提供其中一类数据结构借助thread机制实现复杂状态机

  

Quagga模型是一组独立守护进程，通过unix domain socket进行IPC。单个守护进程架构遵循event-driven模型。

![image-20230306091306587](https://user-images.githubusercontent.com/30233494/223980450-767cacbe-d969-4dc4-9374-726601fea2df.png)


图中显示3个协议进程，和1管理进程。其中ripd进程处理rip协议，ospfd处理ospf协议，进程可分布在同机器，每个进程通过监听不同端口来VTY连接。

组成：

- 协议模块：实现各种协议功能，以子模块方式加载到zebra中
- 守护进程：管理各协议的信号传输、表项操作、系统操作调用等事务，为各协议提供底层信息以及相关硬件处理等功能支持

模块交互采用C/S模式，每个协议子模块都有一个client，和守护进程中的server进行交互。



传统unix的路由配置通过ifconfig和route命令完成，需root权限，单进程模式

Quagga提供的思路是两种用户模式，多进程模式(各个协议路由守护进程+内核路由管理进程)：

- normal mode：查看系统状态
- enable mode：改系统配置

当前Quagga支持常用单播路由协议：BGP、BFD、OSPF、RIP和IS-IS，MPLS也将支持。LDP也将支持。

多进程带来了良好的扩展性、模块化和可维护性，但同时引入了很多配置文件和终端端口(每个协议进程都有自己的配置文件和终端端口)

静态路由通过zebra配置文件完成，bgp网络配置在bgpd进程配置文件完成



## Thread

Quagga event system可以看成一种用户态线程的实现方式(便于平台间移植)，不同数据结构可看做是thread的变形，主上下文数据结构叫threadmaster，event或task在本系统中都称作thread，数据结构是struct thread。区分于kernel thread(特指pthread)。



### Event Architecture

主结构是struct thread_master，对应threadmaster。一个threadmaster是一个全局状态对象和上下文，保存所有任务基本信息(当前挂起任务、已执行任务统计)。event system触发机制是往thread_master中放tasks，然后调用函数(thread_fetch)获取下一个待执行任务，并执行。初始化时，守护进程会创建thread_master(负责基本初始化，然后循环拿任务执行)。

threadmaster管理6个”线程“队列，分别是read、write、timer、ready和unuse。

**任务类型**(thread.h中定义)：

- thread_read：等待文件描述符可用并读取
- thread_write：等待文件描述符可用并写入
- thread_timer：定时任务，自上次执行后一段时间再次执行
- thread_event：通用任务，高优先级，有整型数值表示event type。通常用于实现fsm（有限状态机），比如理由路由协议。
- thread_ready：内部使用类型，表示在ready queue上的任务
- thread_unused：内部使用，任务执行完放到这里，再次需要新任务时，从这里取出去，避免内存申请

不需要显式操作这些类型，仅需要定义某任务，将其添加到对应任务队列。比如新增thread_read类型任务，直接调用thread_add_read函数，对应任务将被创建，并添加threadmaster对应内部结构。

**任务优先级**

thread_event > thread_timer > thread_ready >  thread_read/thread_write

在thread_fetch函数中有拿任务的逻辑，优先按照event, timer, io顺序拿任务执行。与操作系统的线程优先级有本质区别，不存在抢占等逻辑。这里仅仅按序执行任务，仅当前一个任务执行完成，下一个任务才被执行。

TIPs：

- read和write并没有严格的顺序，在执行read或write之前，或进行select，就绪则放入ready队列，再被执行
- 关联同个文件描述符的任务(read/write)，同一时间仅有一个被执行。比如两个write，一个接一个执行，有可能后执行的任务会覆盖前一个写的内容
- timer任务的时间精确度取决于底层操作系统提供的时钟
- caller负责传入schedule的那些参数内存管理

**任务管理**

借助thread结构将事件和对应函数封装，并将其分类到对应线程链表。通过struct thread_list双向链表管理线程，可进行操作：

- thread_list_add：添加thread到链表尾部
- thread_list_add_before：在指定位置前添加thread，比如在进行链表排序时使用
- thread_list_delete：删除指定thread
- thread_list_free：释放指定链表list中所有thread，比如释放read队列中所有thread
- thread_trim_head：移除list中第一个thread，并将该thread返回。

以read线程为例：创建一个socket，listen该socket并读取

```
#1.创建线程管理者
thread_master_create（）
#2.创建读线程，包含读处理函数等，添加到read队列
thread_add_read（）
#3.select所有描述符，有线程就绪，就取回执行
thread_fetch(检测就绪)-->thread_call(执行)
```

Tips：
任务执行完后，将会被放置到unuse队列，如果想要再次执行，需要将其再次放回read队列。原因见【任务基本操作，call和run区别】



其中thread_fetch核心处理逻辑

```
#1. 优先处理event队列
#2. 处理timer队列
#3. 处理ready中的队列
#4. 处理read队列
thread_process_fd
#5. 处理write队列
thread_process_fd

thread_process_fd：侦听到描述符添加到ready链表
- 从read/write队列移除
- 加到ready队列
```

![image-20230306101448307](https://user-images.githubusercontent.com/30233494/223980531-76cc1587-9f09-4cc3-a415-a2def1abfa82.png)






**任务基本操作**

- thread_call： 执行thread 
- thread_execute：直接创建并执行thread，thread_master参数可以为空 
- thread_cancel：取消一个线程，thread_master不可为空 
- thread_cancel_event：取消所有event链表中参数为arg的线程
- thread_run：与thread_call区别，该函数运行thread，并将其放入unuse队列 
- thread_master_free：释放thread_master和其线程链表



**执行逻辑**

![image-20230306093444547](https://user-images.githubusercontent.com/30233494/223980598-42f36238-3ab3-4828-8117-64653623e703.png)


图中task boxes是当前待执行的任务，其他队列中的其他任务没有表示出来：

task：struct thread *

fetch：thread_fetch()

exec：thread_call

cancel：thread_cancel

schedule：不同类型任务的thread_add_*函数







## VTY

Quagga每个守护进程都可通过CLI(也叫VTY, virtual terminal，命令解析引擎)配置，和其他路由软件CLI工作方式相似。提供vtysh工具作为统一入口连接所有守护进程，管理员可通过该工具访问不同守护进程。

daemon和vthsh间的通信，通过unix domain socket连接每个守护进程，以代理方式读取用户输入。

- vtysh从终端读取cli命令，解析，根据defsh或其他发往指定daemon
- vtysh进程会和每个daemon connect，connect分两种
  - vtysh进程main中connect_default
  - vtysh_send之前会connect对应的daemon



```
vty_init:
	- 初始化vtyvec：全局变量，申请空间
	- 初始化Vvty_serv_thread：全局变量，server_thread，在vty_event()将vty_accept添加到read队列
	- install_node：添加当前配置到vty_node
	- install_element：将注册命令添加到某一个node节点下

以who命令为例
DEFUN(config_who, config_who_cmd, "who", "Display who is on vty\n")
#define DEFUN(funcname, cmdname, cmdstr, helpstr) \
	int funcname(struct cmd_element*, struct vty*, int, char**);\
	struct cmd_elememt cmd_name = {cmdstr, funcname, helpstr, DEFUN_FLAG};\
	int funcname(struct cmd_element *self, struct vty *vty, int argc, char **argv)
参数说明：
config_who: 命令执行函数
config_who_cmd：命令名
"who"：命令字符串
"Display who is on vty\n"：命令帮助字符串

定义过程：
1.声明config_who()
2.初始化struct cmd_element={"who", config_who, "Display who is on vty\n", DEFUN_FLAG}
3.定义config_who()：这里具体实现函数


使用insert_element将config_who_cmd注册到vty，
```





