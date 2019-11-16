# zookeeper相关常见面试题

## 1. 说说Zookeeper的zab协议
### 1.1 zab协议
1. 全称：zookeeper atomic broadcast (zookeeper原子广播协议)
2. 不是一个通用的分布式一致性算法，是一种特别为Zookeeper设计的一致性算法
3. 一种主备模式的系统架构，使用单一的主进程来接收并处理来自客户端的所有事务，使用zab
协议将服务器状态的变更以proposal(提议)的形势广播到所有的副本进程上去(其他的zk节点)。
4. zab协议能够保证同一时刻只有一个主进程来广播服务器的状态变更
5. zab协议能够保证全局的变更序列被顺序应用
6. zab协议能够保证主进程出现崩溃或重启的情况下，能够快速选主，整个集群能够快速恢复(300ms就可以恢复)。
### 1.2 zab协议介绍
zab协议包括两部分：
    * 恢复模式
    * 原子广播
#### 1.2.1 恢复模式
1. 进入崩溃恢复的前提条件是：整个集群第一次启动，或者leader出现崩溃
2. 当选举产生新的leader，而且集群中有过半的服务器都与leader的数据同步完毕后，
会退出恢复模式，此时进入原子广播模式
#### 1.2.2 原子广播
1. 如果一台新的zk节点加入一个正常的zk集群中，会自觉地进入恢复模式与主节点同步，同步
完成后会自动参与到原子广播模式中
2. 集群中只有leader能够处理客户端的事务，集群中的其他节点收到事务请求会转发给leader
3. 在开始广播之前，所有的节点都会进入恢复模式与主节点进行同步，同步完毕后进入广播模式
4. 原子广播类事一个简化的二阶段提交策略，leader接收事务后会发起proposal，follower收到
proposal后要么正常反馈ack，要么抛弃leader，只要leader收到过半服务器的ack，就发起
提交，不用等待所有的服务器。
5. 这种简化版二阶段提交无法处理leader崩溃的数据不一致问题，因此会有恢复模式
6. 消息广播协议基于TCP，TCP具有FIFO特性，能够保证消息接收和发送的有序性
7. proposal具有一个全局单调递增的ZXID
8. leader会为每个follower分配一个单独队列，将提议放入队列以FIFO策略发送，follower拿到提议会把
提议写入磁盘，然后返回ack。leader拿到过半的ack之后就会广播commit消息给所有的follower，
follower收到commit也会提交自身事务。
#### 1.2.3 数据一致性
需要解决的2个问题
* zab协议需要确保leader提出的commit请求要被所有的follower执行
  
* 如果leader提出proposal之后就挂了，没有发出对应的commit命令，这类proposal要被丢弃

关于第一个问题：
1. **所以如果出现leader崩溃，新选举出来的leader服务器必须要拥有集群中已提交的proposal中的最大的zxid**
2. 选出新leader之后，新leader会把未被同步的proposal发给follower进行同步，并且
紧接着就发commit，等到follower确认后，leader会把这些同步完毕的follower添加到
真正可用的follower列表中，并开始之后的流程。

第二个问题：
1. 要确保丢弃崩溃leader发出的未commit proposal，奥秘在于zxid。
整个zk集群在处理外界请求时，可能会出现多个leader，每个leader负责不同的时期。

> 例如，一开始节点A是leader，后来节点A挂了，节点B成为Leader，然后节点A恢复，重新加入
集群，此时节点B也挂了，节点A幸运的又成为了leader。这样总共有3个leader，分别对应不同
的时期。

2. 再来说zxid，这个id是一个64位数字，低32位是是一个全局单调递增的计数器，高32位
代表了leader周期(epoch)编号，简单的说，一个zxid是由两部分组成的:epoch code + 流水号。

> 举个栗子，一个合法的zxid可以简化成这样表示: epoch-001-0001，epoch-001代表epoch code，
后面的0001代表在epoch-001这个leader纪元中的第一个proposal id

