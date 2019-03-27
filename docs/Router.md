# Router服务
## 功能简介
- 提供路由管理服务，能让客户端Proxy快速定位要访问的数据位于哪个Cache服务器上。
- 控制Cache服务进行主备切换。
- 当集群性能不足或容量不够的情况下，提供数据迁移的调度功能。
## 原理
### 路由功能
由于数据分布在多个cache服务器上，客户端访问数据时，就需要通过路由来快速定位要访问的数据位于哪个cache服务器。具体来说就是客户端Proxy根据Router提供的路由信息，去定位业务的key落到哪个Cache服务上。这个路由信息就是一张路由表，下面是一个路由表的例子:

| 编号 | 业务模块名 | 开始页 | 结束页 | 路由目的服务器组名 |
| :---: | :----------: | :-----: | :------: | :----------------: |
| 4 | SQQProfile | 0 | 143165 | SQQProfileGroup0 |
| 16 | SQQProfile | 143166 | 286330 | SQQProfileGroup1 |
| 28 | SQQProfile | 286331 | 429497 | SQQProfileGroup2 |

在DCache中，不同的业务模块有自己独立的路由表，而每个模块又分成了多个cache组，每个cache组都对应了一条自己的路由信息，负责存储一段路由页。
路由页：业务数据的key有各种含义，比如QQ号码、电话号码或字符串MD5值等，但我们都可以将key通过一致性hash算法转换到0～4294967295 (unsigned int)范围内，以方便我们做路由。
由于分组数不会太多，所以实际的路由范围是以页为单位，每一页包含了连续的10000个hash值，所以路由范围就是0～429497（页）。[图片:路由查找流程]

### 数据迁移
数据的路由是基于路由表的。但由于机器性能不足，或者容量不够的情况下，数据需要从一台机器迁到另一台机器，路由表也需要做相应的变动。这个过程就叫做数据迁移。
但这个过程中业务还在读写，不能在迁移过程中屏蔽业务的所有读写操作。所以数据迁移的重要目标的就是业务无感知。

迁移采用的是先复制数据后改路由表的策略。每次迁移10页（可以配置），迁移过程中不允许写这10页的数据。虽然这10页的数据不可写，但由于我们有42W的页数，10页的的占比很小，而且proxy在迁移过程中出错会有重试的机制（重试一次），因此由迁移导致的写失败业务侧无感知。

1. 迁移流程先通知目的组，把要迁移的页范围锁住，然后做些数据清理的动作；
2. 然后通知迁移源组，把要迁移的页范围锁住不可写，然后从内存读取数据，把数据发送到目的组，完成之后返回给router；
3. router收到迁移完成之后，把最新路由信息推送给源组和目的组，然后修改DB和内存的路由表信息。
4. 整个迁移流程结束。

其中Proxy是通过定时获取路由的变更状态来获取最新的路由。当迁移结束到proxy获取到最新的路由有一个时间窗口，这个时候proxy还会访问到旧的cache。
这时由于cache拥有最新的路由信息，会判断请求是否属于本cache负责，如果发现请求不属于本cache，会回复路由错误的信息给proxy，proxy就随后立即更新路由信息再次访问。
通过这样的方式来保证路由变更不会影响业务数据的一致性。

### 自动切换
RouterServer中还支持了Cache服务的自动切换，包括主备机的自动切换，镜像的自动切换和备机不可读切换。自动切换的主要逻辑是在RouterServer内部实现。DCache的自动切换功能提高了集群容灾容错能力，减少了系统异常对业务的影响，也使得运维工作更加轻松高效。
自动切换保证的两个基本目标是：
A．防止误切（例如网络波动的影响），这都就要求严格的宕机检测。
B．切换过程中保证所有接入机的写操作不会同时写到两个master（切换前的master，和切换后的master），从而导致数据错乱或丢失。

主备机以及镜像会定时向RouterServer发送自身的心跳信息，心跳超时后即会启动自动切换检测流程，满足一定条件后便会执行自动切换。

具体来说，Router执行主备切换的逻辑是这样的：
1. 判断Cache主机心跳超时。
2. 判断备机心跳没有超时。
3. 向Cache主机发送探测包，确认主机没有响应。
4. 向备机发送探测包，确认备机存活。
5. 通知备机断开与主机之间的连接。
6. 通知主机降级。
7. 修改路由信息并同步新的路由到Proxy，切换完成。