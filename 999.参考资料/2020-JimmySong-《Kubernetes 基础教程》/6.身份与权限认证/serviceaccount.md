---
weight: 44
title: ServiceAccount
date: '2022-05-21T00:00:00+08:00'
type: book
summary: "ServiceAccount 为 Pod 中的进程提供身份信息。"
---

ServiceAccount 为 Pod 中的进程提供身份信息。

{{<callout note 注意>}}
本文档描述的关于 ServiceAccount 的行为只有当你按照 Kubernetes 项目建议的方式搭建集群的情况下才有效。集群管理员可能在你的集群中进行了自定义配置，这种情况下该文档可能并不适用。
{{</callout>}}

当你（真人用户）访问集群（例如使用 `kubectl` 命令）时，API 服务器会将你认证为一个特定的 User Account（目前通常是 `admin`，除非你的系统管理员自定义了集群配置）。Pod 容器中的进程也可以与 API 服务器联系。 当它们在联系 API 服务器的时候，它们会被认证为一个特定的 ServiceAccount（例如`default`）。

## 使用默认的 ServiceAccount 访问 API 服务器

当你创建 pod 的时候，如果你没有指定一个 ServiceAccount，系统会自动得在与该 pod 相同的 namespace 下为其指派一个 `default` ServiceAccount。如果你获取刚创建的 pod 的原始 json 或 yaml 信息（例如使用`kubectl get pods/podename -o yaml`命令），你将看到`spec.serviceAccountName`字段已经被设置为 `default`。

你可以在 pod 中使用自动挂载的 ServiceAccount 凭证来访问 API，如 [Accessing the Cluster](https://kubernetes.io/docs/user-guide/accessing-the-cluster/#accessing-the-api-from-a-pod) 中所描述。

ServiceAccount 是否能够取得访问 API 的许可取决于你使用的 [授权插件和策略](https://kubernetes.io/docs/admin/authorization/#a-quick-note-on-service-accounts)。

在 1.6 以上版本中，你可以选择取消为 ServiceAccount 自动挂载 API 凭证，只需在 ServiceAccount 中设置 `automountServiceAccountToken: false`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

在 1.6 以上版本中，你也可以选择只取消单个 pod 的 API 凭证自动挂载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

如果在 pod 和 ServiceAccount 中同时设置了 `automountServiceAccountToken`, pod 设置中的优先级更高。

## 使用多个ServiceAccount

每个 namespace 中都有一个默认的叫做 `default` 的 ServiceAccount 资源。

你可以使用以下命令列出 namespace 下的所有 serviceAccount 资源。

```bash
$ kubectl get serviceaccounts
NAME      SECRETS    AGE
default   1          1d
```

你可以像这样创建一个 ServiceAccount 对象：

```bash
$ cat > /tmp/serviceaccount.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF
$ kubectl create -f /tmp/serviceaccount.yaml
serviceaccount "build-robot" created
```

如果你看到如下的 ServiceAccount 对象的完整输出信息：

```bash
$ kubectl get serviceaccounts/build-robot -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-06-16T00:12:59Z
  name: build-robot
  namespace: default
  resourceVersion: "272500"
  selfLink: /api/v1/namespaces/default/serviceaccounts/build-robot
  uid: 721ab723-13bc-11e5-aec2-42010af0021e
secrets:
- name: build-robot-token-bvbk5
```

然后你将看到有一个 token 已经被自动创建，并被 ServiceAccount 引用。

你可以使用授权插件来 [设置 ServiceAccount 的权限](https://kubernetes.io/docs/admin/authorization/#a-quick-note-on-service-accounts) 。

设置非默认的 ServiceAccount，只需要在 pod 的 `spec.serviceAccountName` 字段中将name设置为你想要用的 ServiceAccount 名字即可。

在 pod 创建之初 ServiceAccount 就必须已经存在，否则创建将被拒绝。

你不能更新已创建的 pod 的 ServiceAccount。

你可以清理 ServiceAccount，如下所示：

```bash
$ kubectl delete serviceaccount/build-robot
```

## 手动创建 ServiceAccount 的 API token

假设我们已经有了一个如上文提到的名为 ”build-robot“ 的 ServiceAccount，我们手动创建一个新的 secret。

```bash
$ cat > /tmp/build-robot-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
EOF
$ kubectl create -f /tmp/build-robot-secret.yaml
secret "build-robot-secret" created
```

现在你可以确认下新创建的 secret 取代了 “build-robot” 这个 ServiceAccount 原来的 API token。

所有已不存在的 ServiceAccount 的 token 将被 token controller 清理掉。

```bash
$ kubectl describe secrets/build-robot-secret
Name:   build-robot-secret
Namespace:  default
Labels:   <none>
Annotations:  kubernetes.io/service-account.name=build-robot,kubernetes.io/service-account.uid=870ef2a5-35cf-11e5-8d06-005056b45392

Type: kubernetes.io/service-account-token

Data
====
ca.crt: 1220 bytes
token: ...
namespace: 7 bytes
```

{{<callout note 注意>}}
该内容中的`token`被省略了。
{{</callout>}}

## 为 ServiceAccount 添加 ImagePullSecret

首先，创建一个 imagePullSecret，详见[这里](https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod)。

然后，确认已创建。如：

```bash
$ kubectl get secrets myregistrykey
NAME             TYPE                              DATA    AGE
myregistrykey    kubernetes.io/.dockerconfigjson   1       1d
```

然后，修改 namespace 中的默认 ServiceAccount 使用该 secret 作为 imagePullSecret。

```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "myregistrykey"}]}'
```

Vi 交互过程中需要手动编辑：

```bash
$ kubectl get serviceaccounts default -o yaml > ./sa.yaml
$ cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  resourceVersion: "243024"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
$ vi sa.yaml
[editor session not shown]
[delete line with key "resourceVersion"]
[add lines with "imagePullSecret:"]
$ cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
imagePullSecrets:
- name: myregistrykey
$ kubectl replace serviceaccount default -f ./sa.yaml
serviceaccounts/default
```

现在，所有当前 namespace 中新创建的 pod 的 spec 中都会增加如下内容：

```yaml
spec:
  imagePullSecrets:
  - name: myregistrykey
```

## 参考

- [Configure Service Accounts for Pods - kubernetes.io](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
