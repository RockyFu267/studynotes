# calico网络隔离 - profile & policy & iptables
profile 与 policy资源的定义见 Calico资源模型

在默认情况下，calico中的容器间都 不能 互相访问，对应的数据包会在iptables中被丢弃。需要为对应的endpoint设置profile或policy，才能允许互访。

## Profile
每个workload endpoint和host endpoint都可以设置profile字段，选择应用在这个网卡上的profile。

Profile资源的定义如 Calico资源模型#Profile  ，拥有一组ingress（入向流量，外部->endpoint）和egress（出向流量，endpoint->外部）规则。

ingress和egress都可以包含多条规则，在列表中靠前的规则优先级更高。

每条规则由一个action和一组匹配规则组成。action表示处理数据包的方式，包括allow - 放行，deny - 丢弃， log - ???

匹配规则允许对数据包的协议、数据包来源（根据endpoint筛选、IP、端口或端口区间匹配）、数据包目标进行正向/反向的匹配，一个数据包满足规则中的所有匹配规则时，则该数据包符合此条规则。

规则的具体格式见Calico资源模型#定义.5 。

当一个endpoint被指定了多个profile时，在列表中更靠前的profile优先级更高 （测试结论，官方文档无相关描述）

## Policy
policy资源独立于endpoint存在，可以更细粒度地配置容器间的网络访问规则。policy资源的定义见 Calico资源模型#Policy

与profile的区别在于：policy通过label选出该policy应用的endpoint，profile需要应用该profile的endpoint在配置中指定；policy具有优先级，并且policy的优先级高于profile。

policy相比profile，多了一个selector，用于选出应用该policy的endpoint，以及一个order表示该policy的优先级，order越小优先级越高。

此外ingress和egress规则列表与profile基本相同，不同之处在于允许的action多了一个pass，表示转入profile的对应规则。

## iptables实现
calico对网络隔离的实现是通过ipset + iptables实现的。

calico会在iptables的filter表加入以下3条链：cali-INPUT、cali-FORWARD、cali-OUTPUT来处理数据包，决定接受或丢弃。其中主要处理容器相关网络访问的是cali-FORWARD链。

```
Chain cali-FORWARD (1 references)
 pkts bytes target                  prot opt in     out     source               destination         
    0     0 ACCEPT                  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:jxvuJjmmRV135nVu */ mark match 0x1000000/0x1000000 ctstate UNTRACKED
    0     0 DROP                    all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:8YeDX9Z0tXyO0Sp8 */ ctstate INVALID
   72 11176 ACCEPT                  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:1GMSV-PhhZ8QbJg4 */ ctstate RELATED,ESTABLISHED
   14   840 cali-from-wl-dispatch   all  --  cali+  *       0.0.0.0/0            0.0.0.0/0            /* cali:36TkoGXj9EF7Plkv */
    0     0 cali-to-wl-dispatch     all  --  *      cali+   0.0.0.0/0            0.0.0.0/0            /* cali:URMhBRo8ugd8J8Yx */
   14   840 ACCEPT                  all  --  cali+  *       0.0.0.0/0            0.0.0.0/0            /* cali:FyhWsW08U3a5niLK */
    0     0 ACCEPT                  all  --  *      cali+   0.0.0.0/0            0.0.0.0/0            /* cali:G655uIfZuidj1gAw */
    0     0 MARK                    all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:4GbueNC2iWajKnxO */ MARK and 0xf8ffffff
    0     0 cali-from-host-endpoint all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:bq3wVY3mkXk96NQP */
    0     0 cali-to-host-endpoint   all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:G8sjbYXH5_QiYnBl */
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:wYFYRdMhtSYCqKNm */ /* Host endpoint policy accepted packet. */ mark match 0x1000000/0x1000000
```
-  所有容器都通过自己的一个veth与宿主机相连，宿主机端的接口名称为 cali*。因此，cali-from-wl-dispatch和cali-to-wl-dispatch分别匹配从容器出来的数据包和到达容器的数据包。

```
Chain cali-to-wl-dispatch (1 references)
 pkts bytes target                   prot opt in     out     source               destination         
   19  1140 cali-tw-calia2165f5578d  all  --  *      calia2165f5578d  0.0.0.0/0            0.0.0.0/0           [goto]  /* cali:9-Agj3bvNHdG7UMX */
    0     0 DROP                     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:R2WZulaWW-VcPyYn */ /* Unknown interface */

```

-  在cali-to-wl-dispatch下，对于到达每一个该宿主机上的容器（到达对应的veth）的数据包，有一条规则转入对应的chain进行处理。该chain的规则如下
```
Chain cali-tw-calia2165f5578d (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   19  1140 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:sAqqMOm5MZXqQAUe */ MARK and 0xfeffffff
   19  1140 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:UgzfB7pIpttGentJ */ /* Start of policies */ MARK and 0xfdffffff
   19  1140 cali-pi-k8s-policy-no-match  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:oUVuAcRtBM6X0N1K */ mark match 0x0/0x2000000
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:O-cdytwc1nthm-NW */ /* Return if policy accepted */ mark match 0x1000000/0x1000000
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:a_rcCvBkbSz_y1oA */ /* Drop if no policies passed packet */ mark match 0x0/0x2000000
    2   120 cali-pri-_5KWtHrpY20d7dRz64D  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:m7hySYS06-wObnP4 */
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:urjxACrOe5270_-z */ /* Return if profile accepted */ mark match 0x1000000/0x1000000
    0     0 cali-pri-k8s_ns.policy-test  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:KMVpq7-tHQnvUIlK */
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:GdWq0_8iuWPJ0jxI */ /* Return if profile accepted */ mark match 0x1000000/0x1000000
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:1e8NDjdZzoqx-8fv */ /* Drop if no profiles matched */
```

