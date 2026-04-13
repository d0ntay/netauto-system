# NetAuto — Network Automation Platform

A platform that consolidates network inventory, configuration management, change control, and job execution into a single system. Engineers interact with the network exclusively through the dashboard — no direct device access, no untracked changes, no tool sprawl.

> This is a company project. The architecture, workflows, and design decisions are documented here. The codebase remains private.

---

## The Problem

Our network operations relied on disconnected tools and manual processes:

- **No inventory source of truth.** Device data lived in spreadsheets, monitoring tools, and tribal knowledge. Nobody agreed on what we had or where it was.
- **No config source of truth.** Running configs were the only record. If someone changed something and didn't document it, it was invisible.
- **Manual device-by-device changes.** Deploying a policy across 50 devices meant SSHing into each one individually. Slow, error-prone, inconsistent.
- **No change audit trail.** When something broke, figuring out what changed, who changed it, and why was a forensics exercise.
- **No config drift detection.** Devices drifted from intended state silently. We only found out when something broke.
- **No change gates.** Anyone with credentials could make changes. No approval workflow, no validation, no rollback plan.

NetAuto replaces all of that with a single system where every change is validated, approved, tracked, and reversible.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Dashboard (React)                           │
│       The only interface engineers use to interact with the network  │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Service Gateway (FastAPI)                          │
│  Aggregation layer — every request flows through here                │
│  Auth, validation, orchestration, rate limiting                      │
└──────┬──────────┬──────────┬──────────┬──────────┬───────────────────┘
       │          │          │          │          │
  ┌────▼───┐ ┌───▼────┐ ┌───▼───┐ ┌───▼────┐ ┌───▼───┐
  │Nautobot│ │  Git   │ │  AWX  │ │ SNOW   │ │ Redis │
  │        │ │  Repo  │ │       │ │        │ │       │
  └────────┘ └────────┘ └───────┘ └────────┘ └───────┘
  Inventory   Config     Ansible   Change     Cache,
  SOT         SOT        Execution Approval   Locks,
  (devices,   (OpenConfig (playbook (ticket    Sessions
   circuits,  YAML        runs)    validation,
   IPAM)      intents)             work notes)
```

**Two sources of truth, one gateway, everything else is orchestration.**

- **Nautobot** owns inventory: devices, interfaces, circuits, IPAM, sites.
- **Git** owns configuration: OpenConfig YAML intent files, versioned with full commit history.
- **Service Gateway** ties it all together. Every dashboard action goes through the gateway, which coordinates between the backend services.

---

## How It Works

### The Service Gateway

The FastAPI service gateway is the central nervous system. It doesn't store data — it aggregates and orchestrates across all backend services through a single API surface.

**What it handles:**

| Domain | What the Gateway Does |
|--------|----------------------|
| **Devices** | Queries Nautobot GraphQL for device inventory. Normalizes vendor names, formats interface data, exposes search and filtering. |
| **Configuration** | Reads/writes OpenConfig YAML intents to Git. Renders intents to vendor CLI using Jinja2 templates. Submits rendered configs to AWX for deployment. |
| **Change Execution** | Orchestrates the full change pipeline: lock device → commit intent → render config → push via AWX → verify → rollback on failure. |
| **Verification** | Pulls running config from devices via NETCONF or SSH, parses it to OpenConfig, diffs against the Git intent. |
| **Drift Detection** | Compares intended config (Git) against actual running config (device) on demand or in bulk. Caches results per device. |
| **Job Tracking** | Fetches AWX job history, applies filters, auto-tags jobs based on ticket type and action. |
| **ServiceNow** | Validates that a change ticket exists and is approved before allowing any change. Records results as work notes on the ticket after execution. |
| **Auth** | JWT-based authentication with 4-tier role-based access control. Rate-limited login, token blocklist, refresh rotation. |
| **Circuits & IPAM** | Proxies CRUD operations to Nautobot for circuit, prefix, VLAN, and IP address management. All mutations require a validated ServiceNow ticket. |
| **Topology** | Queries Nautobot for sites, devices, interfaces, and circuits, then formats the data for D3.js network graph rendering. |

---

### Config Intent Model

Configuration is managed as **vendor-neutral OpenConfig YAML intents**, stored in Git.

```
intents/
├── vtx/
│   ├── smke-vtx-pe1-r/
│   │   ├── openconfig-interfaces.yaml
│   │   ├── openconfig-routing.yaml
│   │   └── openconfig-bgp.yaml
│   └── smke-vtx-pe2-r/
│       └── ...
├── maglev/
└── xlink/
```

Each device has its own directory under its contract. Intent files describe the desired state of the device in OpenConfig format:

```yaml
# intents/vtx/smke-vtx-pe1-r/openconfig-interfaces.yaml
device: smke-vtx-pe1-r
vendor: juniper
model: interfaces
interfaces:
  - name: ge-0/0/0
    enabled: true
    mtu: 9192
    description: "Link to SMKE-VTX-PE2"
