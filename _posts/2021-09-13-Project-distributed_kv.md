---
layout:     post
title:      分布式K-V缓存系统设计
subtitle:   实习合作项目：分布式K-V缓存系统项目详解
date:       2021-9-13
author:     Zhgaot
header-img: img/Project/distributed_kv/cloud.png
catalog: true
tags:
    - Project
    - Tencent-Internship
---

> **[项目说明]:** 这是暑期我进入腾讯实习时，TEG布置的虚拟项目，此项目由6个实习生共同完成
>
> ------
>
> **[个人分工]:** 除系统设计外，我主要负责Client端的设计与实现，以及LRU类模板的编写
>
> ------
>
> **[文章致谢]:** (●'◡'●)我非常怀念我们共同写项目的时光！十分感谢大家的帮助，和你们合作超级愉快
>
> ------
>
> **[项目源码]:** 由于公司的保密要求，以及我们使用的是内部开源的网络框架，因此无法将源码拷出，所以这篇文章将详细说明我们的设计与实现过程
>
> ------
>
> **[注意事项]:** **如果未使用VPN**，本篇的图片可能无法查看，可直接进入我的笔记[👉notion-分布式K-V缓存系统设计](https://amazing-brow-87c.notion.site/K-V-91244b838ebe4efd82aea134cf677dee)查看

## 1 分布式缓存系统设计概述

### 1.1 设计目的

受磁盘和SQL解析的影响，数据库的检索速度是非常慢的，随着服务器规模的扩大，大量客户端向数据库请求数据，数据库的压力将变得非常大，进而可能无法正常响应。解决方法之一是：**引入一个中间层——缓存**。如果引入的是**单点缓存**，则可以对复杂且业务逻辑不会发生变化的查询进行缓存，当请求再次发起时，就可以先从单点缓存中查询结果，从而大大减少对数据库的查询，进而减小服务器的压力。但是单点缓存存在稳定性较差和可扩展性较差的问题，当流量多大时依然难以应对，因此需要设计**分布式缓存系统**，它具有更好的稳定性和可扩展性。下图对比了上述三种情况下系统的稳定性和扩展性：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_0.png)

### 1.2 系统架构

接下来介绍一下系统的总体架构，如下图所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_1.png)

系统由一个Master-Server、多组Cache-Server、若干个Client组成。

Master-Server负责管理Key-Address(ip&port)形式的数据分布(路由)信息，这里的Address(ip&port)指的是某个Cache-Server的地址，同时对每个Cache-server的生存与死亡、扩容与缩容进行监控。

每个Cache-server缓存Key-Value形式的数据，并以LRU作为淘汰算法；此外，Cache-Server以定时心跳的方式定期向Master-Server汇报存活情况，如果某一个Cache-Server故障，则心跳失效，Master-Server会重新划分数据分布，并以广播的方式通知仍存活的Cache-Server最新的数据分布(路由)信息，Cache-Server的扩容以及缩容逻辑与上述方法类似。

Client将随机生成Key-Value以及GET/SET请求，Client在进行GET/SET操作前，先通过Key查看本地存储的数据分布(路由)信息，如果有效，则Client根据在本地查询到的Address(ip&port)访问Key对应的Cache-Server，如果无效，则Client需要重新从Master-Server拉去最新的数据分布(路由)信息。

## 2 分布式缓存系统设计方案

### 2.1 数据分布信息与负载均衡

#### 2.1.1 问题与解决思路

如上所述，单台Cache-Server的并发量和存储量有限，那么如何提高系统的并发量和存储量呢？答案是：使用多台Cache-Server提高存储量，并在Master-Server上进行一种负载均衡策略将流量平均地分配到不同的Cache-Server上。

同等的，如何存储众多的数据分布信息，即Key及其所对应的Cache-Server的Address(ip&port)呢？答案是：在Master-Server上管理Key-Address(ip&port)的对应关系，且不通过Client的Address(ip&port)进行负载均衡的运算，而是对Key进行负载均衡的运算。

#### 2.1.2 负载均衡方案对比

设计系统时我们共考虑了四种策略，下面分别对四种策略进行简介并阐述其优缺点：

