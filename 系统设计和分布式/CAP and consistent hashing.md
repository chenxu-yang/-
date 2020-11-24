# CAP

分布式系统的最大难点，就是各个节点的状态如何同步。CAP 定理是这方面的基本定理，也是理解分布式系统的起点。

## 一、分布式系统的三个指标

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018071607.jpg)

这三个指标不可能同时做到

## 二、Partition tolerance

先看 Partition tolerance，中文叫做"分区容错"。

大多数分布式系统都分布在多个子网络。每个子网络就叫做一个区（partition）。分区容错的意思是，区间通信可能失败。比如，一台服务器放在中国，另一台服务器放在美国，这就是两个区，它们之间可能无法通信。

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018071601.png)

一般来说，分区容错无法避免，因此可以认为 CAP 的 P 总是成立。CAP 定理告诉我们，剩下的 C 和 A 无法同时做到。

## 三、Consistency

Consistency 中文叫做"一致性"。意思是，写操作之后的读操作，必须返回该值。举例来说，某条记录是 v0，用户向 G1 发起一个写操作，将其改为 v1。

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018071602.png)

接下来，用户的读操作就会得到 v1。这就叫一致性。

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018071603.png)

问题是，用户有可能向 G2 发起读操作，由于 G2 的值没有发生变化，因此返回的是 v0。G1 和 G2 读操作的结果不一致，这就不满足一致性了。

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018071604.png)

为了让 G2 也能变为 v1，就要在 G1 写操作的时候，让 G1 向 G2 发送一条消息，要求 G2 也改成 v1。

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018071605.png)

这样的话，用户向 G2 发起读操作，也能得到 v1。

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018071606.png)

## 四、Availability

Availability 中文叫做"可用性"，意思是只要收到用户的请求，服务器就必须给出回应。

用户可以选择向 G1 或 G2 发起读操作。不管是哪台服务器，只要收到请求，就必须告诉用户，到底是 v0 还是 v1，否则就不满足可用性。

## 五、Consistency 和 Availability 的矛盾

一致性和可用性，为什么不可能同时成立？答案很简单，因为可能通信失败（即出现分区容错）。

如果保证 G2 的一致性，那么 G1 必须在写操作时，锁定 G2 的读操作和写操作。只有数据同步后，才能重新开放读写。锁定期间，G2 不能读写，没有可用性不。

如果保证 G2 的可用性，那么势必不能锁定 G2，所以一致性不成立。

综上所述，G2 无法同时做到一致性和可用性。系统设计时只能选择一个目标。如果追求一致性，那么无法保证所有节点的可用性；如果追求所有节点的可用性，那就没法做到一致性。



在什么场合，可用性高于一致性？

举例来说，发布一张网页到 CDN，多个服务器有这张网页的副本。后来发现一个错误，需要更新网页，这时只能每个服务器都更新一遍。

一般来说，网页的更新不是特别强调一致性。短时期内，一些用户拿到老版本，另一些用户拿到新版本，问题不会特别大。当然，所有人最终都会看到新版本。所以，这个场合就是可用性高于一致性。



# 数据库主从复制

在复制过程中，一台服务器充当主服务器（Master），接收来自用户的内容更新，而一个或多个其他的服务器充当从服务器（Slave），接收来自主服务器binlog文件的日志内容，解析出SQL重新更新到从服务器，使得主从服务器数据达到一致。



## 应用场景

**应用场景1：从服务器作为主服务器的实时数据备份**

主从服务器架构的设置，可以大大加强MySQL数据库架构的健壮性。例如：当主服务器出现问题时，我们可以人工或设置自动切换到从服务器继续提供服务，此时从服务器的数据和宕机时的主数据库几乎是一致的

**应用场景2：主从服务器实时读写分离，从服务器实现负载均衡**

主从服务器架构可通过程序（PHP、Java等）或代理软件（mysql-proxy、Amoeba）实现对用户（客户端）的请求读写分离，即让从服务器仅仅处理用户的select查询请求，降低用户查询响应时间及读写同时在主服务器上带来的访问压力。对于更新的数据（例如update、insert、delete语句）仍然交给主服务器处理，确保主服务器和从服务器保持实时同步。

