# QNAP QTS NAS Operations Skill for Agents

**Skill type:** Operational field guide / platform-specific diagnostic reference  
**Use case:** Help an agent that already has SSH access and local environment context diagnose and safely work on a QNAP NAS running QTS 5.x.  
**Not a standalone agent profile:** This skill does not define the agent’s identity, permissions, host inventory, credential sources, notification routes, or user-specific environment. Those belong in the agent’s normal instructions, repo docs, inventory files, or credential broker.

---

## 1. When to use this skill

Use this skill whenever the task involves a QNAP NAS, QTS, QNAP apps, QPKGs, Storage & Snapshots, Container Station, QVPN, SMB/NFS shares, QNAP cron, QNAP logs, or NAS hardware health.

The agent is expected to already know or discover the target NAS from its normal context, such as:

- local infrastructure documents
- SSH config
- host inventory
- network map
- agent memory
- repository notes such as `docs/servers/nas.md`
- prior approved operator instructions

Do **not** treat this skill as a separate tool that receives host/IP/user parameters. It is guidance for how to behave once the agent has decided it needs to inspect or work on a QNAP system.

If the target host is ambiguous, search the local docs/inventory first. Ask the operator only when the docs conflict or the action would be risky.

---

## 2. Core mental model

QNAP QTS is **appliance Linux**, not a normal Linux distribution.

It has a Linux kernel and shell, but most administration is mediated through QNAP’s own configuration files, QPKG packages, init scripts, Web UI, and proprietary helper commands.

Assume these facts unless verified otherwise on the target:

| Area | QNAP/QTS behavior |
|---|---|
| Init system | QNAP init scripts, usually under `/etc/init.d`; no `systemd` |
| Shell | BusyBox/POSIX-flavored `sh`; do not assume full Bash |
| Package model | QPKGs, not OS packages; tracked mainly in `/etc/config/qpkg.conf` |
| Main persistent config | `/etc/config/` |
| Shares/data volumes | usually under `/share/CACHEDEV*_DATA/` |
| Cron | persistent file is `/etc/config/crontab` |
| Storage management | QNAP Storage & Snapshots, LVM/mdraid/QNAP tools underneath |
| Network management | QNAP `netmgr` / virtual switch stack; not NetworkManager/networkd |
| Containers | Container Station, with Docker/LXC binaries often outside `PATH` |
| Web UI | QTS admin UI, commonly HTTPS 443 and HTTP 8080 |
| Preferred admin interface | Web UI for high-risk storage, network, permissions, firmware, and app changes |

The practical rule: **observe with CLI, change cautiously, and prefer QTS Web UI for high-risk administration unless a QNAP-specific CLI procedure is known.**

---

## 3. Safety stance

The agent may use this skill to decide what to check and how to interpret results. It must not use this skill as blanket permission to modify the NAS.

### Safe by default: read-only diagnostics

Generally allowed when the agent has been asked to diagnose or inspect the NAS:

- identify QTS version, build, model, kernel, uptime
- inspect CPU, RAM, load, disk usage, mounts
- inspect RAID/md state
- inspect QPKG inventory
- inspect network interfaces, bridges, routes, listeners
- inspect SMB/NFS state
- inspect Container Station install path and read-only container lists
- inspect cron entries
- inspect logs
- inspect UPS and backup-app presence
- inspect thermal and fan state

### Requires explicit operator approval

Do not perform these just because the agent thinks they are probably helpful:

- reboot or shutdown
- restart QTS services
- restart SSH, SMB, NFS, QVPN, Container Station, or QTS Web UI
- restart, remove, prune, rebuild, or modify containers
- edit `/etc/config/*`
- change users, groups, ACLs, shares, quotas, or AD/domain membership
- change network, bridge, DNS, gateway, WireGuard, or virtual switch settings
- run performance benchmarks or disk tests that create load
- change fan modes
- install, remove, enable, disable, or update QPKGs
- update firmware
- alter storage pools, RAID groups, snapshots, LUNs, or volumes

### Hard stop conditions

Stop and report before making changes if any are observed:

