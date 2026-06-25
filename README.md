# Road to CKA: Building a Self-Hosted Kubernetes Lab

A working log of my path to the Certified Kubernetes Administrator (CKA) exam, built around the Linux Foundation LFS258 course. Instead of using the course's browser-based lab environment, I stood the whole thing up on my own hardware so I would own every layer, break things on purpose, and learn what actually happens underneath `kubeadm`.

This repo is the journal. Each milestone gets documented here as I go: what I built, what broke, and how I diagnosed it.

---

## Why self-host instead of the browser lab

The provided lab runs on managed cloud instances where swap is already off, the firewall is handled at the project level, and IP addresses are stable. That convenience hides the parts a CKA actually needs to understand. Building on local VMware Workstation forced me to solve the networking, the runtime, and the cluster bring-up by hand, which is where the real learning lives.

## Lab architecture

| Node     | Role          | vCPU | RAM  | Disk  | IP              |
| -------- | ------------- | ---- | ---- | ----- | --------------- |
| `cp`     | control plane | 2    | 4 GB | 25 GB | 192.168.239.50  |
| `worker` | worker        | 2    | 4 GB | 25 GB | 192.168.239.51  |

**Stack**

- Host: VMware Workstation Pro (free personal-use license)
- Guest OS: Ubuntu Server 24.04 LTS
- Kubernetes: v1.34.2 (kubeadm, kubelet, kubectl)
- Container runtime: containerd
- CNI: Cilium 1.19.1
- Networking: VMware NAT subnet, 192.168.239.0/24, static IPs

---

## Milestone 1: Cluster bring-up

The clean-path summary. Full command detail lives in [`docs/01-cluster-build.md`](docs/01-cluster-build.md).

1. Created two Ubuntu Server 24.04 VMs, username `student` to match the course tarball paths.
2. Disabled swap permanently in `/etc/fstab`, not just at runtime.
3. Disabled the host firewall (`ufw`) on both nodes.
4. Loaded `overlay` and `br_netfilter`, set the required `sysctl` bridge and forwarding values.
5. Installed containerd, set `SystemdCgroup = true`.
6. Installed the v1.34 Kubernetes packages and held them with `apt-mark hold`.
7. Added the `k8scp` control-plane alias to `/etc/hosts` on both nodes.
8. Initialized the control plane with a `kubeadm` config file pinning `controlPlaneEndpoint: "k8scp:6443"` and `podSubnet: 192.168.0.0/16`.
9. Deployed Cilium as the CNI.
10. Joined the worker.

Result:

```
NAME     STATUS   ROLES           AGE   VERSION
cp       Ready    control-plane   46h   v1.34.2
worker   Ready    <none>          --    v1.34.2
```

---

## Troubleshooting log

This is the part I care about most. Three real failures, each isolated to a specific layer.

### 1. No network on the VMs

**Symptom:** Both VMs booted with an interface that was `UP` and had a link-local IPv6 address, but no IPv4 and no default route. Every outbound action failed with `Network is unreachable`.

**Diagnosis:** Started at the guest, worked toward the host.

- `ip -br addr` confirmed the interface was up but had no v4 lease.
- `ip route` was empty, so there was no gateway at all.
- Opened VMware's Virtual Network Editor and found there was no Bridged (VMnet0) network defined on the install. The adapter had been set to Bridged, which silently went nowhere.

**Fix:** Switched both VMs to NAT (VMnet8), which runs its own DHCP service. The nodes landed on 192.168.239.0/24 immediately. NAT keeps the two nodes on the same subnet so they can reach each other and the internet, which is all `kubeadm` needs.

**Lesson:** A NIC being "up" tells you nothing about reachability. Walk the stack outward: interface, address, route, then the virtual network layer on the host.

### 2. Netplan would not apply

**Symptom:** Hand-edited netplan config produced `Invalid YAML: inconsistent indentation`, and after the bad file existed, the interface stayed offline even though a previously valid config was still present.

**Diagnosis:** Netplan is all-or-nothing. If any file in `/etc/netplan/` fails to parse, it refuses to apply the entire directory, including the good installer-generated config. The broken file was poisoning the whole apply.

**Fix:** Removed the broken file, let the valid `00-installer-config.yaml` take over for connectivity, then re-pinned a static IP cleanly. Lesson reinforced: type YAML carefully, two spaces per level, never tabs, and validate before assuming a config is live.

### 3. Control plane init failed at the final phase

**Symptom:** `kubeadm init` ran through cert generation, etcd, and all control-plane health checks successfully, then failed:

```
error: ... unable to create ClusterRoleBinding: Post "https://k8scp:6443/...": context deadline exceeded
```

**Diagnosis:** Every passing health check used the raw IP (`https://192.168.239.50:6443/livez` returned healthy). The one step that failed used the `k8scp` alias. That alias is the `controlPlaneEndpoint`, so when `kubeadm` switched to talking through the name, it went nowhere. `getent hosts k8scp` confirmed the alias was missing or wrong on the cp.

**Fix:** Corrected the `/etc/hosts` entry, ran `kubeadm reset -f` to clear the partial state, and re-ran init. It completed.

### 4. Worker join failed: API server unreachable

**Symptom:**

```
couldn't validate the identity of the API Server: Get "https://k8scp:6443/...": Client.Timeout exceeded
```

**Diagnosis:** This is the diagnostic ladder I want to remember for the exam. On the worker I tested the endpoint two different ways:

```
nc -zv 192.168.239.50 6443   # succeeded  -> network path is fine
nc -zv k8scp 6443            # refused, resolved to 192.168.1.51 (wrong subnet)
```

`cat /etc/hosts` showed the alias pointing at `192.168.1.51`, a leftover placeholder address from my notes that did not match the NAT subnet the cluster actually landed on. The name resolved to a host that does not exist on this network.

**Fix:** Removed the stale lines, pointed `k8scp` at the real cp address (192.168.239.50), verified with `nc -zv k8scp 6443` before retrying, then `kubeadm reset -f` and re-joined.

**Lesson, and the single most useful takeaway so far:** when a Kubernetes component cannot reach the API server, always test the endpoint by IP and by name.

- IP works, name fails: it is name resolution (`/etc/hosts` or DNS).
- Both fail: it is the network, firewall, or routing.
- Name works, still refused: it is the service itself.

Reading which layer an error points at is half the fix. A timeout means connectivity. An `x509` or `Unauthorized` error would mean certs or tokens, a completely different path.

---

## What is next

- CNI deep dive: how Cilium handles pod-to-pod and Network Policy
- Workload basics: deployments, services, scaling, rollouts
- Storage: persistent volumes and claims
- The CKA exam domains, worked one at a time with notes here
- Cluster upgrade lab (kubeadm version bump)

---

## Repo structure

```
.
├── README.md                  # this journal
└── docs/
    └── 01-cluster-build.md     # full command-level build guide
```

## Note on security

The join command in any published writeup uses placeholders (`--token <token>`, `--discovery-token-ca-cert-hash sha256:<hash>`). Real bootstrap tokens and CA hashes should never be committed to a public repo, even from a local lab. Treat them like credentials.

---

*Part of an ongoing certification journey. Background: 20 years US Navy, now working in systems engineering and cybersecurity, building toward the CKA.*
