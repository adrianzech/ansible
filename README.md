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

- `site.yml`: Main playbook
- `production.yml`: Inventory
- `roles/`: Roles for system configuration, SSH, Docker, updates, and packages

## Usage

Run the playbook:

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml --ask-become-pass
```

Use an explicit inventory:

```bash
ansible-playbook -i production.yml site.yml --ask-become-pass
```

Dry run without changes:

```bash
ansible-playbook site.yml --check --diff --ask-become-pass
```

Run against a single host:

```bash
ansible-playbook site.yml --limit git.local.zech.co --ask-become-pass
```

## Updates

Package upgrades are an explicit maintenance action and are not run during the
regular configuration playbook. Run upgrades for all hosts and reboot when
required:

```bash
ansible-playbook site.yml --tags updates --ask-become-pass
```

Run upgrades without rebooting:

```bash
ansible-playbook site.yml --tags updates --extra-vars 'update_reboot=false' --ask-become-pass
```

Combine `--tags updates` with `--limit <hostname>` to update one host only.

## Quality checks

Check syntax:

```bash
ansible-playbook site.yml --syntax-check
```

Run linting:

```bash
ansible-lint
```

## Inventory groups

- `homelab`: Shared Debian baseline, including SSH hardening and updates.
- `docker_hosts`: Also receives Docker.
- `qemu_hosts`: Also receives the QEMU Guest Agent.

After the first run, SSH access is available by key only. Access filtering is
handled by the upstream OPNsense system.

Automatic security updates may reboot hosts at 02:00 when required.
