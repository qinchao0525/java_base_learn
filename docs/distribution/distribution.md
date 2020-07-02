##分布式
###1、CAP和BASE
* C()，一致性，分布式环境下，由于网络等原因，各个节点的状态必须保证一致，节点状态要相同，这个中相同就是一致性
* A()，可用性，在网络中部分节点宕机不可用时，整个分布式系统还能不能对外提供服务
* P()，分区容错性，网络问题造成集群出现网络分区时，又应该怎么办呢？
* 一般认为cap只能选其中之二，而分区容错性有是必须的，所以一般是在一致性和分区容错性之间权衡。
* 为什么分区容错性是必须的，分布式系统出现网络分区是必然会出现的！

* BASE基本可用，既然cap不能兼得，那么久退而求其次，选择基本可用
* 基本可用。当出现故障时，允许损失部分可用性，就是例如允许响应时间上损失、功能上损失，对应服务降级等情况。
* 弱状态。允许系统出现中间状态
* 最终一致性。可以不要求强一致性，但是保证经过一段时间的同步后，系统最终会达到一致。
* migration和pigeon用到的分布式理论
	* 如何保证cap理论，如何保证分布式一致性，通过主从结构，由主节点去管理，同时节点没有状态，那么很容易切换主节点。集群的状态是由zookeeper去管理的。
	* 分区容错性，如果集群出现了网络分区怎么办？为什么pigeon会有问题，因为在考虑网络分区问题的时候，为了防止脑裂，在网络抖动的时候就断开链接并自杀，太独断了。我做的改进是，一是增加了DB备份，核心数据放到DB，核心数据写到DB，然后再由DB同步到zk？这样有两个问题
		* DB如果不可用了怎么办？那就真的认为不可用了。
		* 如果ZK挂了，那么状态都先写到DB，但是此时集群中没有leader节点，不能新增任务和机器，当ZK恢复后，新的leader节点会从DB拉取同步消息到zk。
		* 核心数据是先写DB，再写zk，如何保证事务性呢？这两个操作如何保证成功，两阶段提交，插入DB会等待成功，不成功会重试三次。当插入成功后再写入DB，如果写入zk失败也没问题，leader会有后台线程定期同步zk和DB的状态。当然，如果Db写入不成功，那么我们会扔出可检查异常，执行任务的机器catch到异常以后会报警并且重试，对核心逻辑会阻塞不在运行。


###2、两阶段提交协议，三阶段提交协议
* 2pc。一个事务在提交时，会先向要提交的节点发起询问是否可以提交，当所有的都恢复可提交的时候才提交事务，否则回滚事务。
	* 特点是简单易实现。但是缺点很多，会锁掉很多资源；效率太低，有失败的就回滚；网络问题可能造成阻塞。
	* 2pc是一个强一致的算法
	* 2pc的单点问题，提交事务的失败了，整个就跪了，而且不知道是提交还是回滚，整个集群就跪了
* 3pc。比2pc多了一个预提交截断，先询问可不可以提交，都回答可以了，那就准备提交；然后发送提交请求，完成后就提交；如果超时，或者有反馈不能提交的就中断事务。
	* 协调者依旧有单点问题
	* 网络故障
	* 好处是多了超时中断事务机制，不会让整个集群资源一直阻塞。

###3、ACID事务
* 事务是值将一系列对数据状态便跟的操作放在一个执行逻辑单元中执行，数据库事务是最典型的操作
* A是原子性，表明整个执行逻辑单元所有操作都在一次中执行，不会发生分组
* C是一致性，表示事务是让系统状态从一个状态到另一个状态，没有中间值
* I是隔离性，并发事务之间是隔离的，不会互相影响
* D是持久性，就是事务一旦完成提交，状态就不能再改变

###4、分布式实践---zookeeper
* zookeeper是典型的分布式服务，他是怎么保证分布式的一致性等问题的呢？
* 分布式一致性协议Paxos算法
	* 是解决分布式环境下一致性问题的算法，我们基于网络消息不会篡改这个假设做的。
	* 可以解决少数节点离线以后，剩余多节点的情况下系统仍然可以达到一致性

***paxos理论理解***：



***ZAB协议***

* Zookeeper的原子广播协议
	* ZAB支持崩溃恢复
	* zookeeper采用单一的主进程去接受并处理客户端的事务请求，同时通过ZAB协议，将服务器的状态同步到所有的副本进程当中去。
