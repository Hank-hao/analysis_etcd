[TOC]

## raft

- https://raft.github.io/raft.pdf)
- 用来解决一致性问题的共识算法

### raft 如何保证一致性  

- 强一致性算法的分布式存储仓库
- Log Replication
- WAL(Write Ahead Log)预写日志，是数据库系统中常见的一种手段  
- zookeeper使用Paxos算法, 比较复杂
- Raft算法在做决策时需要超半数节点的投票，所以etcd集群一般推荐奇数节点，如3、5或者7个节点构成一个集群
- 推荐集群个数3, 5, 7
- 尽量不要使用lb 作为 etcd endpoints 配置，etcd client 是 grpc 访问，请使用默认的 全量list,
- 客户端做负载均衡的方式


### Leader Election

- [参考](http://www.xuyasong.com/?p=1706)

#### term 任期

#### 状态

- Leader 领袖
    - 负责所有的请求的处理，并向Follower发送心跳检测
- Follower 群众
    - 只响应Leader或Candidate的请求, 不处理用户请求, 收到请求会转发给leader
- Candidate 候选人
    - 一个中间状态,当Follower收不到leader的检测消息,等到选举计时器(election timer)过期,就会把自己的状态设置为Candidate,触发新的选举
- Learner 
    - 不参与投票, v3.4 later

#### 选举过程

- 系统刚启动的时候，每个节点都是follower状态
- 如果节点在follower状态期间，在一个election timeout时间内没有收到来自Leader的消息，则可以假设没有leader，于是启动选举过程
- 新增自己本地的任期, 此时节点转换到了Candidate状态
- 发送RequestVote RPCs给其他follower，让他们支持自己当leader
- 此时在收到投票结果后，可能会出现3种结果
    - 获得了**大多数**的认可，赢得了投票，成为leader
    - 发现了别人已经成为leader了或者自己的任期落后于别人的任期，自动转换为follower
    - 一个选举周期过去了，也没有赢得竞选，开始新一轮竞选

#### 思考

- election timer是一个随机数, 不会连续出现同时成为Candidate的情况
- 如果Leader节点频繁宕机，或者选举反复进行，怎么办？
- 是不是谁先发起了选举请求，谁就得到了Leader？
    - 不会, 还要取决于Candidate节点的日志是不是最新最全的日志


### 日志同步
- 分布式共识算法

#### 过程
- A(Leader),B,C
- A节点接到请求后（如 set a=10）,将本请求计入本地log
- A节点向B和C发送Append Entries消息（set a=10）
- B和C将此信息计入本地log，并返回给A,已ok
- A将日志信息置为 已提交（Commited）,然后状态机处理，返回
- 向B和C发送消息，该信息已提交
- B和C接到消息后，交给自己的状态机处理


### 日志压缩和快照


### 负载均衡


### watch机制


### 是否是强一致性

- 是强一致性
- 线性一致性与顺序一致性都是强一致性
- raft 采用的是线性一致性

- ReadIndex方案


### 设备数量问题

- 每一次写操作需要集群中的大多数节点将日志落盘成功后, 才能修改内部状态机, 所以设备越多性能越差
- 3, 5, 7 个节点即可

### 脑裂问题

- 集群化的软件总会提到脑裂问题
- 脑裂就是同一个集群中的不同节点，对于集群的状态有了不一样的理解
- [etcd不存在脑裂问题](https://www.kubernetes.org.cn/7569.html)
- 发生网络隔离时，会分成大集群和小集群, 那么大集群会提供服务，小集群不会提供服务(写服务无法服务, 读服务可能会时旧数据)
- 可以保证CAP中的C和P, 不符合A





## 参考
- [心跳和选举](https://bbs.huaweicloud.com/blogs/110887)
- [etcd问题,调优,监控](https://www.kubernetes.org.cn/7569.html)
- [etcd技术内幕](http://www.xuyasong.com/?p=1706)
- [grpc的负载均衡](https://grpc.io/blog/grpc-load-balancing/)
- https://www.cnblogs.com/ricklz/p/15155095.html