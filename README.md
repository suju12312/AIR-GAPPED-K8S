# Air-Gapped Kubernetes with Harbor Registry – Summary Overview

## 📌 Project Overview

This project documents a complete **enterprise-grade air-gapped Kubernetes cluster** deployment using Harbor as a private container registry. It provides a secure, offline infrastructure for organizations requiring network isolation and complete control over container images.

---

## 🎯 What This Setup Achieves

| Objective | Status |
|-----------|--------|
| Private Container Registry | ✅ Harbor Registry (no internet needed) |
| Offline Kubernetes Cluster | ✅ Single-node Kubernetes (v1.29.15) |
| Network Isolation | ✅ Air-gapped (no external internet access) |
| Secure HTTPS Communication | ✅ Self-signed certificates |
| Container Image Management | ✅ Harbor stores all K8s images |
| Enterprise Compliance | ✅ Full control over software supply chain |

---

## 🖥️ Three-Tier Infrastructure

### VMs and Their Roles

```
┌─────────────────────────────────────────────┐
│  INTERNET VM (192.168.126.20)               │
│  ✅ Has Internet Access (NAT)               │
│  Role: Download & Transfer All Files        │
└─────────────────────────────────────────────┘
              │ SCP Transfer
              ↓
┌──────────────────────────────────────────────┐
│  HARBOR VM (192.168.126.10)                  │
│  ❌ No Internet (Host-Only Network)          │
│  Role: Private Container Registry            │
└──────────────────────────────────────────────┘
              │ Image Pull
              ↓
┌──────────────────────────────────────────────┐
│  KUBERNETES VM (192.168.126.30)              │
│  ❌ No Internet (Host-Only Network)          │
│  Role: Kubernetes Cluster Master             │
└──────────────────────────────────────────────┘
```

---

## 📊 IP Address Summary

| Component | IP Address | Purpose |
|-----------|-----------|---------|
| **Internet VM** | `192.168.126.20` | Downloads from public registries, transfers files via SCP |
| **Harbor VM** | `192.168.126.10` | Hosts private registry on HTTPS:443 |
| **Kubernetes VM** | `192.168.126.30` | Kubernetes master node, pulls images from Harbor |
| **Network** | `192.168.126.0/24` | Host-Only network (VMnet2) |
| **Gateway** | `192.168.126.1` | VMware gateway |

---

## 🔒 Air-Gapped Network Explained

### What is Air-Gapping?

**Air-gapping** means creating an **isolated network with no internet connection**. It's a security boundary that prevents:
- Unauthorized external access
- Malware downloads
- Uncontrolled software updates
- Data exfiltration

### How It Works in This Setup

| Step | Action | Location |
|------|--------|----------|
| 1 | Download all binaries & container images | Internet VM (has internet) |
| 2 | Transfer files via SCP to Harbor | Harbor VM (no internet) |
| 3 | Harbor stores images privately | Harbor Registry (isolated) |
| 4 | Kubernetes pulls images from Harbor only | Kubernetes VM (no internet) |

**Result:** Kubernetes cluster operates completely offline with zero internet dependency.

---

## 🏗️ Harbor Registry Overview

### What is Harbor?

**Harbor** is a **private Docker registry** – similar to Docker Hub but:
- Runs on your own infrastructure
- No public internet needed
- Full access control
- HTTPS secure by default
- Completely offline capable

### Harbor in This Setup

| Aspect | Details |
|--------|---------|
| **URL** | `https://192.168.126.10` |
| **Port** | 443 (HTTPS) |
| **Storage** | `/data/` directory on Harbor VM |
| **Certificates** | Self-signed, 10-year validity in `/data/cert/` |
| **Admin User** | `admin` / `Harbor12345` |
| **Repositories** | k8s (contains all Kubernetes images) |

### Harbor Image Storage

All Kubernetes container images are stored in Harbor:

```
Harbor Registry (192.168.126.10)
└── Project: k8s
    ├── kube-apiserver:v1.29.15
    ├── kube-controller-manager:v1.29.15
    ├── kube-scheduler:v1.29.15
    ├── kube-proxy:v1.29.15
    ├── etcd:3.5.12-0
    ├── pause:3.9
    ├── coredns:v1.11.1
    ├── flannel:v0.25.5
    └── flannel-cni-plugin:v1.5.1-flannel1
```

---

## 📋 Setup Workflow Summary

### Phase 1: Internet VM (Download & Prepare)

**Duration:** 30-45 minutes

1. Create Internet VM with RHEL 10
2. Configure dual network adapters (NAT + VMnet2)
3. Register with Red Hat and install Docker
4. **Download from public registries:**
   - Kubernetes binaries (kubeadm, kubelet, kubectl)
   - Container runtime (containerd, runc, CNI plugins)
   - Harbor offline installer
   - All K8s container images
   - Docker RPM packages
5. **Transfer via SCP to Harbor VM and Kubernetes VM**

### Phase 2: Harbor VM (Registry Setup)

**Duration:** 20-30 minutes

1. Create Harbor VM with Host-Only network only (no internet)
2. Configure static IP (192.168.126.10)
3. Install Docker from offline RPMs
4. Generate HTTPS self-signed certificates
5. Extract and configure Harbor offline installer
6. Start Harbor containers
7. Harbor now serves private registry at HTTPS:443

### Phase 3: Kubernetes VM (Cluster Initialization)

**Duration:** 25-40 minutes

1. Create Kubernetes VM with Host-Only network only (no internet)
2. Configure static IP (192.168.126.30)
3. Install container runtime (containerd)
4. Configure containerd to trust Harbor's self-signed certificate
5. Install CNI plugins for pod networking
6. Install Kubernetes binaries (kubeadm, kubelet, kubectl)
7. Run `kubeadm init` with Harbor as image repository
8. Install Flannel network plugin from Harbor images
9. Kubernetes cluster ready and fully offline

---

## 🔐 Security Features

### HTTPS Certificate Management

| Feature | Implementation |
|---------|-----------------|
| **Certificate Type** | Self-signed (no CA required) |
| **Validity Period** | 10 years |
| **Location** | `/data/cert/` on Harbor VM |
| **CA Certificate** | Distributed to Internet & Kubernetes VMs |
| **Trust Method** | Docker/containerd certificate store |

### Network Isolation

| VMs | Network | Internet | Purpose |
|-----|---------|----------|---------|
| Internet VM | NAT + VMnet2 | ✅ NAT | Download files |
| Harbor VM | VMnet2 only | ❌ None | Private registry |
| Kubernetes VM | VMnet2 only | ❌ None | Cluster master |

---

## 📦 Key Components & Versions

| Component | Version | Purpose |
|-----------|---------|---------|
| **Kubernetes** | v1.29.15 | Container orchestration |
| **containerd** | v1.7.13 | Container runtime |
| **runc** | v1.1.12 | Low-level container runtime |
| **CNI Plugins** | v1.4.0 | Pod networking |
| **Harbor** | v2.10.1 | Private registry |
| **Flannel** | v0.25.5 | Network plugin |
| **RHEL** | 10 | Host operating system |

---

## 🚀 Deployment Process Overview

### Download Phase (Internet VM)

```bash
# Kubernetes binaries
kubeadm, kubelet, kubectl (v1.29.15)

# Container runtime
containerd (v1.7.13), runc (v1.1.12)

# Networking
cni-plugins-linux-amd64-v1.4.0.tgz
crictl (v1.29.0)

# Registry & Images
harbor-offline-installer-v2.10.1.tgz
All K8s images (8 container images)
All Flannel images (2 container images)

# System packages
Docker RPMs for offline installation
```

### Transfer Phase (SCP)

