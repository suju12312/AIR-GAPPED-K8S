# Harbor-VM.md
Air-Gapped Harbor Registry Setup (RHEL 10)
FINAL COMPLETE ENTERPRISE VERSION

============================================================

ARCHITECTURE

Internet VM     → 192.168.126.20  (Has Internet)
Harbor VM       → 192.168.126.10  (NO Internet)
Kubernetes VM   → 192.168.126.30  (NO Internet)

All VMs communicate via VMnet2 (Host-Only Network)

============================================================
PART 1 – VMware Network Configuration (CRITICAL)

------------------------------------------------------------
STEP 1 – Create VMnet2 (Host-Only Network)

VMware → Edit → Virtual Network Editor
Click → Change Settings (Admin)

Click → Add Network
Select → VMnet2

Set:

Type: Host-Only
Subnet IP: 192.168.126.0
Subnet Mask: 255.255.255.0

Disable DHCP (Recommended)

Click OK

------------------------------------------------------------
STEP 2 – Attach Harbor VM to VMnet2 ONLY

VM Settings → Network Adapter

Select:
Custom → VMnet2

⚠ DO NOT use NAT
⚠ DO NOT use Bridged

Harbor must be fully air-gapped.

============================================================
PART 2 – Configure Static IP (Harbor VM)

nmcli connection modify ens33 \
ipv4.method manual \
ipv4.addresses 192.168.126.10/24 \
ipv4.gateway 192.168.126.1 \
ipv4.dns 8.8.8.8

nmcli connection up ens33

Verify:

ip a
ping 192.168.126.20
ping 192.168.126.30

============================================================
PART 3 – Install Docker (OFFLINE METHOD)

------------------------------------------------------------
On Internet VM (with internet):

mkdir /root/docker-rpms
cd /root/docker-rpms

dnf download --resolve \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin

Transfer to Harbor VM:

scp *.rpm root@192.168.126.10:/root/docker-rpms/

------------------------------------------------------------
On Harbor VM:

cd /root/docker-rpms
dnf install -y *.rpm

Start Docker:

systemctl enable docker --now

Verify:

docker version

============================================================
PART 4 – Harbor Offline Installer

On Internet VM:

scp harbor-offline-installer-v2.10.1.tgz root@192.168.126.10:/root/

------------------------------------------------------------
On Harbor VM:

cd /root
tar -xzf harbor-offline-installer-v2.10.1.tgz
cd harbor

============================================================
PART 5 – Generate HTTPS Certificates

mkdir -p /data/cert
cd /data/cert

------------------------------------------------------------
Generate CA

openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 \
-subj "/CN=harbor.local" \
-key ca.key \
-out ca.crt

------------------------------------------------------------
Generate Harbor Key

openssl genrsa -out harbor.key 4096

openssl req -sha512 -new \
-subj "/CN=192.168.126.10" \
-key harbor.key \
-out harbor.csr

------------------------------------------------------------
Create v3.ext

cat > v3.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
IP.1=192.168.126.10
EOF

------------------------------------------------------------
Sign Certificate

openssl x509 -req -sha512 -days 3650 \
-extfile v3.ext \
-CA ca.crt -CAkey ca.key -CAcreateserial \
-in harbor.csr \
-out harbor.crt

============================================================
PART 6 – Configure harbor.yml

cd /root/harbor

cp harbor.yml.tmpl harbor.yml

Edit harbor.yml:

hostname: 192.168.126.10

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key

============================================================
PART 7 – Install Harbor

./prepare
./install.sh

Verify:

docker ps

Harbor containers should be running.

============================================================
PART 8 – Restart / Manage Harbor

cd /root/harbor

Stop:
docker compose down

Start:
docker compose up -d

Restart:
docker compose restart

============================================================
PART 9 – Fix HTTPS Trust (Internet VM Side)

If pushing images gives:

x509: certificate signed by unknown authority

------------------------------------------------------------
Method 1 – Recommended

On Internet VM:

scp root@192.168.126.10:/data/cert/ca.crt /root/

mkdir -p /etc/docker/certs.d/192.168.126.10

cp ca.crt /etc/docker/certs.d/192.168.126.10/ca.crt

systemctl restart docker

Test:

docker login 192.168.126.10

------------------------------------------------------------
Method 2 – System Wide Trust

cp ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
systemctl restart docker

============================================================
PART 10 – Emergency Fallback (docker save/load)

If push from Internet VM fails:

On Internet VM:

docker pull nginx
docker save nginx -o nginx.tar

scp nginx.tar root@192.168.126.10:/root/

------------------------------------------------------------
On Harbor VM:

docker load -i nginx.tar

docker tag nginx 192.168.126.10/k8s/nginx:latest

docker push 192.168.126.10/k8s/nginx:latest

============================================================
PART 11 – Common Errors & Fixes

❌ x509 error
→ Check ca.crt in /etc/docker/certs.d/
→ Restart Docker

❌ Harbor containers down
docker ps -a
docker logs harbor-core

❌ docker compose missing
docker compose version
Install docker-compose-plugin offline

❌ Port 443 conflict
ss -tulnp | grep 443

❌ Harbor IP changed
Edit harbor.yml
./prepare
docker compose down
docker compose up -d

============================================================
PART 12 – Backup Harbor

All registry data is stored in:

/data/

Backup:

tar -czvf harbor-backup.tar.gz /data

============================================================
FINAL CHECKLIST

✔ VMware VMnet2 created
✔ Harbor VM Host-Only only
✔ Static IP configured
✔ Docker installed offline
✔ Harbor offline installer used
✔ ./prepare executed
✔ ./install.sh successful
✔ HTTPS configured
✔ CA exported to Internet VM
✔ Docker certs.d configured
✔ update-ca-trust configured
✔ docker save/load fallback ready
✔ Harbor containers running
✔ Images push successful

HARBOR VM COMPLETE – FULL ENTERPRISE AIR-GAPPED READY
