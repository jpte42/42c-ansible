# 42c-ansible

Ansible playbooks for managing 42Crunch's servers — mainly Debian, with a
few OpenBSD routers.

## Requirements

- `ansible-core` (tested with 2.21)
- Collections listed in `requirements.yml`: `community.sops`,
  `community.general`, `ansible.posix`
- The `sops` binary and an `age` private key, for decrypting secrets (see
  [Secrets](#secrets) below)

Install the collections:

```sh
ansible-galaxy collection install -r requirements.yml
```

## Layout

```
site.yml           # top-level playbook
inventory/          # hosts, grouped by function
group_vars/          # per-group variables (all/, webservers.yml, ...)
host_vars/           # per-host variables
roles/               # one role per service/purpose
.claude/ansible.md   # detailed design doc — architecture, conventions,
                      # and the reasoning behind each role; keep in sync
                      # with the code when either changes
```

Hosts are grouped by function in `inventory/hosts.yml`
(`hypervisors`, `dockerboxes`, `webservers`, `routers`, `monitoring`).
`site.yml` applies the `common` role to every host, then a group-specific
role (or roles) per group.

## Usage

Run the full playbook:

```sh
ansible-playbook site.yml
```

Target a single host or group:

```sh
ansible-playbook site.yml --limit hv02.42crunch.com
ansible-playbook site.yml --tags hypervisor
```

Check syntax without touching any host:

```sh
ansible-playbook site.yml --syntax-check
```

## Secrets

Secrets (admin SSH keys, password hashes, etc.) are encrypted with
[`sops`](https://github.com/getsops/sops) via the `community.sops`
collection rather than `ansible-vault`. Encrypted files live at
`group_vars/<group>/*.sops.yaml` and `host_vars/<host>/*.sops.yaml` and
are decrypted automatically at run time by the `community.sops.sops` vars
plugin (enabled in `ansible.cfg`). To run playbooks or edit these files
you need:

- the `sops` binary
- the `age` private key referenced in `.sops.yaml`, available locally
  (typically via `SOPS_AGE_KEY_FILE`)

## Roles

| Role | Purpose |
| --- | --- |
| `common` | Baseline for every host: packages, admin users, unattended-upgrades, firewall (Debian), applied to `all` |
| `hypervisor` | `virt-manager`/Cockpit for the `hypervisors` group, plus the internal-network netplan config |
| `docker` | Docker CE for the `dockerboxes` group |
| `apache` | Apache for the `webservers` group |
| `router` | pf/relayd/iked for the OpenBSD `routers` group |
| `elasticsearch`, `logstash`, `kibana`, `prometheus`, `grafana`, `nginx` | The monitoring stack, applied to the `monitoring` group (`nginx` reverse-proxies Kibana/Grafana and is the only one exposed externally) |

See `.claude/ansible.md` for the full design rationale — variable naming
conventions, the firewall model, per-role details, and the secrets flow.
