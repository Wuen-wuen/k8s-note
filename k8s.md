## Pod详解

### Pod的创建过程

在 Kubernetes 中，Pod 的创建是一个多组件协同的过程，涉及 API Server、调度器（kube-scheduler）、kubelet、容器运行时（如 containerd）等核心组件。以下是 Pod 创建的完整流程，从用户提交请求到容器启动的每一步细节：  

**一、触发Pod创建的方式**

1. 直接提交 Pod 配置清单（`kubectl apply -f pod.yaml`）。
2. 由工作负载控制器（Deployment、StatefulSet、DaemonSet 等）自动创建（控制器根据 `replicas` 字段确保期望副本数）。
3. 临时任务（Job）或定时任务（CronJob）创建的 Pod。

**二、Pod 创建的完整流程**

1. **用户提交创建请求**

   - 用户通过 `kubectl` 命令（如 `kubectl apply -f pod.yaml`）或 Kubernetes API 向 **API Server** 发送 Pod 创建请求。
   - 请求内容是包含 Pod 配置的 YAML/JSON 数据（如容器镜像、资源限制、标签等）。
2. **API Server 验证并存储 Pod 信息**

   - **验证请求**：API Server 首先检查请求的合法性（如字段格式、权限、命名规则等），不合法则直接返回错误。	
   - **存储到 etcd**：验证通过后，API Server 将 Pod 的配置信息（`spec`）存储到分布式存储 **etcd** 中，但此时 Pod 状态为 `Pending`（未调度），`status` 字段为空。
   - **返回确认**：API Server 向用户返回 “创建成功” 的响应（此时 Pod 仅在 etcd 中存在记录，尚未实际运行）。
3. **调度器（kube-scheduler）绑定节点**
   - **监听待调度 Pod**：调度器通过监听 API Server 中状态为 `Pending` 且未分配节（`spec.nodeName` 为空）的 Pod。
   - **筛选合适节点**：调度器对所有可用节点执行 两步调度算法：
     - **预选（Predicates）**：过滤不满足 Pod 要求的节点（如资源不足、节点污点与 Pod 容忍度不匹配、节点亲和性规则等）。
     - **优选（Priorities）**：对预选通过的节点打分（如资源剩余量、负载均衡、节点亲和性权重等），选择得分最高的节点。
   - **绑定节点**：调度器向 API Server 发送 **绑定请求**，将选中的节点名称写入 Pod 的 `spec.nodeName` 字段，API Server 更新 etcd 中 Pod 的配置。
4. **kubelet 发现并处理本地 Pod**
   - **监听本地 Pod**：每个节点上的 **kubelet** 会定期向 API Server 查询 “分配给本节点的 Pod”（通过 `spec.nodeName` 匹配自身节点名）。
   - **检查 Pod 状态**：kubelet 发现新分配的 Pod 后，检查其状态是否为 `Pending`，确认需要在本地创建。
5. **kubelet 准备 Pod 运行环境**
   - **创建网络命名空间**：为 Pod 分配独立的网络命名空间（确保 Pod 内容器共享网络栈）。
   - **配置网络**：调用 **CNI 插件**（如 Calico、Flannel）为 Pod 分配 IP 地址、配置路由、设置 veth 对（连接 Pod 与节点网络），确保 Pod 能与集群内其他 Pod 通信。
   - **挂载存储卷**：根据 Pod 的 `spec.volumes` 配置，挂载所需的存储（如 emptyDir、ConfigMap、Secret、PersistentVolume 等），确保容器能访问指定的文件或数据。
6. **拉取容器镜像**
   - kubelet 通过 容器运行时接口（CRI） 调用容器运行时（如 containerd、CRI-O），根据 Pod 中 spec.containers.image 字段拉取所需镜像
7. **启动容器**
   - 容器运行时根据kubelet传递的配置（如镜像、环境变量、资源限制、挂载卷等）启动容器：
     - 先启动 **init容器**（若有），按顺序执行，全部成功后才启动主容器；
     - 启动主容器（`containers` 字段定义的容器），多个主容器并行启动。
   - 容器启动后，容器运行时向 kubelet 反馈容器状态（如 `Running`、`Error`）
8. **更新 Pod 状态并持续监控**
   - **更新状态**：kubelet 收集容器运行状态，向 API Server 发送 Pod 状态更新请求（如 `status.phase` 改为 `Running`，`status.conditions` 记录容器就绪状态等），API Server 更新 etcd 中的 Pod 状态。
   - **持续监控**：kubelet 通过 **容器运行时** 持续监控容器状态（如健康检查 `livenessProbe`、就绪检查 `readinessProbe`），若容器异常退出，根据重启策略（`restartPolicy`）处理。
   - 用户可见：此时通过 `kubectl get pods` 可看到 Pod 状态为 `Running`，表示创建完成。