```

When an engineer makes a change through the dashboard, the gateway:
1. Updates the relevant intent file in Git (merging the changed interface into the existing file)
2. Renders the intent to vendor-specific CLI using Jinja2 templates
3. Pushes the rendered config to the device via Ansible

**Why Git?**
- Every change is a commit with author, timestamp, ticket number, and description
- Rollback is a `git revert` — restore any previous state instantly
- Diff any two points in time to see exactly what changed
- Full history of every config change ever made to every device

---

### Multi-Vendor Abstraction

The system supports Juniper, Cisco, and Arista through a layered abstraction:

**Layer 1 — Intent (vendor-neutral):** OpenConfig YAML describes desired state. Same format regardless of vendor.

**Layer 2 — Rendering (vendor-specific):** Jinja2 templates at `templates/{vendor}/openconfig-{model}.j2` translate intents into vendor CLI. Adding a new vendor means writing new templates — the rest of the pipeline stays the same.

**Layer 3 — Transport (vendor-specific):**
- Juniper → NETCONF (port 830, structured XML)
- Cisco → SSH CLI
- Arista → SSH CLI

**Layer 4 — Parsing (vendor-specific):** A custom multi-vendor config parser reads running configs and normalizes them back to OpenConfig format. This is what makes verification possible — we can diff the intent against the actual device state in a common format.

---

### The Change Pipeline

Every configuration change follows the same pipeline. No shortcuts, no exceptions.

```
Engineer initiates change in dashboard
        │
        ▼
┌─ Gate 1: ServiceNow Validation ──────────────────────────┐
│  Is there an approved ticket (INC/CHG) for this change?  │
│  Is the change window open?                               │
│  NO → Change blocked                                      │
└──────────────────────────┬───────────────────────────────┘
                           │ YES
                           ▼
┌─ Gate 2: Device Lock ────────────────────────────────────┐
│  Acquire Redis lock on target device (600s TTL)          │
│  Prevents concurrent changes to same device              │
│  LOCKED → Change rejected (device in use by another      │
│           ticket)                                         │
└──────────────────────────┬───────────────────────────────┘
                           │ ACQUIRED
                           ▼
