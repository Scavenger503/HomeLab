# vDSM Setup Guide

Step-by-step guide for setting up a Virtual DSM (vDSM) instance on a Synology NAS, installing Docker via Container Manager, connecting it to an existing Portainer instance, and deploying services.

---

## Prerequisites

- Synology NAS with Virtual Machine Manager package installed
- Existing Portainer CE installation on another host
- Static IP assigned to the vDSM instance via router DHCP reservation
- SSH access to both the vDSM and the primary Docker host

---

## 1. Create the vDSM Virtual Machine

In Synology Virtual Machine Manager:

1. Click **Create** → select **Syno** as the guest OS type
2. Configure resources — recommended minimum for a monitoring stack:
   - **vCPU:** 2
   - **RAM:** 4GB minimum, 8-12GB recommended if running ML workloads
   - **Storage:** 32GB minimum
3. Complete the wizard and power on the VM
4. Assign a static IP via your router's DHCP reservation using the VM's MAC address
5. Enable SSH in vDSM: **Control Panel → Terminal & SNMP → Enable SSH**

---

## 2. Install Docker via Container Manager

vDSM does not support standard Linux Docker install scripts. The correct method is through the Synology Package Center.

In the vDSM web interface (`http://VDSM-IP:5000`):

1. Open **Package Center**
2. Search for **Container Manager**
3. Click **Install**

Once installed, Docker is available on the command line:

```bash
docker --version
```

> **Note:** Do not attempt to install Docker using `curl -fsSL https://get.docker.com | sh` on vDSM. The script checks `/etc/os-release` which does not exist on this platform and will fail with `ERROR: Unsupported distribution ''`.

---

## 3. Create the Folder Structure

Synology vDSM stores persistent data under `/volume1/`. Create a consistent folder structure before deploying any containers:

```bash
mkdir -p /volume1/docker/{portainer-agent,uptime-kuma/data,ntopng/{config,data},compose/{uptime-kuma,ntopng}}
```

This creates:

```
/volume1/docker/
├── portainer-agent/
├── uptime-kuma/
│   └── data/
├── ntopng/
│   ├── config/
│   └── data/
└── compose/
    ├── uptime-kuma/
    └── ntopng/
```

Store all `docker-compose.yml` files under `/volume1/docker/compose/<service>/` and all persistent data under `/volume1/docker/<service>/`.

---

## 4. Deploy Portainer Agent

The Portainer Agent allows an existing Portainer instance on another host to manage containers on this vDSM remotely — no separate Portainer installation needed on the vDSM itself.

> **Important:** Do not include the `/var/lib/docker/volumes` bind mount in the agent command on vDSM. That path does not exist on Synology's Container Manager and will cause the container to fail.

```bash
docker run -d \
  --name portainer-agent \
  --restart always \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/agent:latest
```

Verify it is running:

```bash
docker ps
```

Expected output:
```
CONTAINER ID   IMAGE                    COMMAND      STATUS        PORTS
xxxxxxxxxxxx   portainer/agent:latest   "./agent"    Up X seconds  0.0.0.0:9001->9001/tcp
```

---

## 5. Connect vDSM to Existing Portainer

In your existing Portainer instance:

1. Go to **Settings → Environments → Add Environment**
2. Select **Docker Standalone**
3. Select **Agent**
4. Fill in:
   - **Name:** `vDSM` (or any descriptive name)
   - **Environment URL:** `VDSM-IP:9001` (no `https://` prefix)
5. Click **Connect**

The vDSM environment will appear in the Portainer home screen alongside your existing environments. You can now deploy stacks, view logs, and manage containers on the vDSM from the same Portainer dashboard.

---

## 6. Deploy Uptime Kuma

In Portainer → select the vDSM environment → **Stacks → Add Stack**.

Name: `uptime-kuma`

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: always
    ports:
      - "3001:3001"
    volumes:
      - /volume1/docker/uptime-kuma/data:/app/data
