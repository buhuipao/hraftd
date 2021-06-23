[toc]

## hraftd 基本信息

* 基本介绍：[https://www.philipotoole.com/building-a-distributed-key-value-store-using-raft/](https://www.philipotoole.com/building-a-distributed-key-value-store-using-raft/)
* github：[https://github.com/otoolep/hraftd](https://github.com/otoolep/hraftd)
* 依赖的raft lib：[https://github.com/hashicorp/raft](https://github.com/hashicorp/raft)

## hraftd 实现

### 服务层

#### http服务

hraftd实现了一个简单的http服务，主要包括两个接口：

	* key操作接口：GET、SET、DELETE 字符串key-value的操作
	* join接口：raft JOIN操作

### 存储层

#### Store

Store的基本结构：

```go
// Store is a simple key-value store, where all changes are made via Raft consensus.
type Store struct {
	RaftDir  string		// raft数据存储目录
	RaftBind string		// raft协议绑定IP和端口
	inmem    bool			// 是否为内存模式

	mu sync.Mutex				 // 保护字典的互斥锁，其实改成sync.Map更简单，只是需要在快照时做个转换
	m  map[string]string // The key-value store for the system.

	raft *raft.Raft // The consensus mechanism（raft共识）

	logger *log.Logger	// 日志
}
```

为了和服务层对接，实现了httpd.Store接口，如下：

```go
// Store is the interface Raft-backed key-value stores must implement.
type Store interface {
	// Get returns the value for the given key.
	Get(key string) (string, error)

	// Set sets the value for the given key, via distributed consensus.
	Set(key, value string) error

	// Delete removes the given key, via distributed consensus.
	Delete(key string) error

	// Join joins the node, identitifed by nodeID and reachable at addr, to the cluster.
	Join(nodeID string, addr string) error
}
```

**创建raft系统需要六个重要对象**，如下：

```go
// NewRaft is used to construct a new Raft node. It takes a configuration, as well
// as implementations of various interfaces that are required. If we have any
// old state, such as snapshots, logs, peers, etc, all those will be restored
// when creating the Raft node.
// 1、配置；2、FSM对象；3、日志存储；4、标准存储；5、快照存储；6、用于和其他节点通信的网络配适器；
func NewRaft(conf *Config, fsm FSM, logs LogStore, stable StableStore, snaps SnapshotStore, trans Transport) (*Raft, error) {
...
}
```

如果启动hraftd时选择了非内存模式，那么日志存储和标准存储都是使用[raftboltdb](github.com/boltdb/bolt)，其中`raftboltdb`是基于[boltdb](https://github.com/boltdb/bolt)封装实现的。`boltdb`的源码解析参看：[https://youjiali1995.github.io/storage/boltdb/](https://youjiali1995.github.io/storage/boltdb/) 。



#### fsm

fsm其实就是Store的类型别名，它主要实现了raft.FSM接口，如下：

```go
// FSM provides an interface that can be implemented by
// clients to make use of the replicated log.
type FSM interface {
	// Apply log is invoked once a log entry is committed.
	// It returns a value which will be made available in the
	// ApplyFuture returned by Raft.Apply method if that
	// method was called on the same Raft node as the FSM.
	Apply(*Log) interface{}

	// Snapshot is used to support log compaction. This call should
	// return an FSMSnapshot which can be used to save a point-in-time
	// snapshot of the FSM. Apply and Snapshot are not called in multiple
	// threads, but Apply will be called concurrently with Persist. This means
	// the FSM should be implemented in a fashion that allows for concurrent
	// updates while a snapshot is happening.
	Snapshot() (FSMSnapshot, error)

	// Restore is used to restore an FSM from a snapshot. It is not called
	// concurrently with any other command. The FSM must discard all previous
	// state.
	Restore(io.ReadCloser) error
}
```

其中fsm.Snapshot方法里返回了一个**raft.FSMSnapshot**接口对象，这个接口对象实现的方法有：

```go
// FSMSnapshot is returned by an FSM in response to a Snapshot
// It must be safe to invoke FSMSnapshot methods with concurrent
// calls to Apply.
type FSMSnapshot interface {
	// Persist should dump all necessary state to the WriteCloser 'sink',
	// and call sink.Close() when finished or call sink.Cancel() on error.
	Persist(sink SnapshotSink) error

	// Release is invoked when we are finished with the snapshot.
	Release()
}
```

## hraftd 时序图

### 服务启动过程

<img src="/Users/chenhua/Library/Application Support/typora-user-images/image-20210608153124669.png" alt="image-20210608153124669" style="zoom: 50%;" />

### K-V请求处理过程（以Set为例）

<img src="/Users/chenhua/Library/Application Support/typora-user-images/image-20210608153401542.png" alt="image-20210608153401542" style="zoom:50%;" />



### Join（跟随）raft-leader过程

<img src="/Users/chenhua/Library/Application Support/typora-user-images/image-20210608153518414.png" alt="image-20210608153518414" style="zoom:50%;" />





