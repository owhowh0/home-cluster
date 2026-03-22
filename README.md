# home-cluster

Self-hosted Kubernetes cluster on 3 repurposed laptops.

## Hardware

| Hostname | Role | CPU | RAM |
|----------|------|-----|-----|
| acer-8gb | control-plane | Celeron | 8GB |
| asus-4gb | worker | Celeron | 4GB |
| asus-8gb | worker | AMD C-60 | 8GB |

## Progress

### ✅ Phase 1 — OS install (Alpine)

Alpine installer does not always install the kernel to disk.
After `setup-alpine`, verify:
```sh
ls /boot
# if only efi/ and grub/ are present, kernel is missing
```

Fix — chroot into the installed system and install manually:
```sh
mount /dev/sda3 /mnt
mount /dev/sda1 /mnt/boot
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /dev /mnt/dev
mount --bind /run /mnt/run
cp /etc/resolv.conf /mnt/etc/resolv.conf
chroot /mnt
apk add linux-lts
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=alpine --removable
grub-mkconfig -o /boot/grub/grub.cfg
exit
reboot
```

> Note: if network is unreachable inside chroot, bring up eth0 on
> the live ISO first before chrooting:
> ```sh
> ip link set eth0 up
> udhcpc -i eth0
> ```

### ✅ Phase 2 — cgroups (required for k3s)
```sh
apk add curl bash
rc-update add cgroups
rc-service cgroups start
sed -i 's/quiet/quiet cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1/' /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
reboot
```

### ✅ Phase 3 — iptables (required for k3s)

k3s fails silently without iptables. The LTS kernel does not load
ip_tables automatically:
```sh
apk add iptables ip6tables
modprobe ip_tables
echo "ip_tables" >> /etc/modules
```

### ✅ Phase 4 — k3s install

**Control plane (acer-8gb):**
```sh
curl -sfL https://get.k3s.io | sh -
rc-update add k3s default
```

k3s config (`/etc/rancher/k3s/config.yaml`):
```yaml
write-kubeconfig-mode: "644"
```

**Workers (asus-4gb, asus-8gb):**

> Note: installer requires sudo. Alpine uses doas by default so
> sudo must be installed and k8s user added to sudoers:
> ```sh
> apk add sudo
> echo "k8s ALL=(ALL) ALL" >> /etc/sudoers
> ```

> Note: do not put `write-kubeconfig-mode` in worker config.yaml
> — it is a server-only flag and will crash the agent with:
> `flag provided but not defined: -write-kubeconfig-mode`

Worker config (`/etc/rancher/k3s/config.yaml`):
```yaml
server: https://<CONTROL_PLANE_IP>:6443
token: <NODE_TOKEN>
```
```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<IP>:6443 K3S_TOKEN=<TOKEN> sh -
rc-update add k3s-agent default
```

### ✅ Phase 5 — user & SSH setup
```sh
adduser -h /home/k8s -s /bin/ash k8s
apk add doas openssh
echo "permit persist k8s as root" > /etc/doas.d/k8s.conf
# disable root login and password auth
sed -i 's/#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
rc-service sshd restart
rc-update add sshd default
```

Copy kubeconfig to management machine:
```sh
# on the node
doas cp /etc/rancher/k3s/k3s.yaml /home/k8s/k3s.yaml
doas chmod 644 /home/k8s/k3s.yaml

# on management machine (Fedora)
scp k8s@<NODE_IP>:/home/k8s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/<NODE_IP>/g' ~/.kube/config
chmod 600 ~/.kube/config
```

### ✅ Phase 6 — Flux CD bootstrap
```sh
# install flux CLI on management machine
curl -s https://fluxcd.io/install.sh | sudo bash

# bootstrap
export GITHUB_TOKEN=<TOKEN>
export GITHUB_USER=<USERNAME>
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=home-cluster \
  --branch=main \
  --path=clusters/home \
  --personal \
  --private=false
```

### ✅ Phase 7 — SOPS + age
```sh
# install on management machine
sudo dnf install age
curl -LO https://github.com/getsops/sops/releases/download/v3.9.4/sops-v3.9.4.x86_64.rpm
sudo rpm -i sops-v3.9.4.x86_64.rpm

# generate keypair
age-keygen -o age.agekey

# store private key in cluster
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age.agekey
```

`.sops.yaml` in repo root:
```yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: <PUBLIC_KEY>
```

### ✅ Phase 8 — Longhorn storage

Prerequisites on each node:
```sh
apk add open-iscsi nfs-utils util-linux
rc-update add iscsid default
rc-service iscsid start
```

Deployed via Flux + Helm with 2 replicas (3 is too heavy for this hardware).

## 🚧 In progress

- [ ] Pangolin Newt (tunnel client)
- [ ] Keycloak
- [ ] Home Assistant
- [ ] Paperless-ngx
- [ ] CouchDB (Obsidian LiveSync)
- [ ] Ente Auth
- [ ] Your Spotify
- [ ] Portainer
- [ ] Prometheus + Grafana

## Planned

- [ ] Xray (ports 80/443, after confirmation)
- [ ] Proxmox nodes joined as workers