-  对于应用于该网卡的每一个profile或policy，有一条对应的chain匹配。例如，如下的chain对应的是  k8s_ns.policy-test 这个profile。该profile对应的yaml见下
```
- apiVersion: v1
  kind: profile
  metadata:
    name: k8s_ns.policy-test
    tags:
    - k8s_ns.policy-test1
  spec:
    egress:
    - action: allow
      destination: {}
      source: {}
    ingress:
    - action: allow
      destination: {}
      source:
        selector: calico/k8s_ns == 'policy-test'
    - action: allow
      destination: {}
      source:
        selector: calico/k8s_ns == 'policy-test-2'
```

```

Chain cali-pri-k8s_ns.policy-test (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    1    60 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:hdrm8XUZQX4Q1mma */ match-set cali4-s:v1BpSflp0q_NSwif1we7aQF src MARK or 0x1000000
    1    60 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:hrWVQNK9arZcm9rH */ mark match 0x1000000/0x1000000
    5   300 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:gvqkq9Lx7T_N4Spj */ match-set cali4-s:UlKBTJQR9ju1TukewdkIJue src MARK or 0x1000000
    5   300 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:5U0kWqOncGcjg4Hf */ mark match 0x1000000/0x1000000

```

-  每一条target为MARK规则匹配一组ipset，这组IPSet是根据policy中对应的匹配规则计算出来的IP集合。这里有2条MARK规则，对应匹配的IPSet也正是以上profile中两个selector选择出来的 pod的IP。

```
$sudo ipset list
...
Name: cali4-s:v1BpSflp0q_NSwif1we7aQF
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 1048576
Size in memory: 224
References: 1
Members:
10.10.183.105
10.10.55.86

Name: cali4-s:UlKBTJQR9ju1TukewdkIJue
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 1048576
Size in memory: 224
References: 1
Members:
10.10.119.51
10.10.119.50

```

## k8s-policy-controller - 与k8s交互
k8s-policy-controller负责从k8s apiserver读取k8s的namespace与networkpolicy相关配置，并写入calico对应的datastore(如etcd)。

默认情况下，当一个数据包无法匹配任何一条policy或profile时，calico控制的iptables将会丢掉这个包。因此，需要这个组件将k8s的相关配置转换成calico使用的profile和policy，并将对应规则写入iptables，以放行流量。

对于每一个k8s的namespace，k8s的默认行为是放行所有流量，因此对每个k8s namespace，k8s-policy-controller会生成如下的profile：
```
- apiVersion: v1
  kind: profile
  metadata:
    name: k8s_ns.default
    tags:
    - k8s_ns.default
  spec:
    egress:
    - action: allow
      destination: {}
      source: {}
    ingress:
    - action: allow
      destination: {}
      source: {}
```

```
- apiVersion: v1
  kind: profile
  metadata:
    name: k8s_ns.default
    tags:
    - k8s_ns.default
  spec:
    egress:
    - action: allow
      destination: {}
      source: {}
    ingress:
    - action: allow
      destination: {}
      source: {}
```

-  若该namespace配置了isolation = ingress: defaultDeny，则对应的profile如下

```
- apiVersion: v1
  kind: profile
  metadata:
    name: k8s_ns.default
    tags:
    - k8s_ns.default
  spec:
    egress:
    - action: allow
      destination: {}
      source: {}
    ingress:
    - action: deny
      destination: {}
      source: {}
```

拒绝所有入向流量，包括来自同一namespace和不同namespace的任何其他pod的任何数据包。

k8s对于配置了DefualtDeny的namespace，k8s提供了networkpolicy资源来开放部分其他pod的访问。每条networkpolicy属于一个namespace，作用于该namespace的pod。

networkpolicy包含一个podSelector，来指定该namespace中适用该networkpolicy的pod，以及一组访问来源pod和目标端口/协议的白名单。访问来源pod可以使用podSelector来选取与该networkpolicy属于同一namespace的pod，允许这些pod访问；也可以使用namespaceSelector，以允许某个（同一/不同）namespace的所有pod访问。

对于k8s的每条networkpolicy资源，k8s-policy-controller会添加一条对应的policy。例如

```

kind: NetworkPolicy
metadata:
  name: test-network-policy
spec:
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
    ports:
    - port: 80
      protocol: TCP
  podSelector:
    matchLabels:
      run: nginx


```

-  这个networkpolicy资源会被转换成如下的calico policy

```
- apiVersion: v1
  kind: policy
  metadata:
    name: policy-test.test-network-policy
  spec:
    egress:
    - action: allow
      destination: {}
      source: {}
    ingress:
    - action: allow
      destination:
        ports:
        - 80
      protocol: tcp
      source:
        selector: access == 'true' && calico/k8s_ns == 'policy-test'
    order: 1000
    selector: calico/k8s_ns == 'policy-test' && run == 'nginx'
```

-  selector对应以上networkpolicy.spec里的podSelector；ingress中的每条规则对应以上networkpolicy的一条白名单规则，selector对应的即白名单规则的里的podSelector/namespaceSelector。

k8s-policy-controller启动时，还会添加这样一条policy，以匹配没有任何networkpolicy匹配的pod，转入profile进行处理。

```
- apiVersion: v1
  kind: policy
  metadata:
    name: k8s-policy-no-match
  spec:
    egress:
    - action: pass
      destination: {}
      source: {}
    ingress:
    - action: pass
      destination: {}
      source: {}
    order: 2000
    selector: has(calico/k8s_ns)
```

- 表示所有k8s的pod在没有其他policy匹配时，都匹配这条policy，action=pass表示转入profile进行下一步的匹配和处理。

