# kubernets

## docker

- 核心原理就是为待创建的用户进程：
  - 基于namespace做隔离，相比于虚拟机隔离不彻底，比如时间
  - 基于Cgroups做限制，限制一个进程能使用的资源上限，包括cpu、内存、磁盘、网络带宽等等
    - linux将这个接口暴露在文件系统下，/sys/fs/cgroup
    - 在容器里执行top将显示宿主机的cpu和内存数据，因为存当前内核运行状态的/proc文件系统并不了解用户给容器做了怎样的限制。
  - 切换进程的根目录
- linux的文件、配置、目录与内核是分开存放的，开机启动时才加载指定版本的内核。所有容器共享内核。
- 镜像打包了整个操作系统的文件和目录，包括了应用完整的执行环境。
- 用户制作镜像的每一步操作都生成一个层，通过联合文件系统实现。
  - 镜像自下而上包括：只读层、init层、可读写层。
    - 在容器里的修改以增量的方式出现在可读写层，如果要保存使用docker commit和push。
    - 如果要删除只读层的文件，会在可读写层创建一个遮挡
    - 启动时可能要写入一些特定值比如hostname，不希望commit的时候一起提交
- Dockerfile
  - 使用一些标准原语描述所要的镜像
  - 每个原语执行后都会生成一个对应的镜像层，即使本身没有明显地修改文件的操作。
- exec实现原理：通过/proc/{{进程号}}/ns可以获取其所有的namespace，这意味着可以加入到一个已有的namespace中
- 挂载本质上是一个inode的替换，挂载之后commit，镜像里对应位置会多一个空的文件夹

## k8s

- 控制节点由三个独立组件组合而成：负责api服务的kube-apiserver、负责调度的kube-scheduler、负责容器编排的kube-controller-manager，持久化数据由apiserver处理后保存在etcd
- 计算节点上最核心的部分，则kubelet组件，负责和容器运行时交互；通过gRPC同Device Plugin交互管理GPU等宿主机物理设备；调用网络插件和存储插件为容器配置网络和持久化存储
- 运行在大规模集群中的各种任务之间，实际上存在着各种各样的关系。这些关系的处理，才是作业编排和管理系统最困难的地方。
  - 紧密交互的容器划分成一个pod，pod内共享一个network namespace、同一组数据卷
  - 给pod绑定一个service服务，作为pod的代理入口，对外暴露一个固定网络地址。
  - secret是保存在etcd中的kv键值对，用来处理授权关系
  - job(一次性运行的pod)、daemonset(必须且只能运行一个的守护进程服务)、cronjob(定时任务)

## k8s部署

- 一种思路是用容器部署k8s，但问题在于容器化kubelet较为困难，kubeadm选择了折中的方案，直接运行kubelet，其他的组件容器化部署。
- kubeadm init（检查是否可用来部署；生成证书和目录；为其它组件生成配置文件；为master和etcd组件生成配置文件并用static pod的方式拉起；安装默认插件，比如用来提供服务发现的kube-proxy以及dns插件）但etcd、master等都是单点的，不适合生产环境
- `kubectl get nodes`查看节点列表
- `kubectl describe node master`查看"master"节点的详细信息、状态和事件，重要操作都记录在事件中
- `kubectl get pods -n kube-system`查看节点各个系统pod的状态，kube-system是预留的系统pod的工作空间，-l则是获取匹配label的pod
- `kubectl apply -f https://git.io/weave-kube-1.6`部署网络插件
  - `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml`部署可视化插件，更新也应该用这样的形式，这就是声明式api，删除的话改成delete
- `kubectl exec -it xxx -- /bin/bash`进入这个pod
- 一般推荐用配置文件启动容器`kubectl create -f *.yaml`
- 节点可以添加taint来打上“污点”阻止pod运行，pod可以声明tolerations字段来容忍指定的键值对
- metadata是api对象的标识，最经常用到的是labels键值对，用来过滤关心的被控制对象
- annotation与label层级格式相同，用来携带kv格式的内部信息(组件本身关心而用户不关心)
- spec用来存放对象独有的定义，描述要表达的功能
- 哪怕之部署一个节点，也更适合使用replicas=1的deployment(应该是为了重启等策略)

