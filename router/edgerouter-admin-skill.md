---
name: edgerouter-edgeos-network-admin
description: Operational skill for agents diagnosing and safely administering Ubiquiti EdgeRouter / EdgeOS routers. Use for EdgeOS CLI, configuration transactions, interfaces, VLAN-aware switch, DHCP, DNS, firewall, NAT, VPN, DDNS, logs, packet captures, firmware/storage, and recovery workflows. This is not a generic Linux server skill.
version: 1.0.0
---

# EdgeRouter / EdgeOS Network Admin Skill

## 1. Purpose

Use this skill when acting as a network administration agent for a Ubiquiti EdgeRouter running EdgeOS.

This skill teaches the agent how EdgeOS works, which commands to use, where persistent configuration lives, how to make safe transactional changes, and how to diagnose router/network issues without treating the device like a normal Linux server.

This is **not** a personal environment file. Do not hard-code site-specific IPs, credentials, VLANs, interface roles, firewall names, VPN keys, hostnames, or WAN providers in this skill. Discover those from local documentation, inventory, the live configuration, or explicit operator instructions.

Use this skill for:

- EdgeRouter / EdgeOS diagnosis
- interface and VLAN inspection
- DHCP/DNS troubleshooting
- firewall and NAT review
- port forwarding
- dynamic DNS
- VPN tunnels such as OpenVPN, WireGuard, IPsec, or policy-based routing
- system performance and storage checks
- packet captures
- config backup, rollback, and safe change workflows
- post-upgrade scripts and persistent `/config` assets

Do **not** use this skill for UniFi gateways, UniFi OS consoles, Dream Machine, UXG, pfSense, OpenWrt, MikroTik RouterOS, or generic Debian servers unless the operator explicitly says the device is EdgeOS-compatible.

---

## 2. Core mental model

EdgeOS is Vyatta/VyOS-style appliance routing software on small embedded hardware.

Think in layers:

1. **Operational mode:** `show`, `ping`, `traceroute`, `monitor`, logs, interface state.
2. **Configuration mode:** `configure`, `set`, `delete`, `compare`, `commit`, `commit-confirm`, `confirm`, `save`.
3. **Persistent config:** `/config/config.boot`.
4. **Persistent custom assets:** `/config/`, especially `/config/auth/` and `/config/scripts/`.
5. **Underlying Linux:** useful for `tail`, `tcpdump`, `conntrack`, `df`, `top`, but not the source of truth for router config.
6. **Hardware switch layer:** on switch-chip models such as EdgeRouter X, `switch0` and VLAN-aware switch configuration can differ from pure routed-interface models.

The practical rule:

> Observe with operational commands first. Make changes only through EdgeOS configuration mode unless a documented recovery procedure requires otherwise.

---

## 3. EdgeOS is not generic Linux

Do not treat an EdgeRouter like a normal Debian host.

Avoid as first-line actions:

```sh
apt install ...
apt upgrade ...
systemctl ...
journalctl ...
docker ...
snap ...
ufw ...
iptables-save > ...
editing /etc/network/interfaces
editing random files under /etc
```

EdgeOS does contain Linux tools, but router behavior is controlled by the EdgeOS configuration system. Direct Linux changes may be overwritten, may not survive reboot, or may desynchronize the operational state from `config.boot`.

Use Linux commands for diagnostics and temporary inspection, not as the normal configuration path.

---

## 4. Hardware and resource constraints

Many EdgeRouter models, especially EdgeRouter X-class devices, have limited CPU, RAM, and flash. Some are MIPS-based with very small persistent storage.

Rules:

- Do not install packages unless explicitly approved.
- Do not run package upgrades casually.
- Check storage before firmware updates or package work.
- Keep custom scripts and assets under `/config`.
- Do not fill `/config`, `/var`, `/tmp`, or image storage.
- Avoid heavy daemons, monitoring agents, or modern binaries unless known compatible with the target architecture/kernel.
- Do not assume binaries compiled for x86_64 or ARM will run.
- Do not assume modern Go/Rust/static binaries will work on older EdgeOS kernels.

