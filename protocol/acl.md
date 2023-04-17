

# 背景

Access Control List，一种报文过滤技术，由一条或多条规则组成的集合。

工作原理：读取报文，根据配置好的过滤规则，匹配字段，执行对应的动作。

## 目的

安全、QoS。即网络环境安全性和网络服务质量的可靠性

## 原理

一系列规则组成，通过将报文与ACL规则匹配，过滤出特定的报文

### 组成

1. 编号：标识ACL，ACL规则功能不同，分成不同类型，对应编号取值范围也不同
2. 规则：描述报文匹配条件的判断语句
   1. 规则编号：可自行编号，也可系统自动分配。匹配顺序从编号小的到大的，“匹配则停止”
   2. 动作：permit/deny等
   3. 匹配项：报文各个字段等



### 分类

- 基于标识

  - 数字型：传统标识方法，创建时指定唯一数字标识
  - 命名型：名称代替编号，方便记忆。可同时指定数字，否则系统将自动分配一个数字编号。

- IPv4和IPv6划分

  - ACL4：通常就叫做ACL，特指支持过滤ipv4报文
  - ACL6：特指支持过滤ipv6报文

- 基于规则定义方式

  - 基本、高级、二层、用户自定义、基本ACL6、高级ACL6。它们分别对应不同的编号范围

  

### 步长

系统自动分配编号时，相邻规则编号的差值。

首条未指定编号的规则，其编号就是步长值。比如步长为5，那么在添加首条规则，且未指定编号，系统分配的编号就是5，下一条未指定的编号就是10，依次类推。但如果有一条手工添加12的规则，那么一条自动编号则为15。

所以，自动编号一定是步长的整数倍，且比当前最大编号大的区间最小数值。



**作用**：留一定空间，方便后续在旧规则前添加新规则





### 匹配机制

匹配结果

- 命中：存在ACL，且查到匹配规则。命中的动作按匹配规则定义
- 未命中：不存在ACL，或ACL无规则，或遍历ACL未命中任何规则。



### 匹配顺序

规则间可能存在一定的重复和矛盾，因此，需要有一定的匹配顺序规定

- 配置顺序：缺省，按照编号排序，越小越早被匹配
- 自动顺序：深度优先，按精确度从高到低排序，匹配项越严格，优先级越高。



### 生效时间

配置ACL规则生效的时间段，更加合理地规划网络。

- 周期时间段：每天、每周某天中某个时间段
- 绝度时间段：从某个时间到某个时间



### 常用匹配项

- 生效时间段
- ip协议类型
- 源、目的ip：带通配符掩码，如ip192.168.1.0掩码0.0.0.255，则代表192.168.1.*这个网段的地址
- 源、目的mac：带通配符掩码，掩码ffff-ffff-ffff则表示，精确匹配某个mac地址
- vlan：也通配符掩码，0xfff则代表仅匹配配置的vlan
- tcp/udp端口号：
- tcp标志信息：tcp报文头的标志位，如ack, rst, syn, fin等
- ip分片信息：ip分片报文，仅首包带四层信息，后续非首包分片报文均不带。网络设备在收到分片报文后，会检查是否为最后一个分片，不是则继续分配内存，等待最后一个分片来重组报文。利用该机制，可向接收设备发起分片报文共计，始终不发最后一个分片，使得接收设备内存得不到释放。设置none-first-fragment匹配阻塞非首片分片报文，防止攻击。



# kgtsn配置

相关寄存器**DsTcamAclQosMacKey**

目前，关心字段

- src mac：mac地址及mask
- dst mac：mac地址及mask
- vlan：vlan值及mask
- action：discard 0/1，0代表放行，1代表丢弃



## 匹配原理

### 匹配顺序

group是逻辑概念，芯片底层，依然按照entry(rule)下发顺序匹配。可以通过设置entry的优先级，暂不支持。



默认情况，ip报文仅匹配L3 ACL，也就是说L2 ACL对ip报文无效。如果使能forcemac，则会强制匹配L2 ACL，不会再走L3规则。

