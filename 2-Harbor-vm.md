# Harbor-VM.md
Air-Gapped Harbor Registry – FINAL COMPLETE VERSION

============================================================

ARCHITECTURE

Internet VM (192.168.126.20)
→ Has Internet
→ Downloads everything

Harbor VM (192.168.126.10)
→ NO Internet
→ Host-only network only

Kubernetes VM (192.168.126.30)
→ NO Internet
→ Pulls images from Harbor only

============================================================

PART 1 – Harbor VM Network (NO INTERNET)

VMware:
Network Adapter → Custom → VMnet2 ONLY

Static IP:

nmcli connection modify ens33 \
ipv4.method manual \
ipv4.addresses 192.168.126.10/24 \
ipv4.gateway 192.168.126.1 \
ipv4.dns 8.8.8.8

nmcli connection up ens33

Verify:

ip a
ping 192.168.126.20

============================================================

PART 2 – Install Docker (OFFLINE)

On Internet VM:

mkdir /root/docker-rpms
cd /root/docker-rpms

dnf download --resolve \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin

scp *.rpm root@192.168.126.10:/root/docker-rpms/

------------------------------------------------------------

On Harbor VM:

cd /root/docker-rpms
dnf install -y *.rpm

systemctl enable docker --now
docker version

============================================================

PART 3 – Harbor Offline Installer

On Internet VM:

scp harbor-offline-installer-v2.10.1.tgz root@192.168.126.10:/root/

On Harbor VM:

cd /root
tar -xzf harbor-offline-installer-v2.10.1.tgz
cd harbor

============================================================

PART 4 – Generate HTTPS Certificate

mkdir -p /data/cert
cd /data/cert

openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 \
-subj "/CN=harbor.local" \
-key ca.key \
-out ca.crt

openssl genrsa -out harbor.key 4096

openssl req -sha512 -new \
-subj "/CN=192.168.126.10" \
-key harbor.key \
-out harbor.csr

Create extension:

cat > v3.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
IP.1=192.168.126.10
EOF

openssl x509 -req -sha512 -days 3650 \
-extfile v3.ext \
-CA ca.crt -CAkey ca.key -CAcreateserial \
-in harbor.csr \
-out harbor.crt

============================================================

PART 5 – Configure harbor.yml

cd /root/harbor
cp harbor.yml.tmpl harbor.yml

Edit:

hostname: 192.168.126.10

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key

============================================================

PART 6 – Prepare & Install Harbor

./prepare
./install.sh

Verify:

docker ps

============================================================

PART 7 – Restart Harbor

cd /root/harbor

Stop:
docker compose down

Start:
docker compose up -d

Restart only:
docker compose restart

============================================================

PART 8 – If Push Not Working (Workaround Method)

If Internet VM cannot push images:

Method 1 – Fix certificate trust (Recommended)

On Internet VM:

mkdir -p /etc/docker/certs.d/192.168.126.10
cp ca.crt /etc/docker/certs.d/192.168.126.10/ca.crt
systemctl restart docker

OR system-wide:

cp ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
systemctl restart docker

------------------------------------------------------------

Method 2 – Use docker save/load (Emergency Offline Method)

On Internet VM:

docker pull nginx
docker save nginx -o nginx.tar

scp nginx.tar root@192.168.126.10:/root/

On Harbor VM:

docker load -i nginx.tar
docker tag nginx 192.168.126.10/k8s/nginx:latest
docker push 192.168.126.10/k8s/nginx:latest

============================================================

PART 9 – Common Errors & Fixes

------------------------------------------------------------

❌ x509 certificate error

Check:

/etc/docker/certs.d/192.168.126.10/ca.crt exists

Restart docker.

------------------------------------------------------------

❌ certificate has .cert extension issue

Docker expects:

ca.crt

Rename if needed:

mv harbor.crt harbor.cert
mv harbor.cert harbor.crt

------------------------------------------------------------

❌ Harbor IP changed

Update:

harbor.yml

Run:

./prepare
docker compose down
docker compose up -d

------------------------------------------------------------

❌ Harbor containers down

Check:

docker ps -a
docker logs harbor-core

------------------------------------------------------------

❌ docker compose not working

Check:

docker compose version

If missing:
Install docker-compose-plugin

------------------------------------------------------------

❌ Port 443 already in use

ss -tulnp | grep 443

Stop conflicting service.

============================================================

PART 10 – Backup Harbor

All registry data:

/data/

Backup:

tar -czvf harbor-backup.tar.gz /data

============================================================

FINAL CHECKLIST

✔ Harbor VM has no Internet
✔ Docker installed offline
✔ Harbor offline installer used
✔ ./prepare executed
✔ ./install.sh successful
✔ HTTPS certificate configured
✔ CA exported to Internet VM
✔ Docker certs.d configured
✔ update-ca-trust configured
✔ docker save/load fallback method ready
✔ Harbor containers running
✔ Images can be pushed

HARBOR VM COMPLETE – ENTERPRISE AIR-GAPPED READY