Useful checks:

```sh
show version
show system image
show system image storage
show system storage
df -h
free -m 2>/dev/null || cat /proc/meminfo | sed -n '1,30p'
cat /proc/cpuinfo | sed -n '1,80p'
```

If storage is low, stop before uploading firmware, installing packages, or writing large captures.

---

## 5. Safety model

### 5.1 Read-only by default

Generally safe for diagnosis:

```sh
show version
show system image
show system storage
show system resources
show interfaces
show interfaces detail
show configuration
show configuration commands
show firewall
show nat
show dhcp leases
show arp
show ip route
show dns forwarding statistics
show log
show conntrack table ipv4
```

Linux read-only diagnostics are also generally safe:

```sh
ip addr
ip route
df -h
top -b -n1
tail -n 200 /var/log/messages
sudo conntrack -L 2>/dev/null
sudo tcpdump -i <interface> -n -c 50 <filter>
```

### 5.2 Requires explicit approval

Do not do these without explicit approval:

- enter config mode and commit changes
- restart router services
- reboot
- upgrade firmware
- delete old system images
- change WAN/LAN/VLAN/firewall/NAT/DHCP/DNS/VPN settings
- change SSH/Web UI management settings
- run long packet captures
- install or remove packages
- change scripts in `/config/scripts`
- change files in `/config/auth`
- edit `/config/config.boot` directly
- change hardware offload settings
- factory reset
- clear conntrack table
- delete DHCP leases or DNS cache when it may affect users

### 5.3 High-risk changes

Require explicit approval plus rollback plan and maintenance window:

- WAN interface changes
- default route changes
- firewall rules on management/WAN/local access
- VLAN-aware switch changes
- DHCP scope changes on active LANs
- deleting or moving `switch0` or VIF interfaces
- VPN policy-based routing or kill-switch changes
- EdgeOS firmware upgrade/downgrade
- loading/replacing full configuration
- restoring config from backup
- changing admin users/SSH keys
- any change that could lock out the agent

### 5.4 Hard stops

Stop and report if:

- target router identity is unclear
- model/firmware cannot be identified
- config mode is already locked by another session
- storage is nearly full
- route/firewall change could cut off SSH
- requested change affects the interface currently used for management
- `compare` shows unexpected changes
- `commit`/`commit-confirm` returns errors
- the router is already under high CPU/memory pressure
- the command set differs from this skill
- firmware version/model behavior is unknown
- backup status is unknown before a high-risk change

---

## 6. EdgeOS modes and transactional config

EdgeOS has two major CLI modes.

### 6.1 Operational mode

Default mode after login. Commands typically start with `show`, `ping`, `traceroute`, `monitor`, `reset`, etc.

Examples:

```sh
show version
show interfaces
show configuration commands
show log
```

### 6.2 Configuration mode

Enter with:

```sh
configure
```

Inside configuration mode, use:

```sh
set ...
delete ...
compare
commit
commit-confirm <minutes>
confirm
save
discard
exit
```

The configuration lifecycle:

| State | Meaning |
|---|---|
| Working | Pending changes staged in config mode |
| Active/running | Applied configuration currently used by the router |
| Boot/persistent | Saved config in `/config/config.boot` used after reboot |

Important distinction:

- `commit` applies working changes to active running config.
- `save` writes active/running config to `/config/config.boot`.
- Rebooting after `commit` but before `save` usually reverts to last saved boot config.
- `discard` clears uncommitted working changes.
- `compare` shows pending changes before commit.

---

## 7. Remote change safety: commit-confirm

For remote work, especially over SSH, prefer:

```sh
commit-confirm 5
```

This applies the change but starts a rollback timer. If the agent loses access and does not confirm in time, EdgeOS rolls back.

