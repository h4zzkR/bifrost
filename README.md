# xray-ansible: get XTLS/Xray server up & running using Ansible

Get your [XTLS/Xray](https://github.com/xtls) server up and running
with [Ansible](https://www.ansible.com/).

## Requirements

1. An Ubuntu 22.04 LTS machine
2. SSH config to access the Ubuntu machine as a `root` (to make Ansible playbook work)
3. Python & `pipx`

## Provision the Linux machine

1. Clone the [repo](https://github.com/h4zzkR/bivrest)
2. Change directory

```
cd bivrest
```

3. Install requirements

```
sudo apt update && sudo apt install python3-passlib
pipx install ansible-core --include-deps
pipx inject ansible-core passlib
```

4. Add your Linux machine IP address and root credentials to the encrypted Ansible inventory file, e.g. `~/.ansible/hosts`:

```
[xray]
xray-az-label-1 ansible_host=38.180.110.169 ansible_user=root ansible_ssh_pass=your_root_password ansible_become_pass=your_root_password ansible_connection=ssh
```

Encrypt it:

```
ansible-vault encrypt ~/.ansible/hosts
```

5. Generate ssh key for the server auth:

```
ssh-keygen -t ed25519 -f ~/.ssh/public/key/path -N ""
```

6. Modify `playbook-bridge-setup.yaml` if neccessary

7. Create secrets to store new passwords for root and new ssh sudo user:

```
ansible-vault create secrets.yml
```

Contents must be:

```
# Contents of secrets.yml
ssh_user_password: "YOUR_USER_NEW_PASSWORD"
new_root_password: "YOUR_ROOT_NEW_PASSWORD"
```

8. Run the playbook with verbosity 1:

```
ansible-playbook -v --limit xray playbook-bridge-setup.yaml \
    --inventory ~/.ansible/hosts \
    --user root \
    --extra-vars "@secrets.yml" \
    --ask-vault-pass \
    --ask-vault-password \
    -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
```

9. To connect to server via ssh key, first run this:

```
chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys
eval $(ssh-agent)
ssh-add ~/.ssh/public/key/path
```

10. Remember link of this format from Ansible playbook logs:

```
vless://UUID@YOUR_SERVER_IP:443/?encryption=none&type=tcp&sni=SITE_DOMAIN&fp=chrome&security=reality&alpn=h2&flow=xtls-rprx-vision&pbk=YOUR_PUBLIC_KEY&packetEncoding=xudp
```

## Set up the client

1. Using Hiddify or Nekoray, add new connection with the help of link you got from Ansible logs

# Credits

The Ansible playbook and the `xray` role have been developed by
[dmgening](https://github.com/dmgening) and productionized by
[pilosus](https://github.com/pilosus).
Scripts upgraded for new XRay version and VPS hardenings role added by
[h4zzkR](https://github.com/h4zzkR).

The playbook relies heavily on the
[tutorial](https://habr.com/ru/articles/799751/) (in Russian; last
accessed on 22.06.25) by MiraclePtr.
