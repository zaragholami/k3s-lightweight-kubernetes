### K3s Cluster Installation Guide

#### 🖥️ System Requirements
#### OS Compatibility
| Requirement              | Details                                      |
|--------------------------|----------------------------------------------|
| **Architecture**         | 64-bit Linux (x86_64 or ARM64)               |
| **Kernel Version**       | ≥ 4.15                                       |
| **cgroups**              | v1 or v2 (v2 required for rootless mode)     |
| **Network Tools**        | iptables, ipset, socat                       |
| **Firewall**             | UFW/Firewalld must allow K3s ports           |

#### Supported Distributions
| Category       | Operating Systems                            |
|----------------|----------------------------------------------|
| **Debian-based** | Ubuntu, Debian, Raspberry Pi OS (64-bit)     |
| **RHEL-based**   | CentOS, RHEL (7/8/9), Rocky Linux, AlmaLinux |
| **SUSE-based**  | openSUSE Leap (15.x), SLES (15.x)            |
| **Lightweight** | DietPi (recommended for ARM devices)         |

---

### ⚙️ Container Runtime Setup
K3s uses embedded containerd. Install `nerdctl` for container management:
```bash
# Download and install nerdctl
wget https://github.com/containerd/nerdctl/releases/download/v1.7.5/nerdctl-full-1.7.5-linux-amd64.tar.gz
tar zxvf nerdctl-full-1.7.5-linux-amd64.tar.gz -C /usr/local

# Verify installation
nerdctl --version

# Configure for K3s integration
echo 'export CONTAINERD_NAMESPACE=k8s.io' >> ~/.bashrc
echo 'export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock' >> ~/.bashrc
source ~/.bashrc
```
#### Server Node (Control Plane) Installation

**Basic Installation**
```bash
curl -sfL https://get.k3s.io | sh -
```
**Includes**: etcd, Flannel CNI, Traefik Ingress, containerd

**Advanced Options**

| Use Case          | Command                                                                 |
|-------------------|-------------------------------------------------------------------------|
| Specific Version  | `curl -sfL https://get.k3s.io \| INSTALL_K3S_VERSION="v1.28.5+k3s1" sh -` |
| Disable Flannel   | `curl -sfL https://get.k3s.io \| INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy" sh -` |

#### Install Calico CNI (If Flannel Disabled)
```bash
# Apply Calico manifest
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Verify installation
kubectl get pods -n calico-system
```
**Expected Output:**
```text
calico-kube-controllers-xxxxx   Running
calico-node-xxxxx               Running
```
**Custom Pod CIDR Configuration**

```yml
kubectl apply -f - <<EOF
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: custom-pod-cidr
spec:
  cidr: 10.42.0.0/16
  ipipMode: Never
  vxlanMode: Always
  natOutgoing: true
  nodeSelector: all()
EOF
```
**Post-Install Verification**
```bash
sudo systemctl status k3s
kubectl get nodes -o wide
```
#### 🖥️ Agent Node (Worker) Installation

**Get Join Token from Server**
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
**Join Cluster**
```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<SERVER_IP>:6443 \
  K3S_TOKEN=<JOIN_TOKEN> \
  sh -
```
**Verify Agent Status**
```bash
# On server node
kubectl get nodes

# On agent node
sudo systemctl status k3s-agent
```
#### Firewall Configuration (UFW)

**Server Node Rules**
```bash
sudo ufw allow 22/tcp                 # SSH
sudo ufw allow 6443/tcp               # Kubernetes API
sudo ufw allow 8472/udp               # Flannel VXLAN
sudo ufw allow 10250/tcp              # Kubelet API
sudo ufw allow 2379:2380/tcp          # etcd communication
sudo ufw allow 30000:32767/tcp        # NodePort range
```
**Agent Node Rules**
```bash
sudo ufw allow 22/tcp                 # SSH
sudo ufw allow out 6443/tcp           # API Server access
sudo ufw allow 8472/udp               # Flannel VXLAN
sudo ufw allow out 2379:2380/tcp      # etcd client
sudo ufw allow 10250/tcp              # Kubelet API
sudo ufw allow 30000:32767/tcp        # NodePort access
```
**Apply Rules**
```bash
sudo ufw enable
sudo ufw reload
sudo ufw status numbered
```
#### ✅ Verification Checklist
1.All nodes show `Ready` status: `kubectl get nodes`

2.Core services are running: `kubectl get pods -A`

3.Pod networking works: `kubectl run test --image=nginx`

4.NodePort access: `curl http://<NODE_IP>:30080`

**Note**: Replace placeholders (<SERVER_IP>, <JOIN_TOKEN>) with actual values
