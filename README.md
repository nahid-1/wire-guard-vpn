# wire-guard-vpn

## Architecture Overview

```
Local Client (WireGuard app)
        │
        │  UDP 51820 (encrypted tunnel)
        ▼
   vRouter (Public IP: 172.16.30.20)
        │
        ▼
 WireGuard VM
 ├── ens3: 172.16.30.43/24  ──► VMs on 172.16.30.0/24
 └── ens8: 10.0.0.3/16      ──► VMs on 10.0.0.0/16

WireGuard client pool: 10.8.0.0/24
One container. One agent. Both networks reachable.
```

---

## Prerequisites

- Ubuntu 24.04 VM on PICO cloud
- Two interfaces: ens3 (172.16.30.0/24) and ens8 (10.0.0.0/16)
- Public IP associated via vRouter to ens3
- Root access

---

## Step 1 — System Update

```bash
apt update && apt upgrade -y
reboot
```

---

## Step 2 — Install Docker (official method)

> Do NOT use `snap install docker` or `apt install docker.io` — they have
> known issues with iptables and kernel modules required by WireGuard.

```bash
curl -sSL https://get.docker.com | sh
systemctl enable docker
systemctl start docker
docker --version
docker compose version
```

---

## Step 3 — Enable IP Forwarding (persistent)

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Verify
sysctl net.ipv4.ip_forward
# Expected output: net.ipv4.ip_forward = 1
```

---

## Step 4 — Verify Netplan Configuration

```bash
cat /etc/netplan/50-cloud-init.yaml
```

Expected content (no static routes needed for ens8 — it is a directly connected network):

```yaml
network:
  version: 2
  ethernets:
    ens3:
      match:
        macaddress: "fa:16:3e:22:dc:c9"
      dhcp4: true
      set-name: ens3
      mtu: 8942
    ens8:
      dhcp4: false
      addresses:
        - 10.0.0.3/16
```

Apply if changed:

```bash
netplan apply
ip route show
```

Expected routes:

```
default via 172.16.30.1 dev ens3
10.0.0.0/16 dev ens8 proto kernel scope link src 10.0.0.3
172.16.30.0/24 dev ens3 proto kernel scope link src 172.16.30.43
```

---

## Step 5 — Generate Password Hash

> The `$` signs in the bcrypt hash must be escaped as `$$` in docker-compose.yml.
> The heredoc method below handles this automatically.

```bash
docker run --rm -it ghcr.io/wg-easy/wg-easy:14 \
  node -e "console.log(require('bcryptjs').hashSync('YourPasswordHere', 10))"
```

Copy the output. Example output:
```
$2a$10$s68zTr.ey4i0434/pjgarO7fnP1yQz..ekiob34bmtUH9ZG4OMBZW
```

Replace every `$` with `$$` when putting it in the compose file:
```
$$2a$$10$$s68zTr.ey4i0434/pjgarO7fnP1yQz..ekiob34bmtUH9ZG4OMBZW
```

---

## Step 6 — Create Docker Compose File

```bash
mkdir -p /etc/docker/containers/wg-easy

cat > /etc/docker/containers/wg-easy/docker-compose.yml << 'EOF'
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:14
    container_name: wg-easy
    environment:
      - LANG=en
      - WG_HOST=<public IP>
      - PASSWORD_HASH=$$2a$$10$$s68zTr.ey4i0434/pjgarO7fnP1yQz..ekiob34bmtUH9ZG4OMBZW
      - WG_PORT=51820
      - WG_DEFAULT_ADDRESS=10.8.0.x
      - WG_DEFAULT_DNS=8.8.8.8
      - WG_ALLOWED_IPS=172.16.30.0/24,10.0.0.0/16
      - WG_PERSISTENT_KEEPALIVE=25
      - PORT=51821
    volumes:
      - /etc/docker/containers/wg-easy/data:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
EOF
```

> Replace the PASSWORD_HASH value with your own escaped hash from Step 5.

---

## Step 7 — Start the Container

```bash
cd /etc/docker/containers/wg-easy
docker compose up -d

# Verify container is running
docker ps

# Check logs for errors
docker logs wg-easy --tail 30

# Verify password hash is loaded correctly (should show full $2a$10$... hash)
docker exec -it wg-easy env | grep PASSWORD_HASH
```

---

## Step 8 — Configure iptables MASQUERADE

> MASQUERADE rewrites the source IP of VPN client packets (10.8.0.x) to the
> WireGuard VM's own interface IP before forwarding. This is required because
> destination VMs have no route back to 10.8.0.0/24 — they only know how to
> reply to IPs on their own subnet.

```bash
# Clients accessing 172.16.30.0/24 network (via ens3)
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens3 -j MASQUERADE