报文ACL只能查找一次，L2或L3，二选一。

说明：

- 二层和三层的报文，分开查找。L2查找mac ACL，三层 IP ACL
  - 报文有ip信息，L2 ACL无效。如需对mac层也生效，则需要在ip acl规则里添加对应mac字段
    - 未配置L3 acl enable或无规则，则优先匹配L3 ACL，即默认处理（目前放行）
    - 配置forcemac，强行匹配L2 ACL
  - 报文无ip信息，则会匹配L2 ACL
- 



寄存器说明**DsMemSrcPort**：

- l2AclEn：使能l2 acl
- l2AclHiPrio：在label场景（port和vlan label），置位按照port label(l2 label)，否则走vlan label(l3 label)
- ipv4ForceMacKey：使ip报文匹配l2 ACL
- l2AclLabel：分类标签，port和vlan group二选一，label全局唯一。



**举例**

L2 ACL规则： 精确匹配源MAC_A，丢弃。即DsTcamAclQosMacKey中的字段。

- 报文1：源MAC_A，带ip字段
- 报文2：源MAC_A，不带ip字段

结果

- 配置l2AclEn：报文1丢弃，报文2通行。
- 配置l2AclEn + ipv4ForceMacKey：报文1和2均丢弃



## 应用场景

### label使用

```

KGXX> enable
KGXX# configure terminal 
KGXX(config)# configure acl 
KGXX(config-acl)#acl port 1/2 label 2
```



### 错误mac过滤



```
KGXX(config-acl)#acl port 1/2 enable 
KGXX(config-acl)#acl port 1/2 forcemac enable 
KGXX(config-acl)#acl add group 1 global
KGXX(config-acl)#acl mac add entry 1 group 1 da 0000.0000.0000 mask 0000.0000.0000 sa 0100.0000.0000 mask 0100.0000.0000 vlan 0 mask 0000 discard 1
KGXX(config-acl)#acl install entry 1 
KGXX(config-acl)#acl install group 1
```



### 端口mac绑定

```
KGXX(config-acl)#acl port 1/2 enable 
KGXX(config-acl)#acl port 1/2 forcemac enable 
KGXX(config-acl)#acl add group 10 port 2
KGXX(config-acl)#acl mac add entry 10 group 10 da 0000.0000.0000 mask 0000.0000.0000 sa 0000.0012.3010 mask ffff.ffff.ffff vlan 2 mask ffff discard 0
KGXX(config-acl)#acl add group 1 global
KGXX(config-acl)#acl mac add entry 1 group 1 da 0000.0000.0000 mask 0000.0000.0000 sa 0000.0000.0000 mask 0000.0000.0000 vlan 0 mask 0000 discard 1
KGXX(config-acl)#acl install entry 10
KGXX(config-acl)#acl install group 10
KGXX(config-acl)#acl install entry 1
KGXX(config-acl)#acl install group 1
```



# 测试



```
KGXX(config-acl)#acl port 1/2 enable 
KGXX(config-acl)#acl port 1/2 forcemac enable 
KGXX(config-acl)#acl add group 10 port 2
KGXX(config-acl)#acl mac add entry 1 group 10 da 0000.0000.0000 mask 0000.0000.0000 sa 0000.0012.3010 mask ffff.ffff.ffff vlan 2 mask ffff discard 0
KGXX(config-acl)#acl add group 1 global
KGXX(config-acl)#acl mac add entry 2 group 1 da 0000.0000.0000 mask 0000.0000.0000 sa 0100.0000.0000 mask 0100.0000.0000 vlan 0 mask 0000 discard 1
KGXX(config-acl)#acl mac add entry 3 group 10 da 0000.0000.0000 mask 0000.0000.0000 sa 0000.0000.0000 mask 0000.0000.0000 vlan 0 mask ffff discard 0
KGXX(config-acl)#acl install entry 1
KGXX(config-acl)#acl install entry 2
KGXX(config-acl)#acl install entry 3
KGXX(config-acl)#acl install group 10
KGXX(config-acl)#acl install group 1
```

