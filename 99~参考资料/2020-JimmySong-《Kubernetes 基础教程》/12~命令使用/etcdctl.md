---
weight: 82
title: 使用 etcdctl 访问 Kubernetes 数据
date: '2022-05-21T00:00:00+08:00'
type: book
---

Kubenretes1.6 中使用 etcd V3 版本的 API，使用 `etcdctl` 直接 `ls` 的话只能看到 `/kube-centos` 一个路径。需要在命令前加上 `ETCDCTL_API=3` 这个环境变量才能看到 kuberentes 在 etcd 中保存的数据。

```bash
ETCDCTL_API=3 etcdctl get /registry/namespaces/default -w=json|python -m json.tool
```

如果是使用 kubeadm 创建的集群，在 Kubenretes 1.11 中，etcd 默认使用 tls ，这时你可以在 master 节点上使用以下命令来访问 etcd ：

```bash
ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/peer.crt \
--key=/etc/kubernetes/pki/etcd/peer.key \
get /registry/namespaces/default -w=json | jq .
```

- `-w` 指定输出格式

将得到这样的 json 的结果：

```json
{
    "count": 1,
    "header": {
        "cluster_id": 12091028579527406772,
        "member_id": 16557816780141026208,
        "raft_term": 36,
        "revision": 29253467
    },
    "kvs": [
        {
            "create_revision": 5,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvZGVmYXVsdA==",
            "mod_revision": 5,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmIKSAoHZGVmYXVsdBIAGgAiACokZTU2YzMzMDgtMWVhOC0xMWU3LThjZDctZjRlOWQ0OWY4ZWQwMgA4AEILCIn4sscFEKOg9xd6ABIMCgprdWJlcm5ldGVzGggKBkFjdGl2ZRoAIgA=",
            "version": 1
        }
    ]
}
```

使用 `--prefix` 可以看到所有的子目录，如查看集群中的 namespace：

```bash
ETCDCTL_API=3 etcdctl get /registry/namespaces --prefix -w=json|python -m json.tool
```

输出结果中可以看到所有的 namespace。

```bash
{
    "count": 8,
    "header": {
        "cluster_id": 12091028579527406772,
        "member_id": 16557816780141026208,
        "raft_term": 36,
        "revision": 29253722
    },
    "kvs": [
        {
            "create_revision": 24310883,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvYXV0b21vZGVs",
            "mod_revision": 24310883,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmQKSgoJYXV0b21vZGVsEgAaACIAKiQ1MjczOTU1ZC1iMzEyLTExZTctOTcwYy1mNGU5ZDQ5ZjhlZDAyADgAQgsI7fSWzwUQ6Jv1Z3oAEgwKCmt1YmVybmV0ZXMaCAoGQWN0aXZlGgAiAA==",
            "version": 1
        },
        {
            "create_revision": 21387676,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvYnJhbmQ=",
            "mod_revision": 21387676,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmEKRwoFYnJhbmQSABoAIgAqJGNkZmQ1Y2NmLWExYzktMTFlNy05NzBjLWY0ZTlkNDlmOGVkMDIAOABCDAjR9qLOBRDYn83XAXoAEgwKCmt1YmVybmV0ZXMaCAoGQWN0aXZlGgAiAA==",
            "version": 1
        },
        {
            "create_revision": 5,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvZGVmYXVsdA==",
            "mod_revision": 5,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmIKSAoHZGVmYXVsdBIAGgAiACokZTU2YzMzMDgtMWVhOC0xMWU3LThjZDctZjRlOWQ0OWY4ZWQwMgA4AEILCIn4sscFEKOg9xd6ABIMCgprdWJlcm5ldGVzGggKBkFjdGl2ZRoAIgA=",
            "version": 1
        },
        {
            "create_revision": 18504694,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvZGV2",
            "mod_revision": 24310213,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmwKUgoDZGV2EgAaACIAKiQyOGRlMGVjNS04ZTEzLTExZTctOTcwYy1mNGU5ZDQ5ZjhlZDAyADgAQgwI89CezQUQ0v2fuQNaCwoEbmFtZRIDZGV2egASDAoKa3ViZXJuZXRlcxoICgZBY3RpdmUaACIA",
            "version": 4
        },
        {
            "create_revision": 10,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMva3ViZS1wdWJsaWM=",
            "mod_revision": 10,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmcKTQoLa3ViZS1wdWJsaWMSABoAIgAqJGU1ZjhkY2I1LTFlYTgtMTFlNy04Y2Q3LWY0ZTlkNDlmOGVkMDIAOABCDAiJ+LLHBRDdrsDPA3oAEgwKCmt1YmVybmV0ZXMaCAoGQWN0aXZlGgAiAA==",
            "version": 1
        },
        {
            "create_revision": 2,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMva3ViZS1zeXN0ZW0=",
            "mod_revision": 2,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmYKTAoLa3ViZS1zeXN0ZW0SABoAIgAqJGU1NmFhMDVkLTFlYTgtMTFlNy04Y2Q3LWY0ZTlkNDlmOGVkMDIAOABCCwiJ+LLHBRDoq9ASegASDAoKa3ViZXJuZXRlcxoICgZBY3RpdmUaACIA",
            "version": 1
        },
        {
            "create_revision": 3774247,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvc3BhcmstY2x1c3Rlcg==",
            "mod_revision": 3774247,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEoABCmYKDXNwYXJrLWNsdXN0ZXISABoAIgAqJDMyNjY3ZDVjLTM0YWMtMTFlNy1iZmJkLThhZjFlM2E3YzViZDIAOABCDAiA1cbIBRDU3YuAAVoVCgRuYW1lEg1zcGFyay1jbHVzdGVyegASDAoKa3ViZXJuZXRlcxoICgZBY3RpdmUaACIA",
            "version": 1
        },
        {
            "create_revision": 15212191,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMveWFybi1jbHVzdGVy",
            "mod_revision": 15212191,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEn0KYwoMeWFybi1jbHVzdGVyEgAaACIAKiQ2YWNhNjk1Yi03N2Y5LTExZTctYmZiZC04YWYxZTNhN2M1YmQyADgAQgsI1qiKzAUQkoqxDloUCgRuYW1lEgx5YXJuLWNsdXN0ZXJ6ABIMCgprdWJlcm5ldGVzGggKBkFjdGl2ZRoAIgA=",
            "version": 1
        }
    ]
}
```

