# Homelab Ansible

Ansible setup for Debian-based homelab hosts.

## Prerequisites

- Ansible, including `ansible-playbook`
- SSH access to the target hosts
- A user provisioned by the server image with `sudo` permissions and SSH key
  access (currently `adrian`). The playbook does not bootstrap root or
  password-based access.
- The collections listed in `collections/requirements.yml`

## Project structure

- `playbooks/homelab.yml`: Main homelab playbook
- `playbooks/komodo-periphery.yml`: Explicit single-host Komodo Periphery deployment
- `playbooks/komodo-update.yml`: Komodo service image updates
- `inventory.yml`: Homelab inventory
- `roles/`: Roles for system configuration, SSH, Docker, updates, and packages

## Usage

Run the playbook:

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook playbooks/homelab.yml --ask-pass --ask-become-pass
```

Use an explicit inventory:

```bash
ansible-playbook -i inventory.yml playbooks/homelab.yml --ask-pass --ask-become-pass
```

Dry run without changes:

```bash
ansible-playbook playbooks/homelab.yml --check --diff --ask-pass --ask-become-pass
```

Run against a single host:

```bash
ansible-playbook playbooks/homelab.yml --limit git.local.zech.co --ask-pass --ask-become-pass
```

## Updates

Package upgrades are an explicit maintenance action and are not run during the
regular configuration playbook. Run upgrades for all hosts and reboot when
required:

```bash
ansible-playbook playbooks/homelab.yml --tags updates
```

Run upgrades without rebooting:

```bash
ansible-playbook playbooks/homelab.yml --tags updates --extra-vars 'update_reboot=false'
```

Combine `--tags updates` with `--limit <hostname>` to update one host only.

## Komodo Periphery

Komodo Periphery is deployed independently from the regular baseline and only
allows one explicitly limited host per run. Current periphery hosts are
`docker-dev.local.zech.co` and `git.local.zech.co`.

```bash
ansible-playbook -i inventory.yml playbooks/komodo-periphery.yml --limit docker-dev.local.zech.co
```

The onboarding key is managed in an encrypted vault file. Create it once and
commit the encrypted file:

```bash
ansible-vault create roles/komodo_periphery/vars/vault.yml
```

Add the following value when prompted:

```yaml
komodo_periphery_onboarding_key: <onboarding-key-from-komodo>
```

Run the deployment with an interactive vault password:

```bash
ansible-playbook -i inventory.yml playbooks/komodo-periphery.yml --limit docker-dev.local.zech.co --ask-vault-pass
```

Alternatively, store the vault password in a local file outside the repository
with mode `0600` and use it when running the playbook:

```bash
ansible-playbook -i inventory.yml playbooks/komodo-periphery.yml --limit docker-dev.local.zech.co --vault-password-file ~/.config/ansible/homelab.vault-pass
```

The role creates the persistent key directory but does not manage the
periphery public key; the container generates it on first start. The onboarding
key remains in `/opt/komodo/.env`, which is owned by the management user and
has mode `0600`.

## Komodo updates

Komodo updates are performed independently from the baseline playbook. The
update covers Komodo Core, MongoDB and Periphery on Docker-Prod, plus
Periphery on the other Periphery hosts. Affected containers can be briefly
unavailable while they are recreated.

```bash
ansible-playbook -i inventory.yml playbooks/komodo-update.yml
```

Limit the update to one host when needed:

```bash
ansible-playbook -i inventory.yml playbooks/komodo-update.yml --limit docker-prod.local.zech.co
```

Set the desired Komodo and MongoDB image versions in
`group_vars/komodo_update_hosts/main.yml`:

```yaml
komodo_version: "2.3.3"
komodo_mongodb_version: "8.3.8"
```

On Docker-Prod, the role manages only `/opt/komodo/compose.ansible.yaml` as a
Compose override. The existing `compose.yaml` and `.env` files, including
their secrets, remain unmanaged.

## Quality checks

Check syntax:

```bash
ansible-playbook playbooks/homelab.yml --syntax-check
```

Run linting:

```bash
ansible-lint
```

## Inventory groups

- `homelab`: Shared Debian baseline, including SSH hardening and updates.
- `docker_hosts`: Also receives Docker.
- `qemu_hosts`: Also receives the QEMU Guest Agent.
- `komodo_periphery_hosts`: Eligible for the separate, explicitly limited
  Komodo Periphery deployment.
- `komodo_update_hosts`: Receives the separate Komodo update playbook.

After the first run, SSH access is available by key only. Access filtering is
handled by the upstream OPNsense system.

Automatic security updates may reboot hosts at 02:00 when required.
