# bifrǫst — the Rainbow Bridge

Ansible playbook that sets up a **two-hop censorship-resistant VPN** using two VPS:
a foreign exit node and a Russian relay.
All protocols deploy in two commands; ready-to-import client configs appear locally when done.

![image](https://github.com/user-attachments/assets/3b8d8640-9665-43ec-8d9f-5965d3244c02)

---

## Architecture

```
Client
  └─► RU relay VPS  (port 443/TCP, VLESS+REALITY, Russian SNI)
        ├─► [UUID-xhttp] VLESS + XHTTP + REALITY ──► Foreign VPS (TCP 8443)
        ├─► [UUID-hy2]   Hysteria2 QUIC           ──► Foreign VPS (UDP 443)
        └─► [UUID-awg]   AmneziaWG tunnel         ──► Foreign VPS (UDP 51820)
                                                          └─► Internet
```

The client connects to the **Russian relay** on TCP 443 using VLESS+REALITY with a
Russian SNI (e.g. `gosuslugi.ru`). The relay forwards traffic to the **foreign VPS** via
one of three protocols — selected by UUID. All three relay links look identical to
the outside: same server, same port, same SNI.

### Why two hops?

A single foreign VPS is increasingly fingerprinted by Russian mobile operators who
detect traffic patterns where *all* foreign connections flow through a single IP
from a cloud-provider range. The relay breaks that pattern: from the client's perspective
the connection goes to a Russian IP, and server-to-server traffic is far less scrutinised.

---

## Protocols

### Foreign VPS (exit node)

| Protocol | Transport | Port | Notes |
|---|---|---|---|
| **VLESS + XHTTP + Reality** | TCP | 8443 | HTTP stream over Reality with self-steal cert. |
| **Hysteria2** | UDP | 443 | QUIC with Salamander obfuscation. Immune to TCP throttling. |
| **AmneziaWG** | UDP | 51820 | WireGuard with junk-packet + header obfuscation. |

### Relay VPS (entry node)

| Protocol | Transport | Port | Notes |
|---|---|---|---|
| **VLESS + REALITY** | TCP | 443 | Client-facing. Three UUIDs → three exit protocols. |

### Self-steal camouflage (both VPS)

Both servers run a local nginx that serves a realistic login page. Xray REALITY steals
the TLS cert from it — probers hitting TCP 443 see a genuine HTTPS site, not a VPN endpoint.

**Foreign VPS** — nginx has a real Let's Encrypt cert for the foreign domain:

```
443/TCP   → nginx → login page    (TSPU sees this)
443/UDP   → Hysteria2             (UDP, no conflict with TCP)
4443/TCP  → nginx 127.0.0.1      (Xray REALITY cert-steal source)
8443/TCP  → Xray VLESS+XHTTP+Reality
51820/UDP → AmneziaWG
```

**Relay VPS** — nginx has a real Let's Encrypt cert for your Russian domain (`relay_domain`).
Probers see a valid, trusted HTTPS site — indistinguishable from a real Russian web service:

```
80/TCP    → nginx → certbot renewal + HTTP→HTTPS redirect
443/TCP   → Xray REALITY:
              valid REALITY client  → forward to foreign VPS
              prober / unknown TLS → forward to 127.0.0.1:4443 → nginx login page
4443/TCP  → nginx 127.0.0.1  (real Let's Encrypt cert, loopback — Xray cert-steal source)
```

---

## Requirements

* **Two VPS** running Ubuntu 22.04 LTS with SSH root access
  * Foreign VPS — any overseas provider (Hetzner, OVH, etc.)
  * Relay VPS — a Russian provider (Selectel, TimeWeb, etc.) or any cheap VPS reachable from Russia
* **Two domains** with DNS A records (both are used for Let's Encrypt):
  * `foreign_domain` → foreign VPS IP (for nginx, Xray REALITY SNI, TLS cert)
  * `relay_domain` → relay VPS IP (for nginx, Xray REALITY SNI, TLS cert)
* Locally: **Python 3** + **pipx**
* Both VPS must be simultaneously reachable when running `playbook-relay.yaml` (it SSHes into the foreign VPS to fetch credentials and register the AWG peer)

---

## Quick start

### 1. Clone the repo

```bash
git clone https://github.com/h4zzkR/bifrost
cd bifrost
```

### 2. Install Ansible

```bash
sudo apt install pipx
pipx ensurepath && source ~/.zshrc
pipx install ansible-core --include-deps
pipx inject ansible-core passlib
ansible-galaxy collection install ansible.posix
```

### 3. Generate an SSH key (if you don't have one)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_vps -N ""
```

### 4. Create and encrypt secrets.yaml

```bash
cp secrets.yaml.example secrets.yaml
# Fill in the values — see the file for comments on each field
ansible-vault encrypt secrets.yaml
```

Minimum required fields:

```yaml
# Foreign VPS
foreign-vps:
  ansible_host: "1.2.3.4"
  ansible_ssh_pass: "ROOT_PASSWORD"

# Relay VPS
relay-vps:
  ansible_host: "5.6.7.8"
  ansible_ssh_pass: "ROOT_PASSWORD"

# Shared config
foreign_domain: "yourdomain.com"   # DNS A → foreign VPS IP
relay_domain:   "relay.example.ru" # DNS A → relay VPS IP
certbot_email:  "you@example.com"
ssh_public_key_path: "~/.ssh/id_vps.pub"
```

### 5. Deploy — step 1: foreign VPS

```bash
ansible-playbook -i secrets.yaml playbook-foreign.yaml --ask-vault-pass \
    -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
```

Output files saved locally:

| File | Contents |
|---|---|
| `xray_client_connect_link.txt` | VLESS+XHTTP URI (direct, for reference) |
| `awg_client.conf` | AmneziaWG config (direct, for reference) |
| `hysteria2_client_connect_link.txt` | Hysteria2 URI (direct, for reference) |

### 6. Deploy — step 2: relay VPS

Both VPS must be reachable at the same time. The relay playbook SSHes into the foreign
VPS to fetch Xray/Hysteria2/AWG credentials and to register the relay as an AWG peer.

```bash
ansible-playbook -i secrets.yaml playbook-relay.yaml --ask-vault-pass \
    -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
```

Output file saved locally:

| File | Contents |
|---|---|
| `relay_client_connect_link.txt` | Three VLESS URIs — one per exit protocol |

---

## Client setup

All three relay links use **the same** server address, port, and SNI.
The exit protocol is selected by UUID — just pick the link you want.

### relay_client_connect_link.txt

```
vless://UUID-XHTTP@RELAY_IP:443?...#Bifrost-Relay-XHTTP
vless://UUID-HY2  @RELAY_IP:443?...#Bifrost-Relay-Hysteria2
vless://UUID-AWG  @RELAY_IP:443?...#Bifrost-Relay-AWG
```

Supported clients: **Hiddify**, **v2rayNG**, **v2rayN**, **Nekoray**

Import via **Add from clipboard** or **Add from file**.

> **Switching exit protocol**: if one link stops working, switch to another —
> the relay server continues to run all three exits simultaneously.

### AmneziaWG (direct, no relay)

If you want to connect directly to the foreign VPS bypassing the relay:

Clients: **[AmneziaVPN](https://amnezia.org/)** — Windows / macOS / iOS / Android

Open AmneziaVPN → **Add VPN** → **Import config from file** → select `awg_client.conf`.

> Standard WireGuard app does **not** support `Jc`/`S1`/`S2`/`H1`–`H4` fields.

---

## SSH after hardening

```bash
# Foreign VPS
ssh -i ~/.ssh/id_vps -p 42228 vpnuser@FOREIGN_IP

# Relay VPS
ssh -i ~/.ssh/id_vps -p 42228 vpnuser@RELAY_IP
```

---

## Rollback

Removes all VPN services, restores SSH to port 22, re-enables password auth, and reboots.
Run with the **hardened** SSH port and key:

```bash
ansible-playbook -i secrets.yaml playbook-rollback.yaml --ask-vault-pass \
    -e "ansible_port=42228 ansible_user=vpnuser" \
    -e "ansible_ssh_private_key_file=~/.ssh/id_vps"
```

> Your SSH session **will drop** when sshd restarts. Reconnect as `ssh root@IP`.

The rollback also restores the root password to the value stored in `secrets.yaml`
(`ansible_ssh_pass` for each host).

---

## Re-running / updating

Both playbooks are **idempotent**. Credentials are persisted on the servers and reused:

| File | Server | Contents |
|---|---|---|
| `/opt/xray/config_vars.yaml` | foreign | Xray UUID, x25519 keys, XHTTP path |
| `/opt/xray/relay_config_vars.yaml` | relay | Three relay UUIDs, x25519 keys |
| `/etc/amnezia/amneziawg/awg_vars.yaml` | foreign | AWG server keys + obfuscation params |
| `/etc/amnezia/amneziawg/relay_vars.yaml` | relay | AWG client keypair |
| `/opt/hysteria2/vars.yaml` | foreign | Hysteria2 auth + obfs passwords |
| `/etc/letsencrypt/live/<domain>/` | foreign | TLS cert (renewed weekly via cron) |

To rotate credentials for a service, delete its vars file and re-run the playbook.

---

## Variables reference

All secrets go in `secrets.yaml`. Version numbers and ports live in the playbooks.

| Variable | File | Description |
|---|---|---|
| `foreign_domain` | secrets.yaml | Domain for foreign nginx + Let's Encrypt + Xray REALITY SNI |
| `relay_domain` | secrets.yaml | Russian domain for relay nginx + Let's Encrypt + Xray REALITY SNI |
| `certbot_email` | secrets.yaml | Let's Encrypt registration email (used for both domains) |
| `ssh_port` | secrets.yaml | SSH port after hardening (default `42228`) |
| `ssh_user` | secrets.yaml | New sudo user created on both servers (default `vpnuser`) |
| `ssh_public_key_path` | secrets.yaml | Local path to SSH public key |
| `ssh_user_password` | secrets.yaml | Password for new sudo user (empty = key-only) |
| `new_root_password` | secrets.yaml | New root password (empty = keep current) |
| `xray_version` | playbook-foreign.yaml | Xray-core release tag |
| `xray_xhttp_port` | playbook-foreign.yaml | VLESS+XHTTP inbound port (default `8443`) |
| `awg_port` | playbook-foreign.yaml | AmneziaWG UDP port (default `51820`) |
| `hysteria2_port` | playbook-foreign.yaml | Hysteria2 UDP port (default `443`) |
| `awg_relay_ip` | playbook-relay.yaml | Relay's IP inside AWG subnet (default `10.66.66.3`) |

---

## Credits

Original playbook — [dmgening](https://github.com/dmgening), productionised by [pilosus](https://github.com/pilosus).
Hardening, AmneziaWG, XHTTP, Hysteria2, self-steal, 2-hop relay — [h4zzkR](https://github.com/h4zzkR).

https://ntc.party/t/%D1%80%D0%B0%D0%B1%D0%BE%D1%87%D0%B8%D0%B9-%D0%BE%D0%B1%D1%85%D0%BE%D0%B4-%D0%B1%D0%B5%D0%BB%D1%8B%D1%85-%D1%81%D0%BF%D0%B8%D1%81%D0%BA%D0%BE%D0%B2/21884/136

https://web.archive.org/web/20260312164333/https://habr.com/ru/articles/1009542/


MIT License
