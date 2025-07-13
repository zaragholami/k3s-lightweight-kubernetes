### Installing K3s (Server Node)

**OS Requirements:**

64-bit Linux only (ARM64 and x86_64 supported)

Kernel version ≥ 4.15

cgroups v1 or v2 enabled (v2 required for rootless mode)

iptables, ipset, and socat should be available

UFW/Firewalld should allow K3s ports

**supported OS:**

Ubuntu,Debian,CentOS,RHEL (7, 8, 9),Rocky Linux (8, 9),AlmaLinux (8, 9),Oracle Linux (7, 8),openSUSE Leap (15.x),USE Linux Enterprise Server (SLES) (15.x)

**Recommended OS:**

Raspberry Pi OS (64-bit),Ubuntu Server for ARM,DietPi (lightweight and fast)

**Container Runtime Compatibility**
K3s uses containerd by default (built-in), so:
Works out-of-the-box with `nerdctl`
Download and install :
```
wget https://github.com/containerd/nerdctl/releases/download/v1.7.5/nerdctl-full-1.7.5-linux-amd64.tar.gz\
tar zxvf nerdctl-full-1.7.5-linux-amd64.tar.gz -C /usr/local
```
Verify installation:
```
nerdctl --version
```
Add environment for K3s containerd:
For use with K3s's containerd:
```
export CONTAINERD_NAMESPACE=k8s.io
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
```
Add it to your shell config:
```
echo 'export CONTAINERD_NAMESPACE=k8s.io' >> ~/.bashrc\
echo 'export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock' >> ~/.bashrc\
source ~/.bashrc
```
#### 🖥️ 1.Server Node (Control Plane)
#### 🔹 Basic Install
```
curl -sfL https://get.k3s.io | sh -
```
Installs:

Embedded etcd

Flannel CNI

Traefik Ingress Controller

containerd as runtime

#### 🔹Specify K3s Version
```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.28.5+k3s1" sh -
```
Useful for:

Ensuring consistency across nodes

Avoiding unexpected updates

#### 🔹Disable Flannel (for Custom CNI like Calico)
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy" sh -
```
Use if you plan to:

Install another CNI like Calico or Cilium manually

Install Calico using kubectl
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```
 Version v3.27.0 is the latest stable version as of July 2025
 Verify Calico pods
 ```
kubectl get pods -n calico-system
```
You should see something like
```
calico-kube-controllers-XXXXX   Running
calico-node-XXXXX               Running
```
Check node status
```
kubectl get nodes -o wide
```
Ensure `INTERNAL-IP` and `EXTERNAL-IP` are correct and `STATUS` is `Ready`

optional: Customize Calico IP Pools

If you want to define custom pod CIDRs (instead of default `192.168.0.0/16`):
```
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
Make sure this pool doesn't conflict with other networks (e.g. your home LAN)

#### Check status
```
sudo systemctl status k3s
kubectl get nodes
```
---------------------------------------------

#### 🖥️ 2.Agent-Node (Worker)
Get token from server (control-plane):
```
sudo cat /var/lib/rancher/k3s/server/node-token
```
Agent install command
```
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -
```
Verify
On the server node:
```
kubectl get nodes
```
Systemd status
```
sudo systemctl status k3s-agent
```
#### UFW Firewall Settings
Server Node (Control Plane)
```
# Allow SSH
sudo ufw allow 22/tcp

# Allow Kubernetes API Server
sudo ufw allow 6443/tcp

# Allow Flannel VXLAN (default CNI)
sudo ufw allow 8472/udp

# Allow etcd and K3s internal communication
sudo ufw allow 10250/tcp

# Allow metrics-server and webhooks
sudo ufw allow 10255/tcp

# Optional: NodePort range (if used)
sudo ufw allow 30000:32767/tcp
```
Agent Node (Worker)
```
# Allow SSH
sudo ufw allow 22/tcp

# Allow connection to server API
sudo ufw allow out 6443/tcp

# Allow Flannel VXLAN
sudo ufw allow 8472/udp

# Allow etcd client traffic if needed
sudo ufw allow out 2379:2380/tcp

# Allow kubelet communication
sudo ufw allow 10250/tcp

# Optional: NodePort access
sudo ufw allow 30000:32767/tcp
```
