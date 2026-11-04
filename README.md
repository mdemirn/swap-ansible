# swap-ansible

Ansible role and playbook that disable swap on Linux hosts for good and set
`vm.swappiness`, the usual preparation for Kubernetes nodes and for databases that
recommend running without swap.

## What it does

On every host in the `linux_servers` group, the `disable_swap` role:

1. Checks the current state: active swap (`swapon --show`), swap lines in `/etc/fstab`,
   systemd swap units and the current `vm.swappiness`.
2. Skips hosts that are already done and says so.
3. Otherwise runs `swapoff -a`, backs up `/etc/fstab` to `/etc/fstab.ansible.backup`,
   removes the swap lines, stops and masks every systemd swap unit and `swap.target`,
   and verifies that no swap is left.
4. If any of that fails, restores `/etc/fstab` from the backup and stops with an error.
5. Writes `vm.swappiness` to `/etc/sysctl.d/99-swappiness.conf` and reloads it.

Running it again changes nothing on hosts that are already configured.

Supported platforms: EL 7, 8 and 9; Ubuntu 20.04 and 22.04; Debian 11 and 12.

## Requirements

- Ansible (ansible-core 2.10 or later) on the control machine
- The `ansible.posix` collection
- SSH access to the hosts as root, or as a user with sudo

## Usage

```bash
git clone https://github.com/mdemirn/swap-ansible.git
cd swap-ansible
ansible-galaxy collection install -r collections/requirements.yml
```

Put your hosts into `inventory/hosts.yml`:

```yaml
---
all:
  children:
    linux_servers:
      hosts:
        node-01:
          ansible_host: node-01.example.internal
          ssh_private_key: ~/.ssh/id_ed25519
```

Run it on one host first, then on all of them:

```bash
ansible-playbook site.yml --limit node-01
ansible-playbook site.yml
```

Only the swappiness part, or only the swap part:

```bash
ansible-playbook site.yml --tags swappiness
ansible-playbook site.yml --tags swap
```

## Variables

Set them in `inventory/group_vars/all.yml` or per host.

| Variable | Default | Meaning |
|---|---|---|
| `swap_disable_permanently` | `true` | Stop and mask the systemd swap units and `swap.target` |
| `swap_remove_fstab_entry` | `true` | Remove swap lines from `/etc/fstab` |
| `swap_backup_fstab` | `true` | Back up `/etc/fstab` first and restore it if the role fails |
| `swap_set_swappiness` | `true` | Manage `vm.swappiness` |
| `swap_swappiness_value` | `0` | Value for `vm.swappiness` |
| `ssh_auth_method` | `key` | `key` uses each host's `ssh_private_key`; `password` uses `vault_ssh_password` |

## SSH password authentication

With `ssh_auth_method: password`, the playbook reads `vault_ssh_password` from
`inventory/group_vars/vault.yml`. Put the password there and encrypt the file before
you commit anything:

```bash
ansible-vault encrypt inventory/group_vars/vault.yml
ansible-playbook site.yml --ask-vault-pass
```

To avoid the prompt, store the vault password in `.vault_pass` (ignored by git) and
uncomment `vault_password_file` in `ansible.cfg`.

## Tags

| Tag | Tasks |
|---|---|
| `swap` | Swap checks and the disable steps |
| `swappiness` | Swappiness check and configuration |
| `check` | State checks only |
| `disable`, `configure` | The changing steps for swap and swappiness |
| `info` | Status messages |

## Layout

```
site.yml                          playbook
inventory/hosts.yml               hosts (linux_servers group)
inventory/group_vars/all.yml      settings for all hosts
inventory/group_vars/vault.yml    SSH password for password authentication (encrypt it)
roles/disable_swap/               the role
collections/requirements.yml      ansible.posix
```
