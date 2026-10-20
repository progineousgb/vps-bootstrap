<p align="center"><img src="assets/logo.png" alt="Pro Gineous" width="360"></p>

# vps-bootstrap

Two ways to get a new Debian or Ubuntu VPS to the same baseline: a cloud-init file for the first boot, or an Ansible playbook you can run against servers that already exist (and re-run later without breaking anything).

The baseline:

- a sudo user that logs in with an SSH key only
- root login and password authentication off
- UFW: deny incoming, rate-limited SSH, 80 and 443 open
- fail2ban watching sshd through the systemd journal
- unattended security upgrades
- 1 GB swap file, swappiness 10
- timezone set, NTP on

Targets Debian 12/13 and Ubuntu 22.04/24.04.

## cloud-init

If your provider lets you paste user-data when creating the server, use [cloud-init/user-data.yaml](cloud-init/user-data.yaml). Edit the username, the `AllowUsers` line and the SSH public key first. The first line must stay `#cloud-config`.

Check it before you paste it:

```bash
cloud-init schema -c cloud-init/user-data.yaml
```

After the server is up, check what happened with `sudo cloud-init status --long` and `/var/log/cloud-init-output.log`.

## Ansible

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
cp inventory.example.ini inventory.ini        # your server IPs
$EDITOR group_vars/all.yml                    # user name and SSH keys
ansible-playbook site.yml
```

The first run connects as `root` (or whatever user the image gives you). After that, root login is disabled, so change `ansible_user` in the inventory to your new user.

Run one part on its own by limiting roles in a copy of `site.yml`, or with `--start-at-task`.

### Variables

All set in [ansible/group_vars/all.yml](ansible/group_vars/all.yml):

| Variable | Default | |
|----------|---------|-|
| `user_name` | `deploy` | Admin user to create |
| `user_ssh_keys` | placeholder | List of public keys. The play stops if empty. Keys not in this list are removed from the user's `authorized_keys`. |
| `user_passwordless_sudo` | `true` | Set `false` and give the user a password if you want sudo to ask |
| `ssh_port` | `22` | See the note below before changing |
| `ssh_hardening_allow_users` | `[user_name]` | Becomes `AllowUsers` |
| `ufw_allowed_tcp_ports` | `[80, 443]` | SSH is added separately |
| `fail2ban_bantime` / `findtime` / `maxretry` | `1h` / `10m` / `5` | sshd jail |
| `unattended_upgrades_auto_reboot` | `false` | Reboot at `unattended_upgrades_reboot_time` when an update needs it |
| `swap_size_mb` | `1024` | `0` skips swap. Nothing is created if any swap is already active. |
| `timezone_name` | `UTC` | |

### Changing the SSH port

Set `ssh_port`, run the playbook. UFW keeps the port Ansible is currently connected on open, so the run does not cut itself off. Then set `ansible_port` in the inventory to the new port, run again, and delete the old rule by hand:

```bash
sudo ufw status numbered
sudo ufw delete <n>
```

On Ubuntu 22.10 and later sshd is socket-activated; the role disables `ssh.socket` when the port is not 22 so the `Port` setting is honoured.

## Before you run either

Keep the provider's web console open in another tab. If something goes wrong with SSH you will need it. Take a snapshot if the server already holds anything you care about.

## Validation

```bash
cloud-init schema -c cloud-init/user-data.yaml      # Valid schema
cd ansible
ansible-playbook -i inventory.example.ini --syntax-check site.yml
ansible-lint                                        # passes the production profile
```

## License

MIT, see [LICENSE](LICENSE).

---

From [Pro Gineous](https://progineous.com/). The single-script version of the same hardening is in [server-toolkit](https://gitlab.com/pro-gineous-group/server-toolkit).