key 的值是经过 base64 编码，需要解码后才能看到实际值，如：

```bash
$ echo L3JlZ2lzdHJ5L25hbWVzcGFjZXMvYXV0b21vZGVs|base64 -d
/registry/namespaces/automodel
```

## etcd 中 kubernetes 的元数据

我们使用 kubectl 命令获取的 kubernetes 的对象状态实际上是保存在 etcd 中的，使用下面的脚本可以获取 etcd 中的所有 kubernetes 对象的 key：

> 注意，我们使用了 ETCD v3 版本的客户端命令来访问 etcd。

```bash
#!/bin/bash
# Get kubernetes keys from etcd
export ETCDCTL_API=3
keys=`etcdctl get /registry --prefix -w json|python -m json.tool|grep key|cut -d ":" -f2|tr -d '"'|tr -d ","`
for x in $keys;do
  echo $x|base64 -d|sort
done
```

通过输出的结果我们可以看到 kubernetes 的原数据是按何种结构包括在 kuberentes 中的，输出结果如下所示：

```ini
/registry/ThirdPartyResourceData/istio.io/istioconfigs/default/route-rule-details-default
/registry/ThirdPartyResourceData/istio.io/istioconfigs/default/route-rule-productpage-default
/registry/ThirdPartyResourceData/istio.io/istioconfigs/default/route-rule-ratings-default
...
/registry/configmaps/default/namerctl-script
/registry/configmaps/default/namerd-config
/registry/configmaps/default/nginx-config
...
/registry/deployments/default/sdmk-page-sdmk
/registry/deployments/default/sdmk-payment-web
/registry/deployments/default/sdmk-report
...
```

我们可以看到所有的 Kuberentes 的所有元数据都保存在 `/registry` 目录下，下一层就是 API 对象类型（复数形式），再下一层是 `namespace`，最后一层是对象的名字。

以下是 etcd 中存储的 kubernetes 所有的元数据类型：

```ini
ThirdPartyResourceData
apiextensions.k8s.io
apiregistration.k8s.io
certificatesigningrequests
clusterrolebindings
clusterroles
configmaps
controllerrevisions
controllers
daemonsets
deployments
events
horizontalpodautoscalers
ingress
limitranges
minions
monitoring.coreos.com
namespaces
persistentvolumeclaims
persistentvolumes
poddisruptionbudgets
pods
ranges
replicasets
resourcequotas
rolebindings
roles
secrets
serviceaccounts
services
statefulsets
storageclasses
thirdpartyresources
```
