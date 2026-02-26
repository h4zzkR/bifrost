# bifrǫst — the Rainbow Bridge

Ansible playbook that turns a bare Ubuntu VPS into a hardened VPN server.
One command deploys all protocols; you get ready-to-import client configs at the end.

![image](https://github.com/user-attachments/assets/3b8d8640-9665-43ec-8d9f-5965d3244c02)

---

## Supported protocols

| Protocol | Transport | Port | Notes |
|---|---|---|---|
| **VLESS + Reality + XTLS-Vision** | TCP | 443 | Primary. Disguises traffic as TLS to apple.com. Best all-round bypass for Russia 2026. |
| **VLESS + Reality + XHTTP** | TCP | 8443 | Secondary Xray transport. Multiplexed HTTP/2 stream. Effective when persistent TCP is throttled. Same Reality disguise, no Nginx needed. |
| **AmneziaWG** | UDP | 51820 | WireGuard fork with junk-packet obfuscation. Defeats fixed-size handshake fingerprinting used by TSPU. |

Toggle protocols via variables in [playbook-bridge-setup.yaml](playbook-bridge-setup.yaml):

```yaml
enable_xray:      true   # deploys both Vision (443) and XHTTP (8443)
enable_amneziawg: true
```

### Why these protocols?

* **OpenVPN / plain WireGuard** — blocked 100 % within seconds by Russian TSPU (as of 2025).
* **VLESS+Reality+Vision** — the server reuses the real TLS handshake of a target site (apple.com).
  A censor probing the port gets a genuine Apple TLS Hello. No self-signed cert to fingerprint.
* **VLESS+Reality+XHTTP** — same Reality disguise but uses the XHTTP `stream-one` transport.
  Useful when ISPs throttle persistent TCP/TLS connections to VPS IPs (≥ 15 KB threshold seen in Russia).
  Client: v2rayNG / v2rayN / Hiddify — **Sing-box does not support XHTTP yet**.
* **AmneziaWG** — prepends random-sized junk before every WireGuard handshake packet,
  randomises packet sizes with `S1`/`S2`, and uses custom 32-bit magic headers `H1`–`H4`.
  RKN UDP blocking targets fixed-size WireGuard init packets; junk breaks that signature.

---

## Requirements

* A VPS running **Ubuntu 22.04 LTS** outside Russia with SSH root access
* Locally: **Python 3** + **pipx**

---

## Quick start (5 steps)

### 1. Clone and enter the repo

```bash
git clone https://github.com/h4zzkR/bifrost
cd bifrost
```

### 2. Install Ansible

```bash
sudo apt install pipx
pipx ensurepath
source ~/.zshrc
pipx install ansible-core --include-deps
pipx inject ansible-core passlib
ansible-galaxy collection install ansible.posix
```

### 3. Create the inventory file

Copy the example and fill in your VPS details:

```bash
cp inventory.ini.example ~/.ansible/hosts
# edit ~/.ansible/hosts — replace IP and root password
ansible-vault encrypt ~/.ansible/hosts   # encrypt before storing
```

Minimal content (all vars on one line — INI format does not support `\` continuation):

```ini
[vpn]
my-vps ansible_host=1.2.3.4 ansible_user=root ansible_ssh_pass=ROOT_PASSWORD ansible_become_pass=ROOT_PASSWORD
```

### 4. Create the secrets file

```bash
ansible-vault create secrets.yml
```

Contents:

```yaml
ssh_user_password: "STRONG_PASSWORD_FOR_NEW_USER"
new_root_password: "STRONG_NEW_ROOT_PASSWORD"
```

### 5. Generate an SSH key (for key-based login after hardening)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_vps -N ""
```

Update `ssh_public_key_path` in [playbook-bridge-setup.yaml](playbook-bridge-setup.yaml) if your key is elsewhere.

---

## Run

```bash
ansible-playbook -v playbook-bridge-setup.yaml \
    --inventory ~/.ansible/hosts \
    --extra-vars "@secrets.yml" \
    --ask-vault-pass \
    -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
```

When the playbook finishes, two files appear in the current directory:

| File | Use |
|---|---|
| `xray_client_connect_link.txt` | Two VLESS URIs (Vision + XHTTP) — paste into client app |
| `awg_client.conf` | AmneziaWG config — import into AmneziaVPN app |

---

## Playbook variables

All variables live at the top of [playbook-bridge-setup.yaml](playbook-bridge-setup.yaml).
Override any of them with `-e "var=value"` on the command line.

| Variable | Default | Description |
|---|---|---|
| `enable_xray` | `true` | Deploy VLESS+Reality (Vision + XHTTP) |
| `enable_amneziawg` | `true` | Deploy AmneziaWG |
| `xray_version` | `v26.2.4` | Xray-core release tag (check [releases](https://github.com/XTLS/Xray-core/releases)) |
| `xray_path` | `/opt/xray` | Xray installation directory |
| `xray_xhttp_port` | `8443` | Port for VLESS+XHTTP+Reality inbound |
| `awg_port` | `51820` | AmneziaWG UDP listen port |
| `ssh_port` | `42228` | SSH port after hardening |
| `ssh_user` | `vpnuser` | New sudo user created on server |
| `ssh_public_key_path` | `~/.ssh/id_vps.pub` | Your public key deployed to server |

---

## Connecting clients

### VLESS+Reality+Vision (port 443)

Clients: **Hiddify**, **Nekoray**, **v2rayNG**, **v2rayN**, **Sing-Box**

1. Copy the first line of `xray_client_connect_link.txt`
2. In your client choose **Add from clipboard** or **Import link**

```
vless://UUID@IP:443/?encryption=none&type=tcp&sni=files.pythonhosted.org&fp=chrome
  &security=reality&flow=xtls-rprx-vision&pbk=PUBLIC_KEY&sid=SHORT_ID
  &packetEncoding=xudp#Bifrost-VLESS-Vision
```

### VLESS+Reality+XHTTP (port 8443)

Clients: **Hiddify**, **v2rayNG**, **v2rayN** (Sing-Box does not support XHTTP yet)

1. Copy the second line of `xray_client_connect_link.txt`
2. Import the link as above

```
vless://UUID@IP:8443/?encryption=none&type=xhttp&path=%2FPATH&sni=files.pythonhosted.org
  &fp=chrome&security=reality&pbk=PUBLIC_KEY&sid=SHORT_ID#Bifrost-VLESS-XHTTP
```

> Switch to this profile if the Vision profile becomes slow or unstable.

### AmneziaWG

Clients: **[AmneziaVPN](https://amnezia.org/)** — Windows / macOS / iOS / Android

1. Open AmneziaVPN → **Add VPN** → **Import config from file**
2. Select `awg_client.conf`
3. Connect

> The standard WireGuard app does **not** understand `Jc`/`S1`/`S2`/`H1`–`H4` obfuscation fields.

---

## SSH after hardening

The security role changes the SSH port and disables password auth.

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_vps
ssh -i ~/.ssh/id_vps -p 42228 vpnuser@YOUR_VPS_IP
```

---

## Validate the deployment

Run [playbook-validate.yaml](playbook-validate.yaml) to verify all services are operational:

```bash
ansible-playbook playbook-validate.yaml \
    --inventory ~/.ansible/hosts \
    --ask-vault-pass \
    -e "ansible_port=42228 ansible_user=vpnuser" \
    -e "ansible_ssh_private_key_file=~/.ssh/id_vps"
```

What it checks:
- Xray service is active and config passes syntax test (`xray run -test`)
- TCP ports 443 and 8443 are listening
- `awg-quick@awg0` is active, `awg0` interface is UP
- UDP port 51820 is open (`ss -ulnp`)
- IPv4 forwarding is enabled
- UFW rules are in place
- Prints both VLESS connection links

---

## Rollback

[playbook-rollback.yaml](playbook-rollback.yaml) completely removes all VPN services and restores
the server to its pre-deploy state.

```bash
ansible-playbook playbook-rollback.yaml \
    --inventory ~/.ansible/hosts \
    --ask-vault-pass \
    -e "ansible_port=42228 ansible_user=vpnuser" \
    -e "ansible_ssh_private_key_file=~/.ssh/id_vps"
```

What it reverts:
- Stops and removes Xray and AmneziaWG services + config files
- Purges `amneziawg` / `wireguard-tools` packages and PPA
- Reverts sysctl (ip_forward → 0, congestion control → cubic)
- Resets UFW and re-allows port 22
- Restores SSH to port 22 with password auth and root login re-enabled

> Your current SSH session **will drop** when sshd restarts. Reconnect as `ssh root@YOUR_VPS_IP`.

Fine-grained control — disable individual rollback steps:

```bash
-e "rollback_ssh=false"        # keep SSH hardening
-e "rollback_amneziawg=false"  # keep AmneziaWG
```

---

## Re-running / updating

The main playbook is **idempotent**.

* **Xray** — credentials stored in `/opt/xray/config_vars.yaml` on server. Re-running only reinstalls if `xray_version` changes.
* **AmneziaWG** — keys and obfuscation params stored in `/etc/amnezia/amneziawg/awg_vars.yaml`. Re-running preserves existing keys.

To rotate keys, delete the respective vars file on the server and re-run.

---

## Credits

Original Ansible playbook and `xray` role — [dmgening](https://github.com/dmgening),
productionised by [pilosus](https://github.com/pilosus).
Upgraded for newer Xray, VPS hardening role, AmneziaWG role, XHTTP transport — [h4zzkR](https://github.com/h4zzkR).

Reference articles:
- [XTLS/Xray tutorial (Habr, RU)](https://habr.com/ru/articles/799751/) by MiraclePtr
- [Personal proxy with Reality/CDN (Habr, EN)](https://habr.com/en/articles/990542/)
- [XHTTP for VLESS overview (Habr, EN)](https://habr.com/en/articles/990208/)
- [Coexisting Vision + XHTTP on one port](https://henrywithu.com/coexisting-xray-vless-tcp-xtls-vision-and-vless-xhttp-reality-on-a-single-port/)
- [AmneziaWG — bypassing Russia's WireGuard block](https://hub.xeovo.com/posts/27-bypassing-russias-wireguard-block-meet-amneziawg/)
- [Official AmneziaWG docs](https://docs.amnezia.org/documentation/amnezia-wg/)

MIT License
