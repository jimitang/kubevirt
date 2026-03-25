# KubeVirt 网络架构详解

本文档详细介绍 KubeVirt 的网络架构、绑定机制和各种网络模式。

## 目录
- [核心概念](#核心概念)
- [网络架构概览](#网络架构概览)
- [网络配置流程](#网络配置流程)
- [绑定机制 (Binding Mechanisms)](#绑定机制-binding-mechanisms)
- [网络模式详解](#网络模式详解)
- [高级网络功能](#高级网络功能)
- [实践示例](#实践示例)

---

## 核心概念

### Pod 网络 vs VM 网络

**关键理解**：必须区分两个概念：

```
┌─────────────────────────────────────────────┐
│  Pod 网络配置                                │
│  - 由 Kubernetes CNI 负责                    │
│  - 为 virt-launcher Pod 配置网络接口         │
│  - KubeVirt 不负责这部分                     │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│  VM 网络配置 (Binding)                       │
│  - 由 KubeVirt 负责                          │
│  - 将 Pod 的网络接口"绑定"到虚拟机           │
│  - 这是本文档的重点                          │
└─────────────────────────────────────────────┘
```

### 网络配置的两个阶段

KubeVirt 遵循**最小权限原则**，将 VM 网络配置分为两个阶段：

| 阶段 | 执行组件 | 权限 | 职责 |
|------|---------|------|------|
| **Phase 1: 特权配置** | virt-handler (DaemonSet) | 特权 | 创建网络基础设施（bridge、tap等） |
| **Phase 2: 非特权配置** | virt-launcher (Pod) | 最小权限 | 生成 libvirt domain XML 配置 |

---

## 网络架构概览

### 整体架构图

```
┌────────────────────────────────────────────────────────────┐
│  Kubernetes 集群                                            │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  virt-launcher Pod (每个 VM 一个)                     │ │
│  │                                                        │ │
│  │  ┌──────────────┐    ┌──────────────┐                │ │
│  │  │ eth0 (Pod网络)│    │ k6t-eth0     │                │ │
│  │  │ 10.244.1.5   │    │ (bridge)     │                │ │
│  │  └───────┬──────┘    └──────┬───────┘                │ │
│  │          │                   │                         │ │
│  │          │            ┌──────┴───────┐                │ │
│  │          │            │ tap device   │                │ │
│  │          │            │ (vnet0)      │                │ │
│  │          │            └──────┬───────┘                │ │
│  │          │                   │                         │ │
│  │  ┌───────┴───────────────────┴──────────────────────┐ │ │
│  │  │  libvirt + QEMU                                  │ │ │
│  │  │  ┌────────────────────────────────────────────┐  │ │ │
│  │  │  │  Virtual Machine (VM)                      │  │ │ │
│  │  │  │  ┌──────────────┐                          │  │ │ │
│  │  │  │  │ eth0 (VM)    │                          │  │ │ │
│  │  │  │  │ 10.244.1.5   │                          │  │ │ │
│  │  │  │  └──────────────┘                          │  │ │ │
│  │  │  │  Guest OS + Application                    │  │ │ │
│  │  │  └────────────────────────────────────────────┘  │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────┘ │
│                         ↑                                   │
│                   CNI Plugin 配置                           │
└─────────────────────────────────────────────────────────────┘
```

### 组件职责

```
┌─────────────────┐
│ Kubernetes CNI  │  → 配置 Pod 网络（eth0）
└────────┬────────┘
         ↓
┌─────────────────┐
│ virt-handler    │  → 创建 bridge、配置网络基础设施
│ (特权组件)      │     (在 virt-launcher 的 netns 中操作)
└────────┬────────┘
         ↓
┌─────────────────┐
│ virt-launcher   │  → 生成 libvirt domain XML
│ (非特权组件)    │     启动 QEMU/KVM
└────────┬────────┘
         ↓
┌─────────────────┐
│ libvirt/QEMU    │  → 创建 tap 设备，连接到 bridge
└────────┬────────┘
         ↓
┌─────────────────┐
│ Virtual Machine │  → 使用虚拟网卡通信
└─────────────────┘
```

---

## 网络配置流程

### Phase 1: 特权网络配置 (virt-handler)

**执行位置**：virt-handler（在目标 virt-launcher 的 network namespace 中）

**流程**：

```
1. discoverPodNetworkInterface()
   ├─ 读取 Pod 网络接口信息
   ├─ 获取 IP 地址
   ├─ 获取 MAC 地址（仅 bridge 模式）
   ├─ 获取路由信息（仅 bridge 模式）
   ├─ 获取 Gateway
   └─ 获取 MTU

2. preparePodNetworkInterfaces()
   ├─ 根据不同的 Binding Mechanism 执行特定操作
   ├─ 创建 bridge（如果需要）
   ├─ 配置 IP 地址（移除/保留/修改）
   └─ 设置 MAC 地址

3. setCachedInterface()
   └─ 将接口信息缓存到内存

4. setCachedVIF()
   └─ 将 VIF 对象持久化到文件系统
      位置: /proc/<virt-launcher-pid>/root/var/run/kubevirt-private/vif-cache-<iface_name>.json
```

### Phase 2: 非特权网络配置 (virt-launcher)

**执行位置**：virt-launcher Pod（非特权）

**流程**：

```
1. loadCachedInterface()
   └─ 从内存加载接口信息

2. loadCachedVIF()
   └─ 从文件系统加载 VIF 对象

3. decorateConfig()
   ├─ 生成 libvirt domain XML 的 interface 部分
   ├─ 设置 MAC 地址
   ├─ 设置 MTU
   └─ 指定 bridge/network 连接

4. startDHCP() (可选)
   └─ 启动 in-pod DHCP 服务器（某些模式需要）
```

---

## 绑定机制 (Binding Mechanisms)

绑定机制是 **KubeVirt API 和 libvirt domain XML 之间的转换服务**。

### BindMechanism 接口

```go
type BindMechanism interface {
    // Phase 1: 特权操作
    discoverPodNetworkInterface() error
    preparePodNetworkInterfaces() error
    setCachedInterface(pid, name string) error
    setCachedVIF(pid, name string) error

    // Phase 2: 非特权操作
    loadCachedInterface(pid, name string) (bool, error)
    loadCachedVIF(pid, name string) (bool, error)
    decorateConfig() error
    startDHCP(vmi *v1.VirtualMachineInstance) error
}
```

### 三种主要绑定机制

| 绑定机制 | 使用场景 | 网络拓扑 | IP 分配 |
|---------|---------|---------|---------|
| **Bridge** | VM 直接暴露在 Pod 网络 | VM 与 Pod 共享同一网段 | VM 使用 Pod IP |
| **Masquerade** | VM 在 NAT 网络中（默认） | VM 在私有网段，通过 NAT 访问外部 | VM 使用私有 IP |
| **Slirp** | 用户态网络（无需特权） | 完全用户态实现 | 无需特殊权限 |

---

## 网络模式详解

### 1. Bridge 模式

**最直接的网络模式**：VM 直接连接到 Pod 网络。

#### 工作原理

```
┌─────────────────────────────────────────┐
│ virt-launcher Pod                       │
│                                         │
│  eth0 (Pod网络接口, CNI配置)            │
│  └─ IP: 10.244.1.5 (从 CNI 移除)       │
│  └─ MAC: aa:bb:cc:dd:ee:ff              │
│      ↓                                  │
│  k6t-eth0 (bridge)                      │
│      ├─ eth0 (port 1, 随机MAC)         │
│      └─ vnet0 (port 2, VM的tap)        │
│          ↓                              │
│  ┌─────────────────────┐                │
│  │ VM                  │                │
│  │  eth0               │                │
│  │  IP: 10.244.1.5     │ ← DHCP分配    │
│  │  MAC: aa:bb:cc:dd:ee:ff              │
│  └─────────────────────┘                │
└─────────────────────────────────────────┘
```

#### 配置示例

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-bridge
spec:
  domain:
    devices:
      interfaces:
      - name: default
        bridge: {}  # 使用 bridge 模式
  networks:
  - name: default
    pod: {}  # 使用 Pod 网络
```

#### libvirt domain XML

```xml
<interface type='bridge'>
  <mac address='aa:bb:cc:dd:ee:ff'/>
  <source bridge='k6t-eth0'/>
  <target dev='vnet0'/>
  <model type='virtio'/>
  <mtu size='1440'/>
</interface>
```

#### 特点

✅ **优点**：
- VM 直接暴露在 Kubernetes 网络中
- 其他 Pod 可以直接访问 VM IP
- 性能好，没有额外的 NAT 开销

❌ **缺点**：
- VM 使用 Pod 的 IP（IP 地址从 eth0 移除，给 VM 使用）
- 不支持端口映射
- 迁移时 IP 可能改变

---

### 2. Masquerade 模式（默认推荐）

**VM 在私有网络中，通过 NAT 访问外部**。

#### 工作原理

```
┌────────────────────────────────────────────────────┐
│ virt-launcher Pod                                  │
│                                                    │
│  eth0 (Pod网络接口)                                │
│  └─ IP: 10.244.1.5 (保留给Pod使用)                 │
│      ↓                                             │
│  k6t-eth0 (bridge)                                 │
│  └─ IP: 10.0.2.1 (VM的网关)                        │
│      ↓                                             │
│  vnet0 (tap device)                                │
│      ↓                                             │
│  ┌────────────────────────┐                        │
│  │ VM                     │                        │
│  │  eth0                  │                        │
│  │  IP: 10.0.2.2/24       │ ← DHCP分配             │
│  │  Gateway: 10.0.2.1     │                        │
│  └────────────────────────┘                        │
│                                                    │
│  [iptables NAT规则]                                │
│  SNAT: 10.0.2.2 → 10.244.1.5                       │
│  DNAT: 10.244.1.5:port → 10.0.2.2:port             │
└────────────────────────────────────────────────────┘
```

#### 配置示例

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-masquerade
spec:
  domain:
    devices:
      interfaces:
      - name: default
        masquerade: {}  # 使用 masquerade 模式
        ports:          # 端口映射
        - name: http
          port: 80
          protocol: TCP
        - name: ssh
          port: 22
  networks:
  - name: default
    pod: {}
```

#### NAT 规则

```bash
# 出站流量（VM → 外部）
iptables -t nat -A POSTROUTING -s 10.0.2.2 -j MASQUERADE

# 入站流量（外部 → VM）
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 10.0.2.2:80
```

#### 特点

✅ **优点**：
- **VM 有独立的私有 IP**（不占用 Pod IP）
- **支持端口映射**（类似 Docker）
- Pod IP 保持不变，可用于其他服务
- 更好的网络隔离

❌ **缺点**：
- 需要通过端口映射才能从外部访问 VM
- 有 NAT 性能开销（通常可忽略）
- 配置稍复杂

**推荐场景**：大多数情况下的默认选择

---

### 3. Slirp 模式

**纯用户态网络实现**，无需任何特权。

#### 工作原理

```
┌─────────────────────────────────────┐
│ virt-launcher Pod                   │
│                                     │
│  QEMU 内置的用户态网络栈            │
│  ↓                                  │
│  ┌───────────────────┐              │
│  │ VM                │              │
│  │  eth0             │              │
│  │  IP: 10.0.2.15    │ ← QEMU分配  │
│  └───────────────────┘              │
│                                     │
│  所有网络流量通过 QEMU 进程转发     │
└─────────────────────────────────────┘
```

#### 配置示例

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-slirp
spec:
  domain:
    devices:
      interfaces:
      - name: default
        slirp: {}  # 使用 slirp 模式
  networks:
  - name: default
    pod: {}
```

#### 特点

✅ **优点**：
- **完全无需特权**
- 最简单的实现
- 安全隔离性好

❌ **缺点**：
- **性能最差**（用户态网络栈）
- 功能受限（不支持所有协议）
- 只适合测试和开发环境

---

### 模式对比总结

| 特性 | Bridge | Masquerade | Slirp |
|------|--------|-----------|-------|
| **VM IP** | Pod IP（共享） | 私有 IP（10.0.2.x） | 私有 IP（10.0.2.15） |
| **端口映射** | ❌ | ✅ | ✅ |
| **外部可访问性** | 直接访问 | 需要端口映射 | 需要端口映射 |
| **性能** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **特权要求** | 需要 | 需要 | 不需要 |
| **迁移友好** | ❌ IP 会变 | ✅ Pod IP 不变 | ✅ |
| **适用场景** | VM 直接暴露 | **默认推荐** | 测试/开发 |

---

## 高级网络功能

### 1. 多网络接口 (Multus CNI)

**使用 Multus 可以为 VM 添加多个网络接口**。

#### 架构

```
┌────────────────────────────────────────┐
│ VM                                     │
│  ┌────────┐  ┌────────┐  ┌────────┐   │
│  │ eth0   │  │ eth1   │  │ eth2   │   │
│  │(Pod网络)│  │(SR-IOV)│  │(Macvlan)│  │
│  └────────┘  └────────┘  └────────┘   │
└────────────────────────────────────────┘
       ↓            ↓            ↓
  masquerade    bridge       bridge
       ↓            ↓            ↓
    Pod网络      SR-IOV      Macvlan网络
```

#### 配置示例

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-multus
spec:
  domain:
    devices:
      interfaces:
      - name: default
        masquerade: {}      # 第一个网卡：Pod 网络
      - name: sriov-net
        bridge: {}          # 第二个网卡：SR-IOV
      - name: macvlan-net
        bridge: {}          # 第三个网卡：Macvlan
  networks:
  - name: default
    pod: {}
  - name: sriov-net
    multus:
      networkName: sriov-network
  - name: macvlan-net
    multus:
      networkName: macvlan-network
```

---

### 2. SR-IOV（高性能网络）

**Single Root I/O Virtualization** - 将物理网卡虚拟化为多个虚拟功能（VF）。

#### 架构

```
┌──────────────────────────────────────┐
│ 物理服务器                            │
│                                      │
│  ┌────────────────────────────────┐  │
│  │ 物理网卡 (SR-IOV capable)      │  │
│  │                                │  │
│  │  PF (Physical Function)        │  │
│  │   ├─ VF1 ────→ VM1             │  │
│  │   ├─ VF2 ────→ VM2             │  │
│  │   └─ VF3 ────→ VM3             │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

特点：
- 接近物理网卡性能
- 低延迟、高带宽
- 硬件加速
```

#### 配置示例

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-sriov
spec:
  domain:
    devices:
      interfaces:
      - name: default
        masquerade: {}
      - name: sriov
        sriov: {}  # SR-IOV 接口
  networks:
  - name: default
    pod: {}
  - name: sriov
    multus:
      networkName: sriov-network
```

---

### 3. 网络策略 (NetworkPolicy)

**KubeVirt VM 完全支持 Kubernetes NetworkPolicy**。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: vm-network-policy
spec:
  podSelector:
    matchLabels:
      kubevirt.io/vm: my-vm
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 3306
```

---

## 实践示例

### 示例 1：简单的 Bridge 网络 VM

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvm-bridge
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      interfaces:
      - name: default
        bridge: {}
    memory:
      guest: 512Mi
  networks:
  - name: default
    pod: {}
  volumes:
  - containerDisk:
      image: quay.io/kubevirt/cirros-container-disk-demo:latest
    name: containerdisk
```

**创建和测试**：

```bash
# 创建 VM
kubectl apply -f testvm-bridge.yaml

# 查看 VM IP
kubectl get vmi testvm-bridge -o jsonpath='{.status.interfaces[0].ipAddress}'

# 从其他 Pod 访问 VM
kubectl run test-pod --image=busybox -it --rm -- ping <VM-IP>
```

---

### 示例 2：Masquerade 网络 + 端口映射

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvm-web
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
      interfaces:
      - name: default
        masquerade: {}
        ports:
        - name: http
          port: 80
        - name: ssh
          port: 22
    memory:
      guest: 1Gi
  networks:
  - name: default
    pod: {}
  volumes:
  - containerDisk:
      image: quay.io/kubevirt/fedora-with-test-tooling-container-disk-demo:latest
    name: containerdisk
  - cloudInitNoCloud:
      userData: |
        #!/bin/bash
        yum install -y nginx
        systemctl enable --now nginx
    name: cloudinitdisk
```

**创建 Service 暴露服务**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: testvm-web-service
spec:
  selector:
    kubevirt.io/vm: testvm-web
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
# 访问 VM 的 Web 服务
kubectl get svc testvm-web-service
curl http://<EXTERNAL-IP>
```

---

### 示例 3：多网络接口 VM

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvm-multinet
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      interfaces:
      - name: default
        masquerade: {}
      - name: tenant-net
        bridge: {}
    memory:
      guest: 1Gi
  networks:
  - name: default
    pod: {}
  - name: tenant-net
    multus:
      networkName: tenant-network  # 需要预先创建 NetworkAttachmentDefinition
  volumes:
  - containerDisk:
      image: quay.io/kubevirt/cirros-container-disk-demo:latest
    name: containerdisk
```

---

## 网络调试

### 查看 VM 网络信息

```bash
# 查看 VMI 网络接口
kubectl get vmi testvm -o jsonpath='{.status.interfaces}' | jq

# 查看 virt-launcher Pod 网络
kubectl exec virt-launcher-testvm-xxxxx -- ip addr

# 查看 bridge 信息
kubectl exec virt-launcher-testvm-xxxxx -- brctl show

# 查看 iptables NAT 规则（masquerade 模式）
kubectl exec virt-launcher-testvm-xxxxx -- iptables -t nat -L -n -v
```

### 查看 libvirt 网络配置

```bash
# 连接到 libvirt
kubectl exec virt-launcher-testvm-xxxxx -- virsh list

# 查看 VM 网络接口配置
kubectl exec virt-launcher-testvm-xxxxx -- virsh domiflist 1

# 查看完整 domain XML
kubectl exec virt-launcher-testvm-xxxxx -- virsh dumpxml 1
```

### 网络连通性测试

```bash
# 从 VM 内部测试（通过 console）
virtctl console testvm
# 在 VM 内：
ip addr
ping 8.8.8.8
curl https://www.google.com

# 从外部测试访问 VM
kubectl run test-pod --image=busybox -it --rm -- ping <VM-IP>
```

---

## 最佳实践

### 1. 选择合适的网络模式

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 通用 Web 服务 | Masquerade | 端口映射灵活、IP 稳定 |
| 需要固定 IP | Bridge | VM 直接使用 Pod IP |
| 高性能网络 | SR-IOV | 接近物理网卡性能 |
| 开发测试 | Slirp | 简单、无需特权 |
| 多租户隔离 | Multus + NetworkPolicy | 网络隔离 |

### 2. 性能优化

```yaml
spec:
  domain:
    devices:
      interfaces:
      - name: default
        bridge: {}
        model: virtio  # 使用 virtio 半虚拟化驱动
```

### 3. 安全加固

```yaml
# 使用 NetworkPolicy 限制流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-vm-traffic
spec:
  podSelector:
    matchLabels:
      kubevirt.io: virt-launcher
  policyTypes:
  - Ingress
  - Egress
  # 默认拒绝所有流量
```

### 4. 监控网络性能

```bash
# 在 VM 内使用 iperf 测试
# Server (VM1)
iperf3 -s

# Client (VM2)
iperf3 -c <VM1-IP>
```

---

## 总结

### KubeVirt 网络架构的核心理念

1. **复用 Kubernetes 网络**：不重新发明轮子，利用 CNI 生态
2. **分层设计**：Pod 网络 + VM 网络分离
3. **最小权限**：特权操作和非特权操作分离
4. **灵活性**：支持多种网络模式和高级功能

### 关键技术栈

```
┌────────────────────────────────────┐
│ KubeVirt 网络技术栈                │
├────────────────────────────────────┤
│ VM 层：virtio 网卡                 │
│ 虚拟化层：libvirt + QEMU           │
│ 网络层：bridge, NAT, tap           │
│ 容器层：virt-launcher Pod          │
│ CNI 层：Kubernetes CNI 插件        │
│ 物理层：宿主机网卡                 │
└────────────────────────────────────┘
```

### 推荐阅读

- [KubeVirt 官方网络文档](https://kubevirt.io/user-guide/virtual_machines/interfaces_and_networks/)
- [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni)
- [SR-IOV Network Operator](https://github.com/k8snetworkplumbingwg/sriov-network-operator)
- Kubernetes NetworkPolicy 文档

---

**文档版本**：基于 KubeVirt v1.3.x
**最后更新**：2026-03-25