```
Internet VM
    ├─ docker-rpms/*.rpm ──────────→ Harbor VM
    ├─ harbor-offline-installer.tgz ──────────→ Harbor VM
    │
    ├─ kubeadm, kubelet, kubectl ──────────→ Kubernetes VM
    ├─ containerd-*.tar.gz ────────────────→ Kubernetes VM
    ├─ runc.amd64 ─────────────────────────→ Kubernetes VM
    ├─ cni-plugins-*.tgz ──────────────────→ Kubernetes VM
    └─ crictl-*.tar.gz ────────────────────→ Kubernetes VM
```

### Push Phase (Docker Push)

```
Internet VM pulls images from public registries
    └─ docker push to Harbor Registry (192.168.126.10)
           └─ Images stored in Harbor
               └─ Kubernetes VM pulls images from Harbor
                   └─ Kubernetes containers start
```

---

## 📁 File Distribution

### Files Downloaded on Internet VM

| Directory | Contents | Size |
|-----------|----------|------|
| `/root/k8s-binaries/` | kubeadm, kubelet, kubectl | ~200 MB |
| `/root/k8s-binaries/` | containerd, runc, CNI, crictl | ~500 MB |
| `/root/` | harbor-offline-installer.tgz | ~600 MB |
| `/root/docker-rpms/` | Docker packages for offline install | ~800 MB |

### Files on Harbor VM

| Path | Contents |
|------|----------|
| `/root/harbor/` | Harbor application (extracted) |
| `/data/` | Harbor data and images storage |
| `/data/cert/` | HTTPS certificates (ca.crt, harbor.crt, harbor.key) |

### Files on Kubernetes VM

| Path | Contents |
|------|----------|
| `/usr/local/bin/` | kubeadm, kubelet, kubectl |
| `/usr/local/bin/` | containerd, runc, crictl |
| `/opt/cni/bin/` | CNI plugin binaries |
| `/etc/kubernetes/` | Kubernetes configuration |
| `/etc/containerd/` | containerd configuration with Harbor CA |

---

## ✅ Verification Checklist

### Network Connectivity
- [ ] Internet VM can ping Harbor VM (192.168.126.10)
- [ ] Internet VM can ping Kubernetes VM (192.168.126.30)
- [ ] Harbor VM can ping Kubernetes VM
- [ ] Harbor VM cannot access external internet (expected)
- [ ] Kubernetes VM cannot access external internet (expected)

### Internet VM
- [ ] Docker installed and running
- [ ] Can download from public registries (docker pull)
- [ ] All files downloaded successfully
- [ ] All files transferred via SCP

