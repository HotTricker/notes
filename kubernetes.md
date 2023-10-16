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
