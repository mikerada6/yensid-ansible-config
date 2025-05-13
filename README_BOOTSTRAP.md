# Bootstrapping a New Host (`randomland-test`) with Ansible

This guide shows you how to:

1. Generate an SSH keypair named after your new host (`randomland_test_ansible`)
2. Add the host to your Ansible inventory under `[bootstrap]`
3. Run `playbooks/bootstrap.yml` to create the `ansible` user on the remote machine
4. Verify that passwordless SSH and sudo work for the new user

## Prerequisites on Yensid

- Ansible installed on your control node (“Yensid”):
  ```bash
  sudo apt update
  sudo apt install -y ansible
  ```
- Network/SSH access to your new VM (`randomland-test` at IP `10.0.2.50`) via the default user `rez`.
- Project directory layout:
  ```
  my-ansible/
  ├── inventory/
  │   └── hosts.ini
  └── playbooks/
      └── bootstrap.yml
  ```

## Step 1: Generate SSH Keypair

On Yensid, run:
```bash
HOST=randomland-test
# Convert hyphens to underscores for the filename:
KEYBASE=${HOST//-/_}_ansible

ssh-keygen -t ed25519 \
  -f ~/.ssh/${KEYBASE} \
  -C "ansible→${HOST}" \
  -N ""
```
This creates:
- `~/.ssh/randomland_test_ansible` (private key)
- `~/.ssh/randomland_test_ansible.pub` (public key)

## Step 2: Update Inventory

Edit `inventory/hosts.ini` and add your new host:

```ini
[bootstrap]
# first‐touch entries (SSH in as rez)
randomland-test ansible_host=10.0.2.50 ansible_user=rez \
  ansible_ssh_private_key_file=~/.ssh/rez_to_bootstrap

[dev]
# after bootstrap, connect as ansible@…
randomland-test ansible_host=10.0.2.50 ansible_user=ansible \
  ansible_ssh_private_key_file=~/.ssh/randomland_test_ansible

[springapps:children]
dev
# …existing groups…
```

## Step 3: (Optional) Set Defaults for Bootstrap

Create `inventory/group_vars/bootstrap.yml`:
```yaml
bootstrap_user:        ansible
bootstrap_group:       ansible
bootstrap_shell:       /bin/bash
# This will pick up ~/.ssh/randomland_test_ansible.pub automatically
bootstrap_public_key:  "{{ lookup('env','HOME')
                          + '/.ssh/' 
                          + (inventory_hostname|replace('-', '_'))
                          + '_ansible.pub' }}"
```

Or inline these vars under `vars:` in `playbooks/bootstrap.yml`.

## Step 4: Run the Bootstrap Playbook

From your project root:
```bash
cd ~/my-ansible

ansible-playbook \
  -i inventory/hosts.ini \
  playbooks/bootstrap.yml \
  -l bootstrap \
  -kK
```
- `-l bootstrap` limits the run to the `[bootstrap]` group
- `-k` prompts once for `rez`’s SSH password
- `-K` prompts once for `rez`’s sudo password (if required)

## Step 5: Verify the New `ansible` User

Test SSH + passwordless sudo:
```bash
ssh -i ~/.ssh/randomland_test_ansible ansible@10.0.2.50 \
  'sudo -n true && echo "PASSWORDLESS SUDO OK"'
```
You should see:
```
PASSWORDLESS SUDO OK
```

## Next Steps

1. (Optional) Remove the host from `[bootstrap]` once fully bootstrapped.
2. Provision your microservices & metrics:
   ```bash
   ansible-playbook \
     -i inventory/hosts.ini \
     playbooks/springapps-with-metrics.yml \
     --limit dev