# Clients accessing 10.0.0.0/16 network (via ens8)
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens8 -j MASQUERADE

# Verify rules are active
iptables -t nat -L POSTROUTING -n -v
```

---

## Step 9 — Persist iptables Rules (survive reboot)

```bash
apt install iptables-persistent -y
netfilter-persistent save

# Verify saved
cat /etc/iptables/rules.v4
```

---

## Step 10 — Open Ports on vRouter / Security Group

On your PICO cloud vRouter or security group, allow inbound:

| Port  | Protocol | Purpose              |
|-------|----------|----------------------|
| 51820 | UDP      | WireGuard VPN tunnel |
| 51821 | TCP      | wg-easy Web UI       |

---

## Step 11 — Add VPN Clients

1. Open browser: `http://172.16.30.20:51821`
2. Login with your plain text password (e.g. `Pico@2026##`)
3. Click **+ New Client** → give it a name
4. Download the `.conf` file
5. Import into WireGuard app on your local computer
6. Connect — both `172.16.30.0/24` and `10.0.0.0/16` are now reachable

---

## Full Verification Checklist

```bash
# 1. Container running
docker ps

# 2. IP forwarding active
sysctl net.ipv4.ip_forward

# 3. Both MASQUERADE rules present
iptables -t nat -L POSTROUTING -n -v

# 4. Routing table correct
ip route show

# 5. WireGuard interface created by container
ip -br a | grep wg

# 6. iptables rules will survive reboot
cat /etc/iptables/rules.v4
```

---

## How to Add a New Network Interface (e.g. 192.168.0.0/24)

Follow these steps whenever you attach a new interface to the VM and want
VPN clients to reach it.

### Scenario
You attach a new interface `ens9` with IP `192.168.0.5/24` and want VPN
clients to access VMs on `192.168.0.0/24`.

---

### Step A — Verify the new interface is up

```bash
ip -br a
# Should show: ens9  UP  192.168.0.5/24
```

---

### Step B — Add the new subnet to WG_ALLOWED_IPS

```bash
cd /etc/docker/containers/wg-easy
docker compose down
```

Edit the compose file and update `WG_ALLOWED_IPS`:

```yaml
- WG_ALLOWED_IPS=172.16.30.0/24,10.0.0.0/16,192.168.0.0/24
```

Start the container again:

```bash
docker compose up -d
```

> This change only affects NEW client configs generated after the restart.
> Existing clients must re-download their .conf from the web UI to get the
> updated AllowedIPs. Or you can manually edit their local .conf file and
> add 192.168.0.0/24 to the AllowedIPs line.

---

### Step C — Add MASQUERADE rule for the new interface

```bash
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens9 -j MASQUERADE

# Verify
iptables -t nat -L POSTROUTING -n -v
```

---

### Step D — Save the new iptables rule

```bash
netfilter-persistent save
```

---

### Step E — Update netplan if ens9 is not configured (static IP case)

```bash
vim /etc/netplan/50-cloud-init.yaml
```

Add under `ethernets:`:

```yaml
    ens9:
      dhcp4: false
      addresses:
        - 192.168.0.5/24
```

Apply:

```bash
netplan apply
ip route show
# Should show: 192.168.0.0/24 dev ens9 proto kernel scope link src 192.168.0.5
```

---

### Step F — Test from VPN client

After reconnecting the VPN client with the updated config:

```bash
ping 192.168.0.X   # any VM on the new subnet
```

---

## Summary Table — Per-Interface Requirements

| Action                        | ens3 | ens8 | New interface |
|-------------------------------|------|------|---------------|
| IP configured in netplan      | DHCP | Done | Required      |
| Route in kernel routing table | Auto | Auto | Auto (once netplan applied) |
| MASQUERADE iptables rule      | Done | Done | Add new rule  |
| Subnet in WG_ALLOWED_IPS      | Done | Done | Add and restart container |
| Save iptables                 | Done | Done | netfilter-persistent save |
| Client re-downloads .conf     | N/A  | N/A  | Required      |

---

## Quick Reference — Key Values

| Item                  | Value                    |
|-----------------------|--------------------------|
| Public IP             | 172.16.30.20             |
| ens3 IP               | 172.16.30.43/24          |
| ens8 IP               | 10.0.0.3/16              |
| WG client pool        | 10.8.0.0/24              |
| WG tunnel port        | UDP 51820                |
| Web UI port           | TCP 51821                |
| Web UI URL            | http://172.16.30.20:51821|
| Compose file location | /etc/docker/containers/wg-easy/docker-compose.yml |
| WG data/keys location | /etc/docker/containers/wg-easy/data/              |
