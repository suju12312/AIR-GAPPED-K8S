# Kubernetes-VM.md
Air-Gapped Kubernetes Setup (RHEL 10)
FINAL COMPLETE VERSION (VMware + Harbor Integrated)

============================================================

ARCHITECTURE

Harbor VM:      192.168.126.10
K8s Master VM:  192.168.126.30
Internet VM:    192.168.126.20

Kubernetes VM = NO INTERNET
Network Type = Host-Only (VMnet2)

============================================================
PART 1 – Create Kubernetes VM in VMware

1. Open VMware Workstation
2. Click Create New Virtual Machine
3. Select Typical
4. Select RHEL 10 ISO
5. Name: K8s-VM
6. CPU: 2+
7. RAM: 4GB minimum (6GB recommended)
8. Disk: 40GB+
9. Finish

------------------------------------------------------------
NETWORK SETTING (VERY IMPORTANT)

VM Settings → Network Adapter

Select:
Custom → VMnet2 (Host-Only)

DO NOT use NAT
DO NOT use Bridged

============================================================
PART 2 – Configure VMnet2 (If Not Created)

VMware → Edit → Virtual Network Editor
Click Change Settings

Add Network → VMnet2

Type: Host-Only
Subnet: 192.168.126.0
Mask: 255.255.255.0

Disable DHCP (Recommended)

============================================================
PART 3 – Configure Static IP

nmcli connection modify ens33 \
ipv4.method manual \
ipv4.addresses 192.168.126.30/24 \
ipv4.gateway 192.168.126.1 \
ipv4.dns 8.8.8.8

nmcli connection up ens33

Verify:

ip a
ping 192.168.126.10

============================================================
PART 4 – Hostname Fix (CRITICAL)

hostnamectl set-hostname k8s-master

cat <<EOF >> /etc/hosts
192.168.126.30 k8s-master
EOF

Verify:

hostname
hostname -f

============================================================
PART 5 – Disable Swap

swapoff -a
sed -i '/swap/d' /etc/fstab

============================================================
PART 6 – Disable SELinux (Lab Mode)

setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

Check:
getenforce

============================================================
PART 7 – Load Kernel Modules

cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

============================================================
PART 8 – Configure Sysctl

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

============================================================
PART 9 – Install containerd (Offline)

Extract:

tar -C /usr/local -xzf containerd-1.7.13-linux-amd64.tar.gz

Install runc:

install -m 755 runc.amd64 /usr/local/sbin/runc

------------------------------------------------------------
Create containerd service

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

Start containerd:

systemctl daemon-reexec
systemctl daemon-reload
systemctl enable containerd --now

Verify:

ctr version

============================================================
PART 10 – Configure containerd (IMPORTANT)

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

Edit:

vi /etc/containerd/config.toml

Set:

SystemdCgroup = true

sandbox_image = "192.168.126.10/k8s/pause:3.9"

Restart:

systemctl restart containerd

============================================================
PART 11 – Install CNI Plugins

mkdir -p /opt/cni/bin
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.4.0.tgz

Verify:

ls /opt/cni/bin

============================================================
PART 12 – Install crictl

tar -C /usr/local/bin -xzf crictl-v1.29.0-linux-amd64.tar.gz
chmod +x /usr/local/bin/crictl

cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

Verify:

crictl version

============================================================
PART 13 – Install Kubernetes Binaries

mv kubeadm kubelet kubectl /usr/local/bin/

chmod +x /usr/local/bin/kubeadm
chmod +x /usr/local/bin/kubelet
chmod +x /usr/local/bin/kubectl

Verify:

kubeadm version
kubectl version --client

============================================================
PART 14 – Create kubelet Service

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

systemctl daemon-reload
systemctl enable kubelet

============================================================
PART 15 – Initialize Cluster (AIR-GAPPED)

kubeadm init \
--apiserver-advertise-address=192.168.126.30 \
--kubernetes-version=v1.29.15 \
--pod-network-cidr=10.244.0.0/16 \
--image-repository=192.168.126.10/k8s

============================================================
After Successful Init

mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

============================================================
PART 16 – Install Flannel (From Harbor)

Edit flannel YAML image paths:

image: 192.168.126.10/k8s/flannel:v0.25.5
image: 192.168.126.10/k8s/flannel-cni-plugin:v1.5.1-flannel1

Apply:

kubectl apply -f kube-flannel.yml

============================================================
VERIFY CLUSTER

kubectl get nodes
kubectl get pods -A

Node should become Ready.

============================================================
RESET PROCEDURE (If Needed)

kubeadm reset -f

rm -rf /etc/kubernetes
rm -rf /var/lib/etcd
rm -rf /var/lib/kubelet

systemctl restart containerd
systemctl restart kubelet

Re-run kubeadm init.

============================================================
COMMON ERRORS

Node NotReady
→ CNI missing
→ containerd not running
→ sandbox_image wrong

ImagePullBackOff
→ Image not pushed to Harbor

Lease forbidden
→ Hostname mismatch
→ Run kubeadm reset

FailedCreatePodSandbox
→ CNI not installed
→ pause image wrong path

============================================================
FINAL CHECKLIST

✔ VMware Host-Only network used
✔ Static IP configured
✔ Hostname fixed
✔ Swap disabled
✔ SELinux permissive/disabled
✔ Kernel modules loaded
✔ Sysctl applied
✔ containerd running
✔ sandbox_image from Harbor
✔ CNI installed
✔ kubeadm init uses Harbor
✔ Node Ready

KUBERNETES VM COMPLETE – FULLY AIR-GAPPED ENTERPRISE READY
