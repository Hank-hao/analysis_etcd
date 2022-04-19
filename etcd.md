[TOC]

## ETCD

- 各节点数据完全一致，基于Raft的一致性算法

### 术语

Raft：etcd所采用的保证分布式系统强一致性的算法。  
Node：一个Raft状态机实例。  
Member：一个etcd实例。它管理着一个Node，并且可以为客户端请求提供服务。  
Cluster：由多个Member构成可以协同工作的etcd集群。  
Peer：对同一个etcd集群中另外一个Member的称呼。  
Client：向etcd集群发送HTTP请求的客户端。  
WAL：预写式日志，etcd用于持久化存储的日志格式。  
snapshot：etcd防止WAL文件过多而设置的快照，存储etcd数据状态。  
Proxy：etcd的一种模式，为etcd集群提供反向代理服务。  
Leader：Raft算法中通过竞选而产生的处理所有数据提交的节点。  
Follower：竞选失败的节点作为Raft中的从属节点，为算法提供强一致性保证。  
Candidate：当Follower超过一定时间接收不到Leader的心跳时转变为Candidate开始竞选。  
Term：某个节点成为Leader到下一次竞选时间，称为一个Term。  
Index：数据项编号。Raft中通过Term和Index来定位数据。  

- https://cloud.tencent.com/developer/article/1634693


### 应用场景

- 服务发现
- 消息订阅和消费
- 负载均衡
- 分布式通知与协调
- 分布式锁
- 分布式队列
- 集群监控与 Leader 竞选


### 管理

```bash

etcdctl --endpoints=$ENDPOINTS endpoint health
etcdctl --write-out=table --endpoints=$ENDPOINTS endpoint status

# 成员管理
etcdctl --write-out=table --endpoints=$ENDPOINTS member list


etcdctl  user list
etcdctl  user add testuser
etcdctl  role add testrole
etcdctl  user grant-role testuser testrole
etcdctl  user revoke-role testuser testrole

# grant-permission, revoke-permission
etcdctl role grant-permission testrole read /test
etcdctl role revoke-permission testr --prefix=true test

etcdctl role get testrole
# role delete myrolename

# 启动认证, 要有root用户
etcdctl auth enable

```
- https://etcd.io/docs/v3.5/op-guide/authentication/

### 使用

``` bash

etcdctl --endpoints=$ENDPOINTS put key value
# get
etcdctl --endpoints=$ENDPOINTS get key
etcdctl --write-out="json" get key
etcdctl get / --prefix              # 查看所有key & value
etcdctl get / --prefix --keys-only  # 查看所有key

etcdctl --endpoints=$ENDPOINTS del key

etcdctl --endpoints=$ENDPOINTS put /aaa/bbb/ccc/ value  # 可以以目录的形式规划Key

```

### lease,租约

- 是一种检测客户端存活状况的机制
- 一种资源、对象
- 可用来将key绑定在此租约上

```bash
# 创建租期
etcdctl --endpoints=$ENDPOINTS lease grant 100                                 
# 将租期绑定到Key
etcdctl --endpoints=$ENDPOINTS put --lease=3881793005e2463e foo "Hello World"  
# 租约列表
etcdctl --endpoints=$ENDPOINTS lease list
# 查看租约详细信息
etcdctl --endpoints=$ENDPOINTS lease timetolive 26947f2ad7bbcf5a
# 续租
etcdctl --endpoints=$ENDPOINTS lease keep-alive 26947f2ad7bbcf5a 
# 查看租约绑定的key
etcdctl --endpoints=$ENDPOINTS lease timetolive 26947f2ad7bbcf5a --keys


- txn, 事务
```go
txnResp, err := txn.If(clientv3.Compare(clientv3.Value("/hi"), "=", "hello")).
        Then(clientv3.OpGet("/hi")).
        Else(clientv3.OpGet("/test/", clientv3.WithPrefix())).
        Commit()
```

- revision,
    - 每个 key 带有一个 Revision 号，每进行一次事务便+1，它是全局唯一的， 
    - 通过 Revision 的大小就可以知道进行写操作的顺序, 通过它实现分布式锁


### API

- The v3 protocol uses gRPC as its transport instead of a RESTful interface like v2

#### v3
- https://etcd.io/docs/v3.5/learning/api/
- https://etcd.io/docs/v3.5/dev-guide/api_reference_v3/
```bash

```

#### v2
- https://etcd.io/docs/v2.3/api/
```bash
curl http://10.100.56.202:2379/v2/keys/test -XPUT -d value="Hello world"  # 插入key=/test, value="Hello world"

```

### 持久化

- wal
- snap
- 自动压缩
- 碎片整理


## faq

- 先通过 http 建立了 cluster，然后再用自签证书 https 来建立，这样就会报错
    - 连接协议已经写到存储里了。 删除数据，或者使用 snapshot & restore 处理 

- 服务端证书配置，增加 client auth

## Doc

- [官方文档](https://etcd.io/docs/v3.4/)
- [etcdctl](https://etcd.io/docs/v3.3/dev-guide/interacting_v3/)
https://blog.csdn.net/Jmilk/article/details/108912201
- [etcd架构概览](https://bbs.huaweicloud.com/blogs/104633)
- [BoltDB的优点与缺点](https://zhuanlan.zhihu.com/p/47214093)
- [官方中文](https://www.bookstack.cn/read/etcd/README.md)