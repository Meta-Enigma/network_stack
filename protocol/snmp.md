[toc]

# SNMP

Simple Network Management Protocol

广泛应用于TCP/IP网络的网络管理标准协议，该协议能够支持网络管理系统，用以监测连接到网络上的设备是否有任何引起管理上关注的情况。

SNMP采用轮询机制，提供最基本的功能集，适合小型、快速、低价格的环境使用，而且SNMP以用户数据报协议（UDP）报文为承载，因而受到绝大多数设备的支持，同时保证管理信息在任意两点传送，便于管理员在网络上的任何节点检索信息，进行故障排查。



## 背景

网络设备管理困难增加:

- 数量多：数量几何级别增加，同时网络作为分布式系统，覆盖区域增加，急剧增加设备实时监控和故障排查的困难
- 种类多：不同设备厂商提供的管理接口也不同

SNMP是广泛应用于TCP/IP网络的网络管理标准协议，该协议能够支持网络管理系统，用以监测连接到网络上的设备是否有任何引起管理上关注的情况。

通过“利用网络管理网络”的方式：

- 提供集中化平台：完成任意节点信息查询、修改和故障排查等工作
- 提供最基本功能集：管理任务和被管理设备的物理特性、网络类型解耦，屏蔽设备间物理差异
- 设计简单、运行代价低：遵循KISS原则，在设备上添加软硬件、报文种类和格式都力求简单



## 基本组件

