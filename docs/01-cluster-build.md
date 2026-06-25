# 01: Cluster Build (Command-Level Guide)

The full command walkthrough for Milestone 1: a two-node Kubernetes cluster on local VMware Workstation. This documents the environment as actually built, on a VMware NAT subnet (192.168.239.0/24) with static IPs.

All shell commands are single-line on purpose. Indented config blocks are YAML files, not commands.

## Environment

| Node     | Role          | vCPU | RAM  | IP              |
| -------- | ------------- | ---- | ---- | --------------- |
| `cp`     | control plane | 2    | 4 GB | 192.168.239.50  |
| `worker` | worker        | 2    | 4 GB | 192.168.239.51  |

- Host: VMware Workstation Pro (free personal-use license)
- Guest: Ubuntu Server 24.04 LTS, username `student`
- Kubernetes v1.34.2, containerd, Cilium 1.19.1
- NAT gateway on this subnet: 192.168.239.2

Substitute your own addresses throughout if yours differ.

---

## Part 1: VMware and the VMs

1. Install VMware Workstation Pro (free for personal use, no license key, downloaded from the Broadcom support portal after creating a free account).
2. Create two VMs from the Ubuntu Server 24.04 ISO: `cp` (2 vCPU, 4 GB, 25 GB) and `worker` (2 vCPU, 4 GB, 25 GB).
3. Set each VM's Network Adapter to **NAT** (VM > Settings > Network Adapter > NAT). NAT puts both nodes on the same VMware subnet so they reach each other and the internet, and VMware's built-in DHCP hands out leases. Bridged also works if your host has a usable bridged vmnet, but NAT avoids host-adapter guesswork.

## Part 2: Install Ubuntu (both VMs)

Walk the Ubuntu Server installer. Two choices that matter:

- Username: **`student`** (the course tarball unpacks to `/home/student/...`).
- Check **Install OpenSSH server** so you can work over SSH instead of the VMware console.

## Part 3: Networking (both nodes)

Find the interface name (usually `ens33`) and current lease:

```
ip -br addr
```

Pin a static IP so the control-plane address never moves. Edit the netplan file (filename may be `00-installer-config.yaml` or `50-cloud-init.yaml`):

```
sudo nano /etc/netplan/00-installer-config.yaml
```

Set it to this for `cp` (use `.51` for the worker, and your real interface name):

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.239.50/24]
      routes:
        - to: default
          via: 192.168.239.2
      nameservers:
        addresses: [192.168.239.2, 8.8.8.8]
```

Lock permissions and apply:

```
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

```
sudo netplan apply
```

Verify and confirm internet:

```
ip -br addr
```

```
ping -c2 8.8.8.8
```

Netplan is all-or-nothing: if any file in `/etc/netplan/` has a YAML error, nothing applies. Two-space indentation, never tabs.

## Part 4: Host prep (both nodes)

Become root:

```
sudo -i
```

Update:

```
apt-get update && apt-get upgrade -y
```

Dependencies:

```
apt install apt-transport-https tree software-properties-common ca-certificates socat -y
```

Disable swap now and permanently (VMware installs create swap, unlike cloud images):

```
swapoff -a
```

```
sed -i '/\sswap\s/s/^/#/' /etc/fstab
```

Disable the firewall while learning:

```
ufw disable
```

Load modules and make them persistent:

```
modprobe overlay
```

```
modprobe br_netfilter
```

```
printf 'overlay\nbr_netfilter\n' | tee /etc/modules-load.d/k8s.conf
```

Kernel networking:

```
printf 'net.bridge.bridge-nf-call-ip6tables = 1\nnet.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\n' | tee /etc/sysctl.d/kubernetes.conf
```

```
sysctl --system
```

## Part 5: containerd (both nodes)

```
mkdir -p /etc/apt/keyrings
```

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```
apt-get update && apt-get install containerd.io -y
```

```
containerd config default | tee /etc/containerd/config.toml
```

```
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
```

```
systemctl restart containerd
```

## Part 6: Kubernetes packages (both nodes)

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
```

```
apt-get update
```

```
apt-get install -y kubeadm=1.34.2-1.1 kubelet=1.34.2-1.1 kubectl=1.34.2-1.1
```

```
apt-mark hold kubelet kubeadm kubectl
```

## Part 7: Control-plane alias (both nodes)

Both nodes must resolve `k8scp` to the control-plane IP, because the cluster's API endpoint is the alias, not the raw IP. Use your real cp address:

```
printf '192.168.239.50 k8scp\n192.168.239.50 cp\n' | tee -a /etc/hosts
```

Confirm:

```
getent hosts k8scp
```

Leave root:

```
exit
```

## Part 8: Initialize the control plane (cp only)

Download the course tarball (confirm the current filename at https://cm.lf.training/LFS258):

```
wget https://cm.lf.training/LFS258/LFS258_V2026-06-03_SOLUTIONS.tar.xz --user=LFtraining --password=Penguin2014
```

```
tar -xvf LFS258_V2026-06-03_SOLUTIONS.tar.xz
```

Write the cluster config:

```
printf 'apiVersion: kubeadm.k8s.io/v1beta4\nkind: ClusterConfiguration\nkubernetesVersion: 1.34.2\ncontrolPlaneEndpoint: "k8scp:6443"\nnetworking:\n  podSubnet: 192.168.0.0/16\n' | tee kubeadm-config.yaml
```

Initialize:

```
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs --node-name=cp | tee kubeadm-init.out
```

Copy the `kubeadm join` command from the bottom of the output for Part 10.

## Part 9: kubeconfig and CNI (cp only)

```
mkdir -p $HOME/.kube
```

```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Deploy Cilium:

```
kubectl apply -f /home/student/LFS258/SOLUTIONS/s_03/cilium-cni.yaml
```

Watch the control-plane pods reach Running:

```
kubectl get pods -A -w
```

## Part 10: Join the worker (worker only)

Use the join command from Part 8 (your token and hash will differ):

```
sudo kubeadm join k8scp:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

If the token expired (24h lifetime), regenerate it on the cp:

```
sudo kubeadm token create --print-join-command
```

## Verify

On the cp:

```
kubectl get nodes
```

A freshly joined worker shows `NotReady` for a minute or two while Cilium schedules its agent onto it, then flips to `Ready`. Both nodes `Ready` means the cluster is done.

## Reset (start a node over)

A failed `init` or `join` leaves partial state. Always reset before retrying:

```
sudo kubeadm reset -f
```

## Security note

Never commit a real bootstrap token or CA hash to a public repo. The join line here uses `<token>` and `sha256:<hash>` placeholders on purpose. Treat them like credentials.