## 编排与作业管理

### pod

- pod是 Kubernetes 项目的原子调度单位，对应操作系统“进程组”的概念，是一组共享了某些资源的容器(network+volume)
- Kubernetes 项目的调度器，是统一按照 Pod 而非容器的资源需求进行计算的，防止某些容器必须一起但是资源又不足产生死锁
- pod的实现需要一个中间容器，infra将永远被第一个创建，永远处于暂停状态。用户创建的容器会加入到infra的network namespace
- 在 Pod 中，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动(用户容器之间是无序的)。
- 调度、网络、存储、安全，各种namespace相关的属性基本上都是pod级别的
- nodeSelector将pod和node进行绑定，只能运行在携带有规定标签的节点上，否则调度失败
- nodename：一旦赋值就会认为已经经过调度，一般由调度器设置。用户如果设置会骗过调度器。
- hostaliases：定义了pod的hosts文件，如果直接修改hosts文件，删除重建后会覆盖修改。
- ImagePullPolicy：镜像拉取策略，always/never/ifnotpresent
- lifecycle：钩子，postStart(容器启动后立刻执行)/preStop(容器被杀死之前)
- pod状态：
  - pending：已经创建并保存在etcd但是不能顺利创建(比如调度不成功)
  - running：已经调度成功跟具体节点绑定，容器都创建成功，并且至少有一个运行中
  - succeded：所有容器都运行完毕并已经退出
  - failed：pod里至少有一个容器不正常退出
  - unknow：状态不能持续被kubelet汇报给api-server，可能是通信问题
  - 细分出conditions：podScheduled、Ready（pod不仅running，而且可以提供服务了）、Initialized、Unschedulable
- porjected volume投射数据卷，为容器提供预先定义好的数据，比如secret
- 配置文件可以放在configmap
- 创建一个pod的流程：提交到apiserver，controller-manager创建一个资源对象，将pod配置信息存在etcd中心，接着进行调度预选和调优，kubelet根据资源配置单运行pod，运行成功后返回给scheduler，将pod运行状态存到etcd
- 对 Pod 的健康检查可以通过两类探针来检查：LivenessProbe（判断存活） 和ReadinessProbe（判断启动完成）
  - startupProbe 探针：启动检查机制，应用一些启动缓慢的业务，避免业务长时间启动被之前两种探针kill掉

### 控制器

- 所有的控制器统一放在/pkg/controller目录，都遵循一个控制循环(调谐循环)，检查实际状态是否满足期望状态，不满足就调整为期望状态。实际状态来自集群本身，期望状态来自用户提交的yaml文件
- deployment(缩写是deploy)
  - deployment中的template字段定义其实就是一个标准的pod对象定义，所有其管理的实例都根据template创建，这就是pod模板
  - 类似的控制器都是：上半部分的控制器定义和期望状态+下半部分被控制对象模板
  - deployment提供水平扩展/收缩的功能，可以通过滚动更新方式升级，如果新pod有问题，那么滚动更新立刻就停止了，然后开发和运维介入
  - deployment操纵的其实算是replicaset对象(副本数目定义+pod模板)，replicaset的名字由deployment名字+随机字符串组成
  - `kubectl create -f xxx-deployment.yaml --record`record可以记录操作执行的命令
  - `kubectl get deployments`查看deployment状态，`kubectl get rs`查看replicaset状态，deployment只是多了一个up-to-date这个跟版本有关的字段
  - `kubectl rollout status xxx`实时查看状态变化 `kubectl rollout history xxx`查看历史版本 `kubectl rollout undo xxx`回滚到上一版本，加上--to-revision=x回到版本x `kubectl rollout pause xxx`pause暂停，resume恢复，期间的修改只触发一次滚动更新
