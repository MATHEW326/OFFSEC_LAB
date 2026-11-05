## 1. What this lab does

This is a small, isolated cybersecurity practice lab built with Docker Desktop on Windows. It gives you an attacker machine and intentionally vulnerable targets that you can inspect and test without exposing those targets to your home network or the public internet.

The lab currently contains:

- **Kali Linux** — the attacker and testing machine.
- **OWASP Juice Shop** — a deliberately vulnerable web application.
- **Metasploitable 2** — an old Linux system with intentionally vulnerable services.

## 2. Host requirements

The setup was built on Windows using Docker Desktop with its WSL 2 backend and Linux containers.

Verified environment details:

- Docker Engine: 29.3.1
- Docker Compose: 5.1.1
- WSL: 2.6.3.0
- Docker context: `desktop-linux`
- Docker reports 8 CPUs and approximately 7.38 GiB of memory.

The working directory is:

```
C:\docker-cyber-lab
├── docker-compose.yml
├── configs/
├── evidence/
├── kali/
├── monitoring/
└── targets/
```

## 3. Network design

The lab uses three separate Docker bridge networks. Each is configured as `internal: true`, which prevents containers on those networks from using them to reach external networks.

| Network | Subnet | Purpose |
| --- | --- | --- |
| `cyberlab_attacker` | `172.30.10.0/24` | Attacker-side network |
| `cyberlab_targets` | `172.30.20.0/24` | Vulnerable target network |
| `cyberlab_monitoring` | `172.30.30.0/24` | Reserved for monitoring |

Current lab addresses:

| Container | Address | Network |
| --- | --- | --- |
| Kali | `172.30.10.10` | Attacker |
| Kali | `172.30.20.20` | Targets |
| Juice Shop | `172.30.20.10` | Targets |
| Metasploitable 2 | `172.30.20.30` | Targets |

Kali is connected to both the attacker and target networks so it can reach the targets. Its IP forwarding is explicitly disabled with `net.ipv4.ip_forward: "0"`; it should not act as a router between the networks.

**Important:** Vulnerable target services are not published to Windows host ports. Do not add Docker `ports:` mappings to these targets for convenience.

## 4. Build the Kali attacker image

The Kali image is based on `kalilinux/kali-rolling`. Its Dockerfile installs common reconnaissance and network tools:

```docker
FROM kalilinux/kali-rolling

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    nmap \
    libcap2-bin \
    iproute2 \
    iputils-ping \
    dnsutils \
    curl \
    tcpdump \
    netcat-openbsd \
    traceroute \
    whois \
    nikto \
    whatweb \
    gobuster \
    && setcap -r /usr/lib/nmap/nmap \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

CMD ["sleep", "infinity"]
```

The Nmap file capabilities are removed because the container drops all Linux capabilities. This avoids an execution error caused by file capabilities that are not available in the container. Use unprivileged TCP connect scans, for example `nmap --unprivileged -sT`.

The Kali service is configured with a dual network attachment, a long-running command, resource limits, and restrictive security settings. The essential shape is:

```yaml
kali:
  build:
    context: ./kali
    dockerfile: Dockerfile
  container_name: cyberlab-kali
  hostname: kali-attacker
  command: sleep infinity
  restart: unless-stopped
  networks:
    attacker_net:
      ipv4_address: 172.30.10.10
    target_net:
      ipv4_address: 172.30.20.20
  cap_drop:
    - ALL
  security_opt:
    - no-new-privileges:true
  sysctls:
    net.ipv4.ip_forward: "0"
  mem_limit: 2g
  cpus: 2.0
```

Build and start Kali from PowerShell in the project directory:

```powershell
docker compose build kali
docker compose up -d kali
```

## 5. Add OWASP Juice Shop

Juice Shop runs as a target at `172.30.20.10`, listening on container port 3000. It has no host port mapping.

```yaml
juice-shop:
  image: bkimminich/juice-shop:latest
  container_name: cyberlab-juice-shop
  hostname: juice-shop
  restart: unless-stopped
  networks:
    target_net:
      ipv4_address: 172.30.20.10
  mem_limit: 1g
  cpus: 1.0
```

Start it:

```powershell
docker compose up -d juice-shop
```

The container logs confirmed that the server was listening on port 3000. From Kali, connectivity was verified with:

```bash
ping -c 4 172.30.20.10
curl -I http://172.30.20.10:3000
nmap --unprivileged -sT -Pn -p 3000 172.30.20.10
```

Observed results: ping succeeded, HTTP returned `200 OK`, and Nmap reported `3000/tcp open`.

## 6. Add Metasploitable 2

The lab uses the third-party image `tleemcjr/metasploitable2:latest` from the referenced setup guide. It is an old image (created in 2018) and is not an official Rapid7 image. Treat it as untrusted legacy software and keep it isolated.

Its Compose service uses:

- Target address: `172.30.20.30`
- No published host ports
- `cap_drop: ALL`
- Only `NET_BIND_SERVICE`, `SETUID`, and `SETGID` added
- `no-new-privileges:true`
- IP forwarding disabled
- 1 GiB memory limit and 1 CPU

The service's security settings were intentionally kept restrictive; the container was not made privileged.

### A startup issue and the workaround

The image's original command was:

```bash
sh -c "/bin/services.sh && bash"
```

The container repeatedly exited and restarted because the non-interactive shell could exit after the service script completed. Docker's `restart: unless-stopped` policy then restarted it repeatedly.