1. **Key-Address哈希表**：直接建立一个哈希表维护Key与Cache-Server地址的对应关系，此方法实现简单，但随着Key的数量越多，哈希表占用空间可能会越来越大；
2. **哈希求余**：对一个Key算取它的哈希值，之后，假如有`n`个Cache-Server就对此哈希值取模`n`，这样即可得到Key和Cache-Server的对应关系；此方法实现简单且速度快，但单调性不好，比如如果Cache-Server扩容，那么整个的取模结果全部会变化，这将导致大量的Key会被重新分配；
3. **一致性哈希算法**([原理](https://www.zsythink.net/archives/1182))：一致性哈希算法具有良好的可扩展性及单调性，但存在数据倾斜问题，虽然可以通过添加虚拟节点来解决，但无法从源头解决问题；
4. **Hashslot(哈希槽)**：首先设置槽的个数(这里参考Redis取16384)，然后划分槽的分布，即划分每一个槽所属的Cache-Server；当有Key值到来时，先对Key求CRC16校验码，然后将其结果对16384取模，这样Key就会落在一个槽上，进而可知Key在哪个Cache-Server上了。哈希槽的可扩展性、单调性都很好，且没有数据倾斜情况，因此本系统决定采用哈希槽作为

#### 2.1.3 哈希槽分配方案

哈希槽的分配方案如下图所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_2.png)

当Master-Server初始化时设置一个空的哈希槽；当Cache-Server_1上线并发送心跳包给Master-Server时，哈希槽将全部的槽分配给Cache-Server_1；当Cache-Server_2上线并发送心跳包给Master-Server时，哈希槽将Cache-Server_1一般的槽分配给Cache-Server_2；当Cache-Server_3上线并发送心跳包给Master-Server时，哈希槽分别将Cache-Server_1和Cache-Server_2各自1/3的槽分配给Cache-Server_3，以此类推...

### 2.2 Client设计

#### 2.2.1 本地缓存方案

##### 2.2.1.1 本地缓存设计目的

为何在Client端设置本地缓存？目的是给Master-Server减压，即不希望所有的Client在每一次请求查询数据时都先去访问Master-Server获取Key对应的Address后再取访问对应的Cache-Server。

##### 2.2.1.2 缓存实现方案

既然要在Client端设置本地缓存，就要明确缓存维护的数据是什么，以及缓存采取何种数据结构？在本项目中我们考虑了三种缓存方案，并对后两种进行了实现，最终采取了第三种方案：

(1) **Key-Value缓存**

之前在学习计算机网络的应用层时，学习到一种提升访问效率、减少网络流量的方法——web缓存与代理服务器，即将最近访问过的请求和响应暂存在客户机或中间系统的本地磁盘中，当新请求到达时，若发现这个新请求与暂存的请求相同，就返回暂存的响应，而不需要按照URL的地址再去原始服务器访问该资源；其中位于中间系统的web缓存称为代理服务器；因此我们考虑仿照此方法实现Client端的本地缓存。

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_3.png)

如上图所示，将最近访问过的Key-Value以键值对的格式暂存在Client的内存中，并为每一条记录设置一个Last-Modified字段，表示该条记录最近一次的更新时间，当Client产生新的GET(查询)请求时，首先查看本地缓存中是否有此GET请求的Key对应的Value，如果存在则通过检查Last-Modified字段来判断Value是否过期，未过期则可直接得到查询结果(Value)，过期则Client需要重新访问Master-Server获取Key对应的Address后再取访问对应的Cache-Server。

该方法存在诸多弊端：比如，不支持SET请求；比如，当存在海量Client对系统进行GET和SET时，数据的更新程度是很高的，那为保证Client不读到过期数据，则过期时间应设置的很短，那么这将导致Key-Value缓存失效。因此，本项目并没有实现此方案。

(2) **Key-Address缓存**

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_4.png)

如上图所示，既然Key-Value缓存可能会失效，那倒不如直接缓存Key-Address；无论Client产生GET还是SET请求，均可先访问本地缓存，查找Key对应的Cache-Server的Address，然后无需访问Master-Server即可直接访问Key对应的Cache-Server；只有当Client在本地缓存中无法找到结果时才需要取Master-Server查询Key对应的Cache-Server的Address。