Safe transaction pattern:

```sh
configure
# set/delete commands here
compare
commit-confirm 5
# validate connectivity and behavior from the agent's perspective
confirm
save
exit
```

If validation fails:

```sh
# Do not confirm. Let rollback occur, or explicitly rollback if safe.
exit discard
```

Do not use plain `commit` for network/firewall/routing changes that can lock out remote access.

Do not press Ctrl+C during `commit`, `commit-confirm`, `confirm`, `rollback`, or `save`. Interrupting config operations can leave the configuration system in a bad state.

---

## 8. Backups, archives, and rollback

### 8.1 Export current config

Before any meaningful change, collect:

```sh
show configuration commands
show configuration
cat /config/config.boot
```

To copy from router:

```sh
scp <user>@<router>:/config/config.boot ./config.boot.<router>.$(date +%Y%m%d-%H%M%S)
```

### 8.2 Enable commit archive if not already enabled

Check:

```sh
show configuration commands | match commit-revisions
show system commit
```

If operator approves, enable a commit history depth:

```sh
configure
set system config-management commit-revisions 20
commit-confirm 5
confirm
save
exit
```

### 8.3 View and rollback commits

```sh
show system commit
show system commit diff <revision>
```

In config mode:

```sh
configure
rollback <revision>
compare
commit-confirm 5
# validate
confirm
save
exit
```

### 8.4 Direct config.boot editing

Avoid direct edits to `/config/config.boot`. Prefer CLI `set`/`delete`.

Only edit `config.boot` directly for documented recovery workflows, with backup and operator approval.

---

## 9. Standard read-only preflight

Run before diagnosis or proposed changes.

```sh
echo "===== EDGEOS PREFLIGHT ====="
date
hostname
whoami
uptime

echo "===== VERSION / MODEL ====="
show version
show system image

echo "===== STORAGE ====="
show system storage
show system image storage
df -h

echo "===== RESOURCES ====="
show system resources
free -m 2>/dev/null || cat /proc/meminfo | sed -n '1,30p'

echo "===== INTERFACES ====="
show interfaces
show interfaces detail

echo "===== ROUTING ====="
show ip route
ip route 2>/dev/null || true

echo "===== FIREWALL / NAT SUMMARY ====="
show firewall
show nat

echo "===== DHCP / ARP ====="
show dhcp leases
show arp

echo "===== LOGS ====="
show log | tail -n 120
tail -n 200 /var/log/messages 2>/dev/null
```

Interpretation checklist:

- Confirm model and firmware.
- Confirm active/default boot image.
- Confirm storage headroom.
- Identify WAN, LAN, switch, VIF, VPN, and management interfaces.
- Identify default route.
- Check if DHCP leases match expected clients/subnets.
- Check logs for link flaps, DHCP failures, VPN failures, firewall drops, PPPoE/DHCP WAN issues.
- Check whether current configuration uses `switch0`, routed `eth*`, VLAN-aware switch, or zone-based firewall.

---

## 10. Interface diagnostics

### 10.1 Show interface state

```sh
show interfaces
show interfaces detail
ip addr
ip -s link
```

Look for:

- `up/down` vs `up/up`
- missing IP addresses
- RX/TX errors
- link flaps
- wrong MTU
- wrong VLAN interface
- tunnel interface down
- PPPoE/DHCP WAN not receiving address

### 10.2 Logs for link state

```sh
grep -i "link" /var/log/messages | tail -n 100
grep -i "Link Status Changed" /var/log/messages | tail -n 100
```

### 10.3 Interface traffic

```sh
show interfaces detail
sudo tcpdump -i <interface> -n -c 50
```

Do not run open-ended `tcpdump` in an agent session. Use `-c` or timeout and avoid capturing sensitive traffic unless necessary.

---

## 11. VLAN-aware switch and EdgeRouter X-style switch0

Some EdgeRouter models have a hardware switch chip represented by `switch0`.

