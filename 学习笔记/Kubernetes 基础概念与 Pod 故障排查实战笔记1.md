# Kubernetes 核心概念与故障排查笔记

## 一、PVC（PersistentVolumeClaim）

- **PVC** 是持久卷声明，像一份存储“申请单”。

- 它解耦了 Pod 与底层存储（PV），Pod 通过 PVC 使用存储，无需关心具体实现。

- 示例 YAML：

  ```
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: my-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  ```

  

## 二、Pod 循环重启的原因

1. **资源不足**
   - 内存超限 → `OOMKilled` → 反复被杀重启。
   - CPU 极度匮乏 → 健康检查超时 → 被 kubelet 杀死。
   - 节点磁盘/进程数压力导致驱逐。
2. **依赖未就绪**
   - 启动时连接数据库、缓存等失败，之后依赖就绪，删除 Pod 重建后成功。
3. **排查方式**
   - 查看 `Last State` 原因、退出码。
   - 检查 `--previous` 日志。
   - 查看 Pod 与节点资源使用（`kubectl top`）。
   - 检查节点状态（`kubectl describe node`）。

## 三、Node 与 Pod

- **Node**：运行 Pod 的机器（物理机/虚拟机），提供计算、网络、存储资源。
- **Pod**：最小的调度单位，包含一个或多个容器，运行在 Node 上。
- **为什么需要多 Worker 节点？**
  - 高可用与容灾：节点宕机后 Pod 可调度到其他节点。
  - 横向扩展：突破单机资源上限。
  - 负载均衡与隔离。
- **Node 不负责调度**，调度由 Master 上的 `kube-scheduler` 完成。Node 上的 `kubelet` 负责接收指令、管理容器。

## 四、健康检查（探针）

- **Liveness Probe**：检查容器是否存活，失败则重启容器。
- **Readiness Probe**：检查容器是否就绪，决定是否加入 Service 负载均衡。
- **Startup Probe**：保护启动慢的容器，避免过早被存活探针误杀。

## 五、Master 与 Worker 节点

- **Master（控制平面）**：集群大脑，管理集群状态，包含 API Server、etcd、Scheduler、Controller Manager。
- **Worker**：执行者，运行应用容器，包含 kubelet、kube-proxy、容器运行时。
- 关系：Master 决策调度，Worker 执行并上报状态。生产环境两者分离，单机集群可合一。

## 六、kubectl 命令的运行位置

- `kubectl` 是客户端工具，**可在任何能连通 API Server 的机器上执行**，无需一定在 Node 上。
- 它通过 kubeconfig 连接集群，信息来自 Master，不直接访问 Node。

## 七、仅知 Pod IP 和 Node IP 时的排查流程

1. 通过 Pod IP 反查 Pod 名称和命名空间：

   ```
   kubectl get pods --all-namespaces -o wide | grep "<pod-ip>"
   ```

   

2. 获得 Pod 名和 namespace 后，查看详细状态：

   ```
   kubectl describe pod <pod-name> -n <namespace>
   ```

   

3. 若无法使用 kubectl 远程访问，可 SSH 到对应 Node 上，使用节点自带 kubectl（如 `kubelet.conf`）或 `crictl` 检查容器日志、内核 OOM 记录。

## 八、查看容器退出原因（Last State）

`kubectl describe pod` 输出中 `Containers` 段落包含 `Last State`：

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```



- `Reason: OOMKilled` → 内存不足被杀。
- `Reason: Error` + 非零退出码 → 应用自身错误退出。
- `Reason: Evicted` → 节点资源压力被驱逐。

**快速提取关键信息：**

```
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Last State"
```



## 九、命令集

### Pod 信息查询

```
# 查看 Pod 详细状态（含 Last State）
kubectl describe pod <pod-name> -n <namespace>

# 过滤上次退出原因
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Last State"

# 一次性查看状态、退出码和重启次数
kubectl describe pod <pod-name> -n <namespace> | grep -E "State:|Last State:|Reason:|Exit Code:|Restart Count:"

# 查看 Pod 资源实时用量
kubectl top pod <pod-name> -n <namespace>
```



### 通过 IP 查找 Pod

```
# 列出所有命名空间，按 Pod IP 过滤
kubectl get pods --all-namespaces -o wide | grep "<pod-ip>"

# 使用 field-selector 直接过滤（部分集群支持）
kubectl get pods --all-namespaces --field-selector status.podIP=<pod-ip>
```



### 日志查看

```
# 当前容器日志
kubectl logs <pod-name> -c <container-name>

# 上一次崩溃容器的日志
kubectl logs <pod-name> -c <container-name> --previous
```



### 节点与资源状态

```
# 查看节点资源使用
kubectl top node

# 查看节点详细状况（含 MemoryPressure、DiskPressure 等）
kubectl describe node <node-name> | grep -A 5 "Conditions"
```



### 集群事件（Pod 被删除后仍可追查）

```
# 查看涉及特定 Pod 的事件
kubectl get events --all-namespaces --field-selector involvedObject.name=<pod-name>

# 过滤 OOM 相关事件
kubectl get events --all-namespaces --field-selector involvedObject.name=<pod-name> | grep -i oom
```



### Node 上直接排查（无 kubectl 远程权限时）

```
# 使用节点上的 kubectl（需指定 kubeconfig）
kubectl --kubeconfig /etc/kubernetes/kubelet.conf get pods --all-namespaces -o wide | grep <pod-ip>

# 使用 crictl 查看容器（需知道容器运行时端点）
crictl ps -a | grep <pod-name或ID>
crictl logs <container-id>
crictl pods | grep <pod-uid>

# 查看内核 OOM 记录
dmesg | grep -i oom
```