┌─ Step 3: Commit Intent to Git ───────────────────────────┐
│  Merge interface changes into device intent file         │
│  Commit with ticket number + change description          │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌─ Step 4: Render Config ──────────────────────────────────┐
│  Load Jinja2 template for vendor + model                 │
│  Render OpenConfig intent → vendor CLI commands          │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌─ Step 5: Execute via AWX ────────────────────────────────┐
│  Submit Ansible playbook with rendered config            │
│  AWX connects to device (SSH/NETCONF)                    │
│  Applies configuration                                    │
│  Gateway polls job status (5s intervals, 10 min timeout) │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌─ Gate 6: Post-Change Verification ───────────────────────┐
│  Pull running config from device (NETCONF or SSH)        │
│  Parse running config → OpenConfig (custom parser)       │
│  Diff running config vs Git intent                       │
│                                                           │
│  MATCH → Success. Record result to SNOW work notes.      │
│          Release device lock.                             │
│                                                           │
│  MISMATCH → Automatic rollback triggered                 │
│     1. Git reverts intent to previous commit             │
│     2. Re-render config from reverted intent             │
│     3. AWX pushes reverted config to device              │
│     4. If rollback succeeds → device restored            │
│     5. If rollback ALSO fails → permanent device lock    │
│        (requires manual intervention)                     │
└──────────────────────────────────────────────────────────┘
```

**Every step produces an audit record:** Git commit, AWX job output, verification result, SNOW work note. If something goes wrong six months from now, we can trace exactly what happened.

---

### ServiceNow Integration

ServiceNow is the approval gate. No change goes through without a validated ticket.

**Before a change executes, the gateway:**
1. Takes the ticket number the engineer provides (INC-XXXXX or CHG-XXXXX)
2. Authenticates to ServiceNow via OAuth 2.0
3. Queries the ticket table to verify it exists, is approved, and the change window is active
4. If validation fails, the change is blocked

**After a change executes, the gateway:**
1. Writes the result (success, failure, rollback) as a work note on the ServiceNow ticket
2. Includes the job ID, device name, and verification outcome

This means the ServiceNow ticket becomes a complete record of what was requested, approved, executed, and verified — without anyone manually updating it.

---

### Drift Detection

Config drift is when a device's running config doesn't match its intended state. This happens when someone makes an out-of-band change, a device reloads with a different config, or a change partially fails.

**How drift detection works:**
1. Read the device's intended config from Git (OpenConfig YAML)
2. Pull the device's actual running config (NETCONF for Juniper, SSH for Cisco/Arista)
3. Parse the running config through the multi-vendor parser → normalize to OpenConfig
4. Diff the intent against the parsed running config
5. Report: CLEAN (matches), DRIFTED (differences found), UNKNOWN (no intent file), or ERROR

**The diff is field-level:** It doesn't just say "these configs are different" — it reports exactly which interfaces, which fields (MTU, description, enabled state), what the expected value is, and what the actual value is.

Drift checks can run on-demand for a single device or in bulk across the entire inventory with bounded concurrency.

---

### Verification vs Drift Detection

These use the same underlying logic but serve different purposes:

- **Verification** runs immediately after a change is pushed. It confirms the change was applied correctly. If it fails, automatic rollback is triggered.
- **Drift detection** runs independently. It checks whether devices have drifted from their intended state over time. It reports but doesn't auto-remediate.

Both use the same `pull_and_verify()` function: pull running config → parse to OpenConfig → diff against intent.

---

### Device Locking

The gateway uses Redis-based mutual exclusion to prevent concurrent changes to the same device.

**Two lock tiers:**

1. **Standard lock** (TTL 600 seconds): Acquired when a change starts, released when it completes. Prevents two engineers from changing the same device simultaneously. The lock value is the ticket number, so if a lock is held, the error message tells you which ticket is using the device.

2. **Permanent lock** (no TTL): Triggered when a rollback fails — meaning both the original change AND the rollback attempt failed. The device is locked until someone manually investigates and releases it. This prevents cascading failures.

---

### Rollback

Rollback can be triggered automatically or manually.

**Automatic rollback** fires when post-change verification detects a mismatch between the intent and the running config. The gateway:
1. Checks that prior Git state exists for the device
2. Reverts the Git commit (creating a new "ROLLBACK" commit)
3. Re-renders config from the reverted intent
4. Pushes the reverted config via AWX
5. If the rollback job fails → permanent device lock

**Manual rollback** is available to engineers through the dashboard. It follows the same steps but requires a ServiceNow ticket and a reason.

**The key insight:** Because Git is the config source of truth and every change is a commit, rollback is just "revert to the previous commit and re-render." No manual config reconstruction.

---

### Role-Based Access Control

Four roles with hierarchical permissions:

| Role | Can Do |
|------|--------|
| **NOC Tier 1** | View dashboard, devices, jobs, alerts. Read-only. |
| **NOC Tier 2** | Everything above + execute simple operational actions (interface bounce). |
| **Engineer** | Everything above + submit config changes, rollback, manage inventory (IPAM, circuits, devices), view running configs. |
| **Manager** | Everything above + approve engineer-level changes, override workflows. |

**How it's enforced:** Every API endpoint declares a minimum role. The auth middleware extracts the JWT, checks the user's role rank, and rejects requests that don't meet the threshold. The dashboard dynamically shows/hides navigation sections based on the user's role.

**Auth details:**
- JWT access tokens (15 min expiry) + HTTP-only refresh token cookies (7 day expiry)
- Bcrypt password hashing
- Rate-limited login (5 attempts per minute per username, enforced via Redis)
- Token blocklist for logout (Redis TTL matches remaining token life)

---

### Job Tracking & Auto-Tagging

Every Ansible job submitted through AWX is tracked and queryable through the dashboard.

**What engineers see:**
- Job history with status (queued, running, successful, failed)
- Filtering by status, vendor, device, action type, date range
- Full job stdout (command-by-command output from the device)
- Tags for categorization

**Auto-tagging:** When jobs are fetched, the gateway automatically derives tags from the job data:
- Ticket prefix → tag: `INC*` becomes "incident-driven", `CHG*` becomes "cr-driven", no ticket becomes "manual"
- Action type → tag: `set_description` becomes "set-description", `bounce_interface` becomes "bounce-interface"

Tags are stored in PostgreSQL. Auto-tags are read-only (can't be modified or deleted). Engineers can also create and assign manual tags for their own categorization.

---

## Services Summary

| Service | Custom or Off-the-Shelf | Purpose |
|---------|------------------------|---------|
| **Service Gateway (FastAPI)** | Custom | API aggregation, auth, orchestration, change pipeline |
| **Dashboard (React)** | Custom | Engineer-facing UI, single pane of glass |
| **Change Engine** | Custom | Orchestrates lock → commit → render → push → verify → rollback |
| **Drift Service** | Custom | Compares Git intents vs running configs, reports drift |
| **Verification Service** | Custom | Post-change config validation, triggers rollback on failure |
| **Config Parser** | Custom | Multi-vendor config → OpenConfig normalization |
| **Template Renderer** | Custom | OpenConfig YAML → vendor CLI via Jinja2 |
| **Git Client** | Custom | Intent CRUD, merge logic, revert for rollback |
| **Device Lock Service** | Custom | Redis-based mutual exclusion with permanent escalation |
| **ServiceNow Client** | Custom | OAuth ticket validation, work note recording |
| **NETCONF Client** | Custom | Async config pull for Juniper, Cisco, Arista |
| **Auth System** | Custom | JWT, RBAC, rate limiting, token lifecycle |
| **Auto-Tagger** | Custom | Derives job tags from ticket prefix + action type |
| **Nautobot** | Off-the-shelf | Inventory source of truth (API-only, no UI exposed) |
| **Ansible AWX** | Off-the-shelf | Job execution engine (runs playbooks on devices) |
| **Redis** | Off-the-shelf | Caching, device locks, auth tokens, rate limiting |
| **PostgreSQL** | Off-the-shelf | Auth database (users, tags, saved filters) |
| **ServiceNow** | Off-the-shelf (company instance) | Change approval and audit |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS, D3.js, TanStack Query |
| Service Gateway | Python, FastAPI, async httpx, Pydantic |
| Config Management | Git (GitPython), Jinja2, OpenConfig YAML |
| Device Communication | NETCONF (ncclient), SSH (Paramiko/Netmiko), Ansible |
| Inventory | Nautobot 3.x (GraphQL + REST API) |
| Job Execution | Ansible AWX |
| Change Approval | ServiceNow (OAuth 2.0 REST API) |
| Auth | JWT (PyJWT), bcrypt, Redis (token store + blocklist) |
| Database | PostgreSQL (users, tags, filters), Redis (cache, locks, sessions) |
| Infrastructure | Kubernetes (k3s), Docker, Nginx |

---

## What This Project Demonstrates

- **End-to-end network automation architecture** — from intent to device, with validation at every step
- **Custom service development** — 13 custom services built to solve specific operational problems
- **Multi-vendor abstraction** — single workflow for Juniper, Cisco, and Arista through OpenConfig and templating
- **Change control discipline** — mandatory approval gates, post-change verification, automatic rollback
- **Infrastructure as Code** — Git as the config source of truth with full commit history
- **System integration** — coordinating Nautobot, AWX, ServiceNow, Git, Redis, and PostgreSQL through a unified API
- **Operational thinking** — drift detection, device locking, double-failure protection, audit trails