Important rule:

> When `switch0` is VLAN-aware and VLANs are implemented as `switch0.<vlan_id>` VIFs, avoid assigning an IP directly to base `switch0` unless the design explicitly requires it and the model/firmware supports that design.

Discover current design:

```sh
show configuration commands | match "interfaces switch switch0"
show interfaces switch switch0
show interfaces switch switch0 vif
show interfaces ethernet
```

Common patterns:

```sh
set interfaces switch switch0 switch-port vlan-aware enable
set interfaces switch switch0 vif <vlan_id> address <gateway_ip>/<cidr>
set interfaces switch switch0 switch-port interface eth1 vlan pvid <native_vlan>
set interfaces switch switch0 switch-port interface eth1 vlan vid <tagged_vlan>
```

### 11.1 Trunk/access interpretation

For VLAN-aware switch ports:

- `pvid` = untagged/native VLAN for ingress untagged traffic
- `vid` = tagged VLANs allowed on the port
- VIF `switch0.<vlan>` = router L3 gateway for that VLAN
- access port usually has only `pvid`
- trunk port usually has one `pvid` plus one or more `vid`

### 11.2 Safe VLAN change workflow

Before changes:

```sh
show configuration commands | match "interfaces switch switch0"
show configuration commands | match "service dhcp-server"
show configuration commands | match "firewall"
show interfaces detail
```

For any switch/VLAN change:

1. Identify management path.
2. Confirm which VLAN/interface carries SSH.
3. Avoid changing that path remotely unless there is out-of-band access.
4. Use `commit-confirm`.
5. Validate from a client on each affected VLAN before `confirm`.
6. Save only after validation.

---

## 12. DHCP diagnostics

```sh
show dhcp leases
show dhcp server statistics
show configuration commands | match "service dhcp-server"
grep -i dhcp /var/log/messages | tail -n 150
```

Common issues:

- DHCP scope not tied to correct subnet/interface
- default-router mismatch
- DNS server mismatch
- overlapping ranges
- exhausted pool
- client on wrong VLAN
- firewall blocking DHCP relay/forwarding
- stale lease expectations
- service not running after config change

For DHCP packet check:

```sh
sudo tcpdump -i <lan_or_vlan_interface> -n "port 67 or port 68" -vvv -c 50
```

Do not change DHCP scopes without explicit approval; this can affect active clients.

---

## 13. DNS forwarding diagnostics

EdgeOS commonly uses DNS forwarding (`dnsmasq`) for local clients.

Commands:

```sh
show dns forwarding statistics
show configuration commands | match "service dns forwarding"
grep -i dnsmasq /var/log/messages | tail -n 150
```

Check:

- listening interfaces
- cache size
- system name servers
- forwarding options
- local host mappings
- DHCP DNS handed to clients

Common config patterns:

```sh
set service dns forwarding cache-size 10000
set service dns forwarding listen-on switch0.<vlan_id>
set system name-server <resolver_ip>
```

Do not change DNS forwarding or upstream resolvers without considering DHCP clients and split-DNS/VPN behavior.

---

## 14. Routing diagnostics

```sh
show ip route
show ip route forwarding
ip route
show configuration commands | match "protocols static"
show configuration commands | match "firewall modify"
```

For connectivity:

```sh
ping <ip> count 4
traceroute <ip>
```

From config mode, operational commands are usually prefixed with `run`:

```sh
run ping 1.1.1.1 count 4
run show ip route
```

Check:

- default route
- WAN DHCP/PPPoE route
- static routes
- policy-based routing
- VPN route tables
- blackhole fail-safe routes
- asymmetric routing
- missing route for return traffic

---

## 15. Firewall model

EdgeOS firewall rules are directional and applied to interfaces.

Common directions:

| Direction | Meaning |
|---|---|
| `in` | traffic entering an interface and being routed through the router |
| `local` | traffic destined to the router itself |
| `out` | traffic leaving an interface |