每次选举新的leader时，这个leader解析出本地事务中的最大zxid的epoch code，用上面的例子
就是epoch-001(其实是个32位数字，这里便于理解，使用字母)，在epoch-001基础上加1，生成
epoch-002作为自己任期内的高32位，后面如果要广播新的事务，使用epoch-002-0001这个zxid进行广播。
回过头来，再来思考一下，为什么在选新leader时我们要使用本地已提交的proposal的最大zxid，
因为如果你本地有未提交的zxid，这其实是无意义的。我们来思考一个场景：  
leader发生崩溃，产生新的一轮选举，此时一个zk节点A本地有3个proposal，分别称为p1,p2,p3，其中p1已提交，p2,p3均未提交
针对p2,p3，选举的策略如下：
1. 节点A本地未提交p2,但是集群中存在一个quorum(过半机制，一个commit一定要被集群中过半数的节点同步
，否则就丢弃proposal，这个过半集合就是quorum)，这个集合中的节点BCD已经提交了p2,
BCD这三个节点的最大zxid一定比A大，那么A一定不会成为leader。
2. 节点本地未提交p3,但是集群中没有节点提交p3,因为在旧leader提出p3之后还没来得及
提交c3就挂了，由于存在p2的quorum集合，所以A还是不会成为leader
考虑上面的策略，你会发现旧leader未提交的proposal自动的就被中策略给丢弃了，perfect!

### 1.3 深入zab协议
> TODO 以后再深入学习整个协议的数学表示吧，目前看懂策略就好了
## 2. zab协议和paxos算法的区别
> TODO 以后有时间再深入研究paxos算法

## 3. 关于curator你了解吗


## 4. zookeeper的常见应用场景

### 4.1 数据发布/订阅 
基于watcher机制，zookeeper采用**推拉结合**的方式，客户端向zk注册监听某个path，这个path
发生变化后会通知给客户端，此时客户端去拉取最新的数据。
这些应用场景是，zk保存的数据量较小，数据内容在运行时会发生动态变化，且不是非常频繁。

1. 统一配置中心
2. 内存缓存刷新

### 4.2 负载均衡，域名解析
将域名和节点id端口配置到zk的节点下，举个栗子: 
`/dns/test.learning.com/192.168.1.1`，这样发起请求之前先去zk里查一下就行了，
然后随机选出一个节点，进行定位。如果还要监听节点ip的变化，使用数据发布订阅的watcher机制
即可。原理其实都差不多  
### 4.3 分布式锁
最小序号的临时节点原理：多个服务到zk的某个zknode下创建临时有序节点，并查看自己的节点
序号是否是最小的那个，如果是，获得锁，其他的服务器监听比自己节点序号减一的那个节点，
例如，服务器A创建了lock-0001,服务器B创建lock-0002,服务器C创建了lock-0003，
此时A获得锁，B监听lock-0001,C监听lock-0002，当A释放锁后，lock-0001消失，B获得
WatchEvent，B持有锁，B执行完业务逻辑后释放锁，lock-0002消失，此时C获得watchEvent，
C持有锁，如此往复。**之所以选用临时节点，是因为如果持有分布式锁的服务挂了，临时节点能够保证
到了超时时间临时节点能够自动删除，锁最终能够释放，不会阻塞其他的节点。**

### 4.5 master选举(同时创建节点+临时节点)
所有的服务器在某个znode下同时创建一个子node，一定只有一个服务器能够创建成功，这个服务器就是master，
其他的服务器watch这个子node，当master挂了，再次发起一轮新的选举。

### 4.6 分布式协调
**mysql数据同步的例子** 
我需要同步某个业务主表，可以创建znodeA = /sync_data_base/task_sync_user_info/节点。
在znodeA 中再创建三个字段：
* offset   kafka的offset 代表处理进度
* instance 当前在执行任务的进程会在这个目录新增一个临时节点
* status   整个任务的状态，正常运转，暂缓

数据同步有两种模式，采用哪种要看数据一致的延迟：

* 热备份：主要用于低延迟的数据同步，多个同步进程向instance目录下创建临时有序节点，
判断自己的节点是不是序号最小的，如果是，就开始同步。如果两个进程，那么一定有一个不是最小的，
这样它会注册一个watcher到最小的序号节点，将自己的状态设置成standby，当主进程挂掉，standby就成为最小
节点，接管同步任务
* 冷备份：主要用于高延迟的数据同步，有一部分同步进程属于自由人，它们会不断扫描任务列表，
如果这个列表中的任务没有进程工作，就接管同步任务，**此时如果有其他的进程进来发现已经有进程在干活了，
这些其他进程不会在这里standby，而是自动删除自己创建的临时有序节点，继续去扫描其他的任务**，
之前完成同步任务的进程就继续扫描空闲的任务，如此往复。

