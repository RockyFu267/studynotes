# calico模式
- IPIP   
从字面来理解，就是把一个IP数据包又套在一个IP包里，即把 IP 层封装到 IP 层的一个 tunnel，看起来似乎是浪费，实则不然。它的作用其实基本上就相当于一个基于IP层的网桥！一般来说，普通的网桥是基于mac层的，根本不需 IP，而这个 ipip 则是通过两端的路由做一个 tunnel，把两个本来不通的网络通过点对点连接起来。ipip 的源代码在内核 net/ipv4/ipip.c 中可以找到。
- BGP模式  
边界网关协议（Border Gateway Protocol, BGP）是互联网上一个核心的去中心化自治路由协议。它通过维护IP路由表或‘前缀’表来实现自治系统（AS）之间的可达性，属于矢量路由协议。BGP不使用传统的内部网关协议（IGP）的指标，而使用基于路径、网络策略或规则集来决定路由。因此，它更适合被称为矢量性协议，而不是路由协议。BGP，通俗的讲就是将接入到机房的多条线路（如电信、联通、移动等）融合为一体，实现多线单IP，BGP 机房的优点：服务器只需要设置一个IP地址，最佳访问路由是由网络上的骨干路由器根据路由跳数与其它技术指标来确定的，不会占用服务器的任何系统。  
    - mesh模式
        - mesh 模式又称为全互联模式，就是一个 BGP Speaker 需要与其它所有的 BGP Speaker 建立 bgp 连接（形成一个bgp mesh）。BGP Speaker越多，将会消耗越多的连接(N^2)。所以它只能支持小规模集群，一般50-100为上限。
    - RR(Router Reflector)模式
        - 就是在网络中指定一个或多个 BGP Speaker 作为反射路由（Router Reflector），RR与所有的 BGP Speaker 建立 bgp 连接。每个BGP Speaker只与RR建立连接，交换路由信息就能获得全网的路由信息。Calico 中通过Global Peer来实现RR模式。
        - RR 的基本思想是选择一部分节点（一个或者多个）作为 Global BGP Peer，它们和所有的其他节点互联来交换路由信息，其他的节点只需要和 Global BGP Peer 相连就行，不需要之间再两两连接。更多的组网模式也是支持的，不管怎么组网，最核心的思想就是所有的节点能获取到整个集群的路由信息。
# calico组件
- libnetwork-plugin 
    - 是 calico 提供的 docker 网络插件，主要提供的是 IP 管理和网络管理的功能。
    - 默认情况下，当网络中出现第一个容器时，calico 会为容器所在的节点分配一段子网（子网掩码为 /26，比如192.168.196.128/26），后续出现在该节点上的容器都从这个子网中分配 IP 地址。这样做的好处是能够缩减节点上的路由表的规模，按照这种方式节点上 2^6 = 64 个 IP 地址只需要一个路由表项就行，而不是为每个 IP 单独创建一个路由表项。节点上创建的子网段可以在etcd 中 /calico/ipam/v2/host/<node_name>/ipv4/block/ 看到。
    - calico 还允许创建容器的时候指定 IP 地址，如果用户指定的 IP 地址不在节点分配的子网段中，calico 会专门为该地址添加一个 /32 的网段。
        - kubeadm默认的etcd部署方式(也未用自定义配置)，etcd中并未找到/calico*相关的的数据；-------------待考证；

- BIRD（BIRD Internet Routing Daemon） 
    - 是一个常用的网络路由软件，支持很多路由协议（BGP、RIP、OSPF等）。它会在每台宿主机上运行，calico 用它在实现主机间传播路由信息。
    - BIRD 对应的配置文件在 /etc/calico/confd/config/ 目录;
       - kubeadm默认的etcd部署方式(也未用自定义配置)，并未找到该路径以及相关的的数据；~~ -------------待考证；~~
            - 相关的配置在daemonset的node找到了
       - ~~只在calico的daemonset的node里找到了 confd 文件夹和 felix.cfg~~

