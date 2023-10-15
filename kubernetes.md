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
- 在 Pod 中，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。
