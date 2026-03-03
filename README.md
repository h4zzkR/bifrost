# bifrǫst — the Rainbow Bridge

Ansible playbook that turns a bare Ubuntu VPS into a hardened VPN server.
One command deploys all protocols; ready-to-import client configs appear locally when done.

![image](https://github.com/user-attachments/assets/3b8d8640-9665-43ec-8d9f-5965d3244c02)

---

## Protocols

| Protocol | Transport | Port | Notes |
|---|---|---|---|
| **VLESS + Reality + XTLS-Vision** | TCP | 443 | Primary. Steals TLS cert from a target site. Best all-round bypass. |
| **VLESS + Reality + XHTTP** | TCP | 8443 | HTTP/2 over Reality. Effective when persistent TCP is throttled. |
| **AmneziaWG** | UDP | 51820 | WireGuard with junk-packet obfuscation. Defeats TSPU fixed-size packet detection. |
| **Hysteria2** | UDP | 443 | QUIC-based proxy with Salamander obfuscation. Immune to TCP throttling. |

### Why these protocols?

* **OpenVPN / plain WireGuard** — blocked within seconds by Russian TSPU.
* **VLESS+Reality** — server borrows a real TLS certificate from `files.pythonhosted.org`. A censor probing the port sees a genuine TLS handshake.
* **XHTTP** — same Reality disguise but splits traffic into multiplexed HTTP/2 chunks. Use when ISPs throttle long-lived TCP streams (≥ 15 KB threshold seen in Russia). Not supported by Sing-box.
* **AmneziaWG** — prepends random junk before every WireGuard handshake, randomises packet sizes with `S1`/`S2`, and uses custom 32-bit magic headers `H1`–`H4`.
* **Hysteria2** — runs over QUIC (UDP). Russian TSPU TCP-freeze never applies. Salamander obfuscation makes packets look like random UDP noise, defeating QUIC fingerprinting.

### Chain proxy (advanced)

For the most hostile ISPs, deploy a **bridge VPS** inside Russia (whitelisted datacenter IP) that relays to a foreign **exit VPS**. The client-to-bridge leg is domestic so the TCP freeze never fires. Use `playbook-chain-setup.yaml`.

---

## Requirements

* A VPS running **Ubuntu 22.04 LTS** with SSH root access
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
new_root_password: "STRONG_NEW_ROOT_PASSWORD"
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
| `xray_client_connect_link.txt` | VLESS URIs (Vision + XHTTP) |
| `awg_client.conf` | AmneziaWG config |
| `hysteria2_client_connect_link.txt` | Hysteria2 URI |

---

## Variables

| Variable | Default | Description |
|---|---|---|
| `enable_xray` | `true` | Deploy VLESS+Reality (Vision + XHTTP) |
| `enable_amneziawg` | `true` | Deploy AmneziaWG |
| `enable_hysteria2` | `true` | Deploy Hysteria2 |
| `xray_version` | `v26.2.6` | Xray-core release tag |
| `xray_xhttp_port` | `8443` | VLESS+XHTTP port |
| `xray_sni_dest` | `files.pythonhosted.org` | Reality SNI/dest (must match client links) |
| `awg_port` | `51820` | AmneziaWG UDP port |
| `hysteria2_version` | `v2.6.1` | Hysteria2 release tag |
| `hysteria2_port` | `443` | Hysteria2 UDP port |
| `ssh_port` | `42228` | SSH port after hardening |
| `ssh_user` | `vpnuser` | New sudo user created on server |
| `ssh_public_key_path` | `~/.ssh/id_vps.pub` | Public key deployed to server |

---

## Connecting clients

### VLESS+Reality+Vision / XHTTP

Clients: **Hiddify**, **v2rayNG**, **v2rayN**, **Nekoray** (XHTTP not supported in Sing-box)

Import the corresponding line from `xray_client_connect_link.txt` via **Add from clipboard**.

### AmneziaWG

Clients: **[AmneziaVPN](https://amnezia.org/)** — Windows / macOS / iOS / Android

Open AmneziaVPN → **Add VPN** → **Import config from file** → select `awg_client.conf`.

> Standard WireGuard app does **not** support `Jc`/`S1`/`S2`/`H1`–`H4` fields.

### Hysteria2

Clients: **Hiddify**, **Nekoray**, **v2rayN**, **Sing-box**

Import the URI from `hysteria2_client_connect_link.txt` via **Add from clipboard**.

```
hysteria2://AUTH@IP:443?obfs=salamander&obfs-password=PASS&sni=files.pythonhosted.org&insecure=1
```

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

To rotate credentials, delete the respective vars file and re-run.

---

## Credits

Original playbook — [dmgening](https://github.com/dmgening), productionised by [pilosus](https://github.com/pilosus).
Hardening, AmneziaWG, XHTTP, Hysteria2 — [h4zzkR](https://github.com/h4zzkR).

MIT License
