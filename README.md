# CGNAT Bypass with Oracle Cloud + WireGuard

Zero-cost solution to expose services behind CGNAT using an Oracle Cloud Always Free VM as a WireGuard relay. Eliminates the need for a dedicated static IP (~US$40–100/month).

Tested in a production MSP environment serving multiple clients in Brazil, where ISP CGNAT is nearly universal.

---

## The Problem

Most Brazilian ISPs (and many worldwide) put residential and small-business customers behind CGNAT (Carrier-Grade NAT). You share a single public IP with hundreds of other customers, making it impossible to receive inbound connections — no port forwarding, no dynamic DNS workaround.

**Before:**
```
Internet → ISP CGNAT → [no path to your server]
```

**After:**
```
Internet → Oracle Cloud VM (public IP) → WireGuard tunnel → Your server
```

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Internet                              │
└────────────────────────┬─────────────────────────────────────┘
                         │ port 80/443/any
                         ▼
┌──────────────────────────────────────────────────────────────┐
│              Oracle Cloud Always Free VM                     │
│              (ARM Ampere A1 · public IP)                     │
│                                                              │
│  Nginx stream  ──────────────► wg0: 10.0.0.1/24             │
│  (TCP passthrough)             WireGuard server              │
└───────────────────────────────┬──────────────────────────────┘
                                │ WireGuard UDP 51820
                                │ (encrypted tunnel)
                                ▼
┌──────────────────────────────────────────────────────────────┐
│              Local Server (behind CGNAT)                     │
│              wg0: 10.0.0.2/24                                │
│              WireGuard client                                │
│                                                              │
│  Caddy / Nginx / any service ◄── receives proxied traffic   │
└──────────────────────────────────────────────────────────────┘
```

**Traffic flow:** Client → Oracle Cloud public IP → Nginx forwards to WireGuard peer IP → Local server handles request → Response returns the same path.

---

## Requirements

### Oracle Cloud (free tier)
- **Account:** Oracle Cloud Always Free ([cloud.oracle.com](https://cloud.oracle.com))
- **VM:** 1× Ampere A1 Compute (ARM, 1 OCPU, 6 GB RAM — always free)
- **OS:** Ubuntu 22.04 or Oracle Linux 9
- **Outbound:** 10 TB/month included free
- **Security List:** open UDP 51820 (WireGuard) + TCP 80/443 (or your ports) inbound

### Local server
- Linux (tested on Ubuntu/Debian and Docker hosts)
- WireGuard (`apt install wireguard` or `apt install wireguard-tools`)
- Your actual services (Caddy, Nginx, Docker, etc.) listening on the WireGuard interface or `0.0.0.0`

---

## Setup

### 1. Install WireGuard on both machines

```bash
apt update && apt install -y wireguard
```

### 2. Generate key pairs (run on each machine separately)

```bash
wg genkey | tee privatekey | wg pubkey > publickey
cat privatekey publickey
```

### 3. Configure WireGuard on Oracle Cloud VM

Copy [`wireguard/oracle-wg0.conf.example`](wireguard/oracle-wg0.conf.example) to `/etc/wireguard/wg0.conf` and fill in the keys.

Enable IP forwarding:

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

Start and enable:

```bash
systemctl enable --now wg-quick@wg0
```

### 4. Configure WireGuard on local server

Copy [`wireguard/local-wg0.conf.example`](wireguard/local-wg0.conf.example) to `/etc/wireguard/wg0.conf` and fill in the keys and Oracle Cloud public IP.

```bash
systemctl enable --now wg-quick@wg0
```

Verify the tunnel is up:

```bash
wg show
ping 10.0.0.1   # should reach Oracle Cloud VM
```

### 5. Configure Nginx on Oracle Cloud VM

Install Nginx with stream module:

```bash
apt install -y nginx-full
```

Copy [`nginx/stream.conf.example`](nginx/stream.conf.example) to `/etc/nginx/stream.d/cgnat-bypass.conf`.

Enable and test:

```bash
nginx -t && systemctl reload nginx
```

### 6. Configure Oracle Cloud Security List

In the Oracle Cloud Console, add ingress rules for:
- UDP 51820 (WireGuard handshake)
- TCP 80 (HTTP)
- TCP 443 (HTTPS)
- Any other ports your services need

Also open the same ports in the VM's firewall:

```bash
iptables -I INPUT -p udp --dport 51820 -j ACCEPT
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -j ACCEPT
# persist with iptables-persistent or nftables
```

---

## Configuration Files

| File | Purpose |
|---|---|
| [`wireguard/oracle-wg0.conf.example`](wireguard/oracle-wg0.conf.example) | WireGuard server config for Oracle Cloud VM |
| [`wireguard/local-wg0.conf.example`](wireguard/local-wg0.conf.example) | WireGuard client config for local server |
| [`nginx/stream.conf.example`](nginx/stream.conf.example) | Nginx TCP stream proxy config for Oracle Cloud VM |

---

## Cost Breakdown

| Resource | Oracle Always Free | Alternative (paid IP) |
|---|---|---|
| VM (ARM A1) | $0/month | — |
| Public IP | $0/month | — |
| Outbound traffic | 10 TB free | — |
| Static IP (ISP) | not needed | US$40–100/month |
| **Total** | **$0/month** | **US$40–100/month** |

Oracle Cloud Always Free resources do not expire as long as the account remains active.

---

## Production Notes

- This setup runs in production at [Systrix Tecnologia](https://systrix.com.br/) serving multiple clients with Docker-based services (Caddy, Zabbix, Nextcloud, GLPI, NetBox)
- The WireGuard tunnel is monitored via a [custom Zabbix template](https://github.com/lgmferras/zabbix-templates/tree/main/templates/wireguard-tunnel) that tracks interface state, peers, and handshake age
- `PersistentKeepalive = 25` keeps the tunnel alive through double NAT and idle timeouts
- Caddy on the local server handles TLS termination and Let's Encrypt — no TLS configuration needed on the Oracle Cloud VM

---

## Author

**Luis Gustavo Ferras** — Infrastructure Specialist at [Systrix Tecnologia](https://systrix.com.br/)

[![LinkedIn](https://img.shields.io/badge/LinkedIn-lgmferras-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/lgmferras)
[![GitHub](https://img.shields.io/badge/GitHub-lgmferras-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/lgmferras)
[![Zabbix Templates](https://img.shields.io/badge/Zabbix_Templates-D40000?style=flat-square&logoColor=white)](https://github.com/lgmferras/zabbix-templates)

20+ years in enterprise IT infrastructure — Docker · Linux · Zabbix · Oracle Cloud · LoRaWAN