The Compose service was changed to keep a foreground process alive after launching the legacy startup script:

```yaml
command: ["sh", "-c", "/bin/services.sh & exec tail -f /dev/null"]
```

After validating the Compose file and recreating the service, the container showed `Status=running` and `RestartCount=0`.

**Known limitation:** Not every legacy service starts successfully with the restrictive capability set. MySQL currently reports permission errors when reading `/var/run/mysqld/mysqld.pid`. Some other services, such as Tomcat and UnrealIRCd, also reported startup errors earlier. Do not assume those services are available just because the container is running. Inspect the process list and listening ports before testing a particular service.

## 7. Verify the lab

Run Docker administration commands from **Windows PowerShell**, in `C:\docker-cyber-lab`:

```powershell
# Check Compose configuration
docker compose config --quiet

# Start or update the lab services
docker compose up -d

# Show running containers
docker ps

# Check the Metasploitable restart count
docker inspect cyberlab-metasploitable2 --format "Status={{.State.Status}} RestartCount={{.RestartCount}}"

# Confirm the target networks are internal
docker network inspect cyberlab_attacker cyberlab_targets --format '{{.Name}} Internal={{.Internal}}'

# Confirm Kali IP forwarding is disabled
docker exec cyberlab-kali sh -c "cat /proc/sys/net/ipv4/ip_forward"
```

Expected isolation checks:

- Both attacker and target networks report `Internal=true`.
- Kali reports IP forwarding value `0`.
- `docker port cyberlab-juice-shop` and `docker port cyberlab-metasploitable2` return no host port mappings.

From Kali, verify target reachability:

```powershell
docker exec cyberlab-kali ping -c 4 172.30.20.10
docker exec cyberlab-kali curl -I http://172.30.20.10:3000
docker exec cyberlab-kali nmap --unprivileged -sT -Pn -p 3000 172.30.20.10
```

Check Metasploitable's currently exposed TCP services:

```powershell
docker exec cyberlab-kali nmap --unprivileged -sT -Pn -p- --open 172.30.20.30
```

This full TCP-port scan can take a little while. Only test services that the scan actually reports as open.

## 8. Open interactive terminals

From Windows PowerShell:

```powershell
# Open Kali
docker exec -it cyberlab-kali bash

# Open Metasploitable 2 (use another PowerShell window)
docker exec -it cyberlab-metasploitable2 bash
```

The Metasploitable container runs as root in this image. Be careful: it is intentionally vulnerable.

## 9. Transfer a file from Kali to Metasploitable

A simple option is Netcat, which is installed in the Kali image. This example transfers a small text file over the isolated target network.

In Kali:

```bash
mkdir -p /tmp/transfer
echo "Hello from Kali Cyber Lab" > /tmp/transfer/test.txt
nc -lvnp 9000 < /tmp/transfer/test.txt
```

Keep the listener open. In another PowerShell window, enter Metasploitable:

```powershell
docker exec -it cyberlab-metasploitable2 bash
```

Then receive and verify the file:

```bash
nc -nv 172.30.20.20 9000 > /tmp/test.txt
cat /tmp/test.txt
```

Expected content:

```
Hello from Kali Cyber Lab
```

Press Ctrl+C in Kali to stop the listener. If Netcat is missing on the receiving image, use another transfer method rather than assuming it is installed.

## 10. Troubleshooting notes

| Symptom | What it means / what to check |
| --- | --- |
| `docker: command not found` inside Kali | Run Docker and Compose commands from Windows PowerShell, not inside the Kali container. |
| `Connection refused` | The target container may be reachable while the particular service is not listening. Check with Nmap and inspect processes/logs. |
| Metasploitable restart count increases | Inspect its command, exit status, and logs. The foreground `tail -f /dev/null` workaround prevents the startup shell from exiting after launching the service script. |
| MySQL PID permission denied | MySQL is not confirmed healthy. The container's dropped capabilities restrict root's ability to bypass file permissions. Diagnose before changing security settings. |
| `/dev/console: No such file or directory` | Legacy init scripts expect a fuller init environment than a basic container provides. This warning alone does not prove that every service failed. |

## 11. Safety rules

1. Use this lab only for systems you own or have explicit permission to test.
2. Keep vulnerable target containers on the internal Docker networks.
3. Do not publish vulnerable target ports to the Windows host or the internet.
4. Do not enable `privileged: true` as a quick fix.
5. Keep Kali IP forwarding disabled unless you deliberately redesign and understand the network.
6. Be cautious with the old third-party Metasploitable image; do not use it as a general-purpose machine.
7. Keep evidence and notes in the `evidence/`  directory.

## 12. Current status and next work

**Verified working**

- Docker Desktop, WSL 2, and Compose prerequisites.
- Project directory and Compose network definitions.
- Kali container and reconnaissance tools.
- Kali connectivity to Juice Shop.
- Juice Shop HTTP response and port 3000.
- Internal network checks and disabled Kali IP forwarding.
- Metasploitable container remains running after the startup-command workaround.
- Metasploitable target address is `172.30.20.30`.

**Known incomplete**

- Metasploitable MySQL startup.
- Some other legacy services reported permission/configuration errors.
- Monitoring network is defined but monitoring tools/services have not yet been added.
- DVWA has not yet been added.

Keep this document updated as new targets, monitoring tools, and verified service configurations are added.