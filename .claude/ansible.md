# Goals

The goal here is to manage our servers with ansible. We use mainly Debian but also some OpenBSD.
We want to follow the standard ansible tree:

|_ our playbook
|_ inventory
|_ group_vars
 |_ all
 |_ dockerboxes.yml
 |_ webservers.yml
 |_ monitoring.yml
|_ host_vars
|_ roles
 |_ common
  |_ defaults
  |_ files
  |_ handlers
  |_ tasks
   |_ main.yml
   |_ debian.yml
   |_ openbsd.yml
  |_ templates
  |_ vars
   |_ Debian.yml
   |_ OpenBSD.yml
 |_ router
  |_ defaults
  |_ files
   |_ pf
    |_ <hostname>.conf
   |_ relayd
    |_ <hostname>.conf
   |_ iked
    |_ <hostname>.conf
  |_ handlers
  |_ tasks
 |_ docker
 |_ apache
 |_ elasticsearch
 |_ logstash
 |_ kibana
 |_ prometheus
 |_ grafana
 |_ nginx

# Inventory

Hosts are grouped by function, and each group carries the roles/vars it
needs. Known so far:
```yaml
all:
  children:
    dockerboxes:
      hosts:
        dockerbox01.intranet.42crunch.com:
    webservers:
      hosts:
        webserver01.intranet.42crunch.com:
    routers:      # OpenBSD, see `router` role — hosts TBD
      hosts: {}
    monitoring:
      hosts:
        monitoring.intranet.42crunch.com:
```
The playbook applies `common` to `all`, then the group-specific role per
group:
```yaml
- hosts: all
  roles: [common]

- hosts: dockerboxes
  roles: [docker]

- hosts: webservers
  roles: [apache]

- hosts: routers
  roles: [router]

- hosts: monitoring
  roles: [elasticsearch, logstash, kibana, prometheus, grafana, nginx]
```
`common_firewall_rules` overrides for a group's extra ports live in the
matching `group_vars/<group>.yml` (e.g. `group_vars/webservers.yml`
appends 80/443, per the Firewall section above).

# Conventions

Variables defined in a role's `defaults/`/`vars/` (its public interface)
are prefixed with that role's name — `common_firewall_rules`,
`common_admin_group`, `router_interfaces`, etc. — to avoid collisions as
more roles get added to the same play. Inventory-wide values that aren't
owned by a single role (`ssh_allowed_sources`, `admin_users`, set in
`group_vars/all.yml`) stay unprefixed. Apply this to every new role.

# Roles

## common

This role will deploy/setup all common stuffs like:
- install tools (ie. curl, vim, etc.)
- create admin users (sudo/doas, ssh keys included)
- setup the system (ntpd, dns, locale, timezone, etc)

Two groups, two escalation policies:
- `common_admin_group` (`sudo`/`wheel`) — password-required sudo/doas,
  for human admins.
- `common_automation_group` (`ansible`) — passwordless sudo/doas, for
  whichever `admin_users` entries have this group. On Debian, each such
  user gets their own `/etc/sudoers.d/<name>` drop-in
  (`<name> ALL = (ALL) NOPASSWD:ALL`, matching the convention already in
  use — a direct per-user rule, not a `%group` one); on OpenBSD it's the
  second line in `doas.conf`
  (`permit nopass keepenv :{{ common_automation_group }}`, vs.
  `permit persist keepenv :{{ common_admin_group }}` for admins).

