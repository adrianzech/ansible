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

The role creates the persistent key directory but does not manage the
periphery public key; the container generates it on first start.

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

After the first run, SSH access is available by key only. Access filtering is
handled by the upstream OPNsense system.

Automatic security updates may reboot hosts at 02:00 when required.