Common rulesets:

- `WAN_IN`: traffic from WAN to internal networks
- `WAN_LOCAL`: traffic from WAN to the router itself
- VLAN-specific `*_IN` or `*_LOCAL`
- zone-based firewall, if configured
- modify policies for policy-based routing

Inspect:

```sh
show firewall
show configuration commands | match "firewall"
show configuration commands | match "interfaces .* firewall"
```

Good stateful ruleset pattern:

1. Accept established/related.
2. Drop invalid.
3. Specific allows/drops.
4. Default action explicit and understood.

Example:

```sh
set firewall name WAN_IN rule 10 action accept
set firewall name WAN_IN rule 10 state established enable
set firewall name WAN_IN rule 10 state related enable
set firewall name WAN_IN rule 20 action drop
set firewall name WAN_IN rule 20 state invalid enable
```

Do not reorder or replace firewall rules casually. Rule order matters.

---

## 16. NAT and port forwarding

EdgeOS NAT is separate from firewall rules.

Inspect:

```sh
show nat
show configuration commands | match "service nat"
show configuration commands | match "port-forward"
```

Important NAT/firewall rule:

> Destination NAT happens before the routed firewall check. Matching firewall rules for forwarded traffic often need to match the post-translated internal destination IP and internal port, not the original public WAN IP/port.

### 16.1 DNAT pattern

```sh
configure
set service nat rule <rule_id> description "Forward WAN <external_port> to <internal_ip>:<internal_port>"
set service nat rule <rule_id> type destination
set service nat rule <rule_id> inbound-interface <wan_interface>
set service nat rule <rule_id> protocol tcp
set service nat rule <rule_id> destination port <external_port>
set service nat rule <rule_id> inside-address address <internal_ip>
set service nat rule <rule_id> inside-address port <internal_port>
compare
commit-confirm 5
# validate
confirm
save
exit
```

### 16.2 Matching WAN_IN rule

```sh
configure
set firewall name WAN_IN rule <firewall_rule_id> action accept
set firewall name WAN_IN rule <firewall_rule_id> description "Allow forwarded traffic to <internal_ip>:<internal_port>"
set firewall name WAN_IN rule <firewall_rule_id> protocol tcp
set firewall name WAN_IN rule <firewall_rule_id> destination address <internal_ip>
set firewall name WAN_IN rule <firewall_rule_id> destination port <internal_port>
compare
commit-confirm 5
# validate
confirm
save
exit
```

### 16.3 Masquerade/SNAT

Outbound NAT pattern:

```sh
set service nat rule 5010 description "Masquerade outbound internet"
set service nat rule 5010 outbound-interface <wan_interface>
set service nat rule 5010 type masquerade
set service nat rule 5010 protocol all
```

Be careful with rule IDs. Preserve local conventions. Do not overwrite existing NAT rules.

---

## 17. Conntrack diagnostics

EdgeOS uses Linux conntrack/netfilter underneath.

Commands:

```sh
show conntrack table ipv4
sudo conntrack -L 2>/dev/null | head -n 100
sudo conntrack -L -d <destination_ip> 2>/dev/null
sudo conntrack -L -s <source_ip> 2>/dev/null
```

Use conntrack to diagnose:

- whether flows are reaching the router
- NAT translations
- stale states
- too many connections
- VPN/PBR flow direction

Do not flush conntrack globally unless approved. It can disrupt active traffic.

---

## 18. Packet capture

Use bounded captures.

Examples:

```sh
sudo tcpdump -i <interface> -n -c 50
sudo tcpdump -i <interface> -n "host <ip>" -c 100
sudo tcpdump -i <interface> -n "port 67 or port 68" -vvv -c 50
sudo tcpdump -i <wan_interface> -n "udp port 500 or udp port 4500" -c 100
sudo tcpdump -i <interface> -w /tmp/capture.pcap -c 200
```

Rules:

- Use `-c` or external timeout.
- Use filters to minimize data.
- Avoid capturing passwords or sensitive traffic unless necessary.
- Store captures in `/tmp` unless there is an approved reason to write to persistent storage.
- Delete temporary captures after retrieval if approved.

---

## 19. Dynamic DNS

EdgeOS dynamic DNS uses configured services and ddclient-style behavior.

Inspect:

```sh
show dns dynamic status
show configuration commands | match "service dns dynamic"
grep -i ddclient /var/log/messages | tail -n 100
```

Force update:

```sh
update dns dynamic interface <wan_interface>
```

Double-NAT caveat:

- If WAN receives a private address, interface-based DDNS may publish the wrong IP.
- Use provider/web-check configuration only if supported and approved.
- Do not print API tokens/passwords.

Configuration changes are approval-only.

---

## 20. VPN diagnostics

### 20.1 OpenVPN client/server

Common interface names:

```text
vtun0
vtun1
```

Inspect:

```sh
show interfaces openvpn
show interfaces openvpn vtun0
show configuration commands | match "interfaces openvpn"
grep -i openvpn /var/log/messages | tail -n 200
```

Persistent auth/profile files are often stored under:

```text
/config/auth/
```

Rules:

- Do not print credential files.
- Do not overwrite `.ovpn`, certs, keys, or `pass.txt` without approval.
- Use `/config/auth` for persistent VPN files.
- Validate routes and policy-based routing after tunnel changes.

### 20.2 WireGuard

Common interface:

```text
wg0
```

Inspect:

```sh
show interfaces wireguard
show interfaces wireguard wg0
show configuration commands | match "interfaces wireguard"
sudo wg show 2>/dev/null
grep -i wireguard /var/log/messages | tail -n 150
```

Rules:

- Do not print private keys.
- Do not change peer allowed-ips without route/firewall review.
- Ensure WAN_LOCAL allows UDP handshake port if server is inbound.
- Ensure WAN_IN or internal firewall permits routed VPN client traffic only as intended.

### 20.3 Policy-based routing

Inspect:

```sh
show configuration commands | match "firewall modify"
show configuration commands | match "protocols static table"
show ip route table all 2>/dev/null || ip route show table all
```

Common PBR pattern:

```sh
set protocols static table <table_id> interface-route 0.0.0.0/0 next-hop-interface vtun0
set protocols static table <table_id> route 0.0.0.0/0 blackhole distance 255
set firewall modify <policy_name> rule 10 action modify
set firewall modify <policy_name> rule 10 source address <subnet>/<cidr>
set firewall modify <policy_name> rule 10 modify table <table_id>
set interfaces switch switch0 vif <vlan_id> firewall in modify <policy_name>
```

Do not change PBR without validating leak/fail-closed behavior.

---

## 21. WAN troubleshooting playbook

Collect:

```sh
show interfaces detail
show ip route
show dhcp client leases 2>/dev/null || true
show configuration commands | match "interfaces ethernet"
show configuration commands | match "service nat"
show configuration commands | match "system name-server"
show log | tail -n 150
grep -iE "dhcp|pppoe|wan|link|route|dns" /var/log/messages | tail -n 200
```

Check:

- WAN interface link state
- DHCP/PPPoE lease
- public/private WAN address
- default route
- NAT masquerade rule
- DNS resolution
- ISP modem/gateway state
- double NAT
- VLAN tag required by ISP
- MTU/MSS issues
- firewall local rules blocking DHCP/PPPoE

Safe tests:

```sh
ping 1.1.1.1 count 4
ping google.com count 4
traceroute 1.1.1.1
```

---

## 22. LAN/VLAN troubleshooting playbook

Collect:

```sh
show interfaces detail
show configuration commands | match "interfaces switch"
show configuration commands | match "vif"
show configuration commands | match "service dhcp-server"
show dhcp leases
show arp
grep -iE "dhcp|link|switch|vlan" /var/log/messages | tail -n 200
```

