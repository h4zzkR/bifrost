# bifrǫst — the Rainbow Bridge

Ansible playbook that turns a bare Ubuntu VPS into a hardened VPN server.
One command deploys all protocols; ready-to-import client configs appear locally when done.

![image](https://github.com/user-attachments/assets/3b8d8640-9665-43ec-8d9f-5965d3244c02)

---

## Protocols

| Protocol | Transport | Port | Notes |
|---|---|---|---|
| **VLESS + Reality + XHTTP** | TCP | 8443 | Primary. HTTP stream over Reality. Effective against TSPU TCP throttling. |
| **AmneziaWG** | UDP | 51820 | WireGuard with junk-packet obfuscation. Defeats TSPU fixed-size packet detection. |
| **Hysteria2** | UDP | 443 | QUIC-based proxy with Salamander obfuscation. Immune to TCP throttling. |

### Why these protocols?

* **OpenVPN / plain WireGuard** — blocked within seconds by Russian TSPU.
* **VLESS+Reality+XHTTP** — splits traffic into multiplexed HTTP chunks over a Reality TLS session. The server uses a real TLS certificate from your own domain (self-steal), so TSPU probing port 8443 sees a legitimate handshake. Not supported by Sing-box.
* **AmneziaWG** — prepends random junk before every WireGuard handshake, randomises packet sizes with `S1`/`S2`, and uses custom 32-bit magic headers `H1`–`H4`.
* **Hysteria2** — runs over QUIC (UDP). Russian TSPU TCP-freeze never applies. Salamander obfuscation makes packets look like random UDP noise, defeating QUIC fingerprinting.

### Self-steal (domain camouflage)

The server runs nginx on your own domain with a real Let's Encrypt certificate. Nginx serves port 443/TCP publicly — TSPU probing that port sees an ordinary HTTPS website. Xray Reality steals the certificate from this local nginx for the XHTTP inbound on 8443.

```
443/TCP  → nginx → real website  (TSPU sees this)
443/UDP  → Hysteria2             (different protocol, no conflict)
4443/TCP → nginx (127.0.0.1)    → Xray steals cert from here
8443/TCP → Xray VLESS+XHTTP+Reality
51820/UDP → AmneziaWG
```

---

## Requirements

* A VPS running **Ubuntu 22.04 LTS** with SSH root access
* A domain with an **A record pointing to the VPS** (for Let's Encrypt)
* Locally: **Python 3** + **pipx**

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

### 3. Create the inventory

```bash
cp inventory.ini.example ~/.ansible/hosts
# edit ~/.ansible/hosts — replace IP and password
ansible-vault encrypt ~/.ansible/hosts
```

```ini
[vpn]
my-vps ansible_host=1.2.3.4 ansible_user=root ansible_ssh_pass=ROOT_PASSWORD ansible_become_pass=ROOT_PASSWORD
```

### 4. Create the secrets file

```bash
ansible-vault create secrets.yml
```

```yaml
ssh_user_password: "STRONG_PASSWORD_FOR_NEW_USER"
new_root_password:  "STRONG_NEW_ROOT_PASSWORD"

# Self-steal domain camouflage (DNS A record must point to the VPS)
webserver_domain: "yourdomain.com"
certbot_email:    "you@example.com"
xray_sni_dest:    "yourdomain.com"   # must match webserver_domain
```

### 5. Generate an SSH key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_vps -N ""
```

Update `ssh_public_key_path` in [playbook-bridge-setup.yaml](playbook-bridge-setup.yaml) if needed.

---

## Run

```bash
ansible-playbook -v playbook-bridge-setup.yaml \
    --inventory ~/.ansible/hosts \
    --extra-vars "@secrets.yml" \
    --ask-vault-pass \
    -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
```

Output files saved locally when done:

| File | Contents |
|---|---|
| `xray_client_connect_link.txt` | VLESS+XHTTP URI |
| `awg_client.conf` | AmneziaWG config |
| `hysteria2_client_connect_link.txt` | Hysteria2 URI |

---

## Variables

All vars live in [playbook-bridge-setup.yaml](playbook-bridge-setup.yaml). Secrets go in `secrets.yml`.

| Variable | Where | Description |
|---|---|---|
| `webserver_domain` | secrets.yml | Domain for nginx + Let's Encrypt (= `xray_sni_dest`) |
| `certbot_email` | secrets.yml | Email for Let's Encrypt registration |
| `xray_sni_dest` | secrets.yml | Reality SNI / client link SNI (= `webserver_domain`) |
| `ssh_user_password` | secrets.yml | Password for new sudo user |
| `new_root_password` | secrets.yml | New root password |
| `enable_xray` | playbook | Deploy VLESS+XHTTP+Reality |
| `enable_amneziawg` | playbook | Deploy AmneziaWG |
| `enable_hysteria2` | playbook | Deploy Hysteria2 |
| `enable_webserver` | playbook | Deploy nginx + Let's Encrypt (self-steal) |
| `xray_version` | playbook | Xray-core release tag |
| `xray_xhttp_port` | playbook | VLESS+XHTTP port (default `8443`) |
| `xray_reality_dest` | playbook | Xray Reality dest (default `127.0.0.1:4443`) |
| `webserver_nginx_port` | playbook | Internal nginx HTTPS port (default `4443`) |
| `awg_port` | playbook | AmneziaWG UDP port (default `51820`) |
| `hysteria2_port` | playbook | Hysteria2 UDP port (default `443`) |
| `ssh_port` | playbook | SSH port after hardening (default `42228`) |
| `ssh_user` | playbook | New sudo user (default `vpnuser`) |

---

## Connecting clients

### VLESS+XHTTP+Reality

Clients: **Hiddify**, **v2rayNG**, **v2rayN**, **Nekoray** (not supported in Sing-box)

Import the URI from `xray_client_connect_link.txt` via **Add from clipboard**.

### AmneziaWG

Clients: **[AmneziaVPN](https://amnezia.org/)** — Windows / macOS / iOS / Android

Open AmneziaVPN → **Add VPN** → **Import config from file** → select `awg_client.conf`.

> Standard WireGuard app does **not** support `Jc`/`S1`/`S2`/`H1`–`H4` fields.

### Hysteria2

Clients: **Hiddify**, **Nekoray**, **v2rayN**, **Sing-box**

Import the URI from `hysteria2_client_connect_link.txt` via **Add from clipboard**.

---

## SSH after hardening

```bash
ssh -i ~/.ssh/id_vps -p 42228 vpnuser@YOUR_VPS_IP
```

---

## Rollback

```bash
ansible-playbook playbook-rollback.yaml \
    --inventory ~/.ansible/hosts \
    --ask-vault-pass \
    -e "ansible_port=42228 ansible_user=vpnuser" \
    -e "ansible_ssh_private_key_file=~/.ssh/id_vps"
```

Reverts: removes all VPN services, purges packages, resets sysctl/UFW, restores SSH to port 22.

> Your SSH session **will drop** when sshd restarts. Reconnect as `ssh root@YOUR_VPS_IP`.

Fine-grained control:

```bash
-e "rollback_ssh=false"        # keep SSH hardening
-e "rollback_amneziawg=false"  # keep AmneziaWG
```

---

## Re-running / updating

The playbook is **idempotent**. Credentials are persisted on the server and reused on subsequent runs:

- Xray: `/opt/xray/config_vars.yaml`
- AmneziaWG: `/etc/amnezia/amneziawg/awg_vars.yaml`
- Hysteria2: `/opt/hysteria2/hy2_vars.yaml`
- TLS cert: `/etc/letsencrypt/live/<domain>/` (renewed automatically via weekly cron)

To rotate VPN credentials, delete the respective vars file and re-run.

---

## Credits

Original playbook — [dmgening](https://github.com/dmgening), productionised by [pilosus](https://github.com/pilosus).
Hardening, AmneziaWG, XHTTP, Hysteria2, self-steal — [h4zzkR](https://github.com/h4zzkR).

MIT License
