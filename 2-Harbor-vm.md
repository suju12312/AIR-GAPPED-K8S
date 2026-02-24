# 🏗️ Air-Gapped Harbor Registry Setup Guide

> **VMware + RHEL 10** | Full offline Harbor private registry for air-gapped Kubernetes clusters.

---

## 📋 Architecture Overview

| VM | IP Address | Internet Access |
|----|------------|-----------------|
| Internet VM | `192.168.126.20` | ✅ Yes |
| Harbor VM | `192.168.126.10` | ❌ No |
| Kubernetes VM | `192.168.126.30` | ❌ No |

> All VMs communicate via **VMnet2** (Host-Only Network)

---

## Table of Contents

- [Part 1 – VMware Network Configuration](#part-1--vmware-network-configuration)
- [Part 2 – Configure Static IP](#part-2--configure-static-ip)
- [Part 3 – Install Docker (Offline)](#part-3--install-docker-offline)
- [Part 4 – Harbor Offline Installer](#part-4--harbor-offline-installer)
- [Part 5 – Generate HTTPS Certificates](#part-5--generate-https-certificates)
- [Part 6 – Configure harbor.yml](#part-6--configure-harboryml)
- [Part 7 – Install Harbor](#part-7--install-harbor)
- [Part 8 – Restart / Manage Harbor](#part-8--restart--manage-harbor)
- [Part 9 – Fix HTTPS Trust (Internet VM)](#part-9--fix-https-trust-internet-vm-side)
- [Part 10 – Emergency Fallback (docker save/load)](#part-10--emergency-fallback-docker-saveload)
- [Part 11 – Common Errors & Fixes](#part-11--common-errors--fixes)
- [Part 12 – Backup Harbor](#part-12--backup-harbor)
- [Final Checklist](#final-checklist)

---

## Part 1 – VMware Network Configuration

### Step 1 – Create VMnet2 (Host-Only Network)

Navigate to: **VMware → Edit → Virtual Network Editor → Change Settings → Add Network**

| Setting | Value |
|---------|-------|
| Name | `VMnet2` |
| Type | Host-Only |
| Subnet IP | `192.168.126.0` |
| Subnet Mask | `255.255.255.0` |
| DHCP | Disabled |

---

### Step 2 – Attach Harbor VM to VMnet2 Only

Navigate to: **VM Settings → Network Adapter**

Select:

```
Custom → VMnet2
```

> ⚠️ **DO NOT** use NAT  
> ⚠️ **DO NOT** use Bridged  
>
> Harbor must be **fully air-gapped.**

---

## Part 2 – Configure Static IP

**Set static IP on Harbor VM** (replace `ens33` with your actual interface):

```bash
nmcli connection modify ens33 \
  ipv4.method manual \
  ipv4.addresses 192.168.126.10/24 \
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
ping 192.168.126.20
```

```bash
ping 192.168.126.30
```

---

## Part 3 – Install Docker (Offline)

### On Internet VM — Download RPMs

```bash
mkdir /root/docker-rpms
cd /root/docker-rpms
```

```bash
dnf download --resolve \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

**Transfer RPMs to Harbor VM:**

```bash
scp *.rpm root@192.168.126.10:/root/docker-rpms/
```

---

### On Harbor VM — Install from RPMs

```bash
cd /root/docker-rpms
```

```bash
dnf install -y *.rpm
```

**Start and enable Docker:**

```bash
systemctl enable docker --now
```

**Verify:**

```bash
docker version
```

---

## Part 4 – Harbor Offline Installer

### On Internet VM — Transfer Installer

```bash
scp harbor-offline-installer-v2.10.1.tgz root@192.168.126.10:/root/
```

---

### On Harbor VM — Extract Installer

```bash
cd /root
```

```bash
tar -xzf harbor-offline-installer-v2.10.1.tgz
```

```bash
cd harbor
```

---

## Part 5 – Generate HTTPS Certificates

```bash
mkdir -p /data/cert
cd /data/cert
```

---

### Generate CA Key & Certificate

```bash
openssl genrsa -out ca.key 4096
```

```bash
openssl req -x509 -new -nodes -sha512 -days 3650 \
  -subj "/CN=harbor.local" \
  -key ca.key \
  -out ca.crt
```

---

### Generate Harbor Server Key & CSR

```bash
openssl genrsa -out harbor.key 4096
```

```bash
openssl req -sha512 -new \
  -subj "/CN=192.168.126.10" \
  -key harbor.key \
  -out harbor.csr
```

---

### Create v3.ext File

```bash
cat > v3.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
IP.1=192.168.126.10
EOF
```

---

### Sign the Certificate

```bash
openssl x509 -req -sha512 -days 3650 \
  -extfile v3.ext \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -in harbor.csr \
  -out harbor.crt
```

---

## Part 6 – Configure harbor.yml

```bash
cd /root/harbor
```

```bash
cp harbor.yml.tmpl harbor.yml
```

**Edit `harbor.yml` and set the following values:**

```yaml
hostname: 192.168.126.10

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key
```

---

## Part 7 – Install Harbor

```bash
./prepare
```

```bash
./install.sh
```

**Verify containers are running:**

```bash
docker ps
```

> ✅ All Harbor containers should be in a running state.

---

## Part 8 – Restart / Manage Harbor

```bash
cd /root/harbor
```

**Stop Harbor:**

```bash
docker compose down
```

**Start Harbor:**

```bash
docker compose up -d
```

**Restart Harbor:**

```bash
docker compose restart
```

---

## Part 9 – Fix HTTPS Trust (Internet VM Side)

If pushing images gives the error:

```
x509: certificate signed by unknown authority
```

---

### Method 1 – Docker-Specific Trust *(Recommended)*

Run on **Internet VM:**

```bash
scp root@192.168.126.10:/data/cert/ca.crt /root/
```

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

### Method 2 – System-Wide Trust

```bash
cp /root/ca.crt /etc/pki/ca-trust/source/anchors/
```

```bash
update-ca-trust
```

```bash
systemctl restart docker
```

---

## Part 10 – Emergency Fallback (docker save/load)

> Use this if pushing images directly from the Internet VM fails.

### On Internet VM — Save & Transfer Image

```bash
docker pull nginx
```

```bash
docker save nginx -o nginx.tar
```

```bash
scp nginx.tar root@192.168.126.10:/root/
```

---

### On Harbor VM — Load & Push Image

```bash
docker load -i nginx.tar
```

```bash
docker tag nginx 192.168.126.10/k8s/nginx:latest
```

```bash
docker push 192.168.126.10/k8s/nginx:latest
```

---

## Part 11 – Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `x509` certificate error | Check `ca.crt` exists in `/etc/docker/certs.d/` → Restart Docker |
| Harbor containers down | Run `docker ps -a` and `docker logs harbor-core` |
| `docker compose` missing | Install `docker-compose-plugin` offline RPM |
| Port 443 conflict | Run `ss -tulnp \| grep 443` to find the conflicting process |
| Harbor IP changed | Edit `harbor.yml` → Run `./prepare` → `docker compose down` → `docker compose up -d` |

---

## Part 12 – Backup Harbor

All registry data is stored in `/data/`. To create a full backup:

```bash
tar -czvf harbor-backup.tar.gz /data
```

---

## Final Checklist

| Task | Status |
|------|--------|
| VMware VMnet2 created | ✅ |
| Harbor VM set to Host-Only (VMnet2) | ✅ |
| Static IP configured | ✅ |
| Docker installed offline | ✅ |
| Harbor offline installer transferred | ✅ |
| `./prepare` executed | ✅ |
| `./install.sh` successful | ✅ |
| HTTPS configured | ✅ |
| CA certificate exported to Internet VM | ✅ |
| Docker `certs.d` configured | ✅ |
| `update-ca-trust` configured | ✅ |
| `docker save/load` fallback ready | ✅ |
| Harbor containers running | ✅ |
| Images pushed successfully | ✅ |

---

> **Harbor VM setup complete — Full Enterprise Air-Gapped Ready! 🎉**
