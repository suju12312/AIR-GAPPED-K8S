# 🖥️ Air-Gapped Kubernetes – Internet VM Setup Guide

> **VMware + RHEL** | Full offline Kubernetes cluster using Harbor as a private registry.

---

## 📋 IP Address Reference

| VM | IP Address |
|----|------------|
| Harbor VM | `192.168.126.10` |
| Internet VM | `192.168.126.20` |
| Kubernetes SNO VM | `192.168.126.30` |

> ⚠️ Check your `ipconfig` on the host machine and adjust IPs accordingly if configuring manually.

---

## Table of Contents

- [Part 1 – Create Internet VM in VMware](#part-1--create-internet-vm-in-vmware)
- [Part 2 – Create VMnet2](#part-2--create-vmnet2)
- [Part 3 – Configure Static IP](#part-3--configure-static-ip)
- [Part 4 – Verify Internet](#part-4--verify-internet)
- [Part 5 – Register RHEL](#part-5--register-rhel)
- [Part 6 – Install Docker](#part-6--install-docker)
- [Part 7 – Harbor HTTPS Certificate Trust](#part-7--harbor-https-certificate-trust)
- [Part 8 – Download Kubernetes Binaries](#part-8--download-kubernetes-binaries)
- [Part 9 – Download Runtime Components](#part-9--download-runtime-components)
- [Part 10 – Download Harbor Offline Installer](#part-10--download-harbor-offline-installer)
- [Part 11 – SCP Files to Harbor VM](#part-11--scp-files-to-harbor-vm)
- [Part 12 – SCP Files to Kubernetes VM](#part-12--scp-files-to-kubernetes-vm)
- [Part 13 – Login to Harbor](#part-13--login-to-harbor)
- [Part 14 – Pull & Push Kubernetes Images](#part-14--pull--push-kubernetes-images)
- [Part 15 – Push Flannel Images](#part-15--push-flannel-images)
- [Final Checkpoint](#final-checkpoint)

---

## Part 1 – Create Internet VM in VMware

1. Open **VMware Workstation**
2. Click **Create New Virtual Machine** → Select **Typical**
3. Select your **RHEL ISO**
4. Set the following specs:

| Setting | Value |
|---------|-------|
| Name | `Internet-VM` |
| CPU | 2 |
| RAM | 4 GB |
| Disk | 40 GB |
| Adapter 1 | NAT (Internet) |
| Adapter 2 | VMnet2 (Lab Network) |

---

## Part 2 – Create VMnet2

Navigate to: **VMware → Edit → Virtual Network Editor → Change Settings → Add Network**

| Setting | Value |
|---------|-------|
| Name | `VMnet2` |
| Type | Host-only |
| Subnet | `192.168.126.0` |
| Mask | `255.255.255.0` |
| DHCP | Disabled |

---

## Part 3 – Configure Static IP

**Check existing interfaces:**

```bash
ip a
```

```bash
nmcli connection show
```

**Set static IP** (replace `ens37` with your actual interface name):

```bash
nmcli connection modify ens37 \
  ipv4.method manual \
  ipv4.addresses 192.168.126.20/24 \
  ipv4.gateway 192.168.126.1 \
  ipv4.dns 8.8.8.8
```

**Activate the connection:**

```bash
nmcli connection up ens37
```

**Verify:**

```bash
ip a
```

```bash
ip route
```

```bash
cat /etc/resolv.conf
```

---

## Part 4 – Verify Internet

```bash
ping google.com
```

```bash
curl https://registry.k8s.io
```

> **If DNS fails**, add the following to `/etc/resolv.conf`:
>
> ```
> nameserver 8.8.8.8
> ```

---

## Part 5 – Register RHEL

```bash
subscription-manager register
```

```bash
subscription-manager attach --auto
```

```bash
subscription-manager status
```

**Enable repos (RHEL 10):**

```bash
subscription-manager repos --enable=rhel-10-for-x86_64-baseos-rpms
```

```bash
subscription-manager repos --enable=rhel-10-for-x86_64-appstream-rpms
```

**Verify repos:**

```bash
dnf repolist
```

---

## Part 6 – Install Docker

### 6.1 – Internet VM (Has Internet Access)

```bash
dnf install -y yum-utils
```

```bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

```bash
dnf install -y docker-ce docker-ce-cli containerd.io
```

```bash
systemctl enable docker --now
```

```bash
docker version
```

---

### 6.2 – Harbor VM (No Internet – Download & Transfer)

Run these commands on the **Internet VM** to download RPMs and transfer them:

```bash
mkdir /root/docker-rpms
cd /root/docker-rpms
```

```bash
dnf install -y yum-utils
```

```bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

```bash
dnf download --resolve \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

```bash
ls -lh
```

**SCP RPMs to Harbor VM:**

```bash
scp *.rpm root@192.168.126.10:/root/docker-rpms/
```

---

## Part 7 – Harbor HTTPS Certificate Trust

When Harbor uses HTTPS with a self-signed certificate, Docker on the Internet VM will show:

```
x509: certificate signed by unknown authority
```

The Harbor VM stores its CA certificate at `/data/cert/ca.crt`. We must copy it to the Internet VM.

---

### Step 1 – Copy CA Certificate from Harbor VM

Run on **Internet VM**:

```bash
scp root@192.168.126.10:/data/cert/ca.crt /root/
```

**Verify:**

```bash
ls -lh /root/ca.crt
```

---

### Method 1 – Docker-Specific Trust *(Recommended for Lab)*

```bash
mkdir -p /etc/docker/certs.d/192.168.126.10
```

```bash
cp /root/ca.crt /etc/docker/certs.d/192.168.126.10/ca.crt
```

```bash
systemctl restart docker
```

**Test login:**

```bash
docker login 192.168.126.10
```

---

### Method 2 – System-Wide Trust *(Recommended for Enterprise)*

```bash
cp /root/ca.crt /etc/pki/ca-trust/source/anchors/
```

```bash
update-ca-trust
```

```bash
systemctl restart docker
```

**Verify (no certificate error should appear):**

```bash
curl -v https://192.168.126.10
```

---

### Troubleshooting – Certificate Still Failing

```bash
ls /etc/docker/certs.d/192.168.126.10/
```

> Make sure the file is named exactly `ca.crt`

```bash
systemctl restart docker
```

---

## Part 8 – Download Kubernetes Binaries

```bash
wget https://dl.k8s.io/v1.29.15/bin/linux/amd64/kubeadm
```

```bash
wget https://dl.k8s.io/v1.29.15/bin/linux/amd64/kubelet
```

```bash
wget https://dl.k8s.io/v1.29.15/bin/linux/amd64/kubectl
```

```bash
chmod +x kubeadm kubelet kubectl
```

**Verify:**

```bash
ls -lh kube*
```

---

## Part 9 – Download Runtime Components

### containerd

```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz
```

```bash
ls -lh containerd-1.7.13-linux-amd64.tar.gz
```

---

### runc

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
```

```bash
ls -lh runc.amd64
```

---

### CNI Plugins

> ⚠️ **Very Important!** Missing CNI plugins cause pods to fail with `FailedCreatePodSandbox: failed to find plugin "loopback"`

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
```

```bash
ls -lh cni-plugins-linux-amd64-v1.4.0.tgz
```

---

### CRI Tools (crictl)

> Used to debug the container runtime on Kubernetes nodes.

```bash
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.29.0/crictl-v1.29.0-linux-amd64.tar.gz
```

```bash
ls -lh crictl-v1.29.0-linux-amd64.tar.gz
```

---

## Part 10 – Download Harbor Offline Installer

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.1/harbor-offline-installer-v2.10.1.tgz
```

**Verify:**

```bash
ls -lh harbor-offline-installer-v2.10.1.tgz
```

---

## Part 11 – SCP Files to Harbor VM

```bash
scp harbor-offline-installer-v2.10.1.tgz root@192.168.126.10:/root/
```

---

## Part 12 – SCP Files to Kubernetes VM

```bash
scp kube* root@192.168.126.30:/root/
```

```bash
scp containerd-*.tar.gz root@192.168.126.30:/root/
```

```bash
scp runc.amd64 root@192.168.126.30:/root/
```

```bash
scp cni-plugins-*.tgz root@192.168.126.30:/root/
```

```bash
scp crictl-*.tar.gz root@192.168.126.30:/root/
```

---

## Part 13 – Login to Harbor

> ⚠️ Make sure Docker is installed. If not, refer to [Part 6](#part-6--install-docker).

```bash
docker login 192.168.126.10
```

Enter your Harbor admin username and password when prompted.

---

## Part 14 – Pull & Push Kubernetes Images

**List required images:**

```bash
kubeadm config images list --kubernetes-version v1.29.15
```

---

### kube-apiserver

```bash
docker pull registry.k8s.io/kube-apiserver:v1.29.15
docker tag registry.k8s.io/kube-apiserver:v1.29.15 192.168.126.10/k8s/kube-apiserver:v1.29.15
docker push 192.168.126.10/k8s/kube-apiserver:v1.29.15
```

---

### kube-controller-manager

```bash
docker pull registry.k8s.io/kube-controller-manager:v1.29.15
docker tag registry.k8s.io/kube-controller-manager:v1.29.15 192.168.126.10/k8s/kube-controller-manager:v1.29.15
docker push 192.168.126.10/k8s/kube-controller-manager:v1.29.15
```

---

### kube-scheduler

```bash
docker pull registry.k8s.io/kube-scheduler:v1.29.15
docker tag registry.k8s.io/kube-scheduler:v1.29.15 192.168.126.10/k8s/kube-scheduler:v1.29.15
docker push 192.168.126.10/k8s/kube-scheduler:v1.29.15
```

---

### kube-proxy

```bash
docker pull registry.k8s.io/kube-proxy:v1.29.15
docker tag registry.k8s.io/kube-proxy:v1.29.15 192.168.126.10/k8s/kube-proxy:v1.29.15
docker push 192.168.126.10/k8s/kube-proxy:v1.29.15
```

---

### etcd

```bash
docker pull registry.k8s.io/etcd:3.5.12-0
docker tag registry.k8s.io/etcd:3.5.12-0 192.168.126.10/k8s/etcd:3.5.12-0
docker push 192.168.126.10/k8s/etcd:3.5.12-0
```

---

### pause

```bash
docker pull registry.k8s.io/pause:3.9
docker tag registry.k8s.io/pause:3.9 192.168.126.10/k8s/pause:3.9
docker push 192.168.126.10/k8s/pause:3.9
```

---

### coredns

```bash
docker pull registry.k8s.io/coredns/coredns:v1.11.1
docker tag registry.k8s.io/coredns/coredns:v1.11.1 192.168.126.10/k8s/coredns:v1.11.1
docker push 192.168.126.10/k8s/coredns:v1.11.1
```

---

## Part 15 – Push Flannel Images

### flannel

```bash
docker pull flannel/flannel:v0.25.5
docker tag flannel/flannel:v0.25.5 192.168.126.10/k8s/flannel:v0.25.5
docker push 192.168.126.10/k8s/flannel:v0.25.5
```

---

### flannel-cni-plugin

```bash
docker pull flannel/flannel-cni-plugin:v1.5.1-flannel1
docker tag flannel/flannel-cni-plugin:v1.5.1-flannel1 192.168.126.10/k8s/flannel-cni-plugin:v1.5.1-flannel1
docker push 192.168.126.10/k8s/flannel-cni-plugin:v1.5.1-flannel1
```

> ✅ After all pushes, verify all images appear in the **Harbor UI**.

---

## Final Checkpoint

Before shutting down the Internet VM, confirm everything is complete:

| Task | Status |
|------|--------|
| Kubernetes binaries downloaded | ✅ |
| containerd downloaded | ✅ |
| runc downloaded | ✅ |
| CNI plugins downloaded | ✅ |
| CRI tools downloaded | ✅ |
| Harbor installer copied to Harbor VM | ✅ |
| All K8s images pushed to Harbor | ✅ |
| Harbor HTTPS trusted | ✅ |

---

> **Internet VM setup complete! 🎉**
