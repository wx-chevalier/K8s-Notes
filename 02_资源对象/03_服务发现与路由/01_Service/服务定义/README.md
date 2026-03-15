# Kubernetes Service 详解

## 概述

Service 是 Kubernetes 中的一个 REST 对象，用于为一组 Pod 提供统一的访问入口。它使用标签选择器来确定目标 Pod，并为这些 Pod 提供负载均衡和服务发现功能。

## Service 类型

Kubernetes 支持以下四种 Service 类型：

1. **ClusterIP** (默认)

   - 分配集群内部可访问的虚拟 IP
   - 仅在集群内部可访问

2. **NodePort**

   - 基于 ClusterIP
   - 在每个节点上绑定固定端口
   - 可通过 `<NodeIP>:NodePort` 访问

3. **LoadBalancer**

   - 基于 NodePort
   - 创建外部负载均衡器
   - 适用于云环境

4. **ExternalName**
   - 通过 CNAME 记录映射到外部服务
   - 要求 Kubernetes 1.7+ 版本
   - 不创建代理

## Service 端口详解

### 三种端口的定义与关系

1. **port**

   - Service 的端口
   - 集群内部访问 Service 使用的端口
   - 必须指定

2. **targetPort**

   - Pod 的端口
   - 实际应用程序监听的端口
   - 如果不指定，默认与 port 相同

3. **nodePort**
   - Node 节点的端口
   - 外部访问 Service 使用的端口
   - 范围：30000-32767
   - 仅在 type: NodePort 时使用

### 端口访问流程图

```plaintext
                                            +-------------------+
            外部访问                         |                   |
    +--------------------------+            |      Node IP      |
    |      nodePort:30163     |            |   192.168.1.100   |
    +--------------------------+            |                   |
                ↓                          +-------------------+
                |                                    ↓
        +---------------+                    +---------------+
        |  port:8080   |   集群内部访问     |    Service    |
        +---------------+ ---------------→   +---------------+
                ↓                                    ↓
        +---------------+                    +---------------+
        |targetPort:80  |                   |      Pod      |
        +---------------+                   +---------------+
```

### 访问方式示例

1. **集群内部访问**

   ```bash
   # 通过 Service IP 访问
   curl http://10.0.0.1:8080

   # 通过 Service 名称访问
   curl http://my-service:8080
   ```

2. **集群外部访问**
   ```bash
   # 通过任意节点 IP 访问
   curl http://192.168.1.100:30163
   ```

## Service 配置示例

### 基础配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
    - port: 8080 # 集群内访问端口
      nodePort: 30163 # 节点访问端口
      targetPort: 80 # 容器端口
```

### 多端口配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http # 多端口时必须命名
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

## 高级特性

### 1. 无选择器服务

用于以下场景：

- 连接外部数据库
- 跨命名空间服务
- 迁移遗留系统

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  # 无 selector
```

需要手动配置 Endpoints：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```

### 2. ExternalName 服务

用于映射外部服务：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.database.example.com
```

## 最佳实践

1. 为多端口服务提供清晰的端口名称
2. 使用适当的 Service 类型匹配使用场景
3. 合理设置 selector 标签
4. 注意端口映射关系
5. 端口命名规范
   ```yaml
   ports:
     - name: http # 给端口命名，便于管理
       port: 8080
       targetPort: 80
   ```
6. 使用命名端口
   ```yaml
   ports:
     - port: 8080
       targetPort: http # 可以引用 Pod 中的命名端口
   ```

## 常见问题与注意事项

1. **端口访问问题**

   - 检查防火墙规则
   - 确认 Pod 是否正常运行
   - 验证 Service selector 是否正确

2. **端口限制**

   - NodePort 的默认范围是 30000-32767
   - 可通过 API Server 的 `--service-node-port-range` 参数修改

3. **其他注意事项**
   - LoadBalancer 类型需要云提供商支持
   - Endpoint IP 不能使用 loopback、link-local 或 link-local 多播地址
   - 避免端口冲突，nodePort 不指定时系统自动分配
   - 确保 targetPort 与容器实际监听端口一致