- cannot verify the host is the intended NAS
- QTS version/build cannot be identified
- RAID is degraded/rebuilding/unclean
- any relevant filesystem is read-only
- any relevant volume is above 95% full
- recent kernel logs show I/O errors
- command output mentions `degraded`, `rebuilding`, `unclean`, `read-only`, `no space left`, `permission denied`, or similar
- the requested action touches networking and SSH is using that same network path
- recent backup status is unknown and the requested action could affect data
- a QNAP-specific command/config path differs from this skill

---

## 4. Command style on QTS

Use POSIX-compatible shell. Avoid Bash-specific syntax.

Prefer:

```sh
date
hostname
uname -a
uptime
command -v qcli_storage >/dev/null 2>&1 && qcli_storage -d || echo "qcli_storage not found"
```

Avoid unless specifically verified:

```sh
systemctl ...
journalctl ...
apt ...
yum ...
dnf ...
[[ ... ]]
source ...
mapfile ...
set -o pipefail
docker system prune
```

Use command discovery before using optional tools:

```sh
command -v getsysinfo >/dev/null 2>&1 && getsysinfo cputmp || echo "getsysinfo not found"
```

For agent logging, capture:

- command
- timestamp
- exit code
- stdout
- stderr
- short interpretation

Never log credentials, SSH keys, cookies, API session IDs, MFA codes, or recovery codes.

---

## 5. QNAP filesystem and config map

Important locations:

| Path | Purpose / notes |
|---|---|
| `/etc/config/uLinux.conf` | QTS system identity, version/build/model info |
| `/etc/config/qpkg.conf` | QPKG package registry: install paths, versions, enabled flags |
| `/etc/config/crontab` | Persistent QNAP cron source |
| `/etc/config/nfssetting` | QNAP NFS export configuration source |
| `/etc/exports` | Generated NFS exports; do not treat as persistent source |
| `/etc/config/smb.conf` | QNAP SMB configuration candidate; may be registry-backed in newer stacks |
| `/etc/passwd`, `/etc/shadow`, `/etc/group` | Local user/group files, but QTS privilege systems also matter |
| `/share/` | Share root and symlinks |
| `/share/CACHEDEV1_DATA/` | Common primary storage volume path |
| `/share/external/` | External USB/eSATA mounts |
| `/share/*/.qpkg/` | Common QPKG installation location |
| `/var/log/` | System logs, if present/available |
| `/tmp/` | Runtime temp; full `/tmp` can break QTS behavior |
| `/mnt/HDA_ROOT/` | QNAP system/root-related storage area on many models |
| `/proc/mdstat` | mdraid state |
| `/proc/meminfo`, `/proc/loadavg`, `/proc/partitions` | Linux kernel telemetry |

Do not assume every path exists. QNAP paths vary by model, firmware, volume layout, and installed QPKGs.

---

## 6. Standard read-only preflight

Run this first for almost every diagnostic task.

```sh
echo "===== QNAP PREFLIGHT ====="
date
hostname
uname -a
uptime

echo "===== QTS VERSION / MODEL ====="
getcfg System Version -f /etc/config/uLinux.conf 2>/dev/null
getcfg System "Build Number" -f /etc/config/uLinux.conf 2>/dev/null
getcfg System Model -f /etc/config/uLinux.conf 2>/dev/null
sed -n '1,120p' /etc/config/uLinux.conf 2>/dev/null

echo "===== RESOURCE STATE ====="
cat /proc/loadavg
free 2>/dev/null || sed -n '1,30p' /proc/meminfo
df -h

echo "===== STORAGE STATE ====="
cat /proc/mdstat 2>/dev/null
mount | sed -n '1,160p'

echo "===== NETWORK STATE ====="
ip addr show 2>/dev/null || ifconfig -a
ip route show 2>/dev/null || route -n
netstat -tulpen 2>/dev/null || netstat -tulpn 2>/dev/null || netstat -an

echo "===== QPKG INVENTORY QUICK VIEW ====="
grep '^\[' /etc/config/qpkg.conf 2>/dev/null | sed 's/^\[//;s/\]$//' | sed -n '1,200p'
```

Interpretation checklist:

- Confirm the host is the intended NAS.
- Confirm QTS version/build.
- Check if load average is abnormal.
- Check if any volume is near full.
- Check if storage is degraded or rebuilding.
- Check if expected services are listening.
- Check if unexpected remote-access services are listening.
- Check if the QPKG list contains old or risky runtimes.

---

## 7. QTS identity and firmware