- statefulset，对有状态应用，抽象出拓扑状态和存储状态。记录并在重新创建时恢复这些状态
  - 拓扑状态
    - (访问service可以通过虚ip或者dns方式，dns可以用headless service，记录被代理pod的ip)
    - 有状态的应该用dns或者hostname，而不是ip访问
  - 存储状态
    - 定义一个PVC生命想要的Volume属性，在pod中生命使用这个pvc。pv由运维人员绑定具体的实现
    - kubernetes通过headless service为这些有编号的pod生成同样编号的dns记录，分配并创建一个同样编号的pvc
    - pod删除后pvc和pv并不删除，已经写入的数据也会存在远程存储服务里
  - statefulset其实是对现有运维业务的容器化抽象，涉及到升级、版本管理等工程化的能力才更体现k8s的好处。`kubectl patch ...`以补丁的方式升级。修改partion的值为2，则更新时序号小于2的pod不会更新，即使重启。
- `kubectl set image xxx xx=xx`以及`kubectl edit xxx`
- deamonSet，运行一个daemon pod。这个pod运行在每个节点上，每个节点只有一个这样的pod实例，新的节点加入集群会自动创建，旧的节点被删除后自动回收(缩写是ds)
  - 各种网络、存储、监控、日志都需要运行在每一个节点上
  - 在创建pod的时候加上nodeAffinity，找到节点。还要加上tolerations，才能在noscheduable上调度
  - 一般要加上resources字段，防止daemonset占用过多宿主机资源
  - 可以向deployment一样做record和版本管理，是通过controllerRevision记录某种controller对象的版本，其data字段保存该版本对应的daemonset的api对象，annotation记录创建该对象的命令。回滚后其实不会回退版本号而是版本号+1（statefulset也是用的controllerRevision）
- job用来处理短作业，前面主要编排的都是长作业，要保持running，job计算完就会退出。
  - job不立刻要求你定义一个selector来描述控制哪些pod，创建后会自动加上一个controller-uid={{随机字符串}}的label，避免不同job管理的对象重合。
  - 在模板中定义restartPolicy=Never标识永远不重启，否则还会重新计算。Job对象里只允许设置为Never/OnFailure，Deployment里，只允许被设置为Always。失败的话会创建新的pod去计算，有限次并且间隔时间指数中增加。如果是OnFailure则失败后重启pod里的容器尝试
  - 可以设置最长运行时间，超时后终止
  - 控制的也是pod，可以设置最大同时运行数和最小完成数
  - cronJob定时任务，cronjob和job的关系，类似deployment和replicaSet，可以指定时间周期，以及之前的job没执行完时的策略
- 声明式api，就是只需提交定义好的对象声明期望的状态，多个api写端以patch的方式对对象进行修改
- 一个api对象在etcd里的完整资源路径由group+version+resource三个部分组成，例如/apis/batch/v2alpha1/cronjobs，yaml文件可以写
  > apiVersions: batch/v2alpha1
  >
  > kind: CronJob
  - 其中batch是组，v2alpha1是版本，CronJob是资源类型
  - pod、node等核心对象的group是""，非核心对象先找到group，然后进一步匹配版本号，接着匹配资源类型
  - 以创建cronjob为例，发起创建请求后，yaml被提交给apiserver；先过滤请求，完成授权、超时处理等工作；根据路由和hander的绑定，找到对应的cronjob定义；把yaml转换成一个super version的对象，根据定义创建一个CronJob，接着用admission和validation（准入+验证字段合法），接着把验证过的对象转换成最初提交的版本序列化保存
