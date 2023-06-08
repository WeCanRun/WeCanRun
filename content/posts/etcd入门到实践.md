---
title: "Etcd入门到实践"
date: 2022-04-09T11:26:27+08:00
draft: false
toc: true
description: 
keywords: 
  - etcd
categories:
  - 云原生 
tags: 
  - 云原生
---

![https//ilolinet/2021/03/25/k3bIZ94PwCUYijDjpg](https://i.loli.net/2021/03/25/k3bIZ94PwCUYijD.jpg)

## 一、Etcd 简介

> etcd 是一个强一致性的分布式键值对存储，设计用来可靠而快速的保存关键数据并提供访问。通过分布式锁，leader 选举和写屏障 (write barriers) 来实现可靠的分布式协作。etcd 集群是为高可用，持久性数据存储和检索而准备的。
> 
> etcd 的场景默认处理的数据都是系统中的控制数据。所以 etcd 在系统中的角色不是其他 NoSQL 产品的替代品，更不能作为应用的主要数据存储。etcd 中应该尽量只存储系统中服务的配置信息，以此来实现 **服务发现**、**分布式通知和协调**，常见的还会使用 etcd 进行 **主备选举**、**分布式锁**、**分布式数据队列** 以及 **服务健康监控**。对于应用数据只推荐把数据量很小，但是更新和访问频次都很高的数据存储在 etcd 中。官方数据显示，etcd 单实例的写操作性能在 w 级别，读操作在 10w 级别

## 二、Etcd 工作原理

### 1、核心组件

![https//ilolinet/2021/03/23/QdSj6TnVAZtHEu4png](https://i.loli.net/2021/03/23/QdSj6TnVAZtHEu4.png)

Server：用于处理用户发送的 API 请求以及其它 `etcd` 节点的同步与心跳信息请求。

KV Store：用于处理 `etcd` 支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件处理与执行等等，是 `etcd` 对用户提供的大多数 API 功能的具体实现，底层默认使用的是开源的嵌入式键值存储数据库 [bolt](https://github.com/boltdb/bolt)，这个项目目前的状态已经是归档不再维护，如果想要使用这个项目可以使用 CoreOS 的 [bbolt](https://github.com/etcd-io/bbolt) 版本。

Raft：Raft 强一致性算法的具体实现，是 `etcd` 的核心。

WAL：Write Ahead Log（预写式日志），是 `etcd` 的数据持久化存储方式。除了在内存中存有所有数据的状态以及节点的索引以外，`etcd` 就通过 WAL 进行持久化存储。WAL 中，所有的数据提交前都会事先记录日志。snapshot 是为了防止数据过多而进行快照压缩，默认每 10000 条记录做一次 `snapshot`, 经过 `snapshot` 以后的 WAL 文件就可以删除了。而通过 API 可以查询的历史 `etcd` 操作默认为 1000 条(v2 版本，v3 版本只要还未被压缩均能查到)；Entry 表示存储的具体日志内容

通常，一个用户的请求发送过来，会经由 Server 转发给 Store 进行具体的事务处理，如果涉及到节点的修改，则交给 Raft 模块进行状态变更、日志记录，然后再同步给其他的 `etcd` 节点以确认数据提交，最后进行数据的提交，再次同步。

### 2、数据一致性

`etcd` 集群是一个分布式系统，由多个节点相互通信构成整体对外服务，每个节点都存储了完整的数据，并且通过 Raft 协议保证每个节点维护的数据是一致的。在集群中的数据流向只有一个方向，即从 `Leader` （主节点）流向 `Follower`（从节点），当从节点收到写请求后均会自动转发到主节点，从节点甚至不需要知道谁是主节点，并由主节点执行写操作，然后追加日志同步到集群所有节点，也就是说所有 `Follower` 的数据必须与 `Leader` 保持一致，如果不一致则会被覆盖。从用户的角度来说，对 `etcd` 集群中的所有节点进行读写效果是一样的。

既然写操作涉及到节点间的数据同步，那么到底如何界定一个写入操作是成功的？`etcd` 认为写入请求被 `Leader` 节点处理并分发给了`Quorum(N/2+1)`个节点后，就是一个成功的写入。

为了保证数据的一致性，那么无论何时集群中必须有且只有一个有效的 `Leader`, 所以 `etcd` 通过 Raft 协议来选举 `Leader`。首先，需要明确的是在选举过程中会出现三种角色，它们的关系如下图，其中 `Candidate` 是在选举期间才会出现的角色，正常的 `etcd` 集群只有 `Leader` 和 `Folower`。

![https//ilolinet/2021/03/22/VuDn3ceI4k1lpEmpng](https://i.loli.net/2021/03/22/VuDn3ceI4k1lpEm.png)

**大致选举流程：**

1. 集群初始化时，所有节点都处于 `Follower` 状态，当 `Follower` 一段时间 (`选举计时器`超时) 收不到 `Leader` 的心跳消息，就认为 `Leader`出现故障导致其任期 (`Term`) 过期，`Follower` 会转成 `Candidate`；
2. `Candidate` 等`选举计时器`超时后，会先投自己一票，并向集群中其他节点发起选举请求，并带上自己的 Term ID 以及当前日志的 Max Index（确保新选举的 `Leader` 有最全的日志）;
3. 如果其他节点在 Term ID 内还没有投票，就会比较 `Candidate` 的 Max Index 是否比自己小，如果比自己大或者相等，就会投票给 `Candidate`；同时，会将自己的`选举计时器`重置；
4. `Candidate` 在获取到超过半数节点的选票后，升级为 `Leader`；并按照`心跳计时器`向集群中其他节点发送心跳报文，并同步日志等；`Follower` 收到心跳报文后，会重置`选举计时器`。

### 3、数据模型

`ectd` 提供了一个持久化、多版本、并发控制的数据模型，当键值对的值需要被新的数据替代时，`etcd` 还会继续保存先前版本的键值对。键值存储事实上是不可变的；它的操作不会就地更新结构，取而代之的是生成一个新的更新后的结构。在修改之后，`key` 的所有先前版本还是可以访问和监控的。为了防止维护老版本导致数据存储无限增长，存储应该被压缩来移除旧的版本，一旦存储被压缩来恢复空间，在压缩修订版本之前的修订版本都会被移除，即不可访问和监控。

在物理视图上，`etcd` 以 [B+ 树](https://en.wikipedia.org/wiki/B%2B_tree) 键值对的方式来存储物理数据，为了追求效率，存储的是当前修订版本与上一个修订版本间的差异（增量），单个修订版本的数据可能对应到 B+ 树上的多个键。需要说明的是，B+ 树上存储的键是一个三元组 `revision` (`major`, `sub`, `type`)，`major` 是持有 `key` 的存储修订版本、`sub` 区分同一个修订版本的不同 `key`、`type` 是用于特殊值的可选后缀。B+ 树的 key 空间的字符串按 **字典序** 排序，所以可以高效的支持 **范围查询**。

除了使用 B+ 树 这种数据结构来持久化数据之外，`etcd` 还通过在内存中维护一棵 B 树索引来加速键的范围查询，`key` 是存储暴露给用户的 `key`。值是到持久化 B+ 树修改的指针。

### 4、修订版本

- `Revision` 表示改动序号（ID），每次 KV 的变化，leader 节点都会修改 `Revision` 值，因此，这个值在 `cluster` 内是全局唯一的，而且是递增的
- `ModRevison` 记录了某个 `key` 最近修改时的 Revision，即它是与 `key` 关联的
- `CreateRevision` 与`ModRevison`类似、记录的是某个`key`创建时的 `Revision`
- `Version` 表示 KV 的版本号，初始值为 `1`，每次修改 KV 对应的 `Version` 都会加 `1`，也就是说它是作用在 KV 之内的， 与其他 KV 无关

### 5、Etcd v3 的 MVCC 实现

`etcd` 在 BoltDB 中存储的 key 是 `revision`，`value` 是 `etcd` 自己的 `key-value` 组合，也就是说 `etcd` 会在 BoltDB 中保存每个版本，从而实现多版本控制。

所以 `etcd` 在BoltDB 中查询数据时必须通过 `revision`，而客户端是通过 `key` 来查询 `value` 的，因此 `etcd` 还在内存中维护了一个 `kvindex`，用于保存 `key` 与 `revision` 之间的映射关系，也就是前文提到的 B 树索引。这样查询路径就变为：客户端提供 `key` => `etcd` 通过 `key` 在 `kvindex` 中查到对应的 `revision` => 接着通过 `revision` 在 BoltDB 查询 `value` => 最后将 `value` 返回给客户端

在内存的索引中，每个用户的原始 `key`都关联了一个 `keyIndex` 结构， 如下：

```go
type keyIndex struct {
    // 用户的原始 key
    key          []byte
    // 最近一次修改时 key 对应的 revision
    modified     revison
    // key 的生命周期，第一次创建的 key 是 generations[0], 标记为删除后再次创建则为 generations[1]
    generations  []generation
}

type generation struct {
    // key 的版本，每次创建都是从零开始，之后每次更新都自增
    ver       int64
    // key 创建时的 revision
    created   revision
    // 用于记录每次更新时的 key 对应的 revision
    revs      []revision
}

type revision struct {
    // 唯一事务ID, 全局递增且不重复，一个事务里可以对多个 key 进行更新操作，但每个 key 最多只能更新一次
    main int64
    // 标记有更新操作的 key，在每个事务内都是从零开始递增编号
    sub  int64
}
```

综上，内存 B 树维护的是 `key` 到 `keyIndex` 之间的映射关系（`kvindex`），`keyIndex`内维护的是一个 `key` 对应多个版本的 `revision` 信息，而通过 `revision` 就可以在 BoltDB 中查询到对应的 `value`了。

## 三、etcdctl 基本操作

### 1、查看版本

```shell
etcdctl version
```

### 2、写入

```shell
etcdctl put <key> <value>
```

### 3、读取

```
// 读取键和值
etcdctl get <key>
// 只读取键
etcdctl get <key> --keys-only
// 读取某些前缀的键
etcdctl get <key> --prefix --keys-only
// 只读取值
etcdctl get <key> --print-value-only
// 读取某个范围的值
etcdctl get <key1> <key2>
// 读取 /test 为前缀的 key， 并限制数量为 3
etcdctl get /test --prefix --limit=3
// 因为 etcd 集群上键值存储的每个修改都会增加 etcd 集群的全局修订版本，应用可以通过提供旧有的 etcd 修改版本来读取被替代的键。
// 读取历史版本: 修订版本号为 vision=230
etcdctl get <key> --rev=230 --prefix
// 读取大于等于键 /test/key 的 byte 值的键
etcdctl get --from-key /test/key
// 以json格式输出
etcdctl get <key> --prefix --write-out=json
// 输出详细的信息
etcdctl get <key> --prefix --write-out=fields
```

### 4、删除

```shell
// 删除一个键值对
etcdctl del <key>
// 删除键 <key> 并返回删除数以及被删除的键值对
etcdctl del --prev-kv <key>
// 删除 key1 到 key2 之间的键
etcdctl del <key1> <key2>
// 删除 /test 前缀的 key
etcdctl del /test --prefix
// 删除大于等于键 /test/key 的 byte 值的键
etcdctl del --from-key /test/key
```

### 5、监控 watch

```shell
// 监控 <key> 的变化
etcdctl watch <key>
// 监控 <key1> 到 <key2> 范围的键
etcdctl watch <key1> <key2>
// 监控以 /test 为前缀的key
etcdctl watch /test --prefix
// 监控多个键
etcdctl watch -i
etcdctl watch <key1>
etcdctl watch <key2>
// 监控 key 的历史变动, 从 vision 为 3 开始
etcdctl watch --rev=3 <key>
```

### 6、压缩修订版本

```shell
// 被压缩的历史版本不能被查询
etcdctl compact <vision>

eg
$ etcdctl compact 250
compacted revision 250

$ etcdctl get /test --prefix --rev=249
...mvcc: required revision has been compacted
```

### 7、授予租约

应用可以为 `etcd` 集群里面的键授予租约。当键被附加到租约时，它的存活时间被绑定到租约的存活时间，而租约的存活时间相应的被 `time-to-live` (TTL) 管理。在租约授予时每个租约的最小 TTL 值由应用指定。租约的实际 TTL 值是不低于最小 TTL，由 `etcd` 集群选择。一旦租约的 TTL 到期，租约就过期并且所有附带的键都将被删除。

```shell
// 授予租约， TTL 为 10s
$ etcdctl lease grant 10
lease 694d77e7d63d678f granted with TTL(10s)
// 将租约绑定到键上
etcdctl put --lease=694d77e7d63d678f <key> <value>
```

### 8、撤销租约

应用通过租约 id 可以撤销租约， 撤销租约将删除所有它附带的 key。

```shell
etcdctl lease revoke 694d77e7d63d678f
```

### 9、维持租约

租约到期时、会自动续约，此命令会阻塞命令行

```shell
etcdctl lease keep-alive 694d77e7d63d678f
```

### 10、获取租约信息

```shell
// 返回租约时间以及剩余时间
$ etcdctl lease timetolive 694d77e7d63d67a4
lease 694d77e7d63d67a4 granted with TTL(30s), remaining(27s)
// 返回租约时间以、剩余时间以及租约附带的key
$ etcdctl lease timetolive --keys 694d77e7d63d67a4
lease 694d77e7d63d67a4 granted with TTL(30s), remaining(28s), attached keys([/test/bbb /test/aaa])
```

## 四、golang 客户端简单示例

> 使用官方的 etcd/clientv3 包来连接 etcd 并进行相关操作

```go
package etcd

import (
    "context"
    "fmt"
    "github.com/coreos/etcd/clientv3"
    "github.com/coreos/etcd/clientv3/concurrency"
    "log"
    "os"
    "time"
)

var client *clientv3.Client

func init() {
    var err error
    client, err = clientv3.New(clientv3.Config{
        Endpoints:   []string{"127.0.0.1:2379"},
        DialTimeout: 5 * time.Second,
    })

    if err != nil {
        log.Fatal("init client fail")
    }
    log.Printf("connect success, %s", client.Username)
}

func GetClient() *clientv3.Client {
    return client
}

func Put(ctx context.Context, key, value string) (err error) {
    _, err = client.Put(ctx, key, value)
    if err != nil {
        log.Printf("put to etcd failed, err:%v\\\\n", err)
    }
    return
}

// 使用租约
func PutWithLease(ctx context.Context, key, value string, ttl int64) (clientv3.LeaseID, error) {
    grant, err := client.Grant(ctx, ttl)
    if err != nil {
        log.Printf("PutWithLease to etcd failed, err:%v\\\\n", err)
        return 0, err
    }
    _, err = client.Put(ctx, key, value, clientv3.WithLease(grant.ID))
    if err != nil {
        log.Printf("PutWithLease to etcd failed, err:%v\\\\n", err)
        return 0, err
    }
    return grant.ID, nil
}

func Del(ctx context.Context, key string) (err error) {
    _, err = client.Delete(ctx, key)
    if err != nil {
        log.Printf("del to etcd failed, err:%v\\\\n", err)
    }
    return
}

func Get(ctx context.Context, key string) string {
    resp, err := client.Get(ctx, key)
    if err != nil {
        log.Printf("get from etcd failed, err:%v\\\\n", err)
        return ""
    }

    for k, v := range resp.Kvs {
        log.Printf("index: %d, keyValue: %s", k, v)
    }

    return string(resp.Kvs[0].Value)
}

// 获取以key为前缀的kv
func GetWithPrefix(ctx context.Context, key string) (resp []map[string]string) {
    get, err := client.Get(ctx, key, clientv3.WithPrefix())
    if err != nil {
        log.Printf("GetWithPrefix from etcd failed, err:%v\\\\n", err)
        return resp
    }

    for _, v := range get.Kvs {
        //log.Printf("index: %d, keyValue: %s", k, v)
        m := make(map[string]string)
        m[string(v.Key)] = string(v.Value)
        resp = append(resp, m)
    }

    return resp
}

// 监控机制
func Watch(ctx context.Context, key string) {
    watch := client.Watch(ctx, key, clientv3.WithPrefix())
    for w := range watch {
        for _, ev := range w.Events {
            log.Printf("watch type: %s, k: %s, v: %s\\\\n", ev.Type, string(ev.Kv.Key), string(ev.Kv.Value))
        }
    }

}

func Clear(ctx context.Context) {
    _, err := client.Delete(ctx, "/", clientv3.WithPrefix())
    if err != nil {
        log.Printf("clear etcd fail, %v\\\\n", err)
    }
}
```

## 五、基于 Etcd 的分布式锁实践

> 基于 etcd 客户端实现分布式锁，etcd 实现分布式锁的基础是 Lease (租约) 机制、Revision（修订版本号） 机制、Prefix 机制、Watch 机制
> 
> - Lease 机制：即租约机制（TTL，Time To Live），etcd 可以为存储的 key-value 对设置租约，当租约到期，key-value 将失效删除；同时也支持续约续期（KeepAlive）
> - Revision 机制：每个 `key` 带有一个 `Revision` 属性值，Etcd 每进行一次事务对应的全局 `Revision` 值都会加一，因此每个 Key 对应的 `Revision` 属性值都是全局唯一的。通过比较 `Revision` 的大小就可以知道进行写操作的顺序。在实现分布式锁时，多个程序同时抢锁，根据 `Revision` 值大小依次获得锁，可以避免惊群效应，实现公平锁
> - Prefix 机制：即前缀机制（或目录机制）。可以根据前缀（目录）获取该目录下所有的 Key 及对应的属性（包括 Key、Value 以及 `Revision` 等）
> - Watch 机制：即监听机制，Watch 机制支持 Watch 某个固定的 Key，也支持 Watch 一个目录前缀（前缀机制），当被 Watch 的 Key 或目录发生变化，客户端将收到通知

```go
func Lock() {
    // 获取两个session模拟锁竞争
    session1, err := concurrency.NewSession(client)
    if err != nil {
        log.Printf("get session fail, %v\n", err)
    }

    session2, err := concurrency.NewSession(client)
    if err != nil {
        log.Printf("get session fail, %v\n", err)
    }

    lock := "/test/lock"
    locker1 := concurrency.NewLocker(session1, lock)
    locker2 := concurrency.NewLocker(session2, lock)

    locker1.Lock()
    log.Println("get lock for session1")

    locked := make(chan struct{})

    go func() {
        defer close(locked)
        log.Println("before locker2 lock")
        // 阻塞知道等待session1释放锁
        locker2.Lock()
        log.Println("after locker2 lock")
    }()

    time.Sleep(time.Second)
    // session1 释放锁
    locker1.Unlock()
    log.Println("session1 unlocked")

    <-locked
    log.Println("get lock for session2")

    locker2.Unlock()
    log.Println("session2 unlocked")

}
```

```bash
2021-03-23 21:40:10.184063 I | connect success,
2021-03-23 21:40:10.302807 I | watch type: PUT, k: /test/lock/694d785eff30693d, v:
2021-03-23 21:40:10.302807 I | get lock for session1
2021-03-23 21:40:10.302807 I | before locker2 lock
2021-03-23 21:40:10.303807 I | watch type: PUT, k: /test/lock/694d785eff30693f, v:
2021-03-23 21:40:11.304476 I | session1 unlocked
2021-03-23 21:40:11.304476 I | watch type: DELETE, k: /test/lock/694d785eff30693d, v:
2021-03-23 21:40:11.309478 I | after locker2 lock
2021-03-23 21:40:11.309478 I | get lock for session2
2021-03-23 21:40:11.310478 I | session2 unlocked
2021-03-23 21:40:11.310478 I | watch type: DELETE, k: /test/lock/694d785eff30693f, v:
```

## 六、基于 Etcd 的分布式系统主备选举实践

日常开发中，为了避免服务的单点故障，我们通常会采用分布式架构的形式，但在某些场景下我们希望某些功能只由特定的节点执行，这些功同时能被多个节点同时执行的话可能导致数据不一致，所以我们的系统可以通过选举出 `leader` 的方式来避免这些问题，而基于 `Ralf` 协议的 `etcd` 就是一个实现分布式系统选举可靠的中间件，与实现分布式锁类似，实现选举也要依赖于 `lease` (租约) 机制、`revision`（修订版本号） 机制、`prefix` 机制、`watch` 机制

`concurrency` 包实现选举的方法主要如下：

![https//ilolinet/2021/03/25/G8Scuy7LINbxq5mpng](https://i.loli.net/2021/03/25/G8Scuy7LINbxq5m.png)

示例中主要会使用的方法是 concurrency.NewElection, concurrency.Election.Campaign, concurrency.Election.Resign

```go
func Campaign() {
    ctx := context.TODO()
    for {
        // 为client创建session，并绑定租约，租约的ttl为5s, 如传入的ttl<0, 将使用默认的ttl（60s）
        session, err := concurrency.NewSession(client, concurrency.WithTTL(5))
        if err != nil {
            continue
        }

        // campaignPrefix 是需要监听的目录前缀，发起选举会自动在末尾补 `/`
        campaignPrefix := "/test/campaign"
        election := concurrency.NewElection(session, campaignPrefix)
        hostname, _ := os.Hostname()
        campaignId := fmt.Sprintf("%s-%d", hostname, os.Getpid())
        // 1、以 keyPrefix + leaseId 为 key, campaignId 为 value 创建键值对
        // 2、一直阻塞除非成功选举为 Leader 或者 context 过期
        // 3、成功选举为 Leader 的条件是当前 key 比 keyPrefix 目录下的其他 key 的 CreateRevision 小
        // 4、每个 key 都监听（等待）比自己 CreateRevision 小但 CreateRevision 最大的 key 被删除
        if err := election.Campaign(ctx, campaignId); err != nil {
            select {
            case <-ctx.Done():
                return
            default:
            }
            continue
        }

        Leader = campaignId
        log.Printf("leader: %s\\\\n", Leader)
        if LeaderShouldDo != nil {
            LeaderShouldDo()
        }

        select {
        case <-session.Done():
            // 放弃 Leader 的身份，重新发起新一轮选举
            election.Resign(ctx)
            session.Close()
            if QuitLeader != nil {
                QuitLeader()
            }
            log.Printf("%s quit leader\\\\n", Leader)
            Leader = ""
            continue
        case <-ctx.Done():
            session.Close()
            if QuitLeader != nil {
                QuitLeader()
            }
            log.Printf("%s quit leader\\\\n", Leader)
            Leader = ""
            return
        }
    }
}
```

假设系统中有三个节点，如图，3 节点监听 2 节点，2 节点监听 1 节点，1 节点是 Leader

1）假如 2 节点租约过期了，那么 3 节点将监听 1 节点

2） 假如 1 节点的租约过期了，那么 2 节点成为 Leader

![https//ilolinet/2021/03/25/GfwiCYLs3xZgXDzpng](https://i.loli.net/2021/03/25/GfwiCYLs3xZgXDz.png)

**启动三个 client后依次停止client1、client2、client3**

**client1**

```bash
2021-03-25 15:30:44.036311 I | connect success,
2021-03-25 15:30:44.140412 I | watch type: PUT, k: /test/campaign/694d786823c4fd87, v: chenhongbin-25712
2021-03-25 15:30:44.148287 I | leader: chenhongbin-25712
2021-03-25 15:30:50.267831 I | watch type: PUT, k: /test/campaign/694d786823c4fd8b, v: chenhongbin-27112
2021-03-25 15:30:55.430359 I | watch type: PUT, k: /test/campaign/694d786823c4fd8f, v: chenhongbin-24684
2021-03-25 15:31:15.770408 I | chenhongbin-25712 quit leader
2021-03-25 15:31:15.770913 I | watch type: DELETE, k: /test/campaign/694d786823c4fd87, v:
```

**client2**

```bash
2021-03-25 15:30:50.171378 I | connect success,
2021-03-25 15:30:50.268338 I | watch type: PUT, k: /test/campaign/694d786823c4fd8b, v: chenhongbin-27112
2021-03-25 15:30:55.430359 I | watch type: PUT, k: /test/campaign/694d786823c4fd8f, v: chenhongbin-24684
2021-03-25 15:31:15.770408 I | watch type: DELETE, k: /test/campaign/694d786823c4fd87, v:
2021-03-25 15:31:15.775075 I | leader: chenhongbin-27112
2021-03-25 15:31:18.673096 I | chenhongbin-27112 quit leader
2021-03-25 15:31:18.673096 I | watch type: DELETE, k: /test/campaign/694d786823c4fd8b, v:
```

**client3**

```bash
2021-03-25 15:30:55.373742 I | connect success,
2021-03-25 15:30:55.430359 I | watch type: PUT, k: /test/campaign/694d786823c4fd8f, v: chenhongbin-24684
2021-03-25 15:31:15.770408 I | watch type: DELETE, k: /test/campaign/694d786823c4fd87, v:
2021-03-25 15:31:18.673096 I | watch type: DELETE, k: /test/campaign/694d786823c4fd8b, v:
2021-03-25 15:31:18.678095 I | leader: chenhongbin-24684
2021-03-25 15:31:32.872987 I | chenhongbin-24684 quit leader
2021-03-25 15:31:32.872987 I | watch type: DELETE, k: /test/campaign/694d786823c4fd8f, v:
```

**启动三个 client后依次停止 etcd 服务模拟网络异常，等待租约过期重新启动服务**

**client1**

```bash
2021-03-25 15:34:10.450527 I | connect success,
2021-03-25 15:34:10.547423 I | watch type: PUT, k: /test/campaign/694d786823c4fd98, v: chenhongbin-24980
2021-03-25 15:34:10.549421 I | leader: chenhongbin-24980
2021-03-25 15:34:14.898319 I | watch type: PUT, k: /test/campaign/694d786823c4fd9c, v: chenhongbin-21324
2021-03-25 15:34:20.017620 I | watch type: PUT, k: /test/campaign/694d786823c4fda0, v: chenhongbin-13260
2021-03-25 15:34:54.375748 I | watch type: DELETE, k: /test/campaign/694d786823c4fd98, v:
2021-03-25 15:34:54.377772 I | chenhongbin-24980 quit leader
2021-03-25 15:34:54.379744 I | watch type: PUT, k: /test/campaign/694d78684eb7fb06, v: chenhongbin-24980
2021-03-25 15:34:54.530581 I | watch type: DELETE, k: /test/campaign/694d786823c4fd9c, v:
2021-03-25 15:34:54.547818 I | watch type: DELETE, k: /test/campaign/694d786823c4fda0, v:
2021-03-25 15:34:54.559815 I | watch type: PUT, k: /test/campaign/694d78684eb7fb0f, v: chenhongbin-21324
2021-03-25 15:34:54.559815 I | leader: chenhongbin-24980
2021-03-25 15:34:54.562814 I | watch type: PUT, k: /test/campaign/694d78684eb7fb14, v: chenhongbin-13260
```

**client2**

```bash
2021-03-25 15:34:14.798817 I | connect success,
2021-03-25 15:34:14.898319 I | watch type: PUT, k: /test/campaign/694d786823c4fd9c, v: chenhongbin-21324
2021-03-25 15:34:20.017620 I | watch type: PUT, k: /test/campaign/694d786823c4fda0, v: chenhongbin-13260
2021-03-25 15:34:54.381756 I | watch type: DELETE, k: /test/campaign/694d786823c4fd98, v:
2021-03-25 15:34:54.381756 I | watch type: PUT, k: /test/campaign/694d78684eb7fb06, v: chenhongbin-24980
2021-03-25 15:34:54.483051 I | leader: chenhongbin-21324
2021-03-25 15:34:54.534824 I | watch type: DELETE, k: /test/campaign/694d786823c4fd9c, v:
2021-03-25 15:34:54.544816 I | chenhongbin-21324 quit leader
2021-03-25 15:34:54.547818 I | watch type: DELETE, k: /test/campaign/694d786823c4fda0, v:
2021-03-25 15:34:54.559815 I | watch type: PUT, k: /test/campaign/694d78684eb7fb0f, v: chenhongbin-21324
2021-03-25 15:34:54.562814 I | watch type: PUT, k: /test/campaign/694d78684eb7fb14, v: chenhongbin-13260
```

**client3**

```bash
2021-03-25 15:34:19.897700 I | connect success,
2021-03-25 15:34:20.017620 I | watch type: PUT, k: /test/campaign/694d786823c4fda0, v: chenhongbin-13260
2021-03-25 15:34:54.375748 I | watch type: DELETE, k: /test/campaign/694d786823c4fd98, v:
2021-03-25 15:34:54.380743 I | watch type: PUT, k: /test/campaign/694d78684eb7fb06, v: chenhong
bin-24980
2021-03-25 15:34:54.530581 I | watch type: DELETE, k: /test/campaign/694d786823c4fd9c, v:
2021-03-25 15:34:54.543815 I | leader: chenhongbin-13260
2021-03-25 15:34:54.547818 I | watch type: DELETE, k: /test/campaign/694d786823c4fda0, v:
2021-03-25 15:34:54.559815 I | watch type: PUT, k: /test/campaign/694d78684eb7fb0f, v: chenhongbin-21324
2021-03-25 15:34:54.559815 I | chenhongbin-13260 quit leader
2021-03-25 15:34:54.562814 I | watch type: PUT, k: /test/campaign/694d78684eb7fb14, v: chenhongbin-13260
```

## 七、基于 Etcd 服务发现实践

所谓服务发现就是了解集群中是否有进程在监听 UDP 或者 TCP 端口，并且通过名字就可以进行查找和链接，因为在微服务中，一个服务重启后其 `endpoint` 极有可能发生变化

满足服务发现的条件：

1. 服务注册：强一致性、高可用的服务存储目录，基于 Raft 算法的 `etcd` 天生就是这样一个强一致性、高可用的服务存储目录，以此来实现服务注册
2. 健康检查：租约机制可用来实现服务健康状况检测，定时续约以达到监控健康状态的效果
3. 服务发现：watch 机制，可以及时发现注册服的更新、删除

![https//ilolinet/2021/06/04/EGBYDXaNt3ClzQ1png](https://i.loli.net/2021/06/04/EGBYDXaNt3ClzQ1.png)

```go
package etcd

import (
    "context"
    "fmt"
    "github.com/coreos/etcd/clientv3"
    "log"
    "sync"
)

var DiscoveryRoot = "/discovery"

type serviceRegisterDiscovery struct {
    mux              sync.Mutex
    ctx              context.Context
    leaseIdMap       map[string]clientv3.LeaseID
    serviceName      string
    keepAliveChanMap map[string]<-chan *clientv3.LeaseKeepAliveResponse
}

func NewServiceRegisterDiscovery(ctx context.Context, serviceName str    ing) *serviceRegisterDiscovery {
    leaseIdMap, keepAliveChanMap := make(map[string]clientv3.LeaseID), make(map[string]<-chan *clientv3.LeaseKeepAliveResponse)
    return &serviceRegisterDiscovery{
        leaseIdMap:       leaseIdMap,
        serviceName:      serviceName,
        keepAliveChanMap: keepAliveChanMap,
        ctx:              ctx,
    }
}

func (s *serviceRegisterDiscovery) key(endpoint string) string {
    return fmt.Sprintf("%s/%s/%s", DiscoveryRoot, s.serviceName, endpoint)
}

func (s *serviceRegisterDiscovery) Register(endpoint string, lease int64) error {
    s.mux.Lock()
    defer s.mux.Unlock()
    key := s.key(endpoint)
    leaseId, err := PutWithLease(s.ctx, key, endpoint, lease)
    if err != nil {
        return err
    }

    // 自动续约
    keepAliveChan, err := client.KeepAlive(s.ctx, leaseId)
    if err != nil {
        return err
    }

    s.leaseIdMap[key], s.keepAliveChanMap[key] = leaseId, keepAliveChan
    return nil
}

func (s *serviceRegisterDiscovery) ListenLeaseChan() {
    for len(s.keepAliveChanMap) == 0 {
    }

    var length = len(s.keepAliveChanMap)
    for length != 0 {
        var i = 0
        var wg sync.WaitGroup
        wg.Add(length)
        for key, ch := range s.keepAliveChanMap {
            i++
            go func() {
                for c := range ch {
                    log.Println(key, "keepalive successful", c.Revision)
                }
            }()
            if i >= length{
                break
            }
            wg.Done()
        }
        wg.Wait()
        length = len(s.keepAliveChanMap)
    }
    log.Printf("all lease of %s is canceled", s.serviceName)
}

func (s *serviceRegisterDiscovery) CloseService() error {
    var err error
    for _, leasId := range s.leaseIdMap {
        _, err = client.Revoke(s.ctx, leasId)
    }
    s.leaseIdMap = nil
    return err
}

func (s *serviceRegisterDiscovery) CancelEndpoint(endpoint string) bool {
    leaseId, ok := s.leaseIdMap[s.key(endpoint)]
    if !ok {
        return false
    }
    _, err := client.Revoke(s.ctx, leaseId)
    delete(s.leaseIdMap, s.key(endpoint))
    delete(s.keepAliveChanMap, s.key(endpoint))
    if err != nil {
        return false
    }
    return true
}

func (s *serviceRegisterDiscovery) Discovery() []string {
    var services []string
    prefix := GetWithPrefix(s.ctx, s.key(""))
    for _, p := range prefix {
        for _, v := range p {
            services = append(services, v)
        }
    }
    return services
}
```

```go
package etcd

import (
    "fmt"
    "log"
    "math/rand"
    "testing"
    "time"
)

func TestNewServiceRegisterDiscovery(t *testing.T) {
    svcRegisterDiscovery := NewServiceRegisterDiscovery(ctx, "web")
    defer svcRegisterDiscovery.CloseService()
    go func() {
        for {
            discovery := svcRegisterDiscovery.Discovery()
            log.Println(discovery)
            time.Sleep(time.Second * 3)
        }
    }()

    go func() {
        for i := 0; ; i++ {
            endpoint := fmt.Sprintf("127.0.0.1:909%d", i)
            err := svcRegisterDiscovery.Register(endpoint, 5)
            if err != nil {
                log.Printf("err: %v", err)
            }
            if i > 5 {
                delEndpoint := fmt.Sprintf("127.0.0.1:909%d", rand.Intn(i))
                for !svcRegisterDiscovery.CancelEndpoint(delEndpoint) {
                    delEndpoint = fmt.Sprintf("127.0.0.1:909%d", rand.Intn(i))
                }
            }
            time.Sleep(time.Second * 3)
        }
    }()

    go svcRegisterDiscovery.ListenLeaseChan()
    select {}

}
```

## 八、基于 etcd 的共享配置中心实践

应用启动的时候拉取某个特定目录下的存储的配置信息，同时注册一个 watcher 并等待更新，etcd 会实时通知应用更新配置

```go
package etcd

import (
    "context"
    "encoding/json"
    "flag"
    "fmt"
    "github.com/coreos/etcd/mvcc/mvccpb"
    "log"
)

type config struct {
    server
    database
}

type server struct {
    Addr  string `json:"addr"`
    Port  string `json:"port"`
    Name  string `json:"app"`
    Token string `json:"token"`
}

type database struct {
    Type     string `json:"type"`
    Url      string `json:"url"`
    User     string `json:"user"`
    Password string `json:"password"`
}

var (
    // /config/appName/profile
    configKeyFmt = "/config/%s/%s"
    Config       config
    Env          = flag.String("profile", "test", "use profile for config")
)

func configPath(appName string) string {
    return fmt.Sprintf(configKeyFmt, appName, *Env)
}

func InitConfig(appName string) {
    path := configPath(appName)
    resp, err := client.Get(context.Background(), path)
    if err != nil {
        log.Fatalf("get config from etcd fail, err: %v", err)
    }

    if err := json.Unmarshal(resp.Kvs[0].Value, &Config); err != nil {
        log.Fatalf("unmarshal config info fail, err: %v", err)
    }

    log.Println("init config info success...")

    go watchConfig(path)
}

func watchConfig(path string) {
    watch := client.Watch(context.Background(), path)
    for w := range watch {
        log.Printf("w: %v\\\\n", w)
        for _, e := range w.Events {
            switch e.Type {
            case mvccpb.PUT:
                if err := json.Unmarshal(e.Kv.Value, &Config); err != nil {
                    log.Panicf("format err of %s, err: %v", string(e.Kv.Value), err)
                }
                log.Println("update config success...")
            case mvccpb.DELETE:
                log.Panicf("config info is deleted")
            }
        }
    }
}
```

```go
package etcd

import (
    "context"
    "encoding/json"
    "testing"
)

func TestInitConfig(t *testing.T) {
    defer Clear(context.Background())
    c := config{server{
        Addr:  "0.0.0.0",
        Port:  "8080",
        Name:  "test",
        Token: "test",
    }, database{
        Type:     "mysql",
        Url:      "mysql://127.0.0.1:3306?test",
        User:     "root",
        Password: "123456",
    }}
    marshal, err := json.Marshal(&c)
    if err != nil {
        t.Fatalf("err: %v", err)
    }

    appName := "test"
    if err := Put(context.Background(), configPath(appName), string(marshal)); err != nil {
        t.Fatalf("err: %v", err)
    }

    InitConfig(appName)

    if err := Put(context.Background(), configPath(appName), string(marshal)); err != nil {
        t.Fatalf("err: %v", err)
    }

    select {}
}
```

## 九、基于 etcd 的分布式通知与协调实践

### 1、 生命探针检测

需要依赖 `etcd` 的 `watch`机制和 `lease`机制，检测系统和被检测系统通过在 `etcd` 上的某个目录进行关联，从而实现解耦

![https//ilolinet/2021/06/04/xTAZ2OCutDdNYIvpng](https://i.loli.net/2021/06/04/xTAZ2OCutDdNYIv.png)

```go
package etcd

import (
    "context"
    "fmt"
    "github.com/coreos/etcd/clientv3"
    "log"
    "time"
)

var pathTemplate = "/test/probe/%s"

type probe struct {
    ctx      context.Context
    workName string
    ttl      int64
    leaseId  clientv3.LeaseID
}

func NewProbe(ctx context.Context, name string, ttl int64) *probe {
    p := &probe{
        ctx:      ctx,
        workName: name,
        ttl:      ttl,
    }
    leaseId, _ := PutWithLease(ctx, p.probePath(), p.probePath(), ttl)
    p.leaseId = leaseId
    return p
}

func (p *probe) probePath() string {
    return fmt.Sprintf(pathTemplate, p.workName)
}

func (p *probe) postHealth() error {
    _, err := client.KeepAliveOnce(p.ctx, p.leaseId)
    if err != nil {
        log.Printf("keepalive fail, err: %v\\\\n", err)
    }
    return err
}

func (p *probe) watchLifeProbe() {
    for Get(p.ctx, p.probePath()) != "" {
        log.Printf("%s is healthy\\\\n", p.workName)
        time.Sleep(time.Duration(time.Second.Nanoseconds() * (p.ttl - 1)))
    }
    log.Printf("%s is unhealthy\\\\n", p.workName)
}
```

**测试代码**

```go
func TestNewProbe(t *testing.T) {
    probe := NewProbe(context.Background(), "test", 5)
    go probe.watchLifeProbe()
    go func() {
        for {
            if err := probe.postHealth(); err != nil {
                log.Println("post health fail, err: ", err)
                return
            }
            log.Println("post health success ")
            time.Sleep(time.Second * 4)
        }
    }()
    select {}
}
```

### 2、 完成系统调度

依赖于 `watch` 机制，控制台负责特定目录的更新，推送系统监听目录的更新

![https//ilolinet/2021/06/04/HsmdchwnMPGzZkqpng](https://i.loli.net/2021/06/04/HsmdchwnMPGzZkq.png)

```go
package etcd

import (
    "context"
    "fmt"
    "github.com/coreos/etcd/mvcc/mvccpb"
    "log"
)

type TaskState uint8

const (
    Scheduled TaskState = iota + 1
    Starting
    Running
    Finished
)

var stateList = [...]string{"", "Scheduled", "Starting", "Running", "Finished"}

func (t TaskState) String() string {
    return stateList[t]
}

func ParseTaskState(state string) TaskState {
    for i, s := range stateList {
        if s == state {
            return TaskState(i)
        }
    }
    return 0
}

type taskScheduler struct {
    ctx   context.Context
    Name  string    `json:"name"`
    State TaskState `json:"state"`
}

func NewTaskScheduler(ctx context.Context, taskName string) *taskScheduler {
    return &taskScheduler{
        ctx:   ctx,
        Name:  taskName,
        State: 0,
    }
}

func (t *taskScheduler) Key() string {
    return fmt.Sprintf("/scheduler/task/%s", t.Name)
}

func (t *taskScheduler) Notify(state TaskState) error {
    if state < Scheduled || state > Finished {
        return fmt.Errorf("error state: %d", state)
    }
    t.State = state
    log.Printf("console notify task %s to work\\\\n", t.Name)
    return Put(t.ctx, t.Key(), t.State.String())
}

func (t *taskScheduler) WatchNotify() {
    watch := client.Watch(t.ctx, t.Key())
    for w := range watch {
        for _, e := range w.Events {
            switch e.Type {
            case mvccpb.PUT:
                t.Work(string(e.Kv.Value))
            case mvccpb.DELETE:
                t.close()
            }
        }
    }
}

func (t *taskScheduler) Work(state string) {
    t.State = ParseTaskState(state)
    switch state {
    case Scheduled.String():
        log.Printf("task %s is scheduled\\\\n", t.Name)
    case Starting.String():
        log.Printf("task %s begin  working\\\\n", t.Name)
    case Running.String():
        log.Printf("task %s is running\\\\n", t.Name)
    case Finished.String():
        log.Printf("task %s finnish working\\\\n", t.Name)
        Del(t.ctx, t.Key())
    }
}

func (t *taskScheduler) close() {
    t.Name = ""
    t.State = 0
}
```

测试代码

```go
package etcd

import (
    "context"
    "testing"
    "time"
)

func TestTaskScheduler(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    defer client.Close()
    defer Clear(ctx)

    scheduler := NewTaskScheduler(ctx, "taskName")
    go scheduler.WatchNotify()
    scheduler.Notify(Scheduled)
    time.Sleep(time.Second)
    scheduler.Notify(Starting)
    time.Sleep(time.Second)
    scheduler.Notify(Running)
    time.Sleep(time.Second)
    scheduler.Notify(Finished)
}
```

### 3、 完成工作汇报

依赖于 `watch` 机制，启动子任务时建立一个临时目录，子任务可以通过更新这个临时目录的值来汇报自己的工作进度

![https//ilolinet/2021/06/04/WOoDdEi3GRQsA2ppng](https://i.loli.net/2021/06/04/WOoDdEi3GRQsA2p.png)

```go
package etcd

import (
    "context"
    "fmt"
    "github.com/coreos/etcd/mvcc/mvccpb"
    "log"
    "strconv"
    "strings"
)

type workReporter struct {
    ctx      context.Context `json:"ctx"`
    Name     string          `json:"name"`
    progress int             `json:"progress"`
}

func NewWorkReporter(ctx context.Context, name string) *workReporter {
    return &workReporter{
        ctx:      ctx,
        Name:     name,
        progress: 0,
    }
}

func (w *workReporter) Key() string {
    return fmt.Sprintf("/work/%s", w.Name)
}

func (w *workReporter) FmtProgress() string {
    return fmt.Sprintf("%d", w.progress) + "%"
}

func (w *workReporter) ParseProgress(progress string) (int, error) {
    if !strings.HasSuffix(progress, "%") {
        panic("progress isn't end of %")
    }
    parse, err := strconv.Atoi(progress[0 : len(progress)-1])
    return parse, err
}

func (w *workReporter) Report(progress int) error {
    if progress < 0 || progress > 100 {
        return fmt.Errorf("the progress of %s is error", w.Name)
    }
    w.progress = progress
    return Put(w.ctx, w.Key(), w.FmtProgress())
}

func (w *workReporter) WatchNotify() {
    watchChan := client.Watch(w.ctx, w.Key())
    for watch := range watchChan {
        for _, e := range watch.Events {
            switch e.Type {
            case mvccpb.PUT:
                progress := string(e.Kv.Value)
                parseInt, err := w.ParseProgress(progress)
                if err != nil {
                    log.Printf("parseInt paogress of %s fail, progress: %s\\\\n", w.Name, progress)
                    break
                }
                log.Printf("the paogress of %s is %s\\\\n", w.Name, progress)
                w.progress = parseInt
            }
        }
    }
}
```

测试代码

```go
package etcd

import (
    "context"
    "math/rand"
    "testing"
)

func TestWorkReporter(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    defer client.Close()
    defer Clear(ctx)
    reporter := NewWorkReporter(ctx, "workReporter")
    go reporter.WatchNotify()
    for i := 0; i < 100; {
        i += rand.Intn(10)
        if i > 100 {
            i = 100
        }
        err := reporter.Report(i)
        if err != nil {
            t.Fatalf("err: %v\\n", err)
        }
    }
}
```