QTS version/build is usually available through `/etc/config/uLinux.conf`.

Commands:

```sh
echo "===== QTS IDENTITY ====="
getcfg System Version -f /etc/config/uLinux.conf 2>/dev/null
getcfg System "Build Number" -f /etc/config/uLinux.conf 2>/dev/null
getcfg System Model -f /etc/config/uLinux.conf 2>/dev/null
getcfg System "Model Name" -f /etc/config/uLinux.conf 2>/dev/null
cat /etc/config/uLinux.conf 2>/dev/null | sed -n '1,160p'
```

Useful interpretation:

- If firmware is older than expected, flag it.
- If auto-update is enabled in operator docs, do not assume it has succeeded; verify version.
- Firmware changes are high-risk. Recommend QTS Web UI and backup verification before updates.

---

## 8. Storage diagnostics

Storage is the most important area to protect. Prefer read-only CLI inspection and QTS Web UI for changes.

### 8.1 Storage inventory

```sh
echo "===== FILESYSTEM USAGE ====="
df -h

echo "===== MOUNTS ====="
mount | sed -n '1,200p'

echo "===== MD RAID ====="
cat /proc/mdstat 2>/dev/null

echo "===== LVM ====="
command -v pvs >/dev/null 2>&1 && pvs || true
command -v vgs >/dev/null 2>&1 && vgs || true
command -v lvs >/dev/null 2>&1 && lvs || true

echo "===== BLOCK DEVICES ====="
cat /proc/partitions

echo "===== QNAP STORAGE CLI ====="
command -v qcli_storage >/dev/null 2>&1 && qcli_storage -d || echo "qcli_storage not found"
```

What to look for:

- degraded or rebuilding md arrays
- missing disks
- unexpected device renumbering
- read-only mounts
- external drives mounted under `/share/external`
- volumes over 90%, urgent if over 95%
- suspicious growth in `.qpkg`, backup targets, container volumes, logs, or snapshots

### 8.2 SMART / disk health

QNAP’s Web UI is usually the best source of full disk health: **Storage & Snapshots > Storage > Disks/VJBOD > Health**.

CLI checks vary by model.

```sh
echo "===== SMART HEALTH ATTEMPT ====="
for d in /dev/sd? /dev/nvme?n1; do
  [ -e "$d" ] || continue
  echo "--- $d ---"
  command -v smartctl >/dev/null 2>&1 && smartctl -H "$d" 2>&1 | sed -n '1,80p'
done

echo "===== QNAP DISK MAP ====="
command -v qcli_storage >/dev/null 2>&1 && qcli_storage -d || true
```

If `smartctl` is not available or output is incomplete, do not install tools. Report that health should be confirmed in QTS Web UI.

### 8.3 Performance test

Do not run automatically. It can add storage load.

Possible approved command:

```sh
command -v qcli_storage >/dev/null 2>&1 && qcli_storage -t force=1
```

Use only when:

- the operator approved it,
- backups/rebuilds/scans are not active,
- load is reasonable,
- and slow storage is a plausible cause.

### 8.4 External drives

QNAP often mounts external devices under:

```sh
/share/external/
```

Inspect:

```sh
echo "===== EXTERNAL STORAGE ====="
ls -la /share/external 2>/dev/null
mount | grep -i external
df -h | grep -i external
```

Do not add external drives to internal LVM/storage pools unless QTS explicitly supports the intended use. Use QTS Web UI for external disk formatting, encryption, and backup target configuration.

---

## 9. Thermal, fan, and chassis health

Commands:

```sh
echo "===== TEMPERATURES ====="
command -v getsysinfo >/dev/null 2>&1 && getsysinfo cputmp || echo "getsysinfo cputmp unavailable"
command -v getsysinfo >/dev/null 2>&1 && getsysinfo systmp 1 || echo "getsysinfo systmp unavailable"

echo "===== FAN STATUS ====="
if command -v getsysinfo >/dev/null 2>&1; then
  FAN_NUM="$(getsysinfo sysfannum 2>/dev/null)"
  echo "fan_count=$FAN_NUM"
  i=1
  while [ "$i" -le "${FAN_NUM:-0}" ] 2>/dev/null; do
    echo "fan_$i=$(getsysinfo sysfan "$i" 2>/dev/null)"
    i=$((i+1))
  done
fi
```