![image-20230309155001768](https://user-images.githubusercontent.com/30233494/223978631-972dd0f6-6cb6-4910-9913-92a883cda27d.png)



NMS：Network Management System，管理者，运行在NMS服务器，采用SNMP协议管理监控设备

- 向设备上的Agent发出请求，查询或修改一个或多个参数值
- 接收设备上Agent主动发送的Trap信息，来获取当前被管理设备当前状态

Agent：被管理设备中的一个代理进程，用于维护被管理设备的信息数据并响应来自NMS的请求，把管理数据汇总发送请求给NMS

- 接收NMS请求信息，通过MIB表完成相应指令指令，并将操作结果发给NMS
- 设备故障或其他事件时，设备通过Agent主动发送信息给NMS，报告当前设备状态变化

MIB：Management Information Base，数据库，指明被管理设备所维护的变量，是能够被Agent查询和设置的信息。定义了被管理对象一系列属性：对象的名称、状态、访问权限和数据类型等

- Agent查询MIB，获取设备当前状态信息
- Agent修改MIB，设置设备状态参数

Managed Object：每个设备可能包含多个被管理对象，管理对象可以是设备中某个硬件，也可以是在硬件、软件（如路由选择协议）上配置的参数集合



## 协议

SNMP有三种版本：SNMPv1，SNMPv2c和SNMPv3。

- SNMPv1：第一个版本，它提供了一种监控和管理计算机网络的系统方法，它基于团体名认证，安全性较差，且返回报文的错误码也较少。它在RFC 1155和RFC 1157中定义。
- SNMPv2c：第二个版本SNMPv2c引入了GetBulk和Inform操作，支持更多的标准错误码信息，支持更多的数据类型。它在RFC 1901，RFC 1905和RFC 1906中定义。
- SNMPv3：鉴于SNMPv2c在安全性方面没有得到改善，IETF颁布了SNMPv3版本，提供了基于USM（User Security Module）的认证加密和基于VACM（View-based Access Control Model）的访问控制，是迄今为止最安全的版本。SNMPv3在RFC 1905，RFC 1906，RFC 2571，RFC 2572，RFC 2574和RFC 2575中定义。

### 端口

SNMP通信端点

- UDP传输，端口号为161/162。
- TLS或DTLS：10161/10162

### SNMPv3


![image-20230309155422124](https://user-images.githubusercontent.com/30233494/223978690-fd0a1f13-9d16-43f2-a230-4970001730ab.png)



- 版本：表示SNMP的版本，SNMPv3报文则对应字段值为2。
- 报头数据：主要包含消息发送者所能支持的最大消息尺寸、消息采用的安全模式等描述内容。
- 安全参数：包含SNMP实体引擎的相关信息、用户名、认证参数、加密参数等安全信息。
- Context EngineID：SNMP唯一标识符，和PDU类型一起决定应该发往哪个应用程序。
- Context Name：用于确定Context EngineID对被管理设备的MIB视图。
- SNMPv3 PDU：包含PDU类型、请求标识符、变量绑定列表等信息。其中SNMPv3 PDU包括GetRequest PDU、GetNextRequest PDU、SetRequest PDU、Response PDU、Trap PDU和GetBulkRequest PDU。

安全性提升：

- USM：User Security Model，用户安全模块。提供身份验证和数据加密服务，NMS和Agent必须共享同一密钥
  - 身份验证：NMS和Agent双方接收消息先确认信息是否来自有权限NMS或Agent，并且数据未被改变。SNMP使用HMAC分为三种，哈希函数分别是MD5、SHA-1和SHA-2
  - 加密：对称密钥系统，相同密钥加密或解密。加密算法：
    - DES：56bit密钥对64bit明文块加密
    - AES：使用128bit、192bit或256bit密钥长度的AES算法对明文加密
- VACM：View-based Access Control Model，对用户组或者团体名实现基于视图的访问控制。用户必须首先配置一个视图，并指明权限。用户可以在配置用户或者用户组或者团体名的时候，加载这个视图达到限制读写操作或Trap的目的。

HMAC：Hash-based Message Authentication Code，验证信息完整性和身份认证

#### 通信实例

以NMS查询MIB节点sysContact值为例

![image-20230309155513358](https://user-images.githubusercontent.com/30233494/223978738-bc1fc638-49f0-427f-a641-2172bb16b4ab.png)


1. NMS：向Agent发送不带安全参数的Get请求报文，向Agent获取Context EngineID、Context Name和安全参数（SNMP实体引擎的相关信息）。
2. Agent：响应NMS的请求，并向NMS反馈请求的参数。
   1. Engine ID：可配置基于eth1的MAC地址计算出来，可配置基于第一个IP地址计算，也可显式手工指定
3. NMS：再次向Agent发送Get请求报文，报文中各字段的设置如下：
   - 版本：SNMPv3版本。
   - 报头数据：指明采用认证、加密方式。
   - 安全参数：NMS通过配置的算法计算出认证参数和加密参数。将这些参数和获取的安全参数填入相应字段。
   - PDU：将获取的Context EngineID和Context Name填入相应字段，PDU类型设置为Get，绑定变量填入MIB节点名sysContact，并使用已配置的加密算法对PDU进行加密。
4. Agent：首先对消息进行认证，认证通过后对PDU进行解密。解密成功后，Agent根据请求查询MIB中的sysContact节点，得到sysContact的值并将其封装到Response报文中的PDU，并对PDU进行加密，向NMS发送响应。如果查询不成功或认证、解密失败，Agent会向NMS发送出错响应。



## 工作原理

SNMP启动，NMS对设备进行管理。

NMS—》Agent—》MIB

Agent运行在设备上，作为代理，对设备端MIB操作，完成NMS的指令。

工作原理：将协议数据单元（SNMP GET）发送到响应SNMP的网络设备，用户通过网络监控工具可以跟踪所有通信过程，并从SNMP获取数据。


![image-20230309160157510](https://user-images.githubusercontent.com/30233494/223978888-d2a88668-4908-4cfc-b8e4-31ae214ef32c.png)





### SNMP Trap

SNMP Agent主动将设备产生的告警或事件上报给NMS，以便管理员及时了解设备当前运行状态

方式：

- Trap：直接上报，无回复，设备自发行为。比如被管理设备热重启后，SNMP Agent会向NMS发送warmStart的Trap。【如果丢了怎么处理？】
  - 信息受限，只有设备端达到告警等触发条件才会发送
  - 仅在严重事件发生时才发送，减少报文交互产生的流量
- Inform：上报告警或事件后，NMS需回复InformResponse确认
  - 告警或事件暂时保存Inform缓存
  - 重复发送该告警或事件，直到NMS确认收到该告警或达到最大重试次数
  - 被管理设备会生成相应告警或事件日志



## net-snmp

开源snmp协议实现，支持3个版本snmp协议，支持ipv4和ipv6，也包含snmp trap所有相关实现。

### snmpd

nmp agent，绑定某端口，等待NMS来的请求，解析并指定对应操作，返回请求的信息

默认情况，监听所有ipv4接口上端口号为161的UDP SNMP报文，可通过snmpd启动命令修改指定接口，比如

~~~
127.0.0.1:161  只监听环回端口，不接收远端snmp请求。161是默认端口，可不指定
tcp:1161  所有ipv4接口，tcp端口1161的报文，也可指定ipv4地址
~~~

http://www.net-snmp.org/docs/man/snmpd.html





# MIB

采用树结构，下图为MIB的一部分，称为对象命名树。

OID(Object Identifier)对象标识符，对应树中一个管理对象。树中每个分支均包含一个数字和名称，从根到该点的路径即为OID，比如system的OID是1.3.6.1.2.1.1，OID用于唯一标识管理对象在MIB树中位置，由SMI(Structure  of Management Information)保证OID不会冲突。

![image-20230309155208511](https://user-images.githubusercontent.com/30233494/223978933-80f934e6-223e-41a5-a7f4-a9992e2723d9.png)


## 分类

- 公有MIB：一般由RFC定义，主要用来对各种公有协议进行结构化设计和接口标准化处理。大多数的设备制造商都需要按照RFC的定义来提供SNMP接口。
- 私有MIB：是公有MIB的必要补充，当公司自行开发私有协议或者特有功能时，可以利用私有MIB来完善SNMP接口的管理功能，同时对第三方网管软件管理存在私有协议或特有功能的设备提供支持。

MIB节点的最大访问权限表明NMS能够通过该MIB节点对设备进行的操作：不可访问、只读、读写、可读可创建(增删改查)、仅通知(Trap绑定的变量)



## MIB浏览器

工具，方便简洁地访问整个MIB数据库，可遍历整棵树，通常以图形方式显示各个分支和树叶对象。

MIB Browser可作为NMS，配置snmp agent的ip地址以及snmp协议相关参数，可进行Query检查连接状态。可配置Trap告警的UDP接口，这样在交换机侧配置使能Trap，即可接收Trap信息。



## 结构

### 变量

- 标量：scalar variable，一个标量对象只有一个对象实例.
  - 标量对象实例访问方式：OID + .0，比如sysName变量OID=“.iso.org.dod.internet.mgmt.mib-2.system.sysName”，标识便是“.iso.org.dod.internet.mgmt.mib-2.system.sysName.0”
- 表格变量：table，对象的有序集合(标量的集合)，包含若干行，用来完整描述一条信息，比如tcpConnTable需要包含状态，地址，端口号等基本信息。表格对象通常叫列对象
  - 访问方式：OID + 行索引(RowIndexValue)联合标识实例，index的个数可以是1个或多个
  - 常见操作：取值(整表、行值、列值)、修改值、添加行、删除行和遍历整个表【比标量多了增删动作】

https://blog.csdn.net/nineday/article/details/1769376



### 语法

https://www.cnblogs.com/kiko2014551511/p/13365850.html





# Develop

## RPC

### rpcgen原理

~~~
### test.x
program TESTPROG {
	version VERSION {
	string TEST(string) = 1;
	} = 1;
} = 87654321;

87654321是RPC程序编号， VERSION版本号为1，均给RPC服务程序使用，同时指定程序接收一个字符串参数


服务端函数名

### 1.根据规格文件(.x)生成标准源文件，不需要修改
rpcgen test.x --> 生成3个源文件：test_clnt.c test.h test_svc.c

test_clnt.c：客户端调用函数，即远程调用函数接口，include <test.h>，
test.h：一些公用头文件，一些宏定义
test_svc.c：服务器端代码，定义testprog_1函数，并注册到主机

### 2. 生成客户端和服务端代码
#客户端
## 创建指定host的rpc client，
rpcgen -Sc -o test_client.c test.x
#服务端
## 对应rpc接口实现，将客户端需要的结果保存返回
rpcgen -Ss -o test_srv_func.c test.x

#生成服务端程序
gcc -Wall -o test_server test_clnt.c test_srv_func.c test_svc.c
#生成客户端程序
gcc -Wall -o test_client test_client.c test_clnt.c

### 3. 运行
./test_server
./test_client 127.0.0.1



TESTPROG和VERSION将出现在函数名中，即
客户端函数名testprog_1()-->test_1()，test_client.c：
1. clnt_create创建client handler，带remote host地址,program prognum->87654321和version->1等
2. test_1()调用clnt_call远程调用，返回值保存
3. 【添加逻辑来处理服务端的返回值】

服务端函数名test_1_svc()，test_srv_func.c:
1. 【添加具体实现】


clnt_create：指定server地址，program和version num
clnt_call：调用server某个过程，返回码表示是否成功执行，返回值保存在传入的参数中
clnt_destroy：关闭连接
svcudp_create：生成udp服务的svcprt结构指针
svctcp_create：生成tcp服务的svcprt结构指针
svc_register：向主机注册一个服务，参数包含处理请求函数名
svc_run：服务端注册完提供服后，调用该函数等待请求
svc_sendreply：服务端收到请求后，处理后调用该函数向客户端发送报文
~~~

rpcgen用法：https://blog.csdn.net/hitxiaotao/article/details/2267523

### 工作机制

server：主进程uifmd

~~~
ifm_main.c --> uifm_module_init()
#初始化rpc server-->rpc_svc_tcp_init()


rpc_svc.c
rpc_svc_tcp_init()
{
	#1. svctcp_create初始化tcp服务的svcprt结构指针
	#2. setsockopt设置socket tcp相关选项
	#3. svc_register将远程调用过程(rpc函数)注册到主机
	#4. thread_add_read将rpc_svc_socket_sync挂到read队列，同时根据fd挂载rpc_svc_packet_rcv
			到read队列
}
~~~

client：snmpd

~~~
将rpcclinet编成动态库，使用dlmod加载到snmpd，完成通信初始化，即执行init_rpcclnt()【初始化机制参考MIB加载机制】
TIPs：
1. rpcclient要最先加载初始化，否则MIB库无法使用rpc
~~~



问题：socket初始化后，并未释放。通过deinit_rpccln()？





## MIB

### mib2c生成源码

~~~
env MIBS="+RFC1213-MIB.txt" mib2c -c mib2c.iterate.conf ifTable
env MIBS="+RFC1213-MIB.txt" mib2c -c mib2c.scalar.conf interfaces
~~~

静态库操作流程：https://blog.csdn.net/qq_27204267/article/details/51595708



需要具体实现回调函数，即handler_access_method。因为snmp作为独立进程，需要借助rpc从uifmd获取sdk的一些基础信息

~~~
##指定OID对应回调函数
netsnmp_handler_registration *
netsnmp_create_handler_registration(const char *name,
                                    Netsnmp_Node_Handler *
                                    handler_access_method, const oid * reg_oid,
                                    size_t reg_oid_len, int modes)

1. 申请MIB handler结构体空间，用name和handler_access_method回调函数初始化结构体
2. 封装内层handler到外层结构体，使用节点oid和modes来初始化完整的注册信息
~~~



mib2c执行模式

1. 缓存方式，老化时间在.h中定义
2. 



### MIB加载机制

加载方式：

1. 静态加载：将生成的.c和.h文件加到对应位置，重新编译snmp库，不改配置，但每次都需要重新编译
2. 动态加载：将生成的.c和.h编译成.so库，修改snmpd.conf，不需重新编译snmpd，但要借助dlmod命令
3. 子代理扩展：将生成.c和.h编译成可执行文件，同时运行和snmpd，简单，但要运行两个程序

加载参考：https://www.cnblogs.com/shenlinken/p/5316021.html



~~~
#1. 加载共享库到内存
#2. object loader在共享库中查找初始化函数，即init_xxx，xxx是扩展库的名称。比如init_interfaces()
#3. snmpd关闭，调用deinit_xxx函数(init_xxx成对)卸载模块


# cat snmpd.conf 
rocommunity public
rwcommunity public

dlmod rpcclnt /kgmicro/usrfs/lib_mib_rpcclnt.so
dlmod interfaces /kgmicro/usrfs/lib_mib_interfaces.so
dlmod ifTable /kgmicro/usrfs/lib_mib_ifTable.so


#TIPs：
#1. object load仅会调用这两个函数，init和deinit
#2. agent对多次加载相同共享库并未保护，模块并非再加载一次，而是再次执行init，deinit也是
~~~

！！！参考文档：http://www.net-snmp.org/wiki/index.php/TUT%3aWriting_a_Dynamically_Loadable_Object



检查snmpd是否支持dlmod

~~~
#snmpd 编译选项支持动态加载
./configure --enable-shared
#检查是否支持
snmpd -H|grep dlmod
dlmod                    module-name module-path
~~~

http://blog.chinaunix.net/uid-23097562-id-2550089.html





## snmpd



启动方式

~~~
./snmpd -f -Le -c snmpd.conf
~~~