- confd
    - 是一个简单的配置管理工具。bird 的配置文件会根据用户设置的变化而变化，因此需要一种动态的机制来实时维护配置文件并通知 bird 使用最新的配置，这就是 confd 的工作。它会监听 etcd 的数据，用来更新 bird 的配置文件，并重新启动 bird 进程让它加载最新的配置文件。
    - confd 的工作目录是 /etc/calico/confd;
        - conf.d：confd 需要读取的配置文件，每个配置文件告诉 confd 模板文件在什么，最终生成的文件应该放在什么地方，更新时要执行哪些操作等
        - config：生成的配置文件最终放的目录;并未找到该路径以及相关的的数据；-------------待考证；
        - templates：模板文件，里面包括了很多变量占位符，最终会替换成 etcd 中具体的数据
            - 它会监听 etcd 的 /calico/bgp/v1 路径，一旦发现更新，就用其中的内容更新模板文件 bird.cfg.mesh.template，把新生成的文件放在 /etc/calico/confd/config/bird.cfg，文件改变之后还会运行 reload_cmd 指定的命令重启 bird 程序。并未找到该路径以及相关的的数据；-------------待考证；
- felix 
    - 负责最终网络相关的配置，也就是容器网络在 linux 上的配置工作；
        - 更新节点上的路由表项
        - 更新节点上的 iptables 表项
    - 它的主要工作是从 etcd 中读取网络的配置，然后根据配置更新节点的路由和 iptables，felix 的代码在 http://github.com/calico/felix。

# 总结calico工作内容
- 分配和管理 IP
- 配置上容器的 veth pair 和容器内默认路由
- 根据集群网络情况实时更新节点上路由表

# 目前最大的问题 
- 我在测试环境的etcd中找不到calico的数据；-------------待考证；(完全不知道测试环境calico的数据存在哪？)
## calico信息收集
- 获取BGP配置
    - ```
        apiVersion: projectcalico.org/v3
        items:
        - apiVersion: projectcalico.org/v3
        kind: BGPConfiguration
        metadata:
            creationTimestamp: "2111-11-11T02:26:06Z"
            name: default
            resourceVersion: "xx"
            uid: xxx-xx-xx-xx-xxx
        spec:
            asNumber: 63400
            logSeverityScreen: Info
            nodeToNodeMeshEnabled: false
        kind: BGPConfigurationList
        metadata:
        resourceVersion: "xxx"
        ```
    - asNumber  值的含义-------------待考证；
    - nodeToNodeMeshEnabled meshi模式开关；
- 获取node配置
    ```
    apiVersion: projectcalico.org/v3
    kind: Node
    metadata:
    annotations:
        projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"xxx"}'
    creationTimestamp: "2111-11-11T11:11:11Z"
    labels:
        beta.kubernetes.io/arch: amd64
        beta.kubernetes.io/os: linux
        kubernetes.io/arch: amd64
        kubernetes.io/hostname: xxxxx
        kubernetes.io/os: linux
        node-role.kubernetes.io/master: ""
        route-reflector: "true"
    name: xxx
    resourceVersion: "xxxx"
    uid: xxx-x-x-x-xxx
    spec:
    addresses:
    - address: xxx.xxx.xx.xx/xx
    - address: xxx.xxx.xx.xx
    bgp:
        ipv4Address: xxx.xxx.xx.xxx/xx
        ipv4IPIPTunnelAddr: xxx.xxx.xxx.xxx
        routeReflectorClusterID: 224.0.0.1
    orchRefs:
    - nodeName: xxxx
        orchestrator: k8s
    status:
    podCIDRs:
    - xxx.xxx.x.x/xx
    ```
    - route-reflector 如果节点不是rr模式，不会有这个key
    - routeReflectorClusterID 这个是否是RR模式节点特有-------------待考证；(怀疑测试环境目前的问题与这个配置也有关)

## 更改CIDR的方案
- 添加一个新的ippool
    - ```
        apiVersion: projectcalico.org/v3
        kind: IPPool
        metadata:
        name: new-pool
        spec:
        cidr: xxx.xx.0.0/xx
        ipipMode: CrossSubnet
        nodeSelector: all()
        vxlanMode: Never
        natOutgoing: true
       ```  
    - apply之后会存在两个IPPool;
