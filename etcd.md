# ETCD

高可用分布式键值数据库，机器间通过raft算法通信，具有强一致性

## 使用场景

- 键值对存储
- 服务注册与发现
  - 强一致性、高可用的服务存储目录（通过raft算法支持）
  - 注册服务和服务健康状况的机制。 用户可以在 etcd 中注册服务，并且对注册的服务配置 key TTL，定时保持服务的心跳以达到监控健康状态的效果
  - 查找和连接服务的机制。通过在 etcd 指定的主题下注册的服务也能在对应的主题下查找到。可以在每个服务机器上都部署一个 Proxy 模式的 etcd，这样就可以确保访问 etcd 集群的服务都能够互相连接
- 消息发布与订阅
  - etcd集中管理配置信息。应用在启动的时候主动从etcd获取一次配置信息，同时在etcd节点上注册一个Watcher并等待，以后每次配置有更新的时候，etcd都会实时通知订阅者
  - 分布式搜索服务中，索引的元信息和服务器集群机器的节点状态存放在etcd中，供各个客户端订阅使用。使用etcd的key TTL功能可以确保机器状态是实时更新的。
  - 分布式日志收集系统。收集器通常是按照应用（或主题）来分配收集任务单元，因此可以在etcd上创建一个以应用（主题）命名的目录P，并将这个应用（主题相关）的所有机器ip，以子目录的形式存储到目录P上，然后设置一个 etcd 递归的Watcher，递归式的监控应用（主题）目录下所有信息的变动。这样就实现了机器IP（消息）变动的时候，能够实时通知到收集器调整任务分配。
  - 动态自动获取信息与人工干预修改信息请求内容。通常是暴露出接口，例如JMX接口，来获取一些运行时的信息。引入etcd之后，就不用自己实现一套方案了，只要将这些信息存放到指定的etcd目录中即可，etcd的这些目录就可以通过HTTP的接口在外部访问。
- 分布式通知与协调
  - 类似消息发布和订阅，用到了etcd中的Watcher机制，不同系统都在etcd上对同一个目录进行注册，同时设置Watcher观测该目录的变化（如果对子目录的变化也有需要，可以设置递归模式），当某个系统更新了etcd的目录，那么设置了Watcher的系统就会收到通知，并作出相应处理。
  - 低耦合的心跳检测
  - 系统调度。管理人员修改etcd上某些目录节点的状态，而etcd就把这些变化通知给注册了Watcher的客户端，完成调度
  - 工作汇报。子任务启动后，到etcd来注册一个临时工作目录，并且定时将自己的进度进行汇报（将进度写入到这个临时目录）
- 分布式锁
  - 分布式环境下不仅需要保证进程可见，还需要考虑进程与锁之间的网络问题
  - 锁服务有两种使用方式，一是保持独占（所有获取锁的用户最终只有一个可以得到），二是控制时序（所有想要获得锁的用户都会被安排执行，但是获得锁的顺序也是全局唯一的，同时决定了执行顺序）

## 安装使用

### 概念词汇表

- Raft：etcd所采用的保证分布式系统强一致性的算法
- Node：一个Raft状态机实例
- Member： 一个etcd实例。它管理着一个Node，并且可以为客户端请求提供服务
- Cluster：由多个Member构成可以协同工作的etcd集群
- Peer：对同一个etcd集群中另外一个Member的称呼
- Client： 向etcd集群发送HTTP请求的客户端
- WAL：预写式日志，etcd用于持久化存储的日志格式
- snapshot：etcd防止WAL文件过多而设置的快照，存储etcd数据状态
- Proxy：etcd的一种模式，为etcd集群提供反向代理服务
- Leader：Raft算法中通过竞选而产生的处理所有数据提交的节点
- Follower：竞选失败的节点作为Raft中的从属节点，为算法提供强一致性保证
- Candidate：当Follower超过一定时间接收不到Leader的心跳时转变为Candidate开始竞选
- Term：某个节点成为Leader到下一次竞选时间，称为一个Term
- Index：数据项编号。Raft中通过Term和Index来定位数据

### 安装

确认Go环境可用后：

```shell
git clone https://github.com/etcd-io/etcd.git
cd etcd
./build

# 执行测试命令，确保 etcd 编译安装成功：
$ ./etcdctl version
```

### docker+etcd集群启动

```shell
# 这个脚本是三台机器启动三个
REGISTRY=bitnami/etcd

# For each machine
ETCD_VERSION=3.4.7
TOKEN=my-etcd-token
CLUSTER_STATE=new
NAME_1=etcd-node-0
NAME_2=etcd-node-1
NAME_3=etcd-node-2
HOST_1= 192.168.202.128
HOST_2= 192.168.202.129
HOST_3= 192.168.202.130
CLUSTER=${NAME_1}=http://${HOST_1}:2380,${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380
DATA_DIR=/var/lib/etcd

# For node 1
THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --volume=${DATA_DIR}:/etcd-data \
  --name etcd ${REGISTRY}:${ETCD_VERSION} \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name ${THIS_NAME} \
  --initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://0.0.0.0:2380 \
  --advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://0.0.0.0:2379 \
  --initial-cluster ${CLUSTER} \
  --initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# 省略node2 and node3
```

当不知道集群成员的ip时，可以使用动态发现

## etcd安全

etcd 支持通过 TLS 协议进行的加密通信。TLS 通道可用于对等体之间的加密内部群集通信以及加密的客户端流量

HTTP 通信，所有信息明文传播，带来了三大风险：

- 窃听风险（eavesdropping）：第三方可以获知通信内容
- 篡改风险（tampering）：第三方可以修改通信内容
- 冒充风险（pretending）：第三方可以冒充他人身份参与通信

SSL/TLS 协议是为了解决这三大风险而设计的，希望达到：

- 所有信息都是加密传播，第三方无法窃听
- 具有校验机制，一旦被篡改，通信双方会立刻发现
- 配备身份证书，防止身份被冒充。

## 租约

可以为etcd集群里的键授予租约，此时键的存活时间被绑定到租约的存活时间，一旦租约到期，所有附带的键都被删除

```shell
# 授予租约，TTL为100秒
$ etcdctl lease grant 100

# 附加键 foo 到租约694d71ddacfda227
$ etcdctl put --lease=694d71ddacfda227 foo10 bar

# 撤销租约。撤销租约将删除所有它附带的 key。
$ etcdctl lease revoke 694d71ddacfda227

# 刷新租期，通过刷新其TTL来保持租约活着。
$ etcdctl lease keep-alive 694d71ddacfda227

# 查询租期，获取有关租赁信息以及哪些 key 使用了租赁信息：
$ etcdctl lease timetolive 694d71ddacfda22c
$ etcdctl lease timetolive --keys 694d71ddacfda22c
```
