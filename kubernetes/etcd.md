# etcdctl操作
- 下载
    - ```wget https://github.com/etcd-io/etcd/releases/download/v3.3.15/etcd-v3.3.15-linux-amd64.tar.gz```
- 解压
    - ```tar zxvf etcd-v3.3.15-linux-amd64.tar.gz ```
- 添加到bin    
    - ```cp etcdctl /usr/bin/```
- 添加接口版本环境配置
    - ```export ETCDCTL_API=3 ```
- 获取数据
    - ```etcdctl --endpoints=https://100.100.11.252:2379  --cert="/etc/kubernetes/pki/etcd/server.crt" --key="/etc/kubernetes/pki/etcd/server.key" --cacert="/etc/kubernetes/pki/etcd/ca.crt"  --prefix --keys-only=true get / | grep calico ```

## etcdctl操作v2命令
- ```etcdctl --endpoints=https://100.100.11.252:2379  --cert-file="/etc/kubernetes/pki/etcd/server.crt" --key-file="/etc/kubernetes/pki/etcd/server.key" --ca-file="/etc/kubernetes/pki/etcd/ca.crt"  member list ```