**应用场景3：把多个从服务器根据业务重要性进行拆分访问**

可以把几个不同的从服务器，根据公司的业务进行拆分。例如：有为外部用户提供查询服务的从服务器，有内部DBA用来数据备份的从服务器，还有为公司内部人员提供访问的后台、脚本、日志分析及供开发人员查询使用的从服务器。这样的拆分除了减轻主服务器的压力外，还可以使数据库对外部用户浏览、内部用户业务处理及DBA人员的备份等互不影响。

## 实现MySQL主从读写分离的方案

PHP和Java程序都可以通过设置多个连接文件轻松地实现对数据库的读写分离，即当语句关键字为select时，就去连接读库的连接文件，若为update、insert、delete时，则连接写库的连接文件。

通过程序实现读写分离的缺点就是需要开发人员对程序进行改造，使其对下层透明，但这种方式更容易开发和实现，适合互联网业务场景。



## MySQL主从复制原理介绍

MySQL的主从复制是一个异步的复制过程（虽然一般情况下感觉是实时的），数据将将从一个MySQL数据库（我们称之为Master）复制到另一个MySQL数据库（我们称之为Slave），在Master于Slave之间实现整个主从复制的过程是由三个线程参与完成的。其中有两个线程（SQL和IO线程）在Slave端，另外一个线程（I/O线程）在Master端。

要实现MySQL的主从复制，首先必须打开Master端的Binlog记录功能，否则就无法实现。因为整个复制过程实际上就是Slave从Master端获取BInlog日志，然后再在Slave上以相同顺序执行获取的binlog日志中记录的各种SQL操作。

MySQL的主从复制是一个异步的复制过程（虽然一般情况下感觉是实时的），数据将将从一个MySQL数据库（我们称之为Master）复制到另一个MySQL数据库（我们称之为Slave），在Master于Slave之间实现整个主从复制的过程是由三个线程参与完成的。其中有两个线程（SQL和IO线程）在Slave端，另外一个线程（I/O线程）在Master端。

要实现MySQL的主从复制，首先必须打开Master端的Binlog记录功能，否则就无法实现。因为整个复制过程实际上就是Slave从Master端获取BInlog日志，然后再在Slave上以相同顺序执行获取的binlog日志中记录的各种SQL操作。


1）在Slave服务器上执行start slave命令开启主从复制开关，主从复制开始进行。

2）此时，Slave服务器的I/O线程会通过在Master上己经授权的复制用户权限请求连接Master服务器，并请求从指定Binlog日志文件的指定位罝（日志文件名和位置就是在配罝主从复制服务时执行change master命令指定的）之后开始发送Binlog日志内容。

3）Master服务器接收到来自Slave服务器的I/O线程的请求后，其上负责复制的I/O线程会根据Slave服务器的I/O线程请求的信息分批读取指定Binlog日志文件指定位置之后的Binlog日志信息，然后返回给Slave端的I/O线程。返回的信息中除了Binlog日志内容外，还有在Master服务器端记录的新的Binlog文件名称以及在新的Binlog中的下一个 指定更新位置。

4）当Slave服务器的I/O线程获取到Master服务器上I/O线程发送的日志内容及日志文件及位置点后，会将Binlog日志内容依次写入到Slave端自身的Relay Log（即中继日志） 文件(MySQL-relay-bin.xxxxxx)的最末端，并将新的Binlog文件名和位置记录到master-info文件中，以便下一次读取Master端新Binlog日志时能够告诉Master服务器需要从新Binlog 日志的指定文件及位置开始请求新的Binlog日志内容。

5）Slave服务器端的SQL线程会实时地检测本地Relay Log中I/O线程新增加的日志内容，然后及时地把Relay Log文件中的内容解析成SQL语句，并在自身Slave服务器上按解析SQL语句的位置顺序执行应用这些SQL语句，并记录当前应用中继日志的文件名及位置点在relay-log.info中。