在实现上：我采用了**FIFO+LRU([LRU-K](https://segmentfault.com/a/1190000022558044))**来实现Key-Address键值对的淘汰，这样一方面可以防止内存爆掉，另一方面可以解决LRU的“缓存污染”问题。

Key-Address缓存相比于Key-Value缓存，虽然可能会增加Client与Cache-Server之间的网络流量，但将大大减少Client访问Master-Server的次数；但Key-Address缓存依然存在缺陷：在任一Client初期，Client必将多次访问Master-Server以建立本地缓存，若很多Client确实仅仅只查询少量几次数据，则Key-Address缓存很可能根本无法被用到，即在此种情况下Key-Address缓存可能失效。

(3) **Hashslot(哈希槽)缓存**

为解决上述Key-Address缓存的问题，同时考虑Master-Server内已经维护的**Hashslot(哈希槽)**，我们想到**Client端可以直接获取Master-Server维护的哈希槽**。如下图所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_5.png)

Client端在上线时必须访问Master-Server获取哈希槽(即数据分布信息)作为本地缓存，而后每一次GET或SET请求均可首先使用本地缓存的哈希槽计算出Key对应的Cache-Server的Address，再根据Address对Cache-Server进行访问，若访问成功则无需进行其他操作(证明Client端当前缓存的哈希槽未过期)，若访问失败则Client需要重新访问Master-Server获取最新的哈希槽。

Hashslot(哈希槽)缓存的优势在于：Client端仅需要在上线时访问一次Master-Server获取哈希槽即可，之后对哈希槽的更新仅会在Cache-Server群组扩容、缩容、宕机这三种情况下发生，这将在任何时刻都能够降低Client端对Master-Server的访问次数。**因此本系统最终采用Hashslot(哈希槽)缓存作为Client端的本地缓存。**

#### 2.2.2 异常情况处理

当Cache-Server群组出现扩容、缩容、宕机时，Client端如何发现并应对呢？

##### 2.2.2.1 Cache-Server群组缩容或宕机

从Client的视角来看，Cache-Server群组的缩容和宕机可视为同一种情况——某台Cache-Server下线，所以在此将缩容和宕机一并讨论，**Client的应对策略是：先“主动发现”，后“更新哈希槽”**。后者很好理解，当Client发现Cache-Server群组出现缩容或宕机时，只能重新连接Master-Server获取最新的哈希槽(数据分布信息)，在此不予赘述。而对于前者，在本系统中Client通过两种方法来进行“主动发现”：

(1) **Client端无法建立与某台Cache-Server的网络连接即视为该台Cache-Server出现了缩容或宕机情况；**这很好理解，例如使用socket基础API——connect时，如果返回-1则表明连接失败，当再次重连后依然失败即认定该台Cache-Server出现了缩容或宕机情况；在我们编写项目时使用的是组内的网络框架，它的connect是非阻塞的，但是send是阻塞的，connect将立即返回，当send失败时才认定该台Cache-Server出现了缩容或宕机情况。

(2) **Client端在本地维护多个针对于Cache-Server的Package-Array用于进行收发包管理**，即每当Client向Cache-Server发送一次请求包时，就复制一份请求包存在本地的某个对应此Cache-Server的Package-Array中，当Client收到此Cache-Server的回复包时，就像连连看一样消除掉Package-Array中对应的请求包，意为“收到回包”；因此，如果某个Package-Array积蓄了一定数量(当前认为2)的请求包，则说明该Package-Array对应的Cache-Server在没来得及回包时出现了缩容或宕机，那么就需要重新连接Master-Server获取最新的哈希槽(数据分布信息)，然后重新发送之前未收到回复包的请求。Client端的Package-Array如下图所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_6.png)

##### 2.2.2.2 Cache-Server群组扩容

Cache-Server群组扩容分为两阶段：“扩容进行中”和“扩容完成后”。

“扩容进行中”指的是：当Cache-Server群组需要扩容时，Master-Server一方面保存旧的哈希槽，另一方面会重新分配一个新的哈希槽，再将最新的哈希槽广播至每组Cache-Server，Cache-Server依照最新的哈希槽进行数据迁移，当所有Cache-Server完成数据迁移后，Master-Server再将本地的旧哈希槽删除，并更新为最新哈希槽。在此期间，如果Client产生了新的请求，它会依照旧的哈希槽对Cache-Server进行访问，这将导致很多问题，对这些问题的探讨可参见“2.3.2 数据迁移时对Client请求的应对”一节，在此不予赘述，因为这些问题的处理是由Cache-Server来完成的。