- CRD：自定义api资源，D是定义的意思，写好后使用代码生成工具自动生成。
  - [详细情况](https://time.geekbang.org/column/article/41876)
  - 自定义控制器监视api对象变化，然后决定具体工作
    - 编写main函数、编写自定义控制器定义、编写控制器业务逻辑
  - informer可以理解成带有本地缓存和索引机制的可以注册event handler的client
    - [原文链接](https://time.geekbang.org/column/article/42076)
- Rbac基于角色的访问权限控制
  - role角色，subject被作用者(用户)，rolebinding二者的关系。role和rolebinding都是namespace对象，旨在namespace中起作用，想在所有namespace起作用需要用clusterRole和clusterRoleBinding
  - 一般用serviceaccount定义一个用户，然后编写roleBinding分配权限，然后创建这几个对象
- operator
  - 更适合管理有状态应用，用CR来管理应用和组件，用k8s声明式风格的api管理应用及服务
  - statefulset的主要特性是编号并绑定pv，operator可以实现一些自定义的部署工作。二者可以单独也可以混合使用。

## 容器持久化存储

- PersistentVolumeController会不断地查看每一个PVC是不是已经处于绑定状态，如果不是就会不断地寻找可用PV
- hostpath和emptydir类型的volume不具备这个特性，既有可能呗kubelet清理掉，也不能迁移到其他节点，大多数持久化Volume往往依赖于一个远程存储服务
- 如果是远程块存储，需要先把远程磁盘挂载到pod的宿主机上，称为attach；第二步是格式化这个磁盘设备，挂载到宿主机指定的挂载点上，称为mount。如果是远程文件存储，则不需要第一步，直接mount即可。反向就是unmount和dettach
- PV的两阶段处理是独立于kubelet主控制循环的，第一阶段是master上的attachDetachController控制，第二阶段是宿主机上的VolumeManagerReconciler，和kubelet主循环解耦可以避免耗时的远程挂载操作拖慢主控制循环。kubelet的一个主要设计原则：**主控制循环不可以被block**
- 自动创建PV：StorageClass是创建PV的模板，定义PV的属性和创建所需的存储插件
  - PVC描述的，是Pod想要使用的持久化存储的属性，比如存储的大小、读写权限等
  - PV描述的，则是一个具体的Volume的属性，比如Volume的类型、挂载目录、远程存储服务器地址等
  - StorageClass的作用是充当PV的模板。并且只有同属于一个StorageClass的PV和PVC才可以绑定在一起
- 本地持久化存储：读写性能会比大多数远程存储好得多，适合高优先级，I/O敏感的应用，但是如果这些节点宕机不能恢复，就有可能丢失，所以要有数据备份和恢复的能力
  - 第一个难点是怎么把本地磁盘抽象成PV，第二个是调度时考虑Volume分布（如果某个pod要使用node1上的本地PV，就必须运行在nod1上，用nodeAffinity指定这个节点的名字）
  - Local PV暂时不支持自动创建PV。并且绑定PVC和PV要推迟到调度阶段，否则可能会出现“死锁”
  - 手动管理的PV可以通过static Provisioner管理

<!-- 速读 -->
<!-- - CSI插件编写 -->

## 容器网络

- Veth Pair 设备的特点是：它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。
- 被限制在 Network Namespace 里的容器进程，实际上是通过 Veth Pair 设备 + 宿主机网桥的方式，实现了跟同其他容器的数据交换。
- 遇到容器连不通“外网”的时候，你都应该先试试 docker0 网桥能不能 ping 通，然后查看一下跟 docker0 和 Veth Pair 设备相关的 iptables 规则是不是有异常，往往就能够找到问题的答案了。
- Flannel 管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”，子网与宿主机的对应关系，保存在 Etcd 当中
  - 我们在进行系统级编程的时候，有一个非常重要的优化原则，就是要减少用户态到内核态的切换次数，并且把核心的处理逻辑都放在内核态进行
  - 在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信。当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里。而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端
- Kubernetes 是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0。这个网桥的名字就叫作：CNI 网桥，它在宿主机上的设备名称默认是：cni0。
- Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈。
  - 这些 CNI 的基础可执行文件，按照功能可以分为三类：
    - Main 插件，它是用来创建具体网络设备的二进制文件
    - IPAM（IP Address Management）插件，它是负责分配 IP 地址的二进制文件
    - 由 CNI 社区维护的内置 CNI 插件。
- Kubernetes 里的 Pod 默认都是“允许所有”（Accept All）的，如果你要对这个情况作出限制，就必须通过 NetworkPolicy 对象来指定白名单。
- Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成 NetworkPolicy 对应的 iptable 规则来实现的。
- 服务发现
  - Service 是由 kube-proxy 组件，加上 iptables 来共同实现的。
  - 这些 Endpoints 对应的 iptables 规则，正是 kube-proxy 通过监听 Pod 的变化事件，在宿主机上生成并维护的。
  - ube-proxy 通过 iptables 处理 Service 的过程，其实需要在宿主机上设置相当多的 iptables 规则。而且，kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的。一直以来，基于 iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。
  - IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价
- 外界访问service
  - NodePort，kube-proxy 要做就是在每台宿主机上生成一条 iptables 规则，对流出的包作snat操作打上标记（好像是方便原路回复）
  - 第二种是指定一个loadbalancer类型的服务
  - 一个 iptables 模式的 Service 对应的规则包括
    - KUBE-SERVICES 或者 KUBE-NODEPORTS 规则对应的 Service 的入口链，这个规则应该与 VIP 和 Service 端口一一对应；
    - KUBE-SEP-(hash) 规则对应的 DNAT 链，这些规则应该与 Endpoints 一一对应；
    - KUBE-SVC-(hash) 规则对应的负载均衡链，这些规则的数目应该与 Endpoints 数目一致；
    - 如果是 NodePort 模式的话，还有 POSTROUTING 处的 SNAT 链。
- 全局的、为了代理不同后端 Service 而设置的负载均衡服务，就是 Kubernetes 里的 Ingress 服务。Ingress，就是 Service 的“Service”，就是 Kubernetes 项目对“反向代理”的一种抽象。
  - 一个 Nginx Ingress Controller 为你提供的服务，其实是一个可以根据 Ingress 对象和被代理后端 Service 的变化，来自动进行更新的 Nginx 负载均衡器。
  
## 作业调度与资源管理

- 像 CPU 这样的资源被称作“可压缩资源”（compressible resources）。当可压缩资源不足时，Pod 只会“饥饿”，但不会退出。
- 像内存这样的资源，则被称作“不可压缩资源（incompressible resources）。当不可压缩资源不足时，Pod 就会因为 OOM（Out-Of-Memory）被内核杀掉。
- 用户在提交 Pod 时，可以声明一个相对较小的 requests 值供调度器使用，而 Kubernetes 真正设置给容器 Cgroups 的，则是相对较大的 limits 值，因为大多数作业使用到的资源其实远小于它所请求的资源限额
  - 当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等的时候，这个 Pod 就属于 Guaranteed 类别
  - 而当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别
  - 而如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort
  - QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时需要用到的
  - Eviction 在 Kubernetes 里其实分为 Soft 和 Hard 两种模式，一个等一段时间，一个立刻开始eviction，删除的时候先删级别低的
- 使用容器的时候，你可以通过设置 cpuset 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力。减少操作系统在cpu之间的上下文切换
- DaemonSet 的 Pod 都设置为 Guaranteed 的 QoS 类型。否则，一旦 DaemonSet 的 Pod 被回收，它又会立即在原宿主机上被重建出来，这就使得前面资源回收的动作，完全没有意义了。

### 默认调度器

- 默认调度器的主要职责，就是为一个新创建出来的 Pod，寻找一个最合适的节点（Node）。
  - 从集群所有的节点中，根据调度算法挑选出所有可以运行该 Pod 的节点；
    - GeneralPredicates 一组过滤规则，负责的是最基础的调度策略
    - 与 Volume 相关的过滤规则
    - 宿主机相关的过滤规则
    - Pod 相关的过滤规则
  - 从第一步的结果中，再根据调度算法挑选一个最符合条件的节点作为最终结果
  - 调度器对一个 Pod 调度成功，实际上就是将它的 spec.nodeName 字段填上调度结果的节点名字。
- assume：为了不在关键调度路径里远程访问 APIServer，Kubernetes 的默认调度器在 Bind 阶段，只会乐观的更新 Scheduler Cache 里的 Pod 和 Node 的信息。之后异步的发起更新pod请求，bind之后还要admit的操作二次确认是否确实可以运行

### 优先级和抢占

- 优先级和抢占机制，解决的是 Pod 调度失败时该怎么办的问题
- 当一个高优先级的 Pod 调度失败后，该 Pod 并不会被“搁置”，而是会“挤走”某个 Node 上的一些低优先级的 Pod
- 调度器只会通过标准的 DELETE API 来删除被抢占的 Pod，鉴于优雅退出期间，集群的可调度性可能会发生的变化，把抢占者交给下一个调度周期再处理，是一个非常合理的选择
- 在调度队列的实现里，使用了两个不同的队列。第一个队列，叫作 activeQ。里面的pod是下一个调度周期需要调度的对象，第二个队列，叫作 unschedulableQ，专门用来存放调度失败的 Pod
- 在为某一对 Pod 和 Node 执行 Predicates 算法的时候，如果待检查的 Node 是一个即将被抢占的节点，即：调度队列里有 nominatedNodeName 字段值是该 Node 名字的 Pod 存在（可以称之为：“潜在的抢占者”）。那么，调度器就会对这个 Node ，将同样的 Predicates 算法运行两遍。第一遍， 调度器会假设上述“潜在的抢占者”已经运行在这个节点上，然后执行 Predicates 算法；第二遍， 调度器会正常执行 Predicates 算法，即：不考虑任何“潜在的抢占者”。

### GPU管理

- 如果有gpu需求，当用户的容器被创建之后，这个容器里必须出现如下两部分设备和目录：GPU 设备、GPU 驱动目录
- 没有为 GPU 专门设置一个资源类型字段，而是使用了一种叫作 Extended Resource（ER）的特殊字段来负责传递 GPU 的信息
- 为了能够在上述 Status 字段里添加自定义资源的数据，你就必须使用 PATCH API 来对该 Node 对象进行更新，加上你的自定义资源的数量。才能实现资源情况上报

## 容器运行时

- kubelet 的工作核心，就是一个控制循环，而驱动这个控制循环运行的事件，包括四种：Pod 更新事件；Pod 生命周期变化；kubelet 本身设置的执行周期；定时的清理事件。
- kubelet 调用下层容器运行时的执行过程，并不会直接调用 Docker 的 API，而是通过一组叫作 CRI的 gRPC 接口来间接执行的。
- CRI只关注容器，不关注Pod

## 监控

- Prometheus 项目工作的核心，是使用 Pull （抓取）的方式去搜集被监控对象的 Metrics 数据（监控指标数据），然后，再把这些数据保存在一个 TSDB （时间序列数据库，比如 OpenTSDB、InfluxDB 等）当中，以便后续可以按照时间进行检索。
- 第一种 Metrics，是宿主机的监控数据。Exporter代替被监控对象来对 Prometheus 暴露出可以被“抓取”的 Metrics 信息的一个辅助进程。
- 第二种 Metrics，是来自于 Kubernetes 的 API Server、kubelet 等组件的 /metrics API，还包括了各个组件的核心监控指标
- 第三种 Metrics，是 Kubernetes 相关的监控数据。其中包括了 Pod、Node、容器、Service 等主要 Kubernetes 核心概念的 Metrics。
- 自定义监控指标，其实就是一个 Prometheus 项目的 Adaptor
  - HPA 的配置就是设置 Auto Scaling 规则的地方。比如，scaleTargetRef 字段，指定了被监控的对象是名叫 sample-metrics-app 的 Deployment，也就是我们上面部署的被监控应用。并且，它最小的实例数目是 2，最大是 10。在 metrics 字段，我们指定了这个 HPA 进行 Scale 的依据，是名叫 http_requests 的 Metrics。而获取这个 Metrics 的途径，则是访问名叫 sample-metrics-app 的 Service。
  - 对于一个多实例应用来说，通过 Service 来采集 Pod 的 Custom Metrics 其实才是合理的做法
- hpa：通过metrics server获取数据并根据扩缩容规则计算，得到目标pod副本数，然后scale

## 日志

- 日志处理系统与容器、Pod 以及 Node 的生命周期都是完全无关的。无论是容器挂了、Pod 被删除，甚至节点宕机的时候，应用的日志依然可以被正常获取到
- 当应用把日志输出到 stdout 和 stderr 之后，容器项目在默认情况下就会把这些日志输出到宿主机上的一个 JSON 文件里。通过 kubectl logs 命令就可以看到这些容器的日志
- 一种方案是logging agent。一般都会以daemonSet的方式运行在节点上，然后将宿主机上的容器目录挂载进去，最后转发出去
- 第一种方案要求日志输出到stdout和stderr，如果要输出到文件，可以通过sidecar容器重新输出到stdout和stderr
- 第三种方案是通过sidecar直接发送到远程存储，资源消耗可能较多