### 4.7 健康监测和集群管理
服务还可用就去zk创建一个临时节点，服务与zk的session断开后临时节点就没了。
关注这些临时节点可以使用watcher，当节点消失说明节点挂了。

### 4.8 命名服务
和域名解析很像
### 4.9 分布式队列
分布式barrier，主要做并行计算用，一般的业务系统很少用到

## 5. zookeeper的技术内幕

### 5.1 系统模型
#### 5.1.1 数据模型
1. ZNODE代表一棵树
2. ZXID高32位表示epoch，低32位是顺序递增的数据
#### 5.1.2 节点特性 
三种基本节点类型
1. 持久节点 persistent (持久)
2. 临时节点 ephemeral(短暂的，朝生暮死)
3. 顺序节点 sequential

四种组合节点类型：
1. 持久节点
2. 持久有序节点
3. 临时节点
4. 临时有序节点

节点还包括状态信息
* czxid 数据节点被创建时的事务ID
* mzxid 数据节点最后一次被更新时的事务ID
* ctime 节点创建时间
* mtime 节点最后一次被更新的时间
* version 节点的版本号
* cversion 子节点的版本号
* aversion 节点的ACL版本号
* ephemeralOwner 创建该临时节点的会话的sessionID。如果是持久节点，这个值=0
* dataLength 数据内容的长度
* numChildren 节点的子节点个数
* pzxid 节点的子节点列表最后一次被修改时的事务ID。注意，只有子节点列表变更才会变更pzxid，
         内容变更不会影响pzxid。
 
#### 5.1.3 版本-保证分布式数据的原子性操作
上面的列表有几个字段标明是了节点的版本号，version,cversion,aversion
只要节点被变更过，不管值是否改变，version都会加1，version主要表示的是变更的次数。
zookeeper使用版本号来实现乐观锁，如果对某个节点的值进行更新，如果版本号不一致，就会
抛出BadVersionException。

#### 5.1.4 Watcher 数据变更的通知

##### 5.1.4.1 watcher机制简述

watcher机制主要参与角色有三个：
* 客户端线程
* 客户端WatchManager
* zookeeper服务器

客户端在向服务器注册watcher的同时，会将watcher对象存储在客户端的watchManager中。
zookeeper服务器触发watcher事件后，会向客户端发通知，客户端线程从watchManager取出
对应的watcher对象执行回调逻辑。
简单说就是，我看着你呢，你有了变化告诉我，具体有什么变化，我会自己亲自去查。

##### 5.1.4.2 Watcher通知状态

keeperState包括以下四种，每种都包含了可能的几种 eventType
1. SyncConnected 此时客户端和服务器处于连接状态
    * none 客户端与服务器成功建立会话
    * NodeCreated Watcher监听的对应节点被创建
    * NodeDeleted Watcher监听的对应节点被删除
    * NodeDataChanged Watcher监听的对应节点的数据内容发生变更
    * NodeChildrenChanged Watcher监听的节点的子节点列表发生变更
2. Disconnected 此时客户端和服务器处于断开连接状态
    * none 客户端和服务端断开连接
3. expired 此时客户端会话失效，也会收到SessionExpiredException异常
    * none 会话超时
4. authFailed 使用错误的scheme进行权限检查，也会抛出AuthFailedException异常
    * none 使用错误的scheme进行权限检查

##### 5.1.4.3 回调方法process(WatchedEvent event)

参数WatchedEvent包含3个属性：keeperState ,eventType,path，前两个属性就是上面介绍的。
服务端在生成WatchedEvent后，会将其包装成一个WatcherEvent，这两个实体的区别是后者
实现了序列化接口，可以用于网络传输。这样客户端收到WatcherEvent会将其还原成一个watchedEvent
作为参数调用回调方法process，这个实体是非常简单的。
举个栗子：
    keeperstate:SynConnected
    eventType:NodeDataChanged
    path:/test