“扩容完成后”指的是：在Cache-Server数据刚刚迁移完成后，Client端会依照旧的哈希槽找到某个Cache-Server，如果恰好Client请求的Key已经不在该台Cache-Server上(即此数据原本在但被迁移走)，则**Cache-Server将返回一个`OTHER`字段表明Client访问到的Cache-Server已经不再为此Key-Value服务了**，因此Client就会重新连接Master-Server获取最新的哈希槽(数据分布信息)，再根据最新的哈希槽进行GET/SET操作。

#### 2.2.3 应用层的超时重传

虽然TCP有超时重传机制，但发送方应用层仍需要实现超时重传，原因有如下两点：

- **基于内存容量方面的考虑，服务进程可能使用了限长的消息队列**，如果收到的瞬时消息过多，超过了消息队列的可处理个数，所有超出的消息会被它丢弃。注意，在这种情况下，TCP的超时重传确实保证了消息的成功送达，只是接收方应用层不接受而已。
- 应考虑极端情况：接收方确实收到了发送方发送的消息，但还没来的及处理接收方就挂掉了，这时发送方的运输层认为接收方接收到了消息(也确实接收到了)，只是接收方没有正常处理这个消息，只要接收方立刻重启或有其它容灾策略，发送方理应进行超时重传。

在实现上，**使用了信号来实现应用层的超时重传**：首先，定义了信号处理函数(以下称为`sig_handler`)负责重连，定义了`struct sigaction`来注册`__sighandler_t`类型转换后的`sig_handler`，使用`sigaction`系统调用来设置`sig_handler`；其次，定义`struct itimerval`变量来设定重传时间；最后在每次发送消息之后调用`setitimer`函数来发送一个仅一次的定时信号`SIGALRM`用来触发`sig_handler`。

插一句题外话，除了定时信号`SIGALRM`外，Client进程还需要接收进程退出信号(Ctrl+\)`SIGQUIT`来切换至用户的手动输入模式，因此在使用`sigaction`系统调用来设置`SIGQUIT`信号的信号处理函数`sig_handler`时，需要使用sigaddset函数来设定`struct sigaction`变量的`sa_mask`**进程掩码**，以防止定时信号`SIGALRM`误调用超时重传的`sig_handler`。

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_7.png)

由于时间关系，在本项目中，Client仅实现了针对Master-Server的超时重传(如上图所示)，暂未实现针对Cache-Server的超时重传，还有一个原因是在写项目时不知如何脱离当前框架来设定多个处理`SIGALRM`定时信号的*信号处理函数*，不过现在知道了是**可以使用[基于升序链表的定时器容器](https://www.cnblogs.com/cobbliu/p/3627448.html)、时间轮、时间堆来处理多个定时事件**，比如libevent就使用时间堆作为定时器容器来管理多个定时事件。

### 2.3 扩容与缩容

这里的扩容与缩容指的是Cache-Server群组的扩容与缩容。项目背景是：根据对业务量的判断，需要人工对Cache-Server群组进行扩容或缩容操作。那么我们需要解决的问题是：

- 当Cache-Server群组进行扩容或缩容操作时**如何进行数据迁移**；
- 当Cache-Server群组**扩容或缩容正在进行中时如何应对Client的访问请求**；

#### 2.3.1 数据迁移

在本小节，以扩容为例说明数据迁移的过程（缩容同理）。从本小节开始，称“旧哈希槽”为`Old-HashSlot`，称“新哈希槽”为`New-HashSlot`。

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_8.png)

在迁移过程方面：如上图所示，当Cache-Server群组需要扩容时，Master-Server将会收到新的Cache-Server发来的心跳包，表明有新的Cache-Server到来；Master-Server一方面保存`Old-HashSlot`，另一方面会重新分配一个`New-HashSlot`，再将`New-HashSlot`广播至每组Cache-Server；Cache-Server依照Master-Server提供的`New-HashSlot`进行数据迁移，当所有Cache-Server完成数据迁移后，Master-Server才将本地的`Old-HashSlot`删除，并以`New-HashSlot`作为替代。

在代码实现方面：当Primary-Cache-Server(将会在“2.4 宕机与容灾”中讲到)收到Master-Server的`New-HashSlot`时，将**另开一个新线程(使用线程的原因见2.3.2一节)，遍历内存中LRU的哈希表，根据`New-HashSlot`挨个比对每个Key是否依然属于本Primary-Cache-Server，如果某个Key不属于，则会将其假设为一条Client发送的SET请求发送至`New-HashSlot`中此Key对应的那个Primary-Cache-Server，并从本Primary-Cache-Server的LRU中删除此条数据**。

