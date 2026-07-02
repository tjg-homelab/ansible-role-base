# Ansible Role: base

[![CI](https://github.com/tjg-homelab/ansible-role-base/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-base/actions/workflows/ci.yml)

Baseline OS setup for Debian and RedHat family systems: a full package upgrade,
an automatic reboot when the OS signals one is required (Debian/Ubuntu), and a
configurable set of baseline utility packages.

This role deliberately does **one** job — keeping the box current and equipped.
User accounts live in [`ansible-role-users`](https://github.com/tjg-homelab/ansible-role-users);
hardening is delegated to the [`devsec.hardening`](https://github.com/dev-sec/ansible-collection-hardening)
collection. Run them in that order: **base → users → hardening**.

## Requirements

- Debian 12/13, Ubuntu 22.04/24.04, or Enterprise Linux 9
- Uses only `ansible.builtin` modules — no extra collections

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `base_update_packages` | `true` | Run a full package upgrade (`apt upgrade dist` / `package '*' latest`) |
| `base_reboot_if_required` | `true` | Reboot when `/var/run/reboot-required` exists (Debian/Ubuntu only) |
| `base_install_utility_packages` | `true` | Install the baseline utility set |
| `base_utility_packages_debian` | see `defaults/` | Utility packages for Debian/Ubuntu |
| `base_utility_packages_redhat` | see `defaults/` | Utility packages for RedHat/EL (base-repo only) |
| `base_manage_epel` | `false` | Install the EPEL repo on RedHat/EL |

The RedHat utility default is intentionally limited to base-repo packages so the
role works without EPEL. To pull EPEL-only tools (e.g. `htop`, `glances`), set
`base_manage_epel: true` and add them to `base_utility_packages_redhat`.

## Example Playbook

```yaml
- hosts: all
  roles:
    - role: base
      vars:
        base_reboot_if_required: false
        base_utility_packages_debian:
          - vim
          - git
          - tmux
```

Installing via `requirements.yml`:

```yaml
roles:
  - name: base
    src: https://github.com/tjg-homelab/ansible-role-base.git
    version: v1.0.0
```

## Testing

Molecule (Docker driver) runs against Debian 12/13 and Ubuntu 24.04, exercising
the upgrade path, the utility-package install, and the reboot-toggle logic
(reboots are disabled; the marker must survive). The RedHat path is not run in CI
because a full `dnf` system upgrade would dominate the job.

```bash
pip install ansible-core molecule molecule-plugins[docker] docker
ansible-galaxy collection install community.docker
molecule test
```

## License

MIT

## Author

Rodney Nissen ([The Jira Guy](https://thejiraguy.com)) — Senior Atlassian
Consultant & Jira Architect.
