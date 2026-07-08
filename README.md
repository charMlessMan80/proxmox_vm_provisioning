# Proxmox VM Provisioning

Ansible project to provision KVM virtual machines on Proxmox VE by **cloning a
cloud-init enabled template** and configuring each VM with cloud-init, using an
API token for authentication.

## Layout

```
ansible.cfg
requirements.yml                 # required Ansible collections
provision.yml                    # entry-point playbook
inventory/hosts.yml
group_vars/proxmox/
  vars.yml                       # API connection, template, cloud-init + VM list
  vault.yml                      # API token secret (encrypt with ansible-vault)
roles/proxmox_vm/
  defaults/main.yml              # per-VM defaults
  tasks/main.yml                 # clone -> configure cloud-init -> resize -> start
```

## Prerequisites

- A Proxmox VM template that has the cloud-init drive attached (created via
  `qm set <id> --ide2 <storage>:cloudinit`) and a serial console.

## Setup

1. Install the required Ansible collection and Python libraries. The
   `community.general.proxmox_*` modules run on the controller and need the
   `proxmoxer` and `requests` Python libraries installed for the same Python
   interpreter Ansible uses:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   pip install -r requirements.txt
   ```

2. Create an API token in Proxmox: **Datacenter > Permissions > API Tokens**.
   Grant the token's user VM management rights (e.g. `PVEVMAdmin`).

3. Edit `group_vars/proxmox/vars.yml`:
   - `proxmox_api_host`, `proxmox_api_user`
   - `proxmox_default_template` (your cloud-init template name)
   - cloud-init defaults (`proxmox_ci_user`, `proxmox_ci_sshkeys`, ...)
   - the `proxmox_vms` list (name, cores, memory, ipconfig, ...)

4. Put the token secret in `group_vars/proxmox/vault.yml`, then encrypt it:

   ```bash
   ansible-vault encrypt group_vars/proxmox/vault.yml
   ```

## Run

```bash
ansible-playbook provision.yml --ask-vault-pass
```

Dry run:

```bash
ansible-playbook provision.yml --ask-vault-pass --check
```

## Notes

- Each VM is a full clone of the template by default
  (`proxmox_full_clone: true`); set it to `false` for faster linked clones.
- cloud-init handles user, SSH keys, DNS, and IP config (`ip=dhcp` or static
  CIDR per VM via `ipconfig`).
- `disk_size` (e.g. `+20G`) grows the cloned disk after creation.
- The play is idempotent on VM name/vmid: re-running won't recreate existing VMs.
