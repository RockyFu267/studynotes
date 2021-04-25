# tips

## 维护
- 副本的终止状态一直保持；手动加force参数卡住；
    - 删除命令后 加 --grace-period=0
        - 该命令意为延迟删除的宽限期指定为0s，要求立刻强制结束而不是等待容器自然终止；
- coredns添加个别host解析记录(紧急情况下，正常不推荐写host)
    - kubectl edit configmap -n kube-system coredns
        - ```hosts {
            100.100.198.253 rancher.test.5f
            fallthrough
            }```
        - 加在kubernetes下面
- 开启apiserver审计日志
    - 备份、修改api的yaml
        - ```
            - --audit-log-maxsize=100
            - --audit-log-maxbackup=50
            - --audit-log-maxage=30
            - --audit-log-format=json
            - --audit-log-path=/root/test-apiserver/testlog
            - --audit-log-version=audit.k8s.io/v1
            - --audit-policy-file=/root/test-apiserver/policy.yaml
            # volumeMounts
                - mountPath: /root/test-apiserver
                  name: policy-test
                  readOnly: false
            # volumes
                - hostPath:
                    path: /root/test-apiserver
                    type: DirectoryOrCreate
                  name: policy-test
            ```
    - 准备日志路径
    - 准备配置文件policy.yaml，赋权777
        - ```
            apiVersion: audit.k8s.io/v1
            kind: Policy
            rules:
            - level: Metadata
            ```