- 停用旧的CIDR
    - ```
        apiVersion: projectcalico.org/v3
        items:
        - apiVersion: projectcalico.org/v3
            kind: IPPool
            metadata:
                creationTimestamp: "xxxx-xx-xxTxx:xx:xxZ"
                name: default-ipv4-ippool
                resourceVersion: "x"
                uid: x-x-x-x-x
            spec:
                blockSize: 26
                cidr: x.x.0.0/xx
                ipipMode: CrossSubnet
                nodeSelector: all()
                vxlanMode: Never
        kind: IPPoolList
        metadata:
        resourceVersion: "xx"
       ```
    - 添加配置 disabled: true；停用旧的CIDR
- 滚动替换node-CIDR参数;~~ -------------待考证；(具体的操作方案) ~~
    - 以下都是目前的假设
        - 导出机器列表
        - 规划机器对应IP的map关系，比如
            - ```
              node01 172.16.1.0/24
              node02 172.16.2.0/24
              ...
              ```
        - 准备node-yaml模版 跑个脚本？
            - 替换 metadata.hostname
            - 替换 spec.podCIDR
            - 替换 spec.podCIDRs
- 修改kube-proxy配置；~~ -------------待考证；(操作顺序？先改node再改控制组件？)~~
    - clusterCIDR
- 修改kubeadm-config配置;~~ -------------待考证；(操作顺序？先改node再改控制组件？;且是不是只要这里做操作就好，别的组件会监听到这个配置的变更，前提是使用kubeadm部署？)~~
    - podSubnet 
- ~~修改kube-controller-manager配置; ~~ -------------待考证；(操作顺序？先改node再改控制组件？)~~
    - spec.containers.command[cluster-cidr]
- 修改 /etc/cni/net.d/ 下的 10-calico.conflist和calico-kubeconfig 配置；-------------待考证；(网络上有文档提到该步骤，很怀疑这部操作的必要性，尤其是caico-kubeconfig，这个文件的作用不包括放cidr的配置吧？)
    - ipv4_pools

### 更改CIDR的方案验证
- 添加新的ippool并停用就CIDR(使用的calicoctl操作,kubectl edit 操作应该没区别？)
    - 并未对目前现有的集群造成任何影响；
    - 之后副本集内的容器状态出现变更，新创建的容器地址段均为新的地址段
    - 新地址段容器与新地址段容器网络通信正常
    - 新地址段容器与旧地址段容器网络通信正常
    - 旧地址段容器与旧地址段容器网络通信未知(旧地址段镜像太赶紧，容器内无法做相关验证操作)
        - 之后集群重建再测试的时候再测；-------------待考证；猜测大概率也是正常的；
- 改kubeadm-config配置;
    - 并未对目前现有的集群造成任何影响；集群也没有发生任何变化；
    - 变更操作还是有必要做；
        - kubeadm支持导出集群初始化配置；怀疑命令的结果就是取kubeadm的configmap配置文件；
        - 考虑到集群的运维操作(升级，恢复等)，该数据的变更还是有必要记录；
- 修改kube-proxy配置；
    - 并未对目前现有的集群造成任何影响；集群也没有发生任何变化；
    - 变更了配置/etc/kubernetes/manifests中的配置文件或者直接修改configmap，daemonset均未重启；
        - 同级目录的api或者cm组件的如果更改了配置文件均会导致pod重新生成载入新的配置；
        - 手动删除原pod，自动拉起新的正常；集群也未发生任何变化；
        - 小插曲(过程中一台新加的node失联了，去机房看发现网卡停了，怀疑是机器本身的问题)
- 修改kube-controller-manager配置;
    - 修改后，服务不断重启；
        - 能短暂启动running；但是读取到node配置时会报错;读取到node的cidr值与新配置的cluster-cidr的值不符；
        - ```
          I1229 09:17:41.673773       1 range_allocator.go:116] No Secondary Service CIDR provided. Skipping filtering out secondary service addresses.
          E1229 09:17:41.673808       1 controllermanager.go:522] Error starting "nodeipam"
          F1229 09:17:41.673823       1 controllermanager.go:235] error starting controllers: failed to mark cidr[192.168.3.0/24] at idx [0] as occupied for node: node02: cidr 192.168.3.0/24 is out the range of cluster cidr 172.21.0.0/16
          ```