Interpretation:

- Fan `0 RPM` is critical if the model has an active fan and should be spinning.
- Rising temperature under low load is more important than one isolated reading.
- Correlate heat with container load, malware scans, backups, indexing, room temperature, dust, failed fan, and drive temps.

Fan override is risky and model-specific. Do not run unless this is an approved emergency action:

```sh
hal_app --se_sys_set_fan_mode enc_sys_id=root,obj_index=0,mode=7
```

If used, report it clearly and recommend restoring normal Smart Fan control in QTS Web UI.

---

## 10. Network and virtual switch diagnostics

QNAP networking is controlled by QNAP’s own network stack and virtual switch system. Do not assume generic Linux network persistence.

Read-only commands:

```sh
echo "===== INTERFACES ====="
ip addr show 2>/dev/null || ifconfig -a

echo "===== ROUTES ====="
ip route show 2>/dev/null || route -n

echo "===== BRIDGES ====="
brctl show 2>/dev/null || true

echo "===== DNS ====="
cat /etc/resolv.conf 2>/dev/null

echo "===== LISTENING SERVICES ====="
netstat -tulpen 2>/dev/null || netstat -tulpn 2>/dev/null || netstat -an

echo "===== NETWORK ERRORS ====="
netstat -i 2>/dev/null || true
```

Look for:

- unexpected default gateway
- duplicate IP symptoms
- interface down
- bridge/virtual switch mismatch
- QTS admin ports exposed on untrusted interfaces
- FTP, CloudLink, UPnP, or other remote access services
- high RX/TX errors
- Docker/LXC bridges such as `docker0`, `lxcbr0`, `lxdbr0`
- WireGuard interface such as `wg0`

Changing network settings can cut off the agent. Any network change is explicit-approval only, and preferably done through QTS Web UI.

---

## 11. QVPN / WireGuard

QVPN may manage WireGuard and related cron/token refresh tasks.

Read-only checks:

```sh
echo "===== WIREGUARD ====="
command -v wg >/dev/null 2>&1 && wg show || echo "wg command not found"
ip addr show wg0 2>/dev/null || true

echo "===== QVPN QPKG INFO ====="
grep -i -A20 '\[QVPN\]' /etc/config/qpkg.conf 2>/dev/null
grep -i 'qvpn' /etc/config/crontab 2>/dev/null
ps | grep -i '[q]vpn\|[w]ireguard\|[w]g'
```

Do not directly edit QVPN configuration unless there is a known QNAP-specific procedure and explicit approval.

---

## 12. QPKG apps

QPKGs are the QNAP app ecosystem. Treat them as OS-level application packages.

### 12.1 Inventory

```sh
echo "===== QPKG INVENTORY ====="
if [ -f /etc/config/qpkg.conf ]; then
  awk '
    /^\[/ {pkg=$0; gsub(/\[|\]/,"",pkg); print ""; print "[" pkg "]"}
    /^Version/ {print}
    /^Enable/ {print}
    /^Install_Path/ {print}
    /^Shell/ {print}
  ' /etc/config/qpkg.conf
else
  echo "/etc/config/qpkg.conf not found"
fi
```

Flag:

- old Node.js, Python, Java/JRE, PHP, MongoDB, or database runtimes
- QPKGs installed but disabled
- remote-access QPKGs
- packages that appear abandoned
- packages listening on unexpected ports
- packages with large data directories

### 12.2 Service scripts

Discover QPKG init scripts:

```sh
echo "===== INIT / SERVICE SCRIPTS ====="
ls -1 /etc/init.d 2>/dev/null | sed -n '1,240p'
```

Do not restart QPKGs unless the operator approves. Before proposing restart, identify:

- package name
- service script
- process list
- listening ports
- expected impact
- rollback/validation

---

## 13. Container Station

Container Station is QNAP’s Docker/LXC app. The Docker CLI may not be on `PATH`.

### 13.1 Discover install path

```sh
echo "===== CONTAINER STATION DISCOVERY ====="
CS_PATH="$(getcfg container-station Install_Path -f /etc/config/qpkg.conf 2>/dev/null)"
echo "container-station Install_Path=$CS_PATH"

if [ -n "$CS_PATH" ] && [ -d "$CS_PATH" ]; then
  find "$CS_PATH" -maxdepth 5 -type f \( -name docker -o -name docker-compose -o -name ctr -o -name nerdctl \) 2>/dev/null
fi

find /share -path '*/.qpkg/ContainerStation*' -maxdepth 6 2>/dev/null | sed -n '1,120p'
```