可以看出客户端无法从事件中获取变更的详细信息，只能自己去读取变化。
##### 5.1.4.4 客户端设置Watcher
先说角色：
* Zookeeper 一个zookeeper实例，提供getData,getChildren,exist三个方法
* ClientCnxn 为客户端连接管理socket i/o，进行网络传输
* WatchRegistration 保存节点的路径信息和watcher的对应关系
客户端可以使用getData，getChildren,exist三个接口来向服务器注册watcher，注册的原理都是一样的，
调用这三个方法时，可以选择是否传递watcher对象，或者选择一个标志位，是否使用默认的watcher去监听。
zookeeper规定packet为一个最小的通信单元，用于进行客户端和服务端之间的网络传输，任何传输的对象最后
都会被封装成一个packet对象。getData方法最终会将request封装成packet，将WatchRegistration塞进packet，
clientCnxn负责发送request，等待服务器响应，响应成功后，ClientCnxn会执行finishPacket方法，将WatchRegistration
从packet对象中取出，添加到客户端的ZKWatchManager中。注意，WatchRegistration不会被序列化传递到服务端。
##### 5.1.4.5 服务器处理Watcher
* ServerCnxn 代表和客户端的连接，实现了Watcher接口，可以作为Watcher使用
* WatcherManager 负责保存节点path和ServerCnxn的关系，还负责Watcher事件的触发，并移除那些触发的Watcher

客户端发来的请求中会有一个标志位，标志位true时，服务端认为需要处理Watcher注册。
如果需要注册，服务端会把节点path和ServerCnxn对象(可以看做一个Watcher)保存到服务端的
WatchManager中。
##### 5.1.4.6 服务器触发Watcher

如果客户端调用了setData方法，也就是修改某个路径的数据，服务端会按照以下流程处理客户端的请求：
FinalRequestProcessor->ZKDataBase->DataTree

最终会调用服务端的ZKTree的setData方法，因为之前对这个path已经设置Watcher，DataTree会调用dataWatches.triggerWatch
方法触发监听。DataTree对象首先删除自己WatchManager
(有两个集合，一个path->watcher，一个是watcher->path)中这个path的Watcher，同时获取这些Watcher的引用，然后遍历
Watcher，执行watcher.process方法，这些watcher其实就是ServerCnxn，代表了与客户端的连接。
调用ServerCnxn.process方法，最终其实就是将变更事件封装成WatcherEvent发送给了客户端，也就触发了通知。
##### 5.1.4.7 客户端处理回调

客户端会起一个SendThread.readResponse来接收watcher事件，将消息反序列化，还原WatchedEvent
，最终回调Watcher，它回调Watcher其实是讲WatchedEvent对象通过队列交给另一个线程，EventThread。
eventThread不停的从队列取出Event，串行执行回调
##### 5.1.4.8 总结
1. watch是一次性的
2. 客户端是单线程串行执行Watch事件回调的，保证了执行顺序
3. 轻量级，WatchedEvent是通知的实体，只有三个部分，通知状态，事件类型和节点路径。
4. 网络开销和服务端的内存开销都十分廉价，很优秀的设计。

#### 5.1.5 ACL 保证数据安全
其实就是Zookeeper的密码
### 5.2 序列化与协议
#### 5.2.1 jute介绍
#### 5.2.2 使用jute进行序列化
#### 5.2.3 深入jute
#### 5.2.4 通信协议
### 5.3 客户端
#### 5.3.1 会话的创建过程
#### 5.3.2 服务器地址列表
#### 5.3.3 ClientCnxn:网络IO
### 5.4 会话
#### 5.4.1 会话状态
#### 5.4.2 会话创建
#### 5.4.3 会话管理
#### 5.4.4 会话清理
#### 5.4.5 重连
### 5.5 Leader选举
#### 5.5.1 leader选举算法
#### 5.5.2 选举的实现细节
### 5.6 服务器的角色介绍
#### 5.6.1 leader
#### 5.6.2 follower
#### 5.6.3 Observer
### 5.7 请求处理
#### 5.7.1 会话创建请求
#### 5.7.2 setData请求
#### 5.7.3 事务请求转发
#### 5.7.4 GetData请求
### 5.8 数据与存储
#### 5.8.1 内存数据
#### 5.8.2 事务日志
#### 5.8.3 snapshot 数据快照
#### 5.8.4 初始化
#### 5.8.9 数据同步
### 