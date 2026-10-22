# vps-security-check

Single Ansible playbook with basic actions for securing a fresh Debian/Ubuntu VPS.

## What it does

| Tag | Action |
| --- | --- |
| `user_setup` | Creates `vps-user`, adds it to `sudo` (passwordless), installs your public key |
| `ssh_hardening` | Checks that `vps-user` can log in with the key, then sets `PermitRootLogin no` and `PasswordAuthentication no`, restarts sshd and asserts the effective config with `sshd -T` |
| `fail2ban` | Installs fail2ban and configures the `sshd` jail (see below) |

### fail2ban behaviour

- 3 failed attempts within 1 day (`maxretry = 3`, `findtime = 1d`) ban the IP **permanently** (`bantime = -1`).
- `mode = aggressive` also counts pre-auth failures such as invalid users and failed key exchanges.
- A banned IP is blocked on **all ports and all IP protocols** (`banaction_allports`, `protocol = 0-255`),
  not only on SSH. This uses the nftables action; rules live in `table inet f2b-table`.
- Only IPv4 is handled by default. Fail2ban adds IPv6 sets/rules on the first IPv6 ban,
  so nothing is needed for IPv4-only servers.

## Requirements

- Ansible on your machine (tested with `ansible-core` 2.20).
- A Debian/Ubuntu target with SSH access (root for the first run, see below). The playbook uses `apt`
  and the `ssh` service name, so other distributions are not supported.
- `sshpass` on your machine only if you want to use a root *password* for the first run.
- An SSH key pair for `vps-user`. The playbook reads the public key from `<vps_key_file>.pub`
  (default `~/.ssh/id_ed25519_vps.pub`, variable `vps_key_file` in `playbook.yaml`):

  ```sh
  ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_vps
  ```

## Setup

1. Copy the example inventory. `inventory.yaml` is in `.gitignore`, so your host data is not committed:

   ```sh
   cp inventory.yaml.example inventory.yaml
   ```

2. Edit `inventory.yaml`: set the server IP, `ansible_user: vps-user` and
   `ansible_private_key_file` to the `vps-user` **private** key (only the path is stored, never the key):

   ```yaml
   webservers:
     hosts:
       203.0.113.10
     vars:
       ansible_user: vps-user
       ansible_private_key_file: ~/.ssh/id_ed25519_vps
   ```

## Usage

### First run on a fresh server (only root available)

`vps-user` does not exist yet, so override the inventory user for this one run:

```sh
# root with an SSH key
ansible-playbook -i inventory.yaml playbook.yaml -e ansible_user=root --private-key ~/.ssh/<root_key>

# root with a password (needs sshpass)
ansible-playbook -i inventory.yaml playbook.yaml -e ansible_user=root -k
```

Note: `--private-key`/`-k` apply to root only for this run. The playbook still tests
`vps-user` with `vps_key_file`, not with the root key.

The playbook creates `vps-user`, installs its key, then verifies that `vps-user` can log in with that
key. Only after that check passes does it disable root and password login. If the check fails, the run
stops while root access still works, so you cannot lock yourself out this way.

### Later runs

```sh
ansible-playbook -i inventory.yaml playbook.yaml
```

The playbook is idempotent. Apart from the `Update APT package cache` task, which always reports
`changed`, a second run should change nothing.

### Run only part of it

```sh
ansible-playbook -i inventory.yaml playbook.yaml --tags fail2ban
ansible-playbook -i inventory.yaml playbook.yaml --tags ssh_hardening
ansible-playbook -i inventory.yaml playbook.yaml --tags user_setup
```

### Dry run

```sh
ansible-playbook -i inventory.yaml playbook.yaml --check --diff
```

Check mode skips the `sshd -T` and `fail2ban-client` commands, so the final
`Verify SSH hardening settings` assertion fails in a dry run even when the server is fine.
Read the `--diff` output of the earlier tasks instead.

## Verifying the result on the server

```sh
sudo fail2ban-client status sshd                   # jail running, banned count
sudo fail2ban-client get sshd actions              # nftables
sudo fail2ban-client get sshd action nftables type # allports
sudo nft list chain inet f2b-table f2b-chain       # expect: meta l4proto 0-255 ip saddr @addr-set-sshd reject ...
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication'
```

## Managing bans

Bans are permanent, so check before testing with your own address. If you lock yourself out of SSH,
use your provider's web/VNC console.

```sh
sudo fail2ban-client set sshd unbanip <ip>   # remove one ban
sudo fail2ban-client set sshd banip <ip>     # ban manually (e.g. test with 203.0.113.5)
sudo fail2ban-client unban --all             # clear every ban
```

To never ban your own address, add `ignoreip = 127.0.0.1/8 <your_ip>` to the jail section in
`playbook.yaml` and re-run with `--tags fail2ban`.

## Files

- `playbook.yaml` — the playbook
- `inventory.yaml.example` — template inventory
- `inventory.yaml` — your inventory (git-ignored)
