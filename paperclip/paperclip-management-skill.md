---
name: paperclip-management
description: Operational skill for agents managing, diagnosing, and safely administering a self-hosted Paperclip installation. Use for Paperclip control-plane health, companies, agents, goals, issues, heartbeats, budgets, approvals, MCP/API workflows, Docker/process adapters, /paperclip persistent storage, backups, secrets, migrations, and recovery. This is not a generic task-execution skill.  This skill needs the paperclip MCP and also access to the paperclip install for best results.
version: 1.0.0
---

# Paperclip Management Skill

## 1. Purpose

Use this skill when an agent is responsible for supporting a Paperclip installation as an operator, systems administrator, or platform manager.

This skill is for managing the **Paperclip control plane** itself: its health, configuration, companies, agents, goals, issues, heartbeats, budgets, approvals, workspaces, storage, secrets, adapters, and recovery workflows.

This is **not** a normal “do the task assigned in Paperclip” skill. It is for an administrative agent such as Hermes that may need to:

- diagnose a Paperclip server
- inspect or repair agent orchestration
- investigate stuck heartbeat runs
- manage task/issue state
- review companies, agents, goals, budgets, and approvals
- inspect local `/paperclip` persistent data
- check Docker/container health
- verify backups and storage
- diagnose MCP/API failures
- recover from migrations, lockouts, or corrupted local state
- safely coordinate other agents through tickets/issues

Do not hard-code user-specific hostnames, company IDs, API keys, agent IDs, task IDs, file paths, or deployment choices in this skill. Discover those from the local repository, environment, container configuration, MCP tools, API responses, or explicit operator instructions.

---

## 2. Core mental model

Paperclip is a persistent multi-agent control plane. It is not just a chat loop, local script runner, or isolated agent process.

Think in layers:

1. **Instance layer:** Paperclip server, database, auth mode, storage, secrets, backups.
2. **Tenant layer:** companies as namespaces and security boundaries.
3. **Organization layer:** agents, reporting lines, roles, budgets, approvals.
4. **Work layer:** goals, projects, issues/tasks, comments, attachments, blocked links.
5. **Execution layer:** heartbeats, runs, logs, adapters, process/runtime workers.
6. **Filesystem layer:** persistent `/paperclip` data, workspaces, backups, config.
7. **Integration layer:** MCP servers, REST API, external runtimes such as OpenClaw, Claude Code, Codex, or HTTP workers.

The core operating rule:

> Use the Paperclip API/MCP for normal control-plane operations. Use direct filesystem/container access only for diagnosis, recovery, backup, or when the control plane itself is unhealthy.

---

## 3. Generic platform boundaries

This skill is generic. It assumes only that the target is a Paperclip-style installation and that the managing agent has authorized access through one or more of:

- Paperclip MCP tools
- Paperclip REST API token
- SSH to the host
- Docker or Docker Compose access
- mounted `/paperclip` persistent directory
- local deployment repository
- local logs/backups
- operator-provided emergency access

Do not assume:

- server port is always `3100`
- persistent path is always `/paperclip`
- container name is always `paperclip`
- database is always embedded pglite
- authentication is disabled
- MCP tools are installed
- the installation is single-company
- the active company ID is known
- the managing agent has board-level authority

Discover the actual deployment before acting.

---

## 4. Key Paperclip entities

| Entity | Meaning | Agent guidance |
|---|---|---|
| Company | Tenant / namespace / security boundary | Never mix company contexts. Always scope queries and actions to the active company. |
| Agent | Registered worker profile and runtime adapter | Inspect role, reports_to, budget, heartbeat, and assigned issues before acting. |
| Goal | Hierarchical business objective | Tasks should trace to an active goal when possible. |
| Project | Work grouping, often under a company/goal | Use for related task clusters. |
| Issue / Task / Ticket | Atomic work unit assigned to one agent or user | Main coordination object. Use comments and status, not side-channel memory. |
| Comment / Thread | Auditable discussion and status history | Use for updates, blockers, handoffs, and final results. |
| Heartbeat | Scheduled or manual agent execution trigger | Used to wake agents and run assigned work. |
| Heartbeat Run | Concrete execution instance | Inspect for stuck runs, logs, costs, and failures. |
| Approval | Governance checkpoint requiring human/board decision | Do not bypass. Pause and request approval. |
| Budget | Cost/token spend cap | Respect budget thresholds; reduce or halt work near limits. |
| Attachment / Artifact | Uploaded work product stored by Paperclip | Use for durable delivery. Do not rely on local workspace paths. |