对于缩容：与扩容相似，只不过在缩容前Master-Server将收到某个Primary-Cache-Server要缩容的请求，于是Master-Server将会重新分配哈希槽，系统进而开始进行一系列的缩容操作(广播、数据迁移等...)。

#### 2.3.2 数据迁移时对Client请求的应对

当数据迁移正在进行中时，Client无论是通过本地缓存的哈希槽，还是通过Master-Server再获取的哈希槽，都是`Old-HashSlot`，因为本地缓存的是`Old-HashSlot`，Master-Server对外提供的也是`Old-HashSlot`；若此时有Client发送请求给某个Cache-Server，且Client请求的Key是此Cache-Server需要迁移的数据(即此Key已不属于此Cache-Server且正在进行数据迁移)，那么Cache-Server如何保证Client读到的(GET)数据是正确的，或写入的(SET)数据如何保证一致性呢？为方便解释，假如一条数据原本在Cache-Server_1中，数据迁移后该数据移动至Cache-Server_2中：

(1) 读取(GET)请求

- Client请求读取**未迁移**走的数据：Cache-Server_1直接查询内存，返回给Client；
- Client请求读取**已迁移**走的数据：Cache-Server_1将代替Client向此数据新的所属Cache-Server_2要取数据并返回给Client；

(2) 写入(SET)请求

- Client请求写入**未迁移**走的数据：Cache-Server_1将代替Client向此数据新的所属Cache-Server_2发送SET请求，这样Cache-Server_2中关于此数据的Value和顺序均被更新，得到Cache-Server_2的确认后返回结果给Client，最后**Cache-Server_1从本地的LRU中删除此数据**；
- Client请求写入**已迁移**走的数据：Cache-Server_1将代替Client向此数据新的所属Cache-Server_2发送SET请求，得到Cache-Server_2的确认后返回结果给Client；

这里需要呼应2.3.1一节来解释一点，即为什么使用开辟新线程来进行数据迁移：是因为这种方式方便访问共同的内存中的LRU(具体来说是LRU中的哈希表)，这是为了方便主线程在接到Client端的请求时判断Client请求操作的某个Key是否已经迁移完成，如果能在哈希表中找到说明未迁移，如果无法在哈希表中找到说明已迁移。

#### 2.3.3 顺带一提

这里顺带叙述一下，在实现上，我们如何模拟Cache-Server的人工扩容与缩容操作：

- 扩容：**直接上线**一个或多个Cache-Server进程，它们会自动向Master-server发送心跳包，从这一时刻起，扩容开始；
- 缩容：通过**对一个键盘输入信号的监听**来模拟人工进行的缩容操作，这样还可以区分服务器的宕机(直接kill掉进程)与缩容(键盘信号)；

### 2.4 宕机与容灾

Master-Server如何监控Cache-Server的存活？当某个Cache-Server突然宕机后，此Cache-Server内的数据应该怎么办？如果此时Client根据本地缓存的哈希槽访问到宕机的Cache-Server怎么办？对于第三个问题，“2.2.2.1 Cache-Server群组缩容或宕机”一节中已经进行了探讨，下面将对前两个问题进行论述。

#### 2.4.1 Cache-Server存活监控

##### 2.4.1.1 存活监控设计

其实之前章节已说明了Cache-Server存活监控的方法：每个Cache-Server(无论主/备)每隔一段时间(例如500ms)向Master-Server发送**心跳包**，表明自己的存活状态，而Master-Server每隔一段时间(例如2000ms)检测是否有Cache-Server超过2000ms依然没有发送心跳包，如果有，则认为此Cache-Server宕机，进而通知此Cache-Server的备份机(Replica-Cache-Server)转正。

##### 2.4.1.1 存活监控实现

对于Cache-Server，只需要从上线开始不断地向Master-Server发送心跳包即可。

对于Master-Server，在内存中维护一个哈希表，哈希表的长度与所有Cache-Server的数量相同，哈希表的Key为Cache-Server的Address，Value为最近一次收到此Cache-Server心跳包的时间戳；以后，每当Master-Server接收到某个Cache-Server的心跳包后，为其打一个时间戳(这可以通过`gettimeofday()`来实现)并修改本地哈希表；Master-Server再通过框架提供的定时器(或时钟信号等)，每隔一段时间遍历哈希表，检测每隔Cache-Server的时间戳与当前时间是否超过了时限，则认为此Cache-Server宕机，从哈希表中将其删除后进行后续操作即可。

