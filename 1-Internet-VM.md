# Internet VM Setup Guide (VMware + RHEL)
Air-Gapped Kubernetes – Internet Machine
FINAL COMPLETE VERSION
192.168.126.10 == Harbor VM IP ADDRESS
192.168.126.20 == INTERNET VM IP ADDRESS
192.168.126.30 == KUBERNETES SNO IP ADDRESS 
[CHECK YOUR IPCONFING IN HOST MACHINE AND DECIDE ACCORDINGLY IP ADDRESS IF DOING MANUALLY]
============================================================

## PART 1 – Create Internet VM in VMware

1. Open VMware Workstation
2. Create New Virtual Machine
3. Select Typical
4. Select RHEL ISO
5. Name: Internet-VM
6. CPU: 2
7. RAM: 4GB
8. Disk: 40GB
9. Add 2 Network Adapters:
   - Adapter 1 → NAT (Internet)
   - Adapter 2 → VMnet2 (Lab Network)

============================================================

## PART 2 – Create VMnet2 (If Not Created)

VMware → Edit → Virtual Network Editor  
Change Settings  

Add Network → VMnet2  

Type: Host-only  
Subnet: 192.168.126.0  
Mask: 255.255.255.0  
Disable DHCP  

============================================================

## PART 3 – Configure Static IP (VMnet2)

Check interfaces:

ip a
nmcli connection show

Example interface: ens37

Set static IP:

nmcli connection modify ens37 \
ipv4.method manual \
ipv4.addresses 192.168.126.20/24 \
ipv4.gateway 192.168.126.1 \
ipv4.dns 8.8.8.8

Activate:

nmcli connection up ens37

Verify:

ip a
ip route
cat /etc/resolv.conf

============================================================

## PART 4 – Verify Internet

ping google.com
curl https://registry.k8s.io

If DNS fails:
Add in /etc/resolv.conf:

nameserver 8.8.8.8

============================================================

## PART 5 – Register RHEL

subscription-manager register
subscription-manager attach --auto
subscription-manager status

Enable repos (RHEL 10 example):

subscription-manager repos --enable=rhel-10-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-10-for-x86_64-appstream-rpms

Verify:

dnf repolist

============================================================

## PART 6 – Install Docker [FOR INTERNET VM]

dnf install -y yum-utils
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl enable docker --now
docker version

## PART 6.1 – Install Docker [FOR HARBOR VM] [NO INTERNET]

mkdir /root/docker-rpms
cd /root/docker-rpms
dnf install -y yum-utils
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf download --resolve \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
ls -lh
scp *.rpm root@192.168.126.10:/root/docker-rpms/



============================================================

## PART 7 – Harbor HTTPS Certificate Trust (IMPORTANT)

When Harbor uses HTTPS with self-signed certificate,
Docker on Internet VM will show error:

x509: certificate signed by unknown authority

This means:
Internet VM does NOT trust Harbor's certificate.

Harbor VM generated CA certificate here:

/data/cert/ca.crt

We must copy this CA certificate
FROM Harbor VM
TO Internet VM.

------------------------------------------------------------

### STEP 1 – Copy CA Certificate From Harbor VM

Run this command on Internet VM:

scp root@192.168.126.10:/data/cert/ca.crt /root/

Explanation:

Source  → Harbor VM  
Destination → Internet VM  

Now verify:

ls -lh /root/ca.crt

If file exists → OK.

------------------------------------------------------------

### METHOD 1 – Docker-Specific Trust (Recommended for Lab)

Create Docker cert directory:

mkdir -p /etc/docker/certs.d/192.168.126.10

Copy CA certificate:

cp /root/ca.crt /etc/docker/certs.d/192.168.126.10/ca.crt

Restart Docker:

systemctl restart docker

Test:

docker login 192.168.126.10

If login works → HTTPS trust successful.

------------------------------------------------------------

### METHOD 2 – System-Wide Trust (Enterprise Recommended)

Instead of Docker-only trust,
we can trust CA at OS level.

Copy certificate:

cp /root/ca.crt /etc/pki/ca-trust/source/anchors/

Update system trust store:

update-ca-trust

Restart Docker:

systemctl restart docker

Now entire system trusts Harbor CA.

Verify:

curl -v https://192.168.126.10

No certificate error should appear.

------------------------------------------------------------

### If Still Failing

Check:

ls /etc/docker/certs.d/192.168.126.10/

Make sure file name is:

ca.crt

Restart Docker again:

systemctl restart docker


------------------------------------------------------------
## PART 8 – Download Kubernetes Binaries

wget https://dl.k8s.io/v1.29.15/bin/linux/amd64/kubeadm
wget https://dl.k8s.io/v1.29.15/bin/linux/amd64/kubelet
wget https://dl.k8s.io/v1.29.15/bin/linux/amd64/kubectl

chmod +x kubeadm kubelet kubectl

Verify:

ls -lh kube*

============================================================

## PART 9 – Download Runtime Components