Paperclip coordination should happen through issues, comments, assignments, statuses, and attachments, not invisible direct agent-to-agent messages.

---

## 5. Access model

### 5.1 Primary path: MCP/API

Use MCP or REST API for normal Paperclip operations:

- list assigned issues
- get issue details
- checkout/lock an issue
- comment on issues
- create or link blocking issues
- release or complete work
- invoke agent heartbeats
- list agents/goals/projects
- inspect budgets and costs
- review approvals
- upload attachments
- inspect activity/audit logs

This preserves auditability and keeps system state consistent.

### 5.2 Fallback path: direct host/filesystem access

Use direct filesystem/container access for:

- server health checks
- logs
- config discovery
- backup verification
- emergency recovery
- migration diagnosis
- adapter/runtime diagnosis
- checking persistent storage
- inspecting workspaces
- investigating when API/MCP is down

Direct access must not be used to bypass Paperclip governance. If a normal API/MCP route exists and the system is healthy, use it.

---

## 6. Important filesystem map

Typical persistent directory:

```text
/paperclip
```

Actual path may differ. Discover from Docker mounts, compose files, or deployment docs.

Important paths:

| Path | Purpose | Rules |
|---|---|---|
| `/paperclip/config.json` | Instance/server/database/auth/storage config | Read for diagnosis. Write only with approval and backup. |
| `/paperclip/.env` | Environment variables and secrets references | Do not print. Do not leak. Write only for approved config/rotation. |
| `/paperclip/db/` | Embedded database storage, if used | Do not modify while server/database is running. |
| `/paperclip/secrets/master.key` | Master key for encrypted secrets | Never print or rotate casually. Loss can break credential decryption. |
| `/paperclip/data/storage/` | Uploaded assets/artifacts | Read for diagnosis; use API for normal artifact delivery. |
| `/paperclip/data/backups/` | Local backups/dumps | Verify existence, freshness, size, and retention. |
| `/paperclip/workspaces/<agent-id>/` | Agent workspaces/checkouts | Inspect for stuck state; do not use as final delivery. |
| `/paperclip/logs/` | Logs, if deployed that way | Read-only unless approved cleanup. |
| `/app` | Application/source path inside container in many deployments | Treat as ephemeral. Do not store persistent changes here. |

Critical rule:

> `/paperclip` is persistent state. `/app` is usually application/runtime code and may be overwritten by updates.

---

## 7. Safety rules

### 7.1 Read-only by default

Generally safe:

- inspect containers/processes
- read logs
- check health endpoint locally
- list issues through MCP/API
- inspect assigned tasks
- inspect agent profiles
- inspect budgets/cost summaries
- inspect config keys with secrets redacted
- inspect backup directory metadata
- inspect workspace directory names and sizes
- inspect migration status/logs
- inspect active heartbeat runs

### 7.2 Requires explicit approval

Do not do these without approval:

- restart Paperclip server
- stop/start containers
- update images/packages
- edit `/paperclip/config.json`
- edit `/paperclip/.env`
- edit database files directly
- delete or reset `/paperclip/db`
- rotate `secrets/master.key`
- run migrations
- approve/reject governance approvals
- create/delete companies
- create/delete agents
- change budgets
- change auth or signup settings
- reassign many issues
- cancel/delete tasks
- terminate heartbeat runs/processes
- purge workspaces, backups, logs, or storage
- modify adapter configuration
- expose Paperclip to a network or reverse proxy

### 7.3 High-risk actions

Require approval, backup, and rollback plan:

- database reset or restore
- migration repair
- authentication recovery
- changing deployment exposure/auth mode
- rotating encryption keys
- applying schema migrations manually
- deleting stuck run state from database
- changing company/tenant boundaries
- updating Paperclip across breaking releases
- changing a production reverse proxy
- changing persistent volume mappings
- granting board/admin privileges

### 7.4 Hard stops

Stop and report if:

- deployment path is unclear
- database type is unclear
- `/paperclip` mount cannot be identified
- auth mode is unclear
- secrets would need to be printed or copied into chat/logs
- backup status is unknown before a destructive action
- API/MCP response company context conflicts with local config
- task checkout returns conflict/locked state
- migration logs indicate partial failure
- budget is exhausted
- operator asks to bypass an approval workflow
- a proposed action would mix tenant/company contexts
- the agent does not have authority for board-level operation

---

## 8. Secret handling

Never print:

- `.env` values
- API keys
- JWT secrets
- Better Auth secrets
- database URLs
- provider keys
- model keys
- session cookies
- CSRF tokens
- `secrets/master.key`
- encrypted secret payloads that can be replayed

Safe redacted inspection pattern:

```sh
sed -E 's/(KEY|TOKEN|SECRET|PASSWORD|DATABASE_URL|AUTH|COOKIE|JWT|OPENAI|ANTHROPIC|GEMINI|CLAUDE|API).*/\1=<redacted>/Ig' /paperclip/.env
```

Prefer listing variable names without values:

```sh
awk -F= '/^[A-Za-z_][A-Za-z0-9_]*=/ {print $1}' /paperclip/.env | sort
```

If secrets migration tooling exists, do not run it until:

1. current config is backed up,
2. operator approves,
3. the command and expected effect are known,
4. rollback path is clear.

---

## 9. Instance discovery

### 9.1 Docker/container discovery

```sh
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker compose ls 2>/dev/null || true
```

Look for Paperclip-related containers by image/name, but do not assume names.

Likely patterns:

- `paperclip`
- `paperclip-ai`
- `agencyenterprise/paperclip-ai`
- `paperclipai/paperclip`
- app/server/api containers
- database containers if external PostgreSQL is used
- reverse proxy containers such as Caddy/Traefik/Nginx

After identifying the container:

```sh
PC_CONT="<discovered-paperclip-container>"
docker inspect "$PC_CONT" --format '
Name={{.Name}}
Image={{.Config.Image}}
State={{.State.Status}}
StartedAt={{.State.StartedAt}}
Restarting={{.State.Restarting}}
ExitCode={{.State.ExitCode}}
NetworkMode={{.HostConfig.NetworkMode}}
'
docker inspect "$PC_CONT" --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Find the mount mapped to `/paperclip` or the deployment’s persistent equivalent.

### 9.2 Compose/repository discovery

Search likely deployment locations, not the entire filesystem:

```sh
find /opt /srv /home -maxdepth 5 \( -name 'compose.yaml' -o -name 'compose.yml' -o -name 'docker-compose.yml' -o -name 'docker-compose.yaml' -o -name '.env' \) 2>/dev/null
```

Do not print `.env` contents.

### 9.3 Process discovery if not Docker

```sh
ps aux | grep -i '[p]aperclip'
ps aux | grep -i '[n]ode\|[p]npm\|[b]un\|[p]glite'
ss -tulpen 2>/dev/null | grep -i '3100\|paperclip' || true
```

---

## 10. Standard read-only preflight

Run before diagnosis or proposed changes.

```sh
echo "===== PAPERCLIP PREFLIGHT ====="
date
hostname
whoami
uptime
df -h

echo "===== CONTAINERS ====="
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}' 2>/dev/null || true

echo "===== PAPERCLIP CONTAINER ====="
if [ -n "$PC_CONT" ]; then
  docker inspect "$PC_CONT" --format '
Name={{.Name}}
Image={{.Config.Image}}
State={{.State.Status}}
StartedAt={{.State.StartedAt}}
Restarting={{.State.Restarting}}
ExitCode={{.State.ExitCode}}
NetworkMode={{.HostConfig.NetworkMode}}
' 2>/dev/null
  docker inspect "$PC_CONT" --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}' 2>/dev/null
fi