```

> **Note:** Use the `:2` tag rather than `:latest` to ensure you get Uptime Kuma v2. At the time of writing, `:latest` still pulls v1 on some platforms.

Access the web UI at `http://VDSM-IP:3001` and create an admin account on first launch.

### Recommended Monitor Configuration

- **Retries:** 3
- **Heartbeat Retry Interval:** 20 seconds
- **Accepted Status Codes:** `200-299`, `300-399`

For services behind Cloudflare, `403` responses indicate the tunnel is alive and Cloudflare is proxying — not that the service is down. A genuine outage produces a `5xx` or connection timeout. Design accepted codes accordingly.

### Telegram Notifications

1. Go to **Settings → Notifications → Setup Notification**
2. Select **Telegram**
3. Enter your Bot Token and Chat ID
4. Click **Test** to verify delivery
5. Apply to all monitors

---

## 7. Deploy Ntopng

Ntopng requires host networking to monitor real network interfaces and elevated capabilities for packet capture.

In Portainer → vDSM environment → **Stacks → Add Stack**.

Name: `ntopng`

```yaml
services:
  ntopng:
    image: ntop/ntopng:latest
    container_name: ntopng
    restart: always
    volumes:
      - /volume1/docker/ntopng/data:/var/lib/ntopng
      - /volume1/docker/ntopng/config:/etc/ntopng
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    user: root
    command: "--community -i eth0 -m 10.0.0.0/24"
```

Replace `10.0.0.0/24` with your actual local network CIDR.

> **Note:** `network_mode: host` is required. Without it, Ntopng only sees internal Docker bridge traffic and cannot monitor actual LAN hosts.

Before deploying, fix volume permissions:

```bash
chmod 777 /volume1/docker/ntopng/data
chmod 777 /volume1/docker/ntopng/config
```

Access the web UI at `http://VDSM-IP:3000`. Default credentials are `admin` / `admin` — change immediately on first login.

### Initial Configuration

**Interface**
Confirm the top-left dropdown shows `eth0`. If it shows `lo`, the `-i eth0` flag in the command was not applied — redeploy the stack.

**Behavioral Checks**
Navigate to **Settings → Behavioral Checks → Host** and enable:
- Dangerous Host
- ICMP Flood
- Flow Flood
- Suspicious Domain Scan
- Countries Contacts

Navigate to **Behavioral Checks → Flow** and disable:
- IEC Unexpected TypeID (industrial control protocol — not relevant for homelab use)

**Telegram Notifications**
Navigate to **Settings → Notifications** and configure a Telegram recipient with your bot token and chat ID. Test before saving.

**Timeseries Retention**
Navigate to **Settings → Preferences → Timeseries** and set retention to 30 days to avoid unbounded storage growth.

**Timezone**
Navigate to **Settings → Preferences → User Interface** and set the correct local timezone. Ntopng defaults to UTC.

---

## 8. Verify the Setup

From any machine on the local network:

| Service | URL | Expected |
|---|---|---|
| Uptime Kuma | `http://VDSM-IP:3001` | Login page |
| Ntopng | `http://VDSM-IP:3000` | Dashboard with `eth0` selected |
| Portainer Agent | `VDSM-IP:9001` | Connected in Portainer environments |

In Portainer on the primary host, confirm both environments are visible:

```
local     → Docker   unix:///var/run/docker.sock
vDSM      → Docker   tcp://VDSM-IP:9001
```

---

## Notes

- These services are intentionally kept LAN-only. Neither Uptime Kuma nor Ntopng are exposed via Cloudflare Tunnel — network monitoring tools should not be publicly accessible.
- The vDSM runs as a guest on the Synology NAS. Resource allocation (2 vCPU, 12GB RAM) leaves sufficient headroom on the host for NAS duties, rsync backup jobs, and other DSM services.
- Persistent data for all services is stored under `/volume1/docker/` on the vDSM, which is backed by the NAS volume and included in Synology's storage pool redundancy.
