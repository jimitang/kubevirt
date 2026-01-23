# KubeVirt 快速安装和测试指南

本指南提供在 Kubernetes 环境中安装和测试 KubeVirt 的完整步骤。

## 目录
- [方式一：使用 KubeVirt 内置测试集群（推荐用于快速测试）](#方式一使用-kubevirt-内置测试集群)
- [方式二：在现有 Kubernetes 集群上安装](#方式二在现有-kubernetes-集群上安装)
- [创建和管理虚拟机](#创建和管理虚拟机)
- [常用操作](#常用操作)

---

## 方式一：使用 KubeVirt 内置测试集群

这是最快速的测试方式，KubeVirt 项目提供了一键启动测试集群的功能。

### 1. 前置条件

确保系统已安装：
- **Docker** 或 **Podman**（容器运行时）
- **rsync**（文件同步工具）
- **嵌套虚拟化支持**（如果在虚拟机中运行）

```bash
# 检查 Docker
docker --version

# 或检查 Podman
podman --version

# 检查 rsync
rsync --version

# 检查是否支持嵌套虚拟化（输出应该 > 0）
cat /sys/module/kvm_intel/parameters/nested  # Intel CPU
# 或
cat /sys/module/kvm_amd/parameters/nested    # AMD CPU
```

如果使用 Podman，需要启用 socket：
```bash
sudo systemctl enable podman.socket
sudo systemctl start podman.socket
```

### 2. 启动测试集群

```bash
# 进入 kubevirt 项目目录
cd /home/user/kubevirt

# 设置 kubevirtci 版本（可选，使用最新版本）
export KUBEVIRTCI_TAG=$(curl -L -Ss https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirtci/latest)

# 启动 Kubernetes 集群（默认使用 kind-1.34）
make cluster-up

# 等待集群启动完成...
```

**可选：指定 Kubernetes 版本**
```bash
# 使用特定的 K8s 版本
export KUBEVIRT_PROVIDER=k8s-1.33
make cluster-up
```

### 3. 配置 kubectl

```bash
# 设置 kubeconfig 访问集群
export KUBECONFIG=$(./kubevirtci/cluster-up/kubeconfig.sh)

# 验证集群连接
kubectl cluster-info
kubectl get nodes
```

### 4. 部署 KubeVirt

```bash
# 构建并部署 KubeVirt 到测试集群
make cluster-sync

# 这个命令会：
# 1. 编译 KubeVirt 组件
# 2. 构建容器镜像
# 3. 部署到集群
# 4. 等待所有 Pod 就绪
```

### 5. 验证安装

```bash
# 查看 KubeVirt 组件状态
kubectl get pods -n kubevirt

# 应该看到类似输出：
# NAME                               READY   STATUS    RESTARTS   AGE
# virt-api-xxxxx                     1/1     Running   0          2m
# virt-controller-xxxxx              1/1     Running   0          2m
# virt-handler-xxxxx                 1/1     Running   0          2m
# virt-operator-xxxxx                1/1     Running   0          3m

# 检查 KubeVirt 资源定义
kubectl get kubevirt -n kubevirt
kubectl api-resources | grep kubevirt
```

### 6. 清理集群

测试完成后，可以停止集群：
```bash
make cluster-down
```

---

## 方式二：在现有 Kubernetes 集群上安装

如果你已经有一个运行中的 Kubernetes 集群，可以直接安装 KubeVirt。

### 1. 前置条件

- Kubernetes 集群版本 >= 1.27
- kubectl 已配置并能访问集群
- 集群节点支持 KVM 虚拟化（或开启软件模拟）
- 集群有足够的资源（至少 4GB 内存，2 CPU）

```bash
# 验证 kubectl 连接
kubectl version
kubectl get nodes

# 检查节点是否支持虚拟化扩展
kubectl get nodes -o jsonpath='{.items[*].status.allocatable}' | grep devices.kubevirt.io
```

### 2. 安装 KubeVirt Operator

选择合适的 KubeVirt 版本，访问 [KubeVirt Releases](https://github.com/kubevirt/kubevirt/releases)

```bash
# 设置 KubeVirt 版本
export KUBEVIRT_VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)

# 或手动指定版本
export KUBEVIRT_VERSION=v1.4.0

# 安装 KubeVirt Operator
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml

# 等待 operator 就绪
kubectl wait --for=condition=Available --timeout=300s -n kubevirt kube-virt operator
```

### 3. 创建 KubeVirt 自定义资源

```bash
# 部署 KubeVirt CR（触发组件安装）
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml

# 等待所有组件就绪（需要几分钟）
kubectl wait --for=condition=Available --timeout=600s -n kubevirt kubevirt kubevirt
```

### 4. 验证安装

```bash
# 检查所有 Pod 运行状态
kubectl get pods -n kubevirt

# 查看 KubeVirt 版本和状态
kubectl get kubevirt -n kubevirt -o yaml

# 验证 CRD 已注册
kubectl get crd | grep kubevirt.io
```

### 5. 安装 virtctl（可选但推荐）

virtctl 是 KubeVirt 的命令行工具，提供 VM 特定的操作。

```bash
# 下载 virtctl
VERSION=$(kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.observedKubeVirtVersion}")
ARCH=$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/') || windows-amd64.exe
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-${ARCH}

# 添加执行权限
chmod +x virtctl

# 移动到 PATH
sudo mv virtctl /usr/local/bin/

# 验证安装
virtctl version
```

---

## 创建和管理虚拟机

### 示例 1：创建临时虚拟机实例（VMI）

创建一个简单的 CirrOS 虚拟机实例（临时，删除后数据丢失）：

```yaml
# vmi-test.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vmi-test
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
    memory:
      guest: 128Mi
    resources:
      requests:
        memory: 128Mi
  terminationGracePeriodSeconds: 0
  volumes:
  - containerDisk:
      # 使用公共镜像（生产环境请使用私有镜像）
      image: quay.io/kubevirt/cirros-container-disk-demo:latest
    name: containerdisk
```

创建并查看：
```bash
# 创建 VMI
kubectl apply -f vmi-test.yaml

# 查看 VMI 状态
kubectl get vmi
kubectl get vmi vmi-test -o yaml

# 查看关联的 Pod
kubectl get pods | grep virt-launcher

# 删除 VMI
kubectl delete vmi vmi-test
```

### 示例 2：创建有状态虚拟机（VM）

有状态虚拟机可以停止和启动，保留数据和状态：

```yaml
# vm-test.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-test
spec:
  running: false  # 初始状态为停止
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-test
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
        memory:
          guest: 256Mi
        resources:
          requests:
            memory: 256Mi
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo:latest
        name: containerdisk
      - cloudInitNoCloud:
          userData: |
            #!/bin/sh
            echo 'Hello from KubeVirt!'
            echo 'VM started at:' $(date)
        name: cloudinitdisk
```

创建和管理：
```bash
# 创建 VM（不会自动启动）
kubectl apply -f vm-test.yaml

# 查看 VM 状态
kubectl get vm

# 启动 VM
kubectl patch vm vm-test --type merge -p '{"spec":{"running":true}}'
# 或使用 virtctl
virtctl start vm-test

# 查看运行中的 VMI
kubectl get vmi

# 停止 VM
virtctl stop vm-test

# 重启 VM
virtctl restart vm-test

# 删除 VM
kubectl delete vm vm-test
```

---

## 常用操作

### 1. 连接到虚拟机控制台

```bash
# VNC 控制台（图形界面）
virtctl vnc vm-test

# 串口控制台（文本界面）
virtctl console vm-test

# 退出控制台：按 Ctrl+] 或 Ctrl+5
```

### 2. SSH 到虚拟机

```bash
# 通过 virtctl 代理 SSH
virtctl ssh user@vm-test

# 或通过端口转发
virtctl port-forward vm-test 2222:22
ssh -p 2222 user@localhost
```

### 3. 查看虚拟机信息

```bash
# 查看 VM 详细信息
kubectl describe vm vm-test

# 查看 VMI 详细信息
kubectl describe vmi vm-test

# 查看 VM 事件
kubectl get events --field-selector involvedObject.name=vm-test

# 查看 virt-launcher Pod 日志
kubectl logs virt-launcher-vm-test-xxxxx
```

### 4. 虚拟机生命周期管理

```bash
# 启动
virtctl start vm-test

# 停止
virtctl stop vm-test

# 重启
virtctl restart vm-test

# 暂停
virtctl pause vmi vm-test

# 恢复
virtctl unpause vmi vm-test

# 迁移（实时迁移到其他节点）
virtctl migrate vm-test

# 查看迁移状态
kubectl get virtualmachineinstancemigration
```

### 5. 虚拟机快照和克隆

```bash
# 创建快照
kubectl apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  name: vm-test-snapshot
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-test
EOF

# 查看快照
kubectl get vmsnapshot

# 从快照恢复
kubectl apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: vm-test-restore
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-test
  virtualMachineSnapshotName: vm-test-snapshot
EOF
```

### 6. 监控和指标

```bash
# 查看 VM 资源使用
kubectl top pod -l kubevirt.io/vm=vm-test

# 获取虚拟机客户端代理信息
virtctl guestosinfo vm-test

# 查看 Prometheus 指标（如果已安装）
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods/virt-launcher-vm-test-xxxxx
```

---

## 故障排查

### 常见问题

#### 1. VMI 无法启动

```bash
# 检查事件
kubectl get events --sort-by='.lastTimestamp'

# 检查 virt-handler 日志
kubectl logs -n kubevirt -l kubevirt.io=virt-handler

# 检查节点是否支持虚拟化
kubectl describe node <node-name> | grep cpu
```

#### 2. 虚拟机网络不通

```bash
# 检查 VMI 网络配置
kubectl get vmi vm-test -o jsonpath='{.status.interfaces}'

# 检查 Pod 网络
kubectl get pods -o wide | grep virt-launcher
```

#### 3. 性能问题

```bash
# 检查资源限制
kubectl describe vmi vm-test | grep -A5 Resources

# 查看节点资源使用
kubectl top nodes
kubectl top pods
```

### 开启软件模拟（如果没有 KVM）

如果节点不支持硬件虚拟化，可以开启软件模拟（性能较低）：

```bash
kubectl -n kubevirt patch kubevirt kubevirt --type=merge --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

---

## 更多资源

- **官方文档**: https://kubevirt.io/user-guide/
- **API 参考**: https://kubevirt.io/api-reference/
- **示例库**: `/home/user/kubevirt/examples/`
- **社区支持**:
  - Slack: #virtualization @ kubernetes.slack.com
  - GitHub Issues: https://github.com/kubevirt/kubevirt/issues

---

## 快速测试脚本

将以下脚本保存为 `test-kubevirt.sh` 快速验证 KubeVirt 安装：

```bash
#!/bin/bash
set -e

echo "🔍 检查 KubeVirt 组件..."
kubectl get pods -n kubevirt

echo -e "\n✅ 创建测试虚拟机..."
cat <<EOF | kubectl apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: test-vmi
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
    memory:
      guest: 128Mi
    resources:
      requests:
        memory: 128Mi
  volumes:
  - containerDisk:
      image: quay.io/kubevirt/cirros-container-disk-demo:latest
    name: containerdisk
EOF

echo "⏳ 等待虚拟机启动..."
kubectl wait --for=condition=Ready vmi/test-vmi --timeout=300s

echo -e "\n📊 虚拟机状态："
kubectl get vmi test-vmi

echo -e "\n🎉 测试成功！使用以下命令连接虚拟机控制台："
echo "   virtctl console test-vmi"
echo -e "\n🧹 清理测试虚拟机："
echo "   kubectl delete vmi test-vmi"
```

运行测试：
```bash
chmod +x test-kubevirt.sh
./test-kubevirt.sh
```