### Harbor VM
- [ ] Docker installed and running
- [ ] Harbor containers running (docker ps)
- [ ] HTTPS accessible (curl -k https://192.168.126.10)
- [ ] Can login with credentials
- [ ] All K8s images present in Harbor

### Kubernetes VM
- [ ] containerd running and configured
- [ ] Harbor CA certificate installed
- [ ] CNI plugins installed in `/opt/cni/bin/`
- [ ] Kubernetes binaries in `/usr/local/bin/`
- [ ] `kubeadm init` completed successfully
- [ ] Node status shows "Ready"
- [ ] Flannel pods running
- [ ] All system pods in "Running" status

---

## 🎓 Key Concepts

### Air-Gapped vs Connected Deployments

**Connected Deployment:**
```
Kubernetes → Internet → Public Registry (registry.k8s.io)
                    ↓
          Download images on-demand
```

**Air-Gapped Deployment:**
```
Internet VM → Harbor (192.168.126.10)
                    ↓
            Kubernetes ← Harbor
          (offline, no internet needed)
```

### Why Three VMs?

1. **Internet VM** – Needs internet to download; can't be air-gapped
2. **Harbor VM** – Private registry; must be air-gapped for security
3. **Kubernetes VM** – Production cluster; must be air-gapped for compliance

**Alternative:** Could use 2 VMs if air-gapping only Harbor & Kubernetes, but 3 VMs is cleaner.

### Why Self-Signed Certificates?

- No CA or certificate authority needed
- Works completely offline
- Harbor generates its own CA
- Valid for 10 years
- Must be imported to Docker/containerd certificate stores

---

## 🔧 Common Operations

### Restart Harbor

```bash
cd /root/harbor
docker compose down
docker compose up -d
```

### Restart Kubernetes

```bash
systemctl restart kubelet
kubectl get nodes  # Wait for Ready status
```

### Reset Everything

```bash
# On Harbor VM
cd /root/harbor && docker compose down

# On Kubernetes VM
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/etcd
systemctl restart containerd kubelet
```

### Push More Images to Harbor

```bash
# On Internet VM
docker pull [image]
docker tag [image] 192.168.126.10/k8s/[name]
docker push 192.168.126.10/k8s/[name]
```

---

## 📚 Three Documentation Files

This summary is based on three complete guides:

1. **Internet VM Setup** – Downloading and preparing all components
2. **Harbor Registry Setup** – Private registry configuration and HTTPS
3. **Kubernetes VM Setup** – Cluster initialization and network integration

Each file contains detailed step-by-step instructions for its respective component.

---

## 🎯 Use Cases

This air-gapped setup is ideal for:

✅ **Government & Defense** – Isolated networks required by regulation  
✅ **Financial Services** – PCI-DSS compliance  
✅ **Healthcare** – HIPAA compliance  
✅ **Critical Infrastructure** – Maximum security posture  
✅ **Offline/Remote Locations** – No internet connectivity available  
✅ **Supply Chain Security** – Complete control over software  
✅ **Development Labs** – Secure isolated testing environment  

---

## 📈 Performance Characteristics

| Metric | Value |
|--------|-------|
| **Setup Time** | 2-3 hours |
| **Network Latency** | < 1ms (local network) |
| **Registry Access** | < 100ms (Harbor at 192.168.126.10) |
| **Image Pull Speed** | Limited by disk I/O (not network) |
| **Kubernetes API** | Sub-second response (local) |

---

## 🛡️ Security Posture

| Security Aspect | Implementation |
|-----------------|-----------------|
| **Network Isolation** | Air-gapped (no internet routes) |
| **Encrypted Communication** | HTTPS on all services |
| **Certificate Management** | Self-signed, offline-compatible |
| **Access Control** | Harbor admin authentication |
| **Image Integrity** | Stored in Harbor, signed on push |
| **Audit Trail** | Docker logs, Harbor logs |
| **Compliance** | Configurable for HIPAA, PCI-DSS, etc. |

---

## 💡 Key Takeaways

1. **Three separate networks** – Internet VM has internet; Harbor & K8s are air-gapped
2. **Harbor is the registry** – All container images sourced from Harbor only
3. **Offline by design** – Once setup, K8s needs zero internet access
4. **Enterprise-ready** – HTTPS, authentication, audit logging included
5. **Self-contained** – Everything needed is downloaded during setup phase
6. **Secure supply chain** – You control exactly what software runs
7. **Compliant** – Meets regulatory requirements for isolated deployments

---

## 📖 Next Steps

1. Review complete documentation files for detailed procedures
2. Set up lab environment in VMware Workstation
3. Follow each phase: Internet VM → Harbor VM → Kubernetes VM
4. Verify each component before moving to next phase
5. Test offline functionality (disconnect Internet VM)
6. Deploy applications to Kubernetes cluster
7. Monitor logs and maintain Harbor/Kubernetes as needed

---

## 📞 Support Resources

- **Kubernetes Documentation:** https://kubernetes.io/docs/
- **Harbor Project:** https://goharbor.io/
- **containerd GitHub:** https://github.com/containerd/containerd
- **Flannel Networking:** https://github.com/flannel-io/flannel
- **VMware Documentation:** VMware Workstation Pro User Guide

---

**Status:** ✅ Complete Setup Ready for Deployment

*Air-gapped Kubernetes with Harbor Registry – Enterprise-Grade Offline Infrastructure*