### containerd

wget https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz

Verify:
ls -lh containerd-1.7.13-linux-amd64.tar.gz

------------------------------------------------------------

### runc

wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64

Verify:
ls -lh runc.amd64

------------------------------------------------------------

### CNI Plugins (VERY IMPORTANT)

wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz

Verify:
ls -lh cni-plugins-linux-amd64-v1.4.0.tgz

If missing in air-gapped cluster:
Pods fail with:
FailedCreatePodSandbox
failed to find plugin "loopback"

------------------------------------------------------------

### CRI Tools (crictl)

wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.29.0/crictl-v1.29.0-linux-amd64.tar.gz

Verify:
ls -lh crictl-v1.29.0-linux-amd64.tar.gz

Purpose:
Used to debug container runtime in Kubernetes nodes.

============================================================

## PART 10 – Download Harbor Offline Installer

wget https://github.com/goharbor/harbor/releases/download/v2.10.1/harbor-offline-installer-v2.10.1.tgz

Verify:
ls -lh harbor-offline-installer-v2.10.1.tgz

============================================================

## PART 11 – SCP Files to Harbor VM

scp harbor-offline-installer-v2.10.1.tgz root@192.168.126.10:/root/

============================================================

## PART 12 – SCP Runtime & K8s Files to Kubernetes VM

scp kube* root@192.168.126.30:/root/
scp containerd-*.tar.gz root@192.168.126.30:/root/
scp runc.amd64 root@192.168.126.30:/root/
scp cni-plugins-*.tgz root@192.168.126.30:/root/
scp crictl-*.tar.gz root@192.168.126.30:/root/

============================================================

## PART 13 – Login to Harbor on Internet Vm

docker login 192.168.126.10
admin:
password:
[your harbor admin and password]


============================================================


[MAKE SURE U HAVE DOCKER INSTALL IN MACHINES] IF NOT REFER TO STEP 6
## PART 14 – Pull & Push ALL Kubernetes Images

kubeadm config images list --kubernetes-version v1.29.15

docker pull registry.k8s.io/kube-apiserver:v1.29.15
docker tag registry.k8s.io/kube-apiserver:v1.29.15 192.168.126.10/k8s/kube-apiserver:v1.29.15
docker push 192.168.126.10/k8s/kube-apiserver:v1.29.15

docker pull registry.k8s.io/kube-controller-manager:v1.29.15
docker tag registry.k8s.io/kube-controller-manager:v1.29.15 192.168.126.10/k8s/kube-controller-manager:v1.29.15
docker push 192.168.126.10/k8s/kube-controller-manager:v1.29.15

docker pull registry.k8s.io/kube-scheduler:v1.29.15
docker tag registry.k8s.io/kube-scheduler:v1.29.15 192.168.126.10/k8s/kube-scheduler:v1.29.15
docker push 192.168.126.10/k8s/kube-scheduler:v1.29.15

docker pull registry.k8s.io/kube-proxy:v1.29.15
docker tag registry.k8s.io/kube-proxy:v1.29.15 192.168.126.10/k8s/kube-proxy:v1.29.15
docker push 192.168.126.10/k8s/kube-proxy:v1.29.15

docker pull registry.k8s.io/etcd:3.5.12-0
docker tag registry.k8s.io/etcd:3.5.12-0 192.168.126.10/k8s/etcd:3.5.12-0
docker push 192.168.126.10/k8s/etcd:3.5.12-0

docker pull registry.k8s.io/pause:3.9
docker tag registry.k8s.io/pause:3.9 192.168.126.10/k8s/pause:3.9
docker push 192.168.126.10/k8s/pause:3.9

docker pull registry.k8s.io/coredns/coredns:v1.11.1
docker tag registry.k8s.io/coredns/coredns:v1.11.1 192.168.126.10/k8s/coredns:v1.11.1
docker push 192.168.126.10/k8s/coredns:v1.11.1

============================================================

## PART 15 – Push Flannel Images

docker pull flannel/flannel:v0.25.5
docker tag flannel/flannel:v0.25.5 192.168.126.10/k8s/flannel:v0.25.5
docker push 192.168.126.10/k8s/flannel:v0.25.5

docker pull flannel/flannel-cni-plugin:v1.5.1-flannel1
docker tag flannel/flannel-cni-plugin:v1.5.1-flannel1 192.168.126.10/k8s/flannel-cni-plugin:v1.5.1-flannel1
docker push 192.168.126.10/k8s/flannel-cni-plugin:v1.5.1-flannel1


## Once all push check on harbor ui whether all images push or not

============================================================




## FINAL CHECKPOINT (Before Shutting Down Internet VM)

✔ Kubernetes binaries downloaded
✔ containerd downloaded
✔ runc downloaded
✔ CNI plugins downloaded
✔ CRI tools downloaded
✔ Harbor installer copied
✔ All images pushed to Harbor
✔ Harbor HTTPS trusted

Internet VM COMPLETE.