### 13.2 Read-only container checks

After discovering the Docker binary:

```sh
DOCKER_BIN="<discovered-docker-path>"
[ -x "$DOCKER_BIN" ] && "$DOCKER_BIN" ps -a
[ -x "$DOCKER_BIN" ] && "$DOCKER_BIN" images
[ -x "$DOCKER_BIN" ] && "$DOCKER_BIN" system df
```

Also check bridge state:

```sh
ip addr show docker0 2>/dev/null || true
ip addr show lxcbr0 2>/dev/null || true
ip addr show lxdbr0 2>/dev/null || true
brctl show 2>/dev/null || true
ps | grep -i '[d]ocker\|[c]ontainer\|[l]xc\|[l]xd'
```

Do not run:

```sh
docker system prune
docker volume rm
docker rm
docker compose down
```

unless explicitly approved and the target stack is known.

---

## 14. UniFi Network Controller QPKG

If UniFi is installed as a QPKG rather than Docker, treat it as a QNAP app.

Discovery:

```sh
echo "===== UNIFI DISCOVERY ====="
grep -i -A30 '\[.*unifi.*\]' /etc/config/qpkg.conf 2>/dev/null
grep -i 'unifi' /etc/config/qpkg.conf 2>/dev/null
ps | grep -i '[u]nifi\|[m]ongod\|[j]ava'
netstat -tulpen 2>/dev/null | grep -i '8080\|8443\|3478\|10001\|unifi' || true
```

Common UniFi-related signals:

- Java process
- MongoDB process, depending on controller version/package
- ports commonly associated with UniFi controller/adoption
- install path in `/etc/config/qpkg.conf`

Do not upgrade or restart UniFi without operator approval because it can affect AP management and client visibility.

---

## 15. SMB file sharing

SMB is usually core NAS functionality. Be conservative.

Read-only checks:

```sh
echo "===== SMB PROCESSES ====="
ps | grep '[s]mbd\|[n]mbd\|[w]inbind'

echo "===== SMB STATUS ====="
command -v smbstatus >/dev/null 2>&1 && smbstatus || echo "smbstatus not found"

echo "===== SMB CONFIG ====="
ls -l /etc/config/smb.conf /etc/smb.conf 2>/dev/null
testparm -s 2>/dev/null | sed -n '1,220p'

echo "===== SMB LISTENERS ====="
netstat -tulpen 2>/dev/null | grep -E ':(139|445)[[:space:]]' || true
```

Do not directly edit SMB share definitions unless the target QTS behavior is known. Prefer QTS Web UI for:

- share creation/deletion
- permissions
- ACLs
- SMB signing
- recycle bin settings
- access-based enumeration
- guest access
- advanced folder permissions

---

## 16. NFS exports

QNAP NFS persistent config usually belongs in:

```sh
/etc/config/nfssetting
```

`/etc/exports` is generated and should not be the source of truth.

Read-only checks:

```sh
echo "===== NFS CONFIG ====="
cat /etc/config/nfssetting 2>/dev/null
cat /etc/exports 2>/dev/null

echo "===== NFS EXPORTS ====="
/usr/sbin/exportfs -v 2>/dev/null || exportfs -v 2>/dev/null

echo "===== NFS PROCESSES ====="
ps | grep -i '[n]fs\|[r]pc'
```

Approved reload pattern after a backed-up config edit:

```sh
cp -p /etc/config/nfssetting /etc/config/nfssetting.bak.$(date +%Y%m%d-%H%M%S)
/sbin/gen_exports > /etc/exports
/usr/sbin/exportfs -r
```

Do not edit NFS exports without approval.

---

## 17. Users, groups, and permissions

QNAP user/group administration is not normal Linux user administration. Standard Linux commands can desynchronize QTS privilege databases and Samba/NFS behavior.

Read-only checks:

```sh
echo "===== USERS ====="
sed -n '1,180p' /etc/passwd

echo "===== GROUPS ====="
sed -n '1,180p' /etc/group

echo "===== SHARE PERMISSIONS QUICK VIEW ====="
ls -ld /share /share/* 2>/dev/null | sed -n '1,220p'
```

