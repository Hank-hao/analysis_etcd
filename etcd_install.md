

## [Install](https://github.com/etcd-io/etcd/releases)

### 镜像

```bash
curl https://quay.io/repository/coreos/etcd?tab=tags  # 镜像列表
docker pull quay.io/coreos/etcd:v3.5.2   # 拉取镜像
```

### docker standalone

```bash

```


### [docker cluster](https://blog.csdn.net/u011508407/article/details/108549703)

``` bash

# 下载镜像

docker pull quay.io/coreos/etcd:v3.2.32
# 创建新的bridge网络
docker network create --driver bridge --subnet=172.28.0.0/16 --gateway=172.28.0.1 hanketcd
# 查看创建的网络
docker network ls

# 创建node1
docker run -d \
-p 2479:2379 \
-p 2381:2380 \
--name etcdnode1 \
--network=hanketcd \
--ip 172.28.10.1 \
quay.io/coreos/etcd:v3.2.32 \
etcd \
-name node1 \
-advertise-client-urls http://172.28.10.1:2379 \
-initial-advertise-peer-urls http://172.28.10.1:2380 \
-listen-client-urls http://0.0.0.0:2379 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "node1=http://172.28.10.1:2380,node2=http://172.28.10.2:2380,node3=http://172.28.10.3:2380" \
-initial-cluster-state new

# 创建 node2
docker run -d \
-p 2579:2379 \
-p 2382:2380 \
--name etcdnode2 \
--network=hanketcd \
--ip 172.28.10.2 \
quay.io/coreos/etcd:v3.2.32 \
etcd \
-name node2 \
-advertise-client-urls http://172.28.10.2:2379 \
-initial-advertise-peer-urls http://172.28.10.2:2380 \
-listen-client-urls http://0.0.0.0:2379 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "node1=http://172.28.10.1:2380,node2=http://172.28.10.2:2380,node3=http://172.28.10.3:2380" \
-initial-cluster-state new

# 创建 node3
docker run -d \
-p 2679:2379 \
-p 2383:2380 \
--name etcdnode3 \
--network=hanketcd \
--ip 172.28.10.3 \
quay.io/coreos/etcd:v3.2.32 \
etcd \
-name node3 \
-advertise-client-urls http://172.28.10.3:2379 \
-initial-advertise-peer-urls http://172.28.10.3:2380 \
-listen-client-urls http://0.0.0.0:2379 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "node1=http://172.28.10.1:2380,node2=http://172.28.10.2:2380,node3=http://172.28.10.3:2380" \
-initial-cluster-state new

# 查看docker
docker ps 
# 查看集群列表
docker exec -it 695 etcdctl member list

# etcd 参数
-name	                # 设置成员节点的别名，建议为每个成员节点配置可识别的命名
-advertise-client-urls	# 广播到集群中本成员的监听客户端请求的地址
-initial-advertise-peer-urls	# 广播到集群中本成员的Peer监听通信地址
-listen-client-urls	            # 客户端请求的监听地址列表
-listen-peer-urls	            # Peer消息的监听服务地址列表
-initial-cluster-token	# 启动集群的时候指定集群口令，只有相同token的几点才能加入到同一集群
-initial-cluster	    # 所有集群节点的地址列表
-initial-cluster-state	# 初始化集群状态，默认为new，也可以指定为exi-string表示要加入到一个已有集群
```