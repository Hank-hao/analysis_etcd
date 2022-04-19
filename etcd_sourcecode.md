[TOC]

## Source code

### 数据结构

```go

type raftNode struct {
	lg *zap.Logger

	tickMu *sync.Mutex
	raftNodeConfig

	// a chan to send/receive snapshot
	msgSnapC chan raftpb.Message

	// a chan to send out applyxz
	applyc chan apply

	// a chan to send out readState
	readStateC chan raft.ReadState

	// utility
	ticker *time.Ticker
	// contention detectors for raft heartbeat message
	td *contention.TimeoutDetector

	stopped chan struct{}
	done    chan struct{}
}

type raft struct {
    id uint64         // 当前节点在集群中的ID
    Term uint64       // 当前任期号
    Vote uint64       // 当前任期内, 当前节点投票给了谁(节点ID)
    readStates []ReadState

    // the log
    raftLog *raftLog

    maxMsgSize         uint64
    maxUncommittedSize uint64
    // TODO(tbg): rename to trk.
    prs tracker.ProgressTracker  //每个follower节点的日志复制情况(Next Index和Match Index)，都会记录在Progress中

    state StateType   // 当前节点在集群中的状态

    // isLearner is true if the local raft node is a learner.
    isLearner bool

    msgs []pb.Message  // 当前待发送消息

    // the leader id
    lead uint64
    // leadTransferee is id of the leader transfer target when its value is not zero.
    // Follow the procedure defined in raft thesis 3.10.
    leadTransferee uint64
    // Only one conf change may be pending (in the log, but not yet
    // applied) at a time. This is enforced via pendingConfIndex, which
    // is set to a value >= the log index of the latest pending
    // configuration change (if any). Config changes are only allowed to
    // be proposed if the leader's applied index is greater than this
    // value.
    pendingConfIndex uint64
    // an estimate of the size of the uncommitted tail of the Raft log. Used to
    // prevent unbounded log growth. Only maintained by the leader. Reset on
    // term changes.
    uncommittedSize uint64

    readOnly *readOnly

    // number of ticks since it reached last electionTimeout when it is leader
    // or candidate.
    // number of ticks since it reached last electionTimeout or received a
    // valid message from current leader when it is a follower.
    electionElapsed int

    // number of ticks since it reached last heartbeatTimeout.
    // only leader keeps heartbeatElapsed.
    heartbeatElapsed int

    checkQuorum bool
    preVote     bool

    heartbeatTimeout int
    electionTimeout  int      // 选举超时, 一旦超时, 重新新一轮选举
    // randomizedElectionTimeout is a random number between
    // [electiontimeout, 2 * electiontimeout - 1]. It gets reset
    // when raft changes its state to follower or candidate.
    randomizedElectionTimeout int
    disableProposalForwarding bool

    // 推进逻辑时钟的处理方法
    // 如果当前是Leader节点，指向tickHeartbeats方法
    // 如果当前是其他节点，指向tickElection方法
    tick func()

    // 收到消息时的处理方法
    // 如果当前是Leader节点，指向stepLeader方法
    // 如果当前是其他节点，指向stepFollower、stepCandidate方法
    step stepFunc

    logger Logger
}

type raftLog struct {
	// storage contains all stable entries since the last snapshot.
	storage Storage

	// unstable contains all unstable entries and snapshot.
	// they will be saved into storage.
	unstable unstable

	// committed is the highest log position that is known to be in
	// stable storage on a quorum of nodes.
	committed uint64
	// applied is the highest log position that the application has
	// been instructed to apply to its state machine.
	// Invariant: applied <= committed
	applied uint64

	logger Logger

	// maxNextEntsSize is the maximum number aggregate byte size of the messages
	// returned from calls to nextEnts.
	maxNextEntsSize uint64
}

```

### raft
![时序图](http://assets.processon.com/chart_image/623d7c7fe401fd070dbab8fc.png)


- raft.step() 是函数指针, 根据不同角色, 运行stepLeader()/stepFollower()/stepCandidate()


#### readindex查询过程

- EtcdServer.Range() 等数据请求函数
- EtcdServer.linearizableReadNotify() 向 EtcdServer.readwaitc 发送信号
- go EtcdServer.linearizableReadLoop() 处理 EtcdServer.readwaitc 信号
    - 发送 MsgReadIndex 消息
    - node.ReadIndex
    - MsgReadIndex 消息发送给 node.recvc
    - go node.run() 处理 node.recvc 消息
    - 根据角色调用 stepLeader()/stepFollower()/stepCandidate()
    - stepLeader()
        - r.readOnly.addRequest(r.raftLog.committed, m), 保存当前commited
        - 向 followers 广播心跳
		- 处理 followers 返回的 MsgHeartbeatResp
		- 多数派响应后, 将请求的 committed 填充到 raft.ReadState 中
		- newReady()后, ReadState 发送给 raftNode.readStateC
	- 处理 raftNode.readStateC 信号
	- 




#### Listen Client Start

```golang

type serveCtx struct {
	lg       *zap.Logger
	l        net.Listener
	addr     string
	network  string
	secure   bool
	insecure bool

	ctx    context.Context
	cancel context.CancelFunc

	userHandlers    map[string]http.Handler  //http请求处理函数
	serviceRegister func(*grpc.Server)       //grpc请求处理函数
	serversC        chan *servers
}

embed.StartEtcd(){
	e.sctxs(map[string]*serveCtx) = configureClientListeners()  //每个client listener, 一个context
	// 每个context, 监听一个net.Listener

	e.serveClients(){
		sctx.serve(){
			v3client.New(s)  // grpc 

			gs = v3rpc.Server()
		}
	}
}

```

### mvcc
- kv存储实现
```golang
store := NewStore()  
// store结构实现了KV接口
// readView 实现了 ReadView接口
// writeView 实现了 WriteView接口

```

### wal, Write Ahead Log, 预写日志

- write ahead log, 预写日志, 数据库系统中一种常见手段, 保证数据操作的原子性和持久性
- 每个wal文件预设64MB, 超过了创建新的wal文件


#### 数据结构

```go
type WAL struct {
	lg *zap.Logger

	dir string // the living directory of the underlay files

	// dirFile is a fd for the wal directory for syncing on Rename
	dirFile *os.File

	metadata []byte           // metadata recorded at the head of each WAL
	state    raftpb.HardState // hardstate recorded at the head of WAL

	start     walpb.Snapshot // snapshot to start reading
	decoder   *decoder       // decoder to decode records
	readClose func() error   // closer for decode reader

	unsafeNoSync bool // if set, do not fsync

	mu      sync.Mutex
	enti    uint64   // index of the last entry saved to the wal
	encoder *encoder // encoder to encode records

	locks []*fileutil.LockedFile // the locked files the WAL holds (the name is increasing)
	fp    *filePipeline
}

```

### snapshot

- 数据恢复
    - 停止 etcd
    - 恢复数据
    - 启动 etcd


#### 参考
- https://zhuanlan.zhihu.com/p/137512843
- snap https://zhuanlan.zhihu.com/p/29865583?from_voters_page=true