Check:

- client is on expected VLAN
- access/trunk port PVID/VID
- DHCP scope matches VLAN subnet
- gateway IP exists on correct VIF
- firewall on VIF blocks traffic
- switch/AP trunk tagging matches EdgeRouter
- client ARP appears on expected subnet

Use DHCP capture when needed:

```sh
sudo tcpdump -i switch0.<vlan_id> -n "port 67 or port 68" -vvv -c 50
```

---

## 23. Port forwarding troubleshooting playbook

Collect:

```sh
show nat
show firewall
show configuration commands | match "service nat"
show configuration commands | match "firewall name WAN_IN"
show interfaces detail
show ip route
```

Check:

- inbound-interface matches actual WAN interface
- DNAT rule destination port/protocol correct
- inside-address IP reachable from router
- firewall WAN_IN rule allows post-translated internal IP/port
- internal host firewall allows connection
- ISP blocks port
- double NAT upstream
- hairpin NAT if testing from inside LAN
- service actually listening on internal host

Packet capture:

```sh
sudo tcpdump -i <wan_interface> -n "port <external_port>" -c 50
sudo tcpdump -i <lan_interface_or_vif> -n "host <internal_ip> and port <internal_port>" -c 50
```

---

## 24. Performance troubleshooting playbook

Collect:

```sh
show system resources
show system storage
show interfaces detail
show conntrack table ipv4 | wc -l
top -b -n1
grep -iE "oom|memory|cpu|conntrack|nf_conntrack|drop" /var/log/messages | tail -n 200
```

Check:

- high CPU from routing/VPN/tcpdump/logging
- conntrack exhaustion
- packet drops/errors on interfaces
- flash/storage full
- excessive firewall logging
- VPN encryption CPU bottleneck
- hardware offload disabled or incompatible with features
- MTU/MSS problems

Do not enable/disable offload without understanding feature compatibility and impact.

---

## 25. Firmware and image management

Read-only:

```sh
show version
show system image
show system image storage
show system storage
```

Rules:

- Firmware updates are high-risk.
- Confirm model, current version, target version, release notes, backup, power stability, and local recovery path.
- Check image storage before upload.
- Do not delete old images without approval.
- Preserve a known-good fallback image when possible.

Typical commands vary by version/model and should not be improvised. Prefer operator-approved documented process or Web UI.

---

## 26. Persistent scripts and custom files

Persistent custom assets should live under `/config`.

Important directories:

| Path | Purpose |
|---|---|
| `/config/config.boot` | persistent boot configuration |
| `/config/auth/` | VPN certs, keys, auth files, WireGuard keys |
| `/config/scripts/post-config.d/` | scripts run after config is applied at boot |
| `/config/scripts/pre-config.d/` | scripts run before config application, if supported |
| `/config/scripts/firstboot.d/` | scripts that may run after upgrade/first boot, model/version dependent |
| `/config/archive/` | config commit archives if enabled |

Rules:

- Scripts must be small, auditable, and executable.
- Do not print secrets from `/config/auth`.
- Use `vbash` and Vyatta wrappers for scripts that change config.
- Test scripts manually before relying on boot hooks.
- Use absolute paths.

Example script header:

```sh
#!/bin/vbash
source /opt/vyatta/etc/functions/script-template

RUN_OP="/opt/vyatta/bin/vyatta-op-cmd-wrapper"
RUN_CFG="/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper"
```

Do not add self-healing scripts that repeatedly commit changes without strong guardrails.

---

## 27. Private web API

EdgeOS has private web/API endpoints used by the GUI. These are not the preferred interface for general agent administration.

Rules:

- Prefer SSH CLI.
- Use API read-only only when CLI is insufficient and the operator approves.
- Do not log cookies, CSRF tokens, passwords, or session IDs.
- Do not push config changes through private batch APIs unless there is a tested site-specific workflow.
- API behavior may differ by EdgeOS version.

