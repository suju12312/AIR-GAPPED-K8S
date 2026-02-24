# ☸️ Air-Gapped Kubernetes Setup Guide

> **VMware + RHEL 10** | Fully offline Kubernetes cluster integrated with Harbor private registry.

---

## 📋 Architecture Overview

| VM | IP Address | Internet Access |
|----|------------|-----------------|
| Harbor VM | `192.168.126.10` | ❌ No |
| Kubernetes Master VM | `192.168.126.30` | ❌ No |
| Internet VM | `192.168.126.20` | ✅ Yes |

> Network Type: **Host-Only (VMnet2)**

---

## Table of Contents

- [Part 1 – Create Kubernetes VM in VMware](#part-1--create-kubernetes-vm-in-vmware)
- [Part 2 – Configure VMnet2](#part-2--configure-vmnet2)
- [Part 3 – Configure Static IP](#part-3--configure-static-ip)
- [Part 4 – Hostname Fix](#part-4--hostname-fix)
- [Part 5 – Disable Swap](#part-5--disable-swap)
- [Part 6 – Disable SELinux](#part-6--disable-selinux)
- [Part 7 – Load Kernel Modules](#part-7--load-kernel-modules)
- [Part 8 – Configure Sysctl](#part-8--configure-sysctl)
- [Part 9 – Install containerd (Offline)](#part-9--install-containerd-offline)
- [Part 10 – Configure containerd](#part-10--configure-containerd)
- [Part 11 – Install CNI Plugins](#part-11--install-cni-plugins)
- [Part 12 – Install crictl](#part-12--install-crictl)
- [Part 13 – Install Kubernetes Binaries](#part-13--install-kubernetes-binaries)
- [Part 14 – Create kubelet Service](#part-14--create-kubelet-service)
- [Part 15 – Initialize Cluster (Air-Gapped)](#part-15--initialize-cluster-air-gapped)
- [Part 16 – Install Flannel (From Harbor)](#part-16--install-flannel-from-harbor)
- [Verify Cluster](#verify-cluster)
- [Reset Procedure](#reset-procedure)
- [Common Errors & Fixes](#common-errors--fixes)
- [Final Checklist](#final-checklist)

---

## Part 1 – Create Kubernetes VM in VMware

1. Open **VMware Workstation**
2. Click **Create New Virtual Machine** → Select **Typical**
3. Select your **RHEL 10 ISO**
4. Set the following specs:

| Setting | Value |
|---------|-------|
| Name | `K8s-VM` |
| CPU | 2+ |
| RAM | 4 GB minimum (6 GB recommended) |
| Disk | 40 GB+ |
| Network Adapter | Custom → VMnet2 (Host-Only) |

> ⚠️ **DO NOT** use NAT  
> ⚠️ **DO NOT** use Bridged

---

## Part 2 – Configure VMnet2

Navigate to: **VMware → Edit → Virtual Network Editor → Change Settings → Add Network**

| Setting | Value |
|---------|-------|
| Name | `VMnet2` |
| Type | Host-Only |
| Subnet | `192.168.126.0` |
| Mask | `255.255.255.0` |
| DHCP | Disabled |

---

## Part 3 – Configure Static IP

**Set static IP** (replace `ens33` with your actual interface):

```bash
nmcli connection modify ens33 \
  ipv4.method manual \
  ipv4.addresses 192.168.126.30/24 \
  ipv4.gateway 192.168.126.1 \
  ipv4.dns 8.8.8.8
```

**Activate the connection:**

```bash
nmcli connection up ens33
```

**Verify:**

```bash
ip a
```

```bash
ping 192.168.126.10
```

---

## Part 4 – Hostname Fix

> ⚠️ **Critical** — Hostname mismatch causes `kubeadm` failures.

```bash
hostnamectl set-hostname k8s-master
```

```bash
cat <<EOF >> /etc/hosts
192.168.126.30 k8s-master
EOF
```

**Verify:**

```bash
hostname
```

```bash
hostname -f
```

---

## Part 5 – Disable Swap

```bash
swapoff -a
```

```bash
sed -i '/swap/d' /etc/fstab
```

---

## Part 6 – Disable SELinux

```bash
setenforce 0
```

```bash
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

**Verify:**

```bash
getenforce
```

---

## Part 7 – Load Kernel Modules

**Create the modules config file:**

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

**Load modules immediately:**

```bash
modprobe overlay
```

```bash
modprobe br_netfilter
```

---

## Part 8 – Configure Sysctl

**Create sysctl config:**

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

**Apply settings:**

```bash
sysctl --system
```

---

## Part 9 – Install containerd (Offline)

**Extract containerd:**

```bash
tar -C /usr/local -xzf containerd-1.7.13-linux-amd64.tar.gz
```

**Install runc:**

```bash
install -m 755 runc.amd64 /usr/local/sbin/runc
```

---

**Create containerd systemd service:**

```bash
cat <<EOF | tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd
After=network.target

[Service]
ExecStart=/usr/local/bin/containerd
Restart=always
Delegate=yes
KillMode=process
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

**Enable and start containerd:**

```bash
systemctl daemon-reexec
```

```bash
systemctl daemon-reload
```

```bash
systemctl enable containerd --now
```

**Verify:**

```bash
ctr version
```

---

## Part 10 – Configure containerd

**Generate default config:**

```bash
mkdir -p /etc/containerd
```

```bash
containerd config default > /etc/containerd/config.toml
```

**Edit the config file:**

```bash
vi /etc/containerd/config.toml
```

**Set the following values inside `config.toml`:**

```toml
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
```

```toml
[plugins.'io.containerd.cri.v1.images'.pinned_images]
  sandbox = '192.168.126.10/k8s/pause:3.9'
```

**Restart containerd:**

```bash
systemctl restart containerd
```

**Verify the values were applied:**

```bash
cat /etc/containerd/config.toml | grep SystemdCgroup
```

```bash
cat /etc/containerd/config.toml | grep sandbox
```

---

## Part 11 – Install CNI Plugins

```bash
mkdir -p /opt/cni/bin
```

```bash
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.4.0.tgz
```

**Verify:**

```bash
ls /opt/cni/bin
```

---

## Part 12 – Install crictl

**Extract crictl:**

```bash
tar -C /usr/local/bin -xzf crictl-v1.29.0-linux-amd64.tar.gz
```

```bash
chmod +x /usr/local/bin/crictl
```

**Create crictl config:**

```bash
cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

**Verify:**

```bash
crictl version
```

---

## Part 13 – Install Kubernetes Binaries

**Move binaries to PATH:**

```bash
mv kubeadm kubelet kubectl /usr/local/bin/
```

**Set permissions:**

```bash
chmod +x /usr/local/bin/kubeadm
```

```bash
chmod +x /usr/local/bin/kubelet
```

```bash
chmod +x /usr/local/bin/kubectl
```

**Verify:**

```bash
kubeadm version
```

```bash
kubectl version --client
```

---

## Part 14 – Create kubelet Service

**Create the systemd service:**

```bash
cat <<EOF | tee /etc/systemd/system/kubelet.service
[Unit]
Description=kubelet
After=network.target

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

**Enable kubelet:**

```bash
systemctl daemon-reload
```

```bash
systemctl enable kubelet
```

---

## Part 15 – Initialize Cluster (Air-Gapped)

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.126.30 \
  --kubernetes-version=v1.29.15 \
  --pod-network-cidr=10.244.0.0/16 \
  --image-repository=192.168.126.10/k8s
```

---

**After successful init — set up kubeconfig:**

```bash
mkdir -p $HOME/.kube
```

```bash
cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

---

## Part 16 – Install Flannel (From Harbor)

**Edit `kube-flannel.yml`** and update the image paths to point to Harbor:

```yaml
image: 192.168.126.10/k8s/flannel:v0.25.5
```

```yaml
image: 192.168.126.10/k8s/flannel-cni-plugin:v1.5.1-flannel1
```

**Apply the manifest:**

```bash
kubectl apply -f kube-flannel.yml
```

---

## Verify Cluster

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

> ✅ The node should transition to **Ready** status.

---

## Reset Procedure

> Use this if `kubeadm init` fails or you need to start fresh.

```bash
kubeadm reset -f
```

```bash
rm -rf /etc/kubernetes
```

```bash
rm -rf /var/lib/etcd
```

```bash
rm -rf /var/lib/kubelet
```

```bash
systemctl restart containerd
```

```bash
systemctl restart kubelet
```

> Then re-run `kubeadm init` from [Part 15](#part-15--initialize-cluster-air-gapped).

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Node NotReady` | CNI missing / containerd not running / wrong `sandbox_image` | Check CNI, containerd status, and `config.toml` |
| `ImagePullBackOff` | Image not pushed to Harbor | Push missing image to Harbor registry |
| `Lease forbidden` | Hostname mismatch | Fix `/etc/hosts` → Run `kubeadm reset` → Re-init |
| `FailedCreatePodSandbox` | CNI not installed or wrong pause image path | Reinstall CNI plugins, verify `sandbox_image` in `config.toml` |

---

## Final Checklist

| Task | Status |
|------|--------|
| VMware Host-Only (VMnet2) network used | ✅ |
| Static IP configured | ✅ |
| Hostname set and `/etc/hosts` updated | ✅ |
| Swap disabled | ✅ |
| SELinux permissive/disabled | ✅ |
| Kernel modules loaded (`overlay`, `br_netfilter`) | ✅ |
| Sysctl settings applied | ✅ |
| containerd installed and running | ✅ |
| `sandbox_image` pointing to Harbor | ✅ |
| CNI plugins installed | ✅ |
| `kubeadm init` uses Harbor `--image-repository` | ✅ |
| Node status is Ready | ✅ |

---

> **Kubernetes VM setup complete — Fully Air-Gapped Enterprise Ready! 🎉**
