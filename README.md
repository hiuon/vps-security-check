# vps-security-check
Mono ansible playbook with basic actions for securing your vps server

## Usage

Copy `inventory.yaml.example` to `inventory.yaml` and set the host, `ansible_user: vps-user` and the
path to the `vps-user` private key (the matching `.pub` file is installed on the server).

### First run on a fresh server (only root available)

Override the inventory user for this one run:

```sh
ansible-playbook -i inventory.yaml playbook.yaml -e ansible_user=root --private-key ~/.ssh/<root_key>
# or with a root password (needs sshpass):
ansible-playbook -i inventory.yaml playbook.yaml -e ansible_user=root -k
```

The playbook creates `vps-user`, installs its key, then checks that `vps-user` can log in
with that key. Only after that check passes does it disable root and password login.
If the check fails, the run stops while root access still works.

### Later runs

```sh
ansible-playbook -i inventory.yaml playbook.yaml
```