For changes, prefer QTS Web UI unless QNAP-supported CLI commands are confirmed on the target.

Commands that may exist:

```sh
adduser <username>
deluser <username>
addgroup <groupname>
delgroup <groupname>
```

Validate names before proposing changes. Do not use `/usr/sbin/useradd` or generic Linux account-management assumptions without QNAP-specific confirmation.

Permission warning:

- Advanced Folder Permissions and Windows ACL support can conflict.
- Do not toggle either casually.
- Do not recursively `chmod` or `chown` shares unless the scope and impact are understood.

---

## 18. Cron and scheduled jobs

QNAP persistent cron source:

```sh
/etc/config/crontab
```

Read-only inventory:

```sh
echo "===== QNAP CRON ====="
cat /etc/config/crontab 2>/dev/null

echo "===== ACTIVE CRONTAB ====="
crontab -l 2>/dev/null

echo "===== CROND PROCESS ====="
ps | grep '[c]rond'
```

Look for:

- malware scan jobs
- backup jobs
- QVPN token refresh
- log rotation
- time sync
- custom jobs with missing scripts
- jobs writing to full volumes
- jobs that explain load spikes

Editing cron is approval-only. Safe pattern after approval:

```sh
cp -p /etc/config/crontab /etc/config/crontab.bak.$(date +%Y%m%d-%H%M%S)
vi /etc/config/crontab
crontab /etc/config/crontab
/etc/init.d/crond.sh restart
crontab -l
```

If `/etc/init.d/crond.sh` does not exist, stop and report instead of guessing.

---

## 19. Logs and evidence

Read-only collection:

```sh
echo "===== DMESG RECENT ====="
dmesg | tail -n 200

echo "===== STORAGE/KERNEL ERRORS ====="
dmesg | grep -i 'error\|fail\|warn\|ata\|scsi\|md\|raid\|ext4\|btrfs\|i/o\|unclean' | tail -n 160

echo "===== LOG DIRECTORY ====="
ls -ltr /var/log 2>/dev/null | tail -n 100

echo "===== MESSAGES ====="
tail -n 300 /var/log/messages 2>/dev/null
```

If logs are missing, rotated, or not readable, say so. Do not install logging packages.

---

## 20. Backup, HBS, Qsync, and UPS

### 20.1 Backup app presence

```sh
echo "===== BACKUP/QSYNC QPKGS ====="
grep -i 'HybridBackup\|HBS\|Backup\|Qsync' /etc/config/qpkg.conf 2>/dev/null

echo "===== BACKUP/QSYNC PROCESSES ====="
ps | grep -i '[h]bs\|[b]ackup\|[q]sync'
```

Do not claim backups are healthy merely because a backup app is installed. To say backups are healthy, the agent must find recent successful job evidence from logs, QTS/HBS UI data, or operator-provided status.

### 20.2 UPS

```sh
echo "===== UPS CHECKS ====="
ps | grep -i '[u]ps\|[n]ut'
lsusb 2>/dev/null | sed -n '1,160p'
dmesg | grep -i 'ups\|cyber\|battery\|usb' | tail -n 160
```

Do not change UPS auto-protection or shutdown behavior without approval.

---

## 21. Malware Remover and security apps

Inventory:

```sh
echo "===== SECURITY QPKGS ====="
grep -i 'Malware\|Security\|QuFirewall\|QVPN\|CloudLink\|QuFTP' /etc/config/qpkg.conf 2>/dev/null

echo "===== SECURITY-RELATED CRON ====="
grep -i 'malware\|security\|qulog\|license\|cloud\|qvpn' /etc/config/crontab 2>/dev/null
```

Interpretation:

- Daily malware scans can explain load spikes.
- QuFTP installed but disabled is still worth reporting as attack surface.
- CloudLink and remote-access helpers should be reviewed if not intentionally used.
- QVPN/WireGuard should be checked for expected listeners and routes.

---

## 22. Security posture checklist

Run read-only:

```sh
echo "===== VERSION ====="
getcfg System Version -f /etc/config/uLinux.conf 2>/dev/null
getcfg System "Build Number" -f /etc/config/uLinux.conf 2>/dev/null

echo "===== LISTENERS ====="
netstat -tulpen 2>/dev/null || netstat -tulpn 2>/dev/null || netstat -an

echo "===== SSH CONFIG ====="
grep -v '^[[:space:]]*#' /etc/ssh/sshd_config 2>/dev/null | sed -n '1,200p'

echo "===== LEGACY/RISKY QPKGS ====="
grep -i 'node\|java\|jre\|python\|php\|mysql\|mariadb\|mongo\|cloudlink\|ftp\|quftp' /etc/config/qpkg.conf 2>/dev/null

echo "===== DEFAULT ADMIN SHADOW PRESENCE ====="
grep '^admin:' /etc/shadow 2>/dev/null | sed 's/:.*$/:<redacted>/'
```

Report:

- QTS version/build
- SSH password login posture if visible
- whether default `admin` appears enabled/disabled, if determinable
- web admin exposure
- FTP/CloudLink/remote access exposure
- legacy runtimes such as Node 8, old Java, old Python, old MongoDB
- missing firewall/security app, if expected
- outdated firmware or QPKGs, if known

Do not run exploit checks or probe vulnerable CGI endpoints unless explicitly asked. Prefer version-based advisory checks and QNAP Security Counselor/Web UI review.

---

## 23. Web UI and HTTP API guidance

The QTS Web UI is commonly available at:

```text
https://<nas>/
http://<nas>:8080/
```

Use the Web UI as authoritative for:

- storage pools, RAID, volumes, snapshots, LUNs
- disk health and SMART views
- firmware updates
- QPKG installation/removal/upgrades
- network/virtual switch changes
- users/groups/permissions/ACLs
- SMB/NFS share management
- UPS behavior
- QVPN configuration
- Security Counselor and Malware Remover

Web API use is optional and should be handled by a secure credential broker. Do not log session IDs, passwords, cookies, or MFA codes. Do not automate MFA recovery flows. Do not create indefinite sessions. Use API primarily for telemetry, not risky state changes.

---

## 24. Common task playbooks

### 24.1 Slow SMB or file transfers

Collect:

```sh
uptime
df -h
cat /proc/mdstat 2>/dev/null
mount | sed -n '1,200p'
dmesg | grep -i 'error\|fail\|ata\|scsi\|i/o\|raid\|md' | tail -n 160
ip addr show 2>/dev/null || ifconfig -a
ip route show 2>/dev/null || route -n
netstat -i 2>/dev/null
command -v qcli_storage >/dev/null 2>&1 && qcli_storage -d
command -v smbstatus >/dev/null 2>&1 && smbstatus
ps w | sed -n '1,220p'
```

Likely causes:

- degraded RAID or disk errors
- volume nearly full
- read-only remount
- slow client Wi-Fi
- switch/NIC speed/duplex issue
- virtual switch/bridge issue
- backup, malware scan, indexing, Qsync, or container load
- SMB configuration issue

Do not run disk performance tests until approved.

### 24.2 High CPU or high load

Collect:

```sh
uptime
cat /proc/loadavg
ps w | sed -n '1,240p'
top -n 1 2>/dev/null | sed -n '1,100p'
df -h
cat /proc/mdstat 2>/dev/null
dmesg | tail -n 160
grep -i 'malware\|backup\|qsync\|qulog\|qvpn' /etc/config/crontab 2>/dev/null
```

Likely causes:

- Malware Remover
- HBS backup
- Qsync
- media/indexing jobs
- Container Station workload
- UniFi/Java/MongoDB
- RAID rebuild/scrub
- full filesystem
- runaway QPKG

Do not kill processes until the service and impact are identified.

### 24.3 Disk warning or degraded storage

Collect:

```sh
cat /proc/mdstat 2>/dev/null
df -h
mount
dmesg | grep -i 'ata\|scsi\|i/o\|medium\|error\|fail\|raid\|md\|unclean' | tail -n 220
command -v qcli_storage >/dev/null 2>&1 && qcli_storage -d
```

Rules:

- Do not remove/re-add disks.
- Do not run repair commands.
- Do not repartition.
- Do not reboot unless QNAP docs/operator approve.
- Recommend QTS Web UI Storage & Snapshots confirmation.
- Confirm backup state before any rebuild/replace workflow.

### 24.4 QPKG not responding

Collect:

```sh
grep -n '^\[.*\]' /etc/config/qpkg.conf
grep -i -A30 '<package-pattern>' /etc/config/qpkg.conf 2>/dev/null
ps w | grep -i '<package-pattern>'
ls -l /etc/init.d | grep -i '<package-pattern>'
netstat -tulpen 2>/dev/null | grep -i '<port-or-pattern>' || true
tail -n 300 /var/log/messages 2>/dev/null
```

Then propose restart only with approval and impact statement.

### 24.5 Web UI unreachable but SSH works

Collect:

```sh
netstat -tulpen 2>/dev/null || netstat -tulpn 2>/dev/null || netstat -an
ps | grep -i '[t]httpd\|[a]pache\|[n]ginx\|[q]thttp'
df -h
dmesg | tail -n 160
```

Check for:

- full system volume
- Web UI process down
- port conflict
- certificate issue
- firewall/network issue
- load so high Web UI is unresponsive

Do not restart the Web UI stack blindly if storage/network state is unhealthy.

### 24.6 Container Station issue

Collect:

```sh
grep -i -A30 '\[container-station\]' /etc/config/qpkg.conf 2>/dev/null
ps w | grep -i '[c]ontainer\|[d]ocker\|[l]xc\|[l]xd'
ip addr show 2>/dev/null | grep -A4 -E 'docker0|lxcbr0|lxdbr0'
brctl show 2>/dev/null
```

Discover Docker binary before using it. Use read-only Docker commands first.

### 24.7 QVPN/WireGuard issue

Collect:

```sh
command -v wg >/dev/null 2>&1 && wg show || echo "wg unavailable"
ip addr show wg0 2>/dev/null || true
ip route show 2>/dev/null || route -n
grep -i -A30 '\[QVPN\]' /etc/config/qpkg.conf 2>/dev/null
grep -i qvpn /etc/config/crontab 2>/dev/null
ps | grep -i '[q]vpn\|[w]g\|[w]ireguard'
```

Do not edit WireGuard/QVPN config directly unless approved and documented.

---

## 25. Change proposal format

Before any change, produce this:

```markdown
## Proposed QNAP Change

Target:
Observed problem:
Proposed action:
Risk:
Why this action:
Expected impact:
Services affected:
Commands or Web UI path:
Backup/rollback plan:
Validation after change:
Requires approval: yes
```

After approval and execution:

```markdown
## QNAP Change Result

Target:
Action performed:
Start/end time:
Commands or Web UI path used:
Exit codes:
Observed result:
Validation:
Rollback needed:
Remaining concerns:
```

---

## 26. Diagnostic report format

Use this report format after read-only diagnosis.

```markdown
# QNAP Diagnostic Report

## Summary
- Overall state:
- Primary issue:
- Confidence:
- Immediate action needed:

## Identity
- Hostname:
- Model:
- QTS version/build:
- Kernel:
- Uptime:

## Storage
- Volume usage:
- RAID/md state:
- Disk inventory:
- SMART/health evidence:
- Read-only mounts:
- Recent disk/kernel errors:

## Performance
- Load:
- Memory:
- Top suspected processes:
- Scheduled jobs possibly active:

## Network
- Interfaces:
- Default route:
- DNS:
- Bridges/virtual switches:
- Listening services:
- QVPN/WireGuard:

## Services/QPKGs
- Critical QPKGs:
- Container Station:
- UniFi:
- SMB:
- NFS:
- Backup/Qsync:
- UPS:
- Security/Malware tools:

## Security observations
- Firmware/update posture:
- SSH posture:
- Web UI exposure:
- FTP/CloudLink/remote access:
- Legacy runtimes:
- Security concerns:

## Recommended next steps
1.
2.
3.

## Actions intentionally not taken
-
```

---

## 27. Short operating instruction for agents

When working on a QNAP NAS:

1. Treat QTS as appliance Linux.
2. Start with read-only preflight.
3. Use QNAP paths and QNAP tools.
4. Discover QPKG install paths before touching apps.
5. Use QTS Web UI for risky changes.
6. Never modify storage, networking, permissions, firmware, QPKGs, cron, or service state without explicit approval.
7. Stop on degraded storage, read-only mounts, I/O errors, unknown host identity, or ambiguous QNAP behavior.
8. Report evidence and next steps; do not improvise fixes.