## 总结

下面针对MySQL主从复制原理的重点小结

- 主从复制是异步的逻辑的SQL语句级的复制
- 复制时，主库有一个I/O线程，从库有两个线程，I/O和SQL线程。
- 作为复制的所有MySQL节点的server-id都不能相同。
- binlog文件只记录对数据库有更改的SQL语句（来自数据库内容的变更），不记录任何查询（select，slow）语句。

# consistenct hashing

## what is hashing

Hashing is the process of mapping one piece of data — typically an arbitrary size object to another piece of data of fixed size, typically an integer, known as *hash code* or simply *hash*.

## what is distributed hashing

Data is too big to store all information in a hash table。

可以将传入的 Key 按照 `index = hash(key) % N` 这样来计算出需要存放的节点。其中 hash 函数是一个将字符串转换为正整数的哈希映射方法，N 就是节点的数量。

这样可以满足数据的均匀分配，但是这个算法的容错性和扩展性都较差。

比如增加或删除了一个节点时，所有的 Key 都需要重新计算，显然这样成本较高，为此需要一个算法满足分布均匀同时也要有良好的容错性和拓展性。

## 一致性hash算法

一致 Hash 算法是将所有的哈希值构成了一个环，其范围在 `0 ~ 2^32-1`。如下图：

[![img](https://i.loli.net/2019/05/08/5cd1b9fe31560.jpg)](https://i.loli.net/2019/05/08/5cd1b9fe31560.jpg)

之后将各个节点散列到这个环上，可以用节点的 IP、hostname 这样的唯一性字段作为 Key 进行 `hash(key)`，散列之后如下：

[![img](https://i.loli.net/2019/05/08/5cd1ba01ebc22.jpg)](https://i.loli.net/2019/05/08/5cd1ba01ebc22.jpg)

之后需要将数据定位到对应的节点上，使用同样的 `hash 函数` 将 Key 也映射到这个环上。

[![img](https://i.loli.net/2019/05/08/5cd1ba05955b9.jpg)](https://i.loli.net/2019/05/08/5cd1ba05955b9.jpg)

这样按照顺时针方向就可以把 k1 定位到 `N1节点`，k2 定位到 `N3节点`，k3 定位到 `N2节点`。

### 容错性

这时假设 N1 宕机了：

[![img](https://i.loli.net/2019/05/08/5cd1ba07908f0.jpg)](https://i.loli.net/2019/05/08/5cd1ba07908f0.jpg)

依然根据顺时针方向，k2 和 k3 保持不变，只有 k1 被重新映射到了 N3。这样就很好的保证了容错性，当一个节点宕机时只会影响到少少部分的数据。

### 拓展性

当新增一个节点时:

[![img](https://i.loli.net/2019/05/08/5cd1ba0c7519c.jpg)](https://i.loli.net/2019/05/08/5cd1ba0c7519c.jpg)

在 N2 和 N3 之间新增了一个节点 N4 ，这时会发现受印象的数据只有 k3，其余数据也是保持不变，所以这样也很好的保证了拓展性。

## 虚拟节点

到目前为止该算法依然也有点问题:

当节点较少时会出现数据分布不均匀的情况：

[![img](https://i.loli.net/2019/05/08/5cd1ba12a3b62.jpg)](https://i.loli.net/2019/05/08/5cd1ba12a3b62.jpg)

这样会导致大部分数据都在 N1 节点，只有少量的数据在 N2 节点。

为了解决这个问题，一致哈希算法引入了虚拟节点。将每一个节点都进行多次 hash，生成多个节点放置在环上称为虚拟节点:

[![img](https://i.loli.net/2019/05/08/5cd1ba1490a67.jpg)](https://i.loli.net/2019/05/08/5cd1ba1490a67.jpg)

计算时可以在 IP 后加上编号来生成哈希值。

这样只需要在原有的基础上多一步由虚拟节点映射到实际节点的步骤即可让少量节点也能满足均匀性。