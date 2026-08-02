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
| `base_timezone` | `""` | System timezone, e.g. `Etc/UTC` (Debian/Ubuntu). Empty leaves the host alone. Set it on **fresh** hosts — see below |
| `base_assert_package_db_sane` | `true` | Fail the play if `dpkg` reports half-configured packages after the package run |
| `base_apt_environment` | `DEBIAN_FRONTEND=noninteractive`, `DEBIAN_PRIORITY=critical` | Environment applied to every Debian/Ubuntu package operation |

### Fresh hosts and the tzdata trap

On a **freshly imaged** Debian/Ubuntu host, `apt-get dist-upgrade` can die with:

```
tzdata failed to preconfigure, with exit status 10
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

The damage is wider than one package: `tzdata` lands in state `iF` and
everything queued behind it is left `iU` (unpacked, never configured) — on a
real occurrence that included `python3`, on the host Ansible was managing.

Note that `ansible.builtin.apt` *already* exports
`DEBIAN_FRONTEND=noninteractive` for the commands it runs, so "set the frontend"
is not by itself the fix. This role therefore:

1. audits the package database and runs `dpkg --configure -a` **before**
   upgrading, healing a host that arrives mid-configure;
2. sets the timezone when `base_timezone` is non-empty, so `tzdata` has no
   debconf question to ask at all;
3. applies `base_apt_environment` to every package operation, which also covers
   dpkg hook child processes;
4. **fails loudly** afterwards if the database is still inconsistent, rather
   than converging another 150 tasks onto a broken host.

`base_timezone` is empty by default on purpose — a fleet is frequently mixed
(UTC on servers, local time on desktops/SBCs), and a default would silently
re-timezone hosts on the next converge.

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