* ZAB保证集群中存在过半机器的时候
* 主备模式的系统架构来保持集群中个副本数据之间的一致性
* 只用单一的主进程来接收处理客户端的所有事物请求，采用zab协议将服务器的状态变更以事物proposal的形式广播到所有副本
* 所有事务请求都必须由一个全局唯一的服务器来协调处理，这个服务器就是leader，其他服务器为follower。
* leader负责把一个状态便跟的协议生成proposal，然后分发给集群中其他的follower，然后leader等待所有的follower反馈状态，超过半数的follower进行正确反馈以后，leader就再次向所有的follower发送commit请求，要求将前一个proposal事务进行提交。
* 如果过半的follower跟leader的状态完成了同步，那么久开启广播模式。
* 崩溃恢复模式
	* 要成为leader，必须有超过一半的进程支持。
	* 进入崩溃恢复模式以后，只要过半服务器能碧彼此正常通信就可以产生一个新的leader，然后再次进入消息广播模式。

（1）消息广播

* 简单的二阶段提交协议，先发送proposal，然等待过半的follower反馈正确的ack
* 当然如果leader此时挂了，整个集群的状态就会不一致。因此加入崩溃恢复
* 崩溃恢复模式会首先选取一个leader，然后这个leader开始检查事务日志中所有的proposal是否完成了同步，

***Raft协议***



***如何解决脑裂问题***

* 设置仲裁机制，slave在准备接管master的时候需要仲裁机制保障是否正常
* 根据lease判断，过期的请求失效
* 隔离

***高可用架构设计***

* 主备架构-->yarn的resourceManager，hdfs的namenode都是主备架构
* 互备，双活架构
* 集群模式，pigeon和migration都是集群模式

***负载均衡***

* 轮询模式，pigeon和migration都是轮询模式
* 最少链接，根据服务上连接数
* 最小负载，pigeon一开始就有2c和4c的机器，基于负载去查找使用的机器
* 地址hash

***全局唯一id生成器***

* uuid，随机性，不具有顺序性
* id生成表，简单易用，使用数据库事务保证
* snowflake算法
	* 数据有三部分组成，41位的时间序列，10位的机器表示，12位的计数顺序
	* 41位之间序列精确到ms，支持使用69年
	* 10位的机器表示，可以支持扩容到1024台机器
	* 12位的计数顺序，支持每个节点每ms生成4096个id
	* 这里的数据是有特殊含义的，很容易从id推断出机器，集群和时间戳信息

***一致性hash***

* 将整个hash空间组织成一个虚拟的圆环
* 将各个节点使用hash进行计算，可以确认某个节点在hash环上的位置
* 定位的时候，将数据key使用相同的hash函数计算hash值，并确定这个数据在环上的位置，然后从这个位置沿着圆环顺时针行走，遇到的第一台服务器就是应该定位的服务器。
* 如果一个节点不可用，受影响的只是其前一台服务器之间的数据，其他不收影响
* 如果加入一个节点，受影响的只是新服务器到前一台服务器之间的数据

###5、分布式实践---分布式事务
* 实现原理


###6、分布式锁实现
（1）基于数据库实现分布式锁

* 利用数据库实现排它锁，想要获取锁的时候，需要去数据库更新状态，数据库可以保证并发情况下只有一个数据状态更新成功
* 依赖数据库，当数据库挂掉以后，服务即不可用。解决办法数据库主从库
* 锁没有失效时间，锁失败以后，其他线程无法获取锁。增加超时数据清理线程
* 非阻塞的锁。可以一直重试，造成阻塞
* 不可重入。在数据库中加字段标记主机信息和进程信息，如果需要重入场景，进行对比就可以了。

（2）基于缓存实现分布式锁

* 不会，没写过

（3）基于zookeeper实现分布式锁

* 锁获取，首先在ZK上创建永久节点所谓锁所在的path
* 需要锁的时候在该永久节点下创建临时顺序节点
* 客户端在请求锁的时候，查找锁路径下的临时节点并排序，判断自己是不是最靠前的，如果是，就成功获得锁。
* 如果client2请求，直接在path下创建新的顺序临时节点即可
* 同时，client2有获取锁的请求的时候，client2如果发现自己并不是最靠前的，那么就只向比自己靠前的节点注册一个watcher，如果有新的请求过来，同样是上面的逻辑。
* 当client1释放锁的时候，client2会获得通知，然后client2会再次查找自己是不是最小的，如果是就获得所，如果client2奔溃消失，client3就会获得通知。
* zookeeper的顺序节点是在节点后默认加上了数字后缀，上限是整形的最大值

###7、限流策略
* 应对高并发，限流、降级、消息队列
* 限流算法：漏桶算法、令牌桶算法、计数算法
* 漏桶算法的流出量是平稳的，不会出现突发状况
* 令牌桶允许峰值出现，但是保证整体是满足限流要求的
* Guava的ratelimiter是令牌桶算法，以固定频率向通中放入令牌

###8、全链路压测
* 数据库角色区分，专门的压测库
* 压测标区分，压测走到压测库

###9、