`admin_users` (defined in `group_vars/all.yml`) is a list of
`{name, group, ssh_keys, password_hash}` — each user gets their own
personal primary group (the `ansible.builtin.user` default), with
`group` added as a *supplementary* group picking which escalation
policy they get, defaulting to `common_admin_group` if omitted.
`ssh_keys` is a list (one or more keys per user — e.g. a laptop key and
a YubiKey/security-key one) joined with `\n` and passed to
`ansible.posix.authorized_key` in a single call per user, per its
"multiple keys in one batch" convention. `password_hash` is optional
(only meaningful for `common_admin_group` members, since
`common_automation_group` uses passwordless sudo/doas) and passed
straight through to `ansible.builtin.user`'s `password:` — must already
be a `crypt(3)` hash for the target OS (`openssl passwd -6` for
Debian/glibc's sha512crypt, `encrypt(1)` for OpenBSD's bcrypt), never a
plaintext password. Omitting it (or leaving it `""`) leaves the account
locked — `default(omit, true)` in the task treats both "key absent" and
falsy values the same way:
```yaml
admin_users:
  - name: ansible
    group: "{{ common_automation_group }}"
    ssh_keys:
      - "..."
  - name: jp
    group: "{{ common_admin_group }}"
    ssh_keys:
      - "..."
      - "..."
    password_hash: "$6$..."
```
Keys are added, not synced exclusively — removing an entry from
`ssh_keys` doesn't revoke it from a host's `authorized_keys` (the task
doesn't set `exclusive: true`), so a key that's actually compromised
still needs manual removal or a one-off cleanup task. `password_hash`
is a real secret (crackable offline, unlike the public keys in
`ssh_keys`) and belongs in a sops-encrypted
`group_vars/all/secrets.sops.yaml` per the Secrets section below, not
committed in the clear in `group_vars/all.yml`.
- unattended-upgrades (Debian only): installed and enabled, with
  `/etc/apt/apt.conf.d/20auto-upgrades` turning on daily
  update+unattended-upgrade
- baseline firewall: deny by default, allow ssh only (Debian only — the
  only OpenBSD hosts are routers, see `router` role below, which owns
  the firewall entirely on those hosts)

Debian and OpenBSD differ on package manager, privilege escalation, service
management, etc. so `common` handles both via OS-specific files rather than
a separate role per OS:
- `tasks/main.yml` explicitly loads `vars/{{ ansible_os_family }}.yml`
  (via `include_vars` — role vars aren't auto-loaded per OS) then includes
  `tasks/{{ ansible_os_family | lower }}.yml` for the actual per-OS steps.
- `vars/Debian.yml` / `vars/OpenBSD.yml` hold the per-OS values (package
  names, admin group, etc.).

# Firewall

`common` sets a deny-by-default baseline (ssh only, restricted to trusted
source IPs). Other roles extend it rather than override it, driven by one
variable holding a list of rules (not just ports, so a rule can restrict
`src`):
- `roles/common/defaults/main.yml`:
  ```yaml
  common_firewall_rules:
    - port: 22
      src: "{{ ssh_allowed_sources }}"   # e.g. ['203.0.113.0/24']
  ```
  Each rule is `{port, proto (default tcp), src (default "any")}`.
- A role/group needing more ports/rules sets `common_extra_firewall_rules`
  (also `roles/common/defaults/main.yml`, defaults to `[]`) instead of
  redefining `common_firewall_rules`, e.g. `group_vars/webservers.yml`:
  ```yaml
  common_extra_firewall_rules:
    - port: 80
    - port: 443
  ```
  `common`'s normalize task loops
  `common_firewall_rules + common_extra_firewall_rules`. This is a
  separate variable rather than `common_firewall_rules: "{{
  common_firewall_rules + [...] }}"` self-reference — that pattern looks
  like it should work (role defaults are lower precedence than
  group_vars) but doesn't: ansible-core flattens same-named vars to a
  single entry per precedence rules *before* templating, so the group_vars
  definition ends up referencing itself and ansible-core raises
  "Recursive loop detected in template" rather than falling back to the
  role default. Confirmed with a minimal repro
  (`role default x: [1,2]` + `group_vars x: "{{ x + [3] }}"` → recursion
  error; renaming the group_vars one to a distinct var name fixes it).
  Omitting `src` means "any".
- Enforcement (Debian only, via `common`): a generated `nftables.conf`
  (`roles/common/templates/nftables.conf.j2`, rendered from the
  normalized `common_firewall_rules + common_extra_firewall_rules`)
  loaded with `nft -f` via the
  `nftables` systemd service — `table inet filter` with `input` policy
  `drop` (loopback/established/related/icmp always allowed, plus one
  accept rule per normalized `{port, proto, src}`), `forward` policy
  `drop`, `output` policy `accept`. Because the whole ruleset is loaded
  atomically in one `nft -f`, there's no window where the drop policy is
  live without the SSH allow rule, unlike sequential firewall-cli calls.
  A handler (`roles/common/handlers/main.yml`) reloads the service on
  change. OpenBSD hosts are all routers and don't use this abstraction
  at all — see `router` below.
- Docker inserts its own rules (via the `DOCKER-USER` chain, using the
  `iptables-nft` compatibility layer by default) and bypasses this
  baseline by default — worth checking when the `docker` role (below)
  lands, so a container's exposed port doesn't end up reachable
  regardless of `common_firewall_rules`.

## router

All OpenBSD hosts are routers/proxies, so `common` only gives them the
generic baseline (admin users, tools, ntpd); everything network-specific
lives in a dedicated `router` role layered on top:
- Enable IP forwarding (`net.inet.ip.forwarding=1` via `sysctl`,
  persisted in `/etc/sysctl.conf`).
- Configure interfaces (`/etc/hostname.if` per interface).
- Deploy a hand-written `pf.conf` per host
  (`files/pf/{{ inventory_hostname }}.conf`, copied with
  `ansible.builtin.copy`) rather than generating it — router rulesets
  involve NAT/redirection (`nat-to`, `rdr-to`) across multiple
  interfaces, which doesn't fit the generic `common_firewall_rules` list
  model used for Debian.
- A handler reloads on change: `pfctl -f /etc/pf.conf`.
- `relayd` (reverse proxy / load balancing) and `iked` (IKEv2 VPN):
  same pattern as `pf.conf` — hand-written per-host config
  (`files/relayd/{{ inventory_hostname }}.conf`,
  `files/iked/{{ inventory_hostname }}.conf`), copied into place,
  enabled and started via `rcctl enable/start`, with handlers running
  `rcctl reload relayd` / `rcctl reload iked` on change.
- `iked.conf` holds PSKs/private key references — treat it as a secret
  and manage it through the `sops` flow (see Secrets below) rather than
  a plain file, even though `pf.conf`/`relayd.conf` are fine in the
  clear.

## docker

Installs/configures Docker (Debian only), from the official Docker apt
repository (`download.docker.com/linux/debian`, added via
`deb822_repository`) rather than Debian's bundled `docker.io` package —
gets `docker-ce`/`docker-ce-cli`/`containerd.io`/`docker-buildx-plugin`/
`docker-compose-plugin`, tracking upstream releases instead of whatever
Debian happens to package. Extends `common_extra_firewall_rules`
with whatever ports containers need to expose — but see the
`DOCKER-USER` caveat above: allow rules in `common`'s `inet filter`
table alone don't govern published container ports, so this role also
needs to add a `DOCKER-USER` jump rule (or publish containers bound to
specific interfaces/IPs) to actually enforce `common_firewall_rules`
for anything containerized. By default Docker manages its rules through
the `iptables-nft` compatibility layer, which lands them in its own
`ip`/`ip6` family tables rather than `common`'s `inet filter` table —
still one nftables ruleset in the kernel, but a separate table, so
`DOCKER-USER` needs its own jump into (or reference to) `common`'s
table rather than assuming shared rules for free. Docker's newer native
nftables support (`"iptables": false` + the native driver) would avoid
the compat layer entirely — worth revisiting once that's stable enough
to pin.

## hypervisor

Installs `virt-manager`, `cockpit`, and `cockpit-machines` (Debian only)
on the `hypervisors` group — Cockpit's web console (with the
`cockpit-machines` plugin for libvirt VM management) plus the desktop
`virt-manager` GUI. `cockpit.socket` is enabled/started (socket
activation, per the Debian package's own unit) rather than
`cockpit.service` directly.

Cockpit is restricted to `hypervisor_cockpit_listen_address:
hypervisor_cockpit_listen_port` (default `127.0.0.1:9090`,
`roles/hypervisor/defaults/main.yml`) via a systemd socket drop-in
(`/etc/systemd/system/cockpit.socket.d/override.conf`, rendered from
`cockpit-socket-override.conf.j2` — an empty `ListenStream=` first to
clear the package's own `0.0.0.0`/`[::]` default, then the actual
address) rather than a `common_firewall_rules` entry, since the intent
is localhost-only access (tunnel/VPN), not "open to the world but
firewalled to a source list". A handler restarts `cockpit.socket` (with
`daemon_reload`) when the override changes — required because an
already-active socket unit doesn't pick up a new `ListenStream` on
`daemon-reload` alone. No `common_firewall_rules` entry, same posture as
the monitoring stack's backend services below. Note this role doesn't
install/configure `libvirtd` itself — hosts in `hypervisors` are assumed
to already have a working libvirt setup for
`virt-manager`/`cockpit-machines` to manage.

Also deploys `/etc/netplan/51-internal-network.yaml`
(`templates/51-internal-network.yaml.j2`) — the vlan420/vlan422 trunk
interfaces off `eno2np1`, bridged (`42cbr0`/`42cbr1`) over per-vlan VXLAN
tunnels (multicast group `239.1.1.1`/`239.1.2.1`, same across hosts) for
the internal 42Crunch network. The vlan addresses are per-host (each
hypervisor gets a distinct `192.168.42.0/29`-style address on vlan420 and
vlan422), so they're `hypervisor_vlan420_address`/
`hypervisor_vlan422_address`, left empty in `defaults/main.yml` and set
per host in `host_vars/<host>/main.yml` (only `hv02.42crunch.com` is
filled in so far — `hv01`/`hv03` need their addresses added before the
role can run cleanly on them). A handler runs `netplan apply` on change.

`group_vars/hypervisors.yml` sets `common_extra_firewall_rules` with
`4789/udp` (VXLAN) restricted to the two internal-network subnets
(`192.168.42.0/29`, `192.168.42.8/29` — the vlan420/vlan422 networks
above), same pattern as `apache`/`nginx` (see Firewall above). `src`
takes a list, normalized in `common`'s `tasks/debian.yml` and rendered
as one `nft` accept line per address (`nftables.conf.j2` loops
`rule.src`) rather than a single-line `saddr { a, b }` set — same
firewall effect, no template changes needed.

## apache

Installs/configures Apache (Debian only). Appends its ports via
`common_extra_firewall_rules` the same way the earlier `webservers`
example did:
```yaml
common_extra_firewall_rules:
  - port: 80
  - port: 443
```

## monitoring stack

`monitoring.intranet.42crunch.com` (Debian) runs one role per service —
`elasticsearch`, `logstash`, `kibana`, `prometheus`, `grafana`, `nginx` —
same one-role-per-service convention as `docker`/`apache`, all applied to
the `monitoring` group.

Only `nginx` is externally reachable:
- `nginx` reverse-proxies Kibana and Grafana; 80/443 is opened via
  `group_vars/monitoring.yml`'s `common_extra_firewall_rules`, same
  pattern as `apache`.
- `elasticsearch`, `logstash`, `kibana`, `prometheus`, `grafana` all bind
  their listeners to `localhost` and get no `common_firewall_rules` entry
  at all — reachable only through the nginx proxy (or an SSH tunnel for
  ad-hoc access), not directly from the network.

## nginx

Installs/configures nginx (Debian only) and reverse-proxies to Kibana and
Grafana (each on its own `server_name`/path). 80/443 is opened via
`group_vars/monitoring.yml`'s `common_extra_firewall_rules` (see
monitoring stack above), the same way `apache` does — this is the only
monitoring-stack role that touches the firewall.

## elasticsearch / logstash / kibana / prometheus / grafana

Install/configure each service (Debian only), binding to `localhost`
rather than opening a port via `common_firewall_rules` — `nginx` is the
only path in from outside the host.

# Secrets

Secrets (admin SSH keys, passwords, etc.) are encrypted with `sops` via the
`community.sops` collection instead of `ansible-vault`.
- Add `community.sops` to `requirements.yml`.
- `.sops.yaml` at the repo root defines which `age` keys can decrypt which
  paths.
- `community.sops` ships a vars plugin that auto-decrypts
  `host_vars/<host>/*.sops.yaml` and `group_vars/<group>/*.sops.yaml` at
  run time — needs enabling in `ansible.cfg`
  (`[vars_plugins] enable_plugins = community.sops.sops`).
- Control node (and anyone running playbooks) needs the `sops` binary plus
  the relevant `age` private key available locally
  (`SOPS_AGE_KEY_FILE`).