#### 2.4.2 Cache-Server容灾设计

##### 2.4.2.1 Cache-Server容灾策略

Cache-Server的容灾策略是：**冗余备份**。即每一个Cache-Server都有一个备份的Cache-Server，它们一个被称为**Primary-Cache-Server(主节点)**，一个被称为**Replica-Cache-Server(从节点)**，可以将它们统一看作一个Cache-Server组。

当Cache-Server宕机时，要分两种情况讨论：即“Primary-Cache-Server(主节点)宕机”与“Replica-Cache-Server(从节点)”宕机。

Primary-Cache-Server宕机时：Primary-Cache-Server将不再给Master-Server发送心跳包，Master-Server将在一段时间后发现此情况，并给Replica-Cache-Server发送转正通知，**Replica-Cache-Server转正为Cache-Server组的新的Primary-Cache-Server**；之后Master-Server写入日志并通知管理员。

Replica-Cache-Server宕机时：Replica-Cache-Server将不再给Master-Server发送心跳包，Master-Server将在一段时间后发现此情况，写入日志并通知管理员。

##### 2.4.2.2 Client主从访问

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/Project/distributed_kv/kv_9.png)

如上图所示，在这种结构下，Primary-Cache-Server负责接收Client的GET/SET请求，而Replica-Cache-Server负责接收Primary-Cache-Server复制并转发的Client的GET/SET请求，为保证强一致性(这里参考的是Google分布式文件系统——GFS)，只有当Primary-Cache-Server收到了Replica-Cache-Server的返回消息后，才对Client进行回复。

##### 2.4.2.3 Cache-Server数据恢复

数据恢复指的是：某个Cache-Server宕机后立刻重启，那么无论此Cache-Server之前是Primary还是Replica，当前它都将作为Replica-Cache-Server工作，那么当前的Primary-Cache-Server将对其进行数据恢复的操作。

在本系统中，使用**全量复制**来实现数据恢复，即当Replica-Cache-Server重新上线后，Primary-Cache-Server需要`fork()`一个**子进程**（基于父子进程的**写时复制机制**，子进程暂时不会复制一份父进程的物理空间），子进程将依次倒序遍历内存中的LRU链表，并将每条数据通过SET请求发送至Replica-Cache-Server，完成全量备份。

若在进行全量复制时，Client访问了Primary-Cache-Server，Primary-Cache-Server会在本地直接进行GET/SET请求并返回结果给Client而不会复制并转发Client的请求给Replica-Cache-Server；重要的是：Primary-Cache-Server会在内存中开辟一段**缓冲区**（基于写时复制，此时子进程才需要复制一份父进程的物理空间），记录下Client的GET/SET请求，等到全量复制完成后，再依次将缓冲区内的请求发送至Replica-Cache-Server以进行数据同步。

恢复数据时，fork子进程而非开辟新线程的原因是：假如Client真的在数据恢复时访问Primary-Cache-Server，则此时Primary-Cache-Server的父进程与子进程之间才会分离物理空间，二者各自的工作将互不影响，而如果Client并未在数据恢复时访问Primary-Cache-Server，父子进程依然共享物理空间，也不会造成浪费。

## 3 不足与优化

### 3.1 Cache-Server宕机导致的短时服务不可用

#### 3.1.1 问题描述

在本系统中，考虑了Cache-Server扩容、缩容、备份时如果Client发送请求该如何应对，但唯独没有考虑Cache-Server宕机后、Master-Server发现其宕机前这段时间服务不可用的情况。即某个Cache-Server宕机后，Master-Server无法立刻发现其宕机，此时若Client访问宕机的Cache-Server，将无法连接，那么Client会去Master-Server拉取最新哈希槽，但此时Master-Server并不知道Cache-Server宕机，因此其内部的哈希槽依然是旧的。

#### 3.1.2 问题解决

可以在哈希槽中同时记录Primary-Cache-Server(主节点)与Replica-Cache-Server(从节点)的Address，当Client通过哈希槽访问不到Primary-Cache-Server(主节点)时，将会依次访问其Replica-Cache-Server(从节点)。

### 3.2 Master-Server的容灾

本项目没有设计Master-Server的容灾，关于Master-Server的容灾可以从以下几个方面考虑：

- Master-Server备份
- 去中心化