---

## 28. Change proposal format

Before any config change:

```markdown
## Proposed EdgeRouter Change

Target:
Model / EdgeOS version:
Current access path:
Observed problem:
Proposed change:
Commands to run:
Risk:
Could this affect SSH/management access:
Rollback method:
commit-confirm timer:
Validation tests before confirm:
Save to boot config after validation:
Maintenance window needed:
Requires approval: yes
```

After execution:

```markdown
## EdgeRouter Change Result

Target:
Change performed:
Commands run:
compare output summary:
commit-confirm used:
Validation performed:
Confirmed:
Saved:
Remaining concerns:
Rollback needed:
```

---

## 29. Diagnostic report format

```markdown
# EdgeRouter Diagnostic Report

## Summary
- Overall state:
- Primary issue:
- Confidence:
- Immediate action needed:

## Identity
- Hostname:
- Model:
- EdgeOS version:
- Active/default image:
- Uptime:

## Resources
- CPU/load:
- Memory:
- Storage:
- Image storage:

## Interfaces
- WAN:
- LAN/switch:
- VLAN/VIFs:
- VPN/tunnels:
- Link errors/flaps:

## Routing
- Default route:
- Static routes:
- Policy-based routing:
- VPN routes:

## DHCP/DNS
- DHCP server status:
- Lease observations:
- DNS forwarding:

## Firewall/NAT
- WAN_LOCAL:
- WAN_IN:
- Inter-VLAN rules:
- NAT/masquerade:
- Port forwards:

## Logs
- Link events:
- DHCP/DNS events:
- Firewall drops:
- VPN events:
- System errors:

## Recommended next steps
1.
2.
3.

## Actions intentionally not taken
-
```

---

## 30. First response pattern

When asked to diagnose an EdgeRouter:

```text
I’ll treat this as an EdgeOS router diagnosis, not a generic Linux task. I’ll start read-only: identify model/version/images, storage, resources, interfaces, routing, DHCP/DNS, firewall/NAT, VPN state, and recent logs. I won’t enter configuration mode, commit changes, restart services, reboot, update firmware, or modify firewall/VLAN/WAN settings unless you approve the specific change.
```

When asked to make a change:

```text
I’ll first inspect the current EdgeOS configuration and management path, then propose the exact set/delete commands. If approved, I’ll use configure → compare → commit-confirm → validate → confirm → save, so a failed remote change can roll back instead of locking us out.
```

---

## 31. What not to do

Never do these as first-line fixes:

```sh
reboot
poweroff
commit
save
delete interfaces ...
delete firewall ...
delete service nat ...
rm /config/config.boot
rm -rf /config/auth
rm -rf /config/scripts
apt upgrade
apt dist-upgrade
iptables -F
conntrack -F
tcpdump -i any -w /config/large.pcap
```

Never assume:

- WAN is `eth0`
- LAN is `switch0`
- management is untagged
- firewall names are `WAN_IN` / `WAN_LOCAL`
- NAT rule IDs are unused
- VLAN-aware switch is enabled
- EdgeRouter model has a switch chip
- WireGuard exists on every EdgeOS version
- config changes are persistent before `save`
- `commit` is safe over remote SSH
- direct Linux edits survive reboot

---

## 32. Success criteria

A good EdgeRouter network admin agent:

- distinguishes operational mode from configuration mode
- uses `commit-confirm` for risky remote changes
- runs `compare` before committing
- validates before `confirm`
- saves only after validation
- preserves `/config/config.boot`
- understands `switch0`, VIFs, PVID/VID, and VLAN-aware switching
- understands firewall direction and NAT-before-firewall behavior
- diagnoses with `show` commands before Linux internals
- uses bounded packet captures
- avoids installing packages on constrained routers
- protects secrets under `/config/auth`
- reports uncertainty clearly
- refuses to improvise changes that could lock out management access
