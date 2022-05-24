### docker核心技术及实现原理
原文引用：[linke](https://draveness.me/docker/)
#### namespace
- 基本概念：用于分离进程树，网络接口，挂载点，进程间通讯等资源的方法，实现资源隔离
- 种类：7种不同的命名空间：clone_newCgroup,clone_newIPC,clone_newNet,clone_newNS,clone_newPid,clone_newUser,clone_newUTS
通过七种命名空间，可以设置哪些资源与宿主机隔离

#### 进程
- 上帝进程idle，pid=0，创建两个特殊进程/sbin/init pid=1 和 进程kthread  pid=2
- /sbin/init负责初始化工作和系统配置，也会创建getty等注册进程，而kthread主要负责调度
- docker 的进程树：init -> dockerd -> docker containerd -> docker containerd shim 然后通过clone(2)时传入clone_newPid创建docker内部进程
- 创建container时主要调用的函数：
    ```go
    containerRouter.postContainersStart
    └── daemon.ContainerStart
        └── daemon.createSpec // 创建
            └── setNamespaces
                └── setNamespace
    // 最后所有命名空间相关的设置 Spec 最后都会作为 Create 函数的参数，创建新的容器，完成容器内进程/网络等资源与宿主机的隔离
    daemon.containerd.Create(context.Background(), container.ID, spec, createOptions)
    ```
#### 网络
- 首要问题：通过命名空间创建独立的网络空间后怎么对外通讯？
- 四种网络模式：Host Container Node 和Bridge(默认)
    - 默认模式：bridge网桥，docker启动时会建立网桥docker0
    - 容器默认会创建两个虚拟网卡，一个放容器一个加入docker0网桥
        ```sh
        $ brctl show
        bridge name	bridge id		STP enabled	interfaces
        docker0		8000.0242a6654980	no		veth3e84d4f
                                                veth9953b75
        ```
     - docker0会为容器分配ip并通过iptables将该容器的网关设置为docker0
- libNetwork 提供了连接不同容器的实现接口，主要有三个组件：Sandbox,Endpoint,Network
