# Troubleshooting Log

Real issues encountered during homelab operation and how they were diagnosed and resolved. Entries are added as problems occur.

---

## 2026-05-13

### High CPU load and swap exhaustion on M710q

**Symptoms**
- Load average sustained above 5.0 on a 4-core machine
- Swap usage at 3.95G / 4.00G (nearly exhausted)
- System sluggish, containers responding slowly

**Initial assumption**
With ~80 containers running, the assumption was that a container workload was responsible. Ran `docker stats --no-stream` to identify the top memory consumers.

**Actual cause**
Netdata was running as a system service (not a Docker container) and was continuously polling every subsystem on the machine. It did not appear in `docker stats` output, which is why it was initially overlooked. Once identified in `htop` with multiple processes consuming 6-7% CPU each, the impact became clear.

**Resolution**
```bash
sudo systemctl stop netdata
sudo systemctl disable netdata
```
Load average dropped from 5.49 to 0.88 immediately. Netdata was replaced by Ntopng on the vDSM for network-level monitoring and Gatus for service health checks — both more targeted tools with significantly lower overhead.

**Lesson**
When diagnosing resource issues on a Docker host, always check system services alongside containers. `docker stats` only shows container processes — `htop` shows the full picture.

---

### Immich machine learning container causing swap pressure

**Symptoms**
- Swap usage remained high even after stopping Netdata
- `immich-machine-learning` visible in `htop` with high resident memory

**Cause**
Immich's ML container loads computer vision models into memory on startup and keeps them resident continuously, even when not actively processing photos. On a host already under memory pressure this contributed significantly to swap usage.

**Resolution**
Commented out the `immich-machine-learning` service block in the compose file and redeployed. Initial attempt using `docker compose up -d` after editing the file did not remove the existing container — it continued running as an orphan.

Correct sequence:
```bash
cd /opt/docker/immich
sudo docker compose down --remove-orphans
sudo docker compose up -d
```

The `--remove-orphans` flag is required to remove containers that existed in a previous compose definition but are no longer present in the current file.

**Lesson**
`docker compose up -d` creates and updates — it does not remove. Always use `--remove-orphans` when removing services from a compose file.

---

### Docker installation fails on vDSM

**Symptoms**
```
ERROR: Unsupported distribution ''
```
Running the standard Docker install script on a fresh vDSM instance failed immediately. Follow-up commands also failed:
```
sudo systemctl enable docker
Failed to execute operation: No such file or directory
```

**Cause**
vDSM uses a stripped Synology kernel (`synology_kvmx64_virtualdsm`) with no `/etc/os-release` file and no `systemd`. The Docker install convenience script checks the OS distribution via `/etc/os-release` and cannot proceed without it.

**Diagnosis**
```bash
uname -a
# Linux ScavNode-02 4.4.302+ synology_kvmx64_virtualdsm
cat /etc/os-release
# cat: /etc/os-release: No such file or directory
which apt || which yum || which synopkg
# /usr/syno/bin/synopkg
```

**Resolution**
Installed Docker through the Synology Package Center GUI by searching for and installing **Container Manager**. This provides a properly integrated Docker environment for the vDSM platform. After installation, `docker` is available on the command line as expected.

**Lesson**
vDSM is not a standard Linux distribution. Standard Linux tooling and install scripts cannot be assumed to work. Always check the platform first before running installation commands.

---

### Portainer agent bind mount fails on vDSM

**Symptoms**
```
docker: Error response from daemon: Bind mount failed: '/var/lib/docker/volumes' does not exist.
```
Running the standard Portainer agent command (including the volumes bind mount) failed on vDSM.

**Cause**
Synology's Container Manager does not use the standard Docker volumes path `/var/lib/docker/volumes`. The directory does not exist on this platform.

**Resolution**
Removed the volumes bind mount from the agent command. The socket mount is sufficient for container management:
```bash
docker run -d \
  --name portainer-agent \
  --restart always \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/agent:latest
```

**Lesson**
Portainer documentation assumes a standard Docker installation. On non-standard platforms, test each mount individually and remove those that reference paths that don't exist.

---

### Ntopng volume permission denied on vDSM

**Symptoms**
```
ERROR: Unable to write on /var/lib/ntopng as 'ntopng' [/var/lib/ntopng/.test]: Permission denied.
```
Ntopng started but could not write to the mounted data volume.

**Cause**
The host directory `/volume1/docker/ntopng/data` was created with root ownership. The Ntopng process runs as the `ntopng` user inside the container and does not have write permission to the mounted path.

**Resolution**
```bash
chmod 777 /volume1/docker/ntopng/data
chmod 777 /volume1/docker/ntopng/config
```
Added `user: root` and the required capabilities to the compose file:
```yaml
cap_add:
  - NET_ADMIN
  - NET_RAW
user: root
```

**Lesson**
When mounting host directories into containers, check that the container's runtime user has write access to the host path. On Synology volumes, permissions behave differently than on standard Linux filesystems.

---

### Ntopng monitoring loopback interface instead of LAN

**Symptoms**
Ntopng dashboard showed only `127.0.0.0/8` network with no LAN hosts visible.

**Cause**
Without specifying an interface, Ntopng defaults to the first available interface — which on the vDSM was `lo` (loopback) rather than `eth0`.

**Resolution**
Updated the compose command to explicitly specify the interface and local network:
```yaml
command: "--community -i eth0 -m 192.168.1.0/24"
```

**Lesson**
Always explicitly specify the monitoring interface when deploying Ntopng. The default interface selection is not predictable across environments.

---

### Uptime Kuma false positives — services showing as down

**Symptoms**
Multiple monitors showing as down despite services being accessible in a browser. Cloudflare-fronted services returning 403 status codes being flagged as down.

**Cause 1 — Retries set to 0**
With retries set to 0, a single failed check immediately marks the service as down. One timeout or transient Cloudflare hiccup triggers an alert.

**Resolution:** Set retries to 3 with a 20 second retry interval.

**Cause 2 — Cloudflare returning 403 to monitoring requests**
Cloudflare's bot protection flags Uptime Kuma's HTTP requests as automated traffic and returns 403 Forbidden. The service itself is up and serving real users normally.

**Resolution:** Added `300-399` to accepted status codes to handle redirects. Left `403` as acceptable since Cloudflare returning 403 confirms the tunnel is alive and the service is reachable — a genuine outage produces a 5xx or connection timeout instead, which still triggers an alert.

**Lesson**
When monitoring services behind Cloudflare, 403 responses indicate the edge is up and proxying — not that the service is down. Design accepted status codes around what a real outage actually looks like, not what a healthy response ideally looks like.
