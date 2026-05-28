# K3s Cluster Setup Guide

Step-by-step guide for setting up a K3s single-node cluster on Ubuntu Server 22.04, including OS preparation, static IP configuration, and K3s installation.

---

## Prerequisites

- Dedicated physical machine (Lenovo ThinkCentre M715q or similar)
- Ubuntu Server 22.04 LTS flashed to USB
- Network access
- Router admin access for static IP assignment

---

## 1. Install Ubuntu Server 22.04

Flash Ubuntu Server 22.04 LTS to a USB drive using Balena Etcher or `dd` and boot from it on the target machine. Follow the installer prompts:

- Select **Ubuntu Server (minimized)** for a leaner install
- Set hostname during install (or change it after)
- Create user `scavenger` with a strong password
- Enable OpenSSH during install for remote access

---

## 2. Initial System Update

After first boot, SSH in and update:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

After reboot confirm the OS version:

```bash
lsb_release -a
```

Expected output:
```
Description: Ubuntu 22.04.5 LTS
```

---

## 3. Set Static IP

Find the netplan config file:

```bash
ls /etc/netplan/
```

The file will be named `50-cloud-init.yaml` or similar. Edit it:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Replace the contents with:

```yaml
network:
  version: 2
  ethernets:
    enp2s0f0:
      dhcp4: false
      addresses:
        - K3S-NODE-IP/24
      routes:
        - to: default
          via: GATEWAY-IP
      nameservers:
        addresses:
          - ADGUARD-PRIMARY-IP
          - ADGUARD-SECONDARY-IP
```

Replace placeholders with your actual values. Apply the config:

```bash
sudo netplan apply
```

Verify:

```bash
ip a | grep K3S-NODE-IP
```

---

## 4. Set Hostname

```bash
sudo hostnamectl set-hostname k3s-node-01
```

Update `/etc/hosts` to match:

```bash
sudo nano /etc/hosts
```

Change the `127.0.1.1` line to:

```
127.0.1.1 k3s-node-01
```

Verify:

```bash
hostnamectl
```

Expected output includes:
```
Static hostname: k3s-node-01
```

---

## 5. Disable Swap

K3s requires swap to be disabled:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

The `sed` command comments out the swap entry in `/etc/fstab` so it stays disabled after reboot.

---

## 6. Install Basic Tools

```bash
sudo apt install -y curl wget git htop net-tools
```

---

## 7. Reboot

```bash
sudo reboot
```

SSH back in after reboot and confirm everything looks good before proceeding to K3s installation.

---

## 8. Install K3s

```bash
curl -sfL https://get.k3s.io | sh -
```

K3s installs itself as a systemd service and starts automatically. The install script:
- Downloads the K3s binary
- Creates kubectl, crictl, and ctr symlinks
- Creates a systemd service file
- Starts the K3s service

Wait about 60 seconds for the cluster to initialize, then verify:

```bash
sudo kubectl get nodes
```

Expected output:
```
NAME          STATUS   ROLES           AGE   VERSION
k3s-node-01   Ready    control-plane   76s   v1.35.5+k3s1
```

---

## 9. Verify System Pods

```bash
sudo kubectl get pods -A
```

All pods should show `Running` or `Completed`:

```
NAMESPACE     NAME                                      READY   STATUS
kube-system   coredns-xxx                               1/1     Running
kube-system   helm-install-traefik-xxx                  0/1     Completed
kube-system   helm-install-traefik-crd-xxx              0/1     Completed
kube-system   local-path-provisioner-xxx                1/1     Running
kube-system   metrics-server-xxx                        1/1     Running
kube-system   svclb-traefik-xxx                         2/2     Running
kube-system   traefik-xxx                               1/1     Running
```

---

## 10. Configure DNS in AdGuard Home

For each K3s service, add a DNS rewrite in AdGuard Home:

- **Domain:** `*.k3s.scavenger` (or per-service entries)
- **Answer:** K3s node IP

This allows accessing services via `http://service.k3s.scavenger` from any device on the LAN using AdGuard as DNS.

Also add entries to `/etc/hosts` on your workstation as a fallback:

```bash
sudo nano /etc/hosts
```

Add one line per service:
```
K3S-NODE-IP homepage.k3s.scavenger
K3S-NODE-IP argocd.k3s.scavenger
K3S-NODE-IP grafana.k3s.scavenger
K3S-NODE-IP traefik.k3s.scavenger
```

---

## 11. Allow kubectl Without sudo (Optional)

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

After this `kubectl` works without `sudo`.

---

## Notes

- K3s includes Traefik as the default ingress controller — no separate ingress installation needed
- K3s uses `containerd` as the container runtime, not Docker
- The kubeconfig is stored at `/etc/rancher/k3s/k3s.yaml` — keep this file secure
- K3s runs as a systemd service: `sudo systemctl status k3s`
- To uninstall K3s: `/usr/local/bin/k3s-uninstall.sh`