echo "===== PAPERCLIP PERSISTENT DIR ====="
if [ -d /paperclip ]; then
  ls -la /paperclip
  du -sh /paperclip/* 2>/dev/null | sort -h
fi

echo "===== BACKUPS ====="
if [ -d /paperclip/data/backups ]; then
  ls -lah /paperclip/data/backups | tail -n 30
fi

echo "===== RECENT LOGS ====="
if [ -n "$PC_CONT" ]; then
  docker logs --tail=300 "$PC_CONT" 2>&1 | tail -n 300
fi
```

Interpretation:

- If disk is nearly full, prioritize storage/backup/log cleanup planning.
- If container is restarting, inspect logs before any restart.
- If `/paperclip` is not mounted/persistent, verify deployment before running.
- If backups are absent/stale, avoid destructive repair.
- If server is exposed beyond localhost/private network, perform security review.

---

## 11. API and MCP usage

### 11.1 Prefer MCP when available

Use available Paperclip MCP tools for ordinary platform operations. Tool names vary by MCP server, but common operations include:

- list issues
- get issue
- create issue
- checkout issue
- release issue
- comment on issue
- list goals
- list agents
- list companies
- get cost summary
- invoke agent heartbeat
- get activity/audit log
- approve/reject approvals

If MCP tools exist, prefer them over raw API calls because they preserve the agent runtime’s normal authorization, auditing, and tool semantics.

### 11.2 REST API fallback

Use API only with an authorized token/cookie from a secure broker. Do not print token values.

Common local base URL pattern:

```text
http://127.0.0.1:3100
```

Actual port/host may differ. Discover from config, `.env`, container ports, or reverse proxy.

Safe health check pattern:

```sh
curl -fsS http://127.0.0.1:3100/api/health
```

Authenticated pattern:

```sh
curl -fsS \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  http://127.0.0.1:3100/api/<endpoint>
```

If auth is required and no secure token is available, stop and ask for an authorized access method. Do not scrape UI sessions or bypass auth.

---

## 12. Company and tenant isolation

Every administrative action must be scoped to the correct company.

Rules:

- Resolve active `company_id` from agent identity, MCP context, explicit task context, or operator instruction.
- Do not create issues/goals/agents without company context.
- Do not copy data between companies unless explicitly requested and approved.
- Do not summarize cross-company data to a company-scoped agent unless authorized.
- When uncertain, ask for the intended company/tenant or inspect the issue context.

For multi-company operations, use board/admin-level tooling only if the managing agent has that authority.

---

## 13. Agent heartbeat lifecycle

When operating as a Paperclip worker during a scheduled/manual heartbeat:

### 13.1 Wake and identify

Resolve identity on every run. Do not rely on stale memory.

Collect:

- agent ID
- company ID
- role
- manager / `reports_to`
- current budget/spend
- active task if `PAPERCLIP_TASK_ID` exists
- wake reason if `PAPERCLIP_WAKE_REASON` exists
- run ID if `PAPERCLIP_RUN_ID` or equivalent exists

### 13.2 Read inbox

Prioritize:

1. Explicit task ID from environment/operator.
2. Existing in-progress task assigned to the agent.
3. Todo tasks assigned to the agent.
4. Mention/handoff requiring response.
5. Nothing else: sleep/exit.

If no assigned work exists, do not browse aimlessly or create work just to stay busy.

### 13.3 Checkout before work

Before modifying workspace/files or doing substantive task work, checkout/lock the issue through MCP/API.

If checkout returns conflict/locked by someone else:

- do not retry aggressively
- do not work on the issue locally
- move to another task or sleep

### 13.4 Execute in workspace

Use the designated workspace, often:

```text
/paperclip/workspaces/<agent-id>/
```

Do not deliver final work by leaving files only in the workspace. Attach artifacts or commit changes through the expected platform workflow.

### 13.5 Update thread

Use comments for:

- start/claim note if appropriate
- meaningful progress
- blockers
- validation results
- final summary
- handoff instructions

Avoid comment spam. Do not repeatedly post the same blocked message if nothing changed.

### 13.6 Release, review, or complete

At the end:

- attach final artifacts if any
- update status as appropriate
- release lock if not complete
- assign to reviewer/human when requested
- link blockers when blocked
- leave a clear next-action comment

---

## 14. Issue/task operations

### 14.1 Listing and selecting work

Filter by:

- company
- assigned agent
- status
- project/goal
- priority
- explicit task ID
- blocked status

Do not pull full history for every issue unless needed. Use compact views first.

### 14.2 Blocked task deduplication

Before posting another blocked update:

1. Fetch latest comments/events.
2. Check whether the agent’s last comment is still the newest meaningful update.
3. If nothing changed, skip the task.
4. Do not re-checkout or post repetitive noise.

When blocked:

- explain the blocker
- identify what is needed
- link blocking issue IDs if supported
- assign/escalate to manager or human owner
- release the task if no further work is possible

### 14.3 Creating issues

Create a new issue only when:

- it represents a distinct unit of work,
- it belongs to an active company/goal/project,
- the current issue would become cluttered without it,
- and the assignee/owner is clear.

Do not create duplicate issues. Search first.

### 14.4 Comments

Good comment format:

```markdown
## Update

What changed:
Evidence:
Validation:
Blocked by:
Next owner:
Next step:
```

Final comment format:

```markdown
## Completed

Summary:
Artifacts:
Validation:
Risks/notes:
Recommended next step:
```

---

## 15. Budgets and cost governance

Resolve current budget/spend at run start when available.

Guidance:

- Under 80% spend: normal execution.
- 80–99% spend: prioritize critical assigned tasks, reduce exploratory loops, avoid broad research.
- At or above 100% spend: stop non-critical execution, comment budget warning, release or escalate task.

Do not create new agent runs/heartbeats to bypass budget limits.

---

## 16. Approvals and governance

Approvals represent intentional governance. Do not bypass them through direct DB/API edits.

Use approval tools only when:

- the agent has board/admin authority,
- the approval is clearly understood,
- required context is present,
- and the decision is explicitly authorized.

If a task requires human approval:

1. summarize the requested decision,
2. state risks,
3. recommend approve/reject/defer if appropriate,
4. wait for decision.

---

## 17. Artifact handling

Final work products should be attached/uploaded to Paperclip through the API/MCP when possible.

Rules:

- Do not rely on `/paperclip/workspaces/<agent-id>/...` as durable delivery.
- Do not attach secrets.
- Do not upload huge logs without trimming/redaction.
- Include enough metadata to understand the artifact.
- If artifact upload fails, comment with the failure and preserve the local path temporarily.

Local staging may occur under:

```text
/paperclip/data/storage/
```

but the control plane should own the final artifact.

---

## 18. Git and code workspaces

When Paperclip agents produce git commits, ensure traceability.

Rules:

- Work only in assigned workspace/repository.
- Check task context before committing.
- Include issue/task reference in commit message if local convention supports it.
- Add required co-author attribution when Paperclip requires it:

```text
Co-Authored-By: Paperclip <noreply@paperclip.ing>
```

- Do not commit secrets, `.env`, local DB files, or workspace temp files.
- Do not push to remote unless task explicitly requires it and credentials are authorized.

---

## 19. Health and logs

### 19.1 Container logs

```sh
docker logs --tail=300 "$PC_CONT"
docker logs --since=1h "$PC_CONT"
docker logs -f --tail=200 "$PC_CONT"
```

Avoid long-running follows in unattended runs.

### 19.2 Host/system logs

```sh
journalctl -u docker --since "1 hour ago" 2>/dev/null || true
dmesg | tail -n 120
df -h
```

### 19.3 Paperclip log themes

Search for:

```sh
docker logs --tail=1000 "$PC_CONT" 2>&1 | grep -iE 'error|exception|traceback|failed|migration|database|auth|token|permission|heartbeat|adapter|workspace|budget|approval|pglite|postgres|storage|backup' | tail -n 250
```

Interpretation:

- auth errors may be config/session/token related
- migration errors may require version/schema review
- adapter errors may be runtime/tool-specific
- workspace errors may be permissions/path/git problems
- pglite/postgres errors may indicate database health issues
- repeated heartbeat errors may indicate stuck runs or bad agent config

---

## 20. Backup management

Backups may be stored under:

```text
/paperclip/data/backups/
```

Verify:

```sh
ls -lah /paperclip/data/backups 2>/dev/null | tail -n 50
find /paperclip/data/backups -type f -maxdepth 1 -printf '%TY-%Tm-%Td %TH:%TM %s %p\n' 2>/dev/null | sort | tail -n 20
du -sh /paperclip/data/backups 2>/dev/null
```

Check:

- newest backup timestamp
- file size nonzero
- backup cadence matches expected schedule
- retention not filling disk
- restore procedure is known

Do not delete old backups unless approved. If disk is full, propose retention cleanup with exact files and expected recovered space.

---

## 21. Database handling

Paperclip may use embedded pglite or external PostgreSQL. Discover before acting.

### 21.1 Identify database mode

Inspect config with secrets redacted:

```sh
if [ -f /paperclip/config.json ]; then
  python -m json.tool /paperclip/config.json | sed -E 's/("?(password|token|secret|key|url|connection|database)"?[[:space:]]*:[[:space:]]*)".*"/\1"<redacted>"/Ig'
fi

awk -F= '/^[A-Za-z_][A-Za-z0-9_]*=/ {print $1}' /paperclip/.env 2>/dev/null | sort
```

Look for database-related variables and config keys, but do not print credentials.

### 21.2 Embedded DB safety

If using embedded DB under `/paperclip/db/`:

- do not modify while server is running
- do not delete as a generic migration fix
- back up first
- stop service/container first
- prefer official migration/restore tooling

### 21.3 External PostgreSQL safety

If using external PostgreSQL:

- do not run destructive SQL without approval
- do not print connection strings
- use standard backups before migrations
- verify schema migration command matches the installed version

### 21.4 Migration recovery

Migration repair is high risk. Proposed workflow:

1. Identify current Paperclip version/image.
2. Capture logs.
3. Confirm DB type.
4. Confirm backup exists.
5. Stop Paperclip if direct DB work is needed.
6. Run only documented migration commands for that version.
7. Restart.
8. Validate API, UI, issue list, heartbeat runs, and logs.

Do not blindly run `rm -rf /paperclip/db/` in production. Treat local DB reset as last-resort development/sandbox recovery unless the operator explicitly approves data loss and backup/restore path.

---

## 22. Authentication and exposure

Paperclip deployments may be:

- local/trusted/private
- authenticated
- reverse-proxied
- tunnel-only
- publicly exposed

Security checks:

```sh
ss -tulpen 2>/dev/null | grep -E '3100|paperclip' || true
docker ps --format 'table {{.Names}}\t{{.Ports}}'
awk -F= '/PAPERCLIP|AUTH|SIGN|EXPOSURE|HOST|PORT/ {print $1"=<redacted>"}' /paperclip/.env 2>/dev/null
```

Rules:

- Local/trusted/no-auth mode must not be exposed to untrusted networks.
- Signup should be disabled after initial admin bootstrap when applicable.
- Reverse proxy must use HTTPS for remote access.
- Prefer private network/tunnel access such as VPN/Tailscale over public exposure.
- Do not open firewall ports or change proxy routing without approval.
- Review known advisories for the installed version when exposure/auth changes are involved.

If an auth bypass vulnerability is suspected or the version is known affected, restrict network exposure first, then plan upgrade/patch.

---

## 23. Admin lockout recovery

Admin/auth recovery is high risk.

Before any recovery:

1. Confirm deployment exposure/auth mode.
2. Confirm current admin state.
3. Confirm operator authority.
4. Back up database and config.
5. Prefer documented bootstrap/recovery endpoint or admin tooling.
6. Do not grant admin to an unknown session.
7. Do not expose recovery endpoints publicly.

If a bootstrap claim endpoint is used, call it only against localhost/private access and only with the operator’s approved active session.

Never print session cookies.

---

## 24. Heartbeat and run recovery

Symptoms:

- agent stuck in running state
- task locked indefinitely
- heartbeats no longer scheduling
- run logs stop mid-command
- database locks
- excessive spend from repeated loops

Read-only investigation:

```sh
# API shape may vary by version.
curl -fsS -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  http://127.0.0.1:3100/api/heartbeat-runs
```

Logs:

```sh
docker logs --tail=1000 "$PC_CONT" 2>&1 | grep -iE 'heartbeat|run|adapter|process|lock|timeout|killed|SIGTERM|SIGKILL' | tail -n 250
```

If log byte-offset endpoints exist, use them through API/MCP rather than reading raw process files.

Terminating a stuck run/process is approval-only unless the operator has pre-authorized emergency recovery.

Before termination, capture:

- run ID
- agent ID
- issue ID
- start time
- last log line
- process ID if known
- expected side effects

After termination:

- verify lock release
- comment on affected issue
- requeue/release task if needed
- inspect cost impact

---

## 25. Adapter/runtime diagnostics

Paperclip agents may run through process adapters, HTTP adapters, OpenClaw gateways, Claude Code, Codex, Cursor, or custom services.

Inspect agent configuration through MCP/API first. Direct config file inspection is fallback.

Check:

- adapter type
- command or endpoint
- working directory
- environment variables present by name only
- model/provider config
- timeout
- heartbeat schedule
- reports_to
- budget limits
- last run status

Common failures:

- missing binary
- wrong working directory
- missing API key
- invalid model name
- runtime image not installed
- network endpoint unavailable
- permissions in workspace
- stale checkout/lock
- runaway loop
- budget cap exceeded

Do not fix by increasing timeouts/budgets blindly. Diagnose root cause.

---

## 26. Workspace management

Workspaces are usually under:

```text
/paperclip/workspaces/<agent-id>/
```

Read-only checks:

```sh
du -sh /paperclip/workspaces/* 2>/dev/null | sort -h | tail -n 30
find /paperclip/workspaces -maxdepth 2 -type d -name .git 2>/dev/null | sed -n '1,120p'
```

Rules:

- Workspaces are execution state, not permanent delivery.
- Do not delete workspaces while agents are running.
- Cleanups require approval and should target stale/large workspaces only.
- Preserve evidence for failed tasks before cleanup.

---

## 27. System update workflow

Before updating Paperclip:

1. Identify install method and current version/image.
2. Read release notes if available.
3. Confirm backup exists and is restorable.
4. Check disk space.
5. Stop or quiesce agents if required.
6. Export config and note env variable names.
7. Pull/build update.
8. Run migrations only as documented.
9. Start service.
10. Validate health, UI/API, companies, agents, issues, heartbeats, and logs.
11. Keep rollback path until validated.

Docker-style checks:

```sh
docker inspect "$PC_CONT" --format '{{.Config.Image}} {{.Image}}'
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
df -h
```

Do not update during active critical tasks unless approved.

---

## 28. Private API / internal endpoint caution

Some Paperclip endpoints may be private/internal and version-specific.

Rules:

- Prefer official MCP/API tools.
- Treat internal endpoints as unstable.
- Do not rely on internal endpoint paths unless confirmed on the installed version.
- Do not expose internal endpoints outside trusted networks.
- Do not use private endpoints to bypass workflow state, approvals, or auth.

---

## 29. Common playbooks

### 29.1 Paperclip UI/API down

Collect:

```sh
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker logs --tail=500 "$PC_CONT"
df -h
ss -tulpen 2>/dev/null | grep -E '3100|paperclip' || true
```

Check:

- container not running
- crash loop
- port conflict
- disk full
- database migration failure
- missing env/config
- bad auth secret/session config
- reverse proxy issue
- database unavailable

Do not restart repeatedly without reading logs.

### 29.2 Agent not waking

Collect via MCP/API when possible:

- agent profile
- heartbeat schedule
- last heartbeat run
- last error
- budget/spend
- enabled/disabled state
- adapter config
- assigned issues

Host checks:

```sh
docker logs --tail=1000 "$PC_CONT" 2>&1 | grep -iE 'heartbeat|cron|schedule|agent|adapter' | tail -n 250
```

Check:

- no assigned issues
- heartbeat disabled
- schedule misconfigured
- budget exhausted
- adapter command broken
- runtime missing credentials
- stuck previous run
- issue checkout conflict

### 29.3 Issue locked/stuck

Steps:

1. Get issue details.
2. Get current lock/checkout owner if available.
3. Inspect current heartbeat run.
4. Inspect last logs.
5. Determine whether process is still active.
6. Comment status.
7. Release/terminate only with approval or documented emergency policy.

### 29.4 Budget overrun

Steps:

1. Get cost summary for company/agent/date range.
2. Identify expensive runs.
3. Identify repeated loops/failures.
4. Pause non-critical heartbeats if approved.
5. Comment on affected tasks.
6. Recommend budget/routing/model changes.

### 29.5 Backups missing/stale

Collect:

```sh
ls -lah /paperclip/data/backups 2>/dev/null | tail -n 50
du -sh /paperclip/data/backups 2>/dev/null
docker logs --tail=1000 "$PC_CONT" 2>&1 | grep -iE 'backup|dump|archive|retention' | tail -n 200
```

Do not attempt database changes until backup issue is addressed.

### 29.6 Auth/signup exposure concern

Collect redacted config:

```sh
awk -F= '/PAPERCLIP|AUTH|SIGN|EXPOSURE|HOST|PORT/ {print $1"=<redacted>"}' /paperclip/.env 2>/dev/null
ss -tulpen 2>/dev/null | grep -E '3100|paperclip' || true
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Recommend:

- bind to localhost/private interface
- disable signup after bootstrap
- require auth for network access
- place behind HTTPS reverse proxy/VPN
- patch if installed version has known auth advisories

---

## 30. Change proposal template

Before any change:

```markdown
## Proposed Paperclip Change

Target:
Install/deployment type:
Observed problem:
Company/agent/issue scope:
Proposed change:
Files/API objects affected:
Commands/API/MCP calls:
Risk level:
Backup plan:
Rollback plan:
Expected downtime:
Validation plan:
Approval required: yes
```

After change:

```markdown
## Paperclip Change Result

Target:
Change performed:
Files/API objects changed:
Backup created:
Commands/API/MCP calls run:
Validation result:
Logs reviewed:
User-visible effect:
Rollback needed:
Remaining concerns:
```

---

## 31. Diagnostic report template

```markdown
# Paperclip Diagnostic Report

## Summary
- Overall state:
- Primary issue:
- Confidence:
- Immediate action needed:

## Instance
- Host:
- Install type:
- Container/process:
- Image/version:
- Base URL/port:
- Auth/exposure posture:
- Persistent path:

## Storage / Database
- Disk usage:
- Database mode:
- Backup status:
- Storage/artifact status:
- Workspace size:

## Control Plane
- Companies:
- Agents:
- Goals/projects:
- Issues:
- Approvals:
- Activity/audit:

## Heartbeats / Runs
- Scheduler status:
- Stuck/running runs:
- Recent failures:
- Adapter/runtime errors:
- Budget/cost state:

## Security
- Signup status:
- Token/secret handling:
- Network exposure:
- Known advisory concerns:
- Secrets migration status:

## Logs
- Main errors:
- Migration/database errors:
- Auth errors:
- Adapter errors:
- Backup errors:

## Recommended next steps
1.
2.
3.

## Actions intentionally not taken
-
```

---

## 32. First response patterns

When asked to diagnose Paperclip:

```text
I’ll treat this as Paperclip control-plane administration, not normal task execution. I’ll start read-only: identify the install/container, persistent path, config mode, auth/exposure posture, database mode, backup status, logs, heartbeat runs, agent status, budgets, and stuck issues. I won’t restart services, edit /paperclip files, touch the database, terminate runs, change auth, or modify companies/agents/issues unless you approve the specific change.
```

When asked to manage an issue/task:

```text
I’ll use the Paperclip control plane first: resolve the company and agent context, fetch the issue, check current status/lock/thread, checkout only if appropriate, perform the work in the assigned workspace, then comment, attach artifacts, and release/complete the task through Paperclip.
```

When asked to recover a broken install:

```text
I’ll first capture evidence and confirm backups before recovery. I will not reset the database, rotate keys, edit auth state, or delete persistent files unless you explicitly approve the exact recovery path and data-loss risk.
```

---

## 33. What not to do

Never do these as first-line fixes:

```sh
rm -rf /paperclip/db
rm -rf /paperclip/data/storage
rm -rf /paperclip/workspaces
rm -rf /paperclip/secrets
cat /paperclip/.env
cat /paperclip/secrets/master.key
docker compose down -v
docker volume rm ...
kill -9 <paperclip-process>
chmod -R 777 /paperclip
```

Never assume:

- `/paperclip` is the correct persistent path before discovery
- `/app` is persistent
- pglite reset is safe
- local/trusted mode is safe on a network
- signup is disabled
- admin is already claimed correctly
- MCP tools have board-level authority
- company context can be inferred from memory
- a workspace file is a delivered artifact
- a stuck issue should be force-released without checking run state
- budget limits can be bypassed by invoking another agent

---

## 34. Success criteria

A good Paperclip management agent:

- uses MCP/API for normal control-plane actions
- uses filesystem/container access only for diagnosis and recovery
- protects tenant/company boundaries
- never leaks secrets
- checks backups before destructive recovery
- understands agents, goals, issues, heartbeats, runs, budgets, approvals, and artifacts
- avoids duplicate/noisy blocked updates
- respects locks and checkout conflicts
- attaches final artifacts to issues
- handles stuck runs with evidence and approval
- treats `/paperclip` as persistent and `/app` as ephemeral
- avoids direct database edits unless in a documented recovery
- reports uncertainty clearly
- does not bypass governance or budget controls
