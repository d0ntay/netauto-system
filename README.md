# NetAuto — Network Automation Platform

**A vendor-neutral, change-driven network automation system where infrastructure changes flow through a single pane of glass with built-in validation gates, audit trails, and rollback.**

> **Portfolio Note:** This is a company project documenting the system architecture and workflows. The actual codebase and infrastructure remain private.

## The Problem It Solves

Traditional network operations suffer from fragmentation:
- **Manual processes:** Engineers SSH to individual devices, make changes ad-hoc
- **Tool sprawl:** Inventory in one system, configs in another, jobs in a third
- **Weak change gates:** No standardized approval workflow, high risk of errors
- **Lost audit trail:** Hard to track who changed what, when, and why
- **Slow rollback:** Reverting changes is manual and error-prone

**NetAuto's solution:** All network changes flow through a single platform with mandatory validation, real-time job tracking, and automated rollback.

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Engineer Dashboard (React)                  │
│  Single Pane of Glass — Devices, Inventory, Jobs, Configs      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              FastAPI Backend (Python)                            │
│  API aggregation layer — auth, validation, orchestration        │
└────────┬──────────────┬──────────────┬──────────────┬───────────┘
         │              │              │              │
    ┌────▼──┐     ┌─────▼──┐    ┌─────▼──┐    ┌─────▼──┐
    │ Git   │     │Nautobot│    │ Redis  │    │ AWS X  │
    │ Repo  │     │(Inv)   │    │(Cache) │    │(Jobs)  │
    │(SOT)  │     │        │    │        │    │        │
    └─────┬─┘     └────┬───┘    └────┬───┘    └────┬───┘
          │            │             │             │
          │ OpenConfig │ Devices,    │ Job Status, │ Ansible
          │ YAML       │ Circuits,   │ Auth, Rate  │ Tasks
          │ Intents    │ IPAM        │ Limiting   │
          │            │             │             │
         ▼            ▼             ▼             ▼
    ┌──────────────────────────────────────────────────────┐
    │         Change Execution Pipeline                    │
    │                                                      │
    │  1. Config Render (Jinja2 templates)               │
    │  2. Pre-deployment Validation (Batfish)            │
    │  3. ServiceNow Change Gate (approval)               │
    │  4. AWX Execution (device push via Ansible)        │
    │  5. Post-deployment Verification (NETCONF/SSH)    │
    │  6. Auto-rollback (on failure)                     │
    └──────────────────────────────────────────────────────┘
          │
          ├─ NETCONF (Juniper devices)
          ├─ SSH CLI (Cisco, Arista)
          └─ gRPC (internal services)
```

## Core Workflows

### 1. Device Configuration Change Workflow

**Scenario:** Engineer needs to change interface MTU on a router.

**Step-by-Step:**

1. **Engineer Views Device** (Dashboard → Devices → Search hostname)
   - Real-time device status pulled from Nautobot
   - Current running config fetched via NETCONF
   - Interface list with current MTU, status, description shown

2. **Engineer Initiates Change** (Click interface → "Set MTU" action)
   - Form opens with device-specific validation rules
   - Pre-fills current values
   - Validates new value against vendor constraints

3. **Change Request Created** 
   - System renders config change using Jinja2 templates
   - Device-specific OpenConfig YAML template applied
   - Produces configuration snippet (vendor CLI)

4. **Pre-Deployment Validation**
   - Batfish analyzes rendered config for logical errors:
     - Route conflicts?
     - MTU mismatches across path?
     - Policy violations?
   - If validation fails → change rejected with error explanation
   - If valid → proceeds

5. **ServiceNow Gate**
   - User selects existing ServiceNow change ticket (CHG-XXXXX)
   - System verifies ticket is approved and within change window
   - Creates change record in audit log

6. **Execution via AWX**
   - FastAPI submits Ansible playbook to AWX with rendered config
   - AWX connects to device (SSH/NETCONF)
   - Applies configuration
   - Returns job status + device output

7. **Post-Deployment Verification**
   - System pulls running config from device (NETCONF for Juniper, SSH for Cisco/Arista)
   - Parses config using multi-vendor parser → OpenConfig model
   - Diffs OpenConfig intent vs running config
   - If diff = 0 → change successful, logged
   - If diff > 0 → automatic rollback triggered

8. **Rollback on Failure** (automatic)
   - Git reverts config intent to previous version
   - System re-renders config from reverted intent
   - AWX executes rollback playbook
   - Device returns to pre-change state
   - Engineer notified with failure reason

**Timeline:** 30 seconds to 2 minutes end-to-end (depends on device response time)

**Audit Trail:**
- Change request timestamp, author, ticket ID
- Pre-change config snapshot
- Rendered config sent to device
- AWX job output
- Verification result (passed/failed)
- Rollback record (if triggered)

---

### 2. Bulk Device Onboarding Workflow

**Scenario:** New 10-device site added to network. Need to provision all devices with base config.

**Step-by-Step:**

1. **Inventory Import** (Settings → Import Devices)
   - CSV upload: hostname, IP, device_type, site, contract
   - System validates against naming conventions
   - Creates devices in Nautobot inventory
   - Assigns roles based on device_type (router, switch, firewall)

2. **IPAM Allocation** (IPAM page)
   - Allocate IP prefixes from reserved pools
   - Assign loopback IPs, management IPs
   - DNS entries auto-created
   - Circuit associations added

3. **Config Generation** (Config page → Generate)
   - System renders base config for all 10 devices
   - Uses device-specific templates (Juniper vs Cisco vs Arista)
   - Renders interfaces, routing, OSPF, BGP based on site topology
   - Preview diff before applying

4. **Batch Change Request**
   - One ServiceNow ticket (CHG-XXXXX) covers all 10 devices
   - Engineer selects "bulk apply" option
   - System creates change records for each device

5. **Parallel Execution**
   - AWX launches 10 concurrent playbooks (one per device)
   - Each playbook:
     - Connects to device
     - Applies base config
     - Verifies connectivity
   - Progress tracked in real-time (dashboard shows % complete)

6. **Verification & Rollback**
   - For each device:
     - Pull running config
     - Parse to OpenConfig
     - Verify against intent
     - On failure: auto-rollback individual device
   - Summary report: 9/10 successful, 1 failed (with reason)

7. **Post-Onboarding**
   - Devices appear in topology visualization
   - Health checks run (OSPF state, BGP session status)
   - Alerts triggered if any session down
   - Devices ready for production

**Timeline:** 5-10 minutes for 10 devices (parallel execution)

**Audit Trail:** Per-device log of config, job ID, verification result, author, ticket

---

### 3. Rollback Workflow

**Scenario:** Deployed config broke routing. Need to rollback immediately.

**Triggered By:**
- Automatic: Post-deployment verification failed (config doesn't match intent)
- Manual: Engineer clicks "Rollback" button

**Automatic Rollback (Post-failure):**
1. Verification detected mismatch (e.g., rendered config for MTU 9000 but device shows 1500)
2. System diffs actual vs expected → mismatch confirmed
3. Git automatically reverts config intent to previous commit
4. Re-renders config from reverted intent
5. AWX applies reverted config
6. Verification confirms match
7. Device returns to pre-change state
8. Change marked as "ROLLED_BACK" in audit log

**Manual Rollback (Emergency):**
1. Engineer: Dashboard → Jobs → [select failed job] → "Rollback"
2. System requires ServiceNow ticket + reason
3. Same steps as automatic rollback
4. Logged as manual rollback with engineer name

**Safety Mechanisms:**
- Double-failure protection: If rollback itself fails → device locked (requires manual intervention)
- Prevents cascading failures
- OnCall engineer notified immediately
- All changes require ServiceNow gate (prevents accidental deploys)

**Timeline:** 30-60 seconds to return device to stable state

---

### 4. Network Inventory & IPAM Workflow

**Scenario:** Need IP address for new customer circuit interface.

**Step-by-Step:**

1. **View Inventory** (Dashboard → Inventory)
   - See all sites, devices, circuits, contracts
   - Real-time sync with Nautobot

2. **Allocate IP Address** (IPAM page)
   - Select site and subnet
   - Drag-and-drop IP allocation (visual IPAM)
   - Assign to device interface
   - Circuit data links IP to service contract

3. **Config Auto-Generation**
   - System detects new IP allocation
   - Auto-updates interface config intent
   - Renders new interface config
   - Awaiting engineer approval

4. **Engineer Approves & Deploys**
   - Review rendered config change
   - Attach ServiceNow change ticket
   - Deploy via dashboard (AWX executes)
   - Verification confirms IP is operational

**Why This Matters:**
- IPAM is source of truth for addressing
- Config intents auto-generate from IPAM changes
- No manual config writing for interface IPs
- Eliminates duplicate IPs and address conflicts

---

### 5. Job Tracking & Monitoring Workflow

**Scenario:** Engineer deploys config across 50-device site. Needs real-time visibility.

**Dashboard Experience:**

1. **Batch Job Submitted**
   - Engineer: Config page → "Apply to all devices" → selects site
   - System submits 50 parallel AWX jobs
   - Dashboard shows: Job ID, site, device count, status

2. **Real-Time Polling**
   - Dashboard polls job status every 5 seconds
   - Shows:
     - **Pending:** Waiting for AWX
     - **Running:** N/50 devices completed
     - **Verifying:** Post-deployment validation in progress
     - **Success:** Device config matches intent
     - **Failed:** Device verification failed, rollback triggered

3. **Drill-Down**
   - Click device in list → see:
     - AWX job output (command-by-command)
     - Pre-change config snapshot
     - Rendered config applied
     - Verification result
     - Diff (if mismatch)

4. **Alerts**
   - Failure notification (Slack/email)
   - Automatic rollback notification
   - Time threshold alerts (job taking too long)

5. **Historical Tracking**
   - Jobs page shows all changes over past 90 days
   - Filter by status (success, failed, rolled back)
   - Filter by engineer, device, site, ticket
   - Export audit trail for compliance

**Key Metrics Visible:**
- Success rate (49/50 devices)
- Average deployment time (3m 22s)
- Failure details with root cause (e.g., "device unreachable")
- Rollback triggers (auto vs manual)

---

## Multi-Vendor Support

The system abstracts vendor differences through layered architecture:

### Layer 1: Config Intents (Vendor-Neutral)
```yaml
# intents/vtx/smke-vtx-pe1-r/openconfig-interfaces.yaml
interfaces:
  - name: ge-0/0/0
    enabled: true
    mtu: 9192
    description: "Link to SMKE-VTX-PE2"
```

### Layer 2: Vendor-Specific Templates (Jinja2)
```jinja2
{# templates/juniper/openconfig-interfaces.j2 #}
set interfaces {{ interface.name }} mtu {{ interface.mtu }}
set interfaces {{ interface.name }} description "{{ interface.description }}"
set interfaces {{ interface.name }} unit 0 family inet address {{ interface.ipv4 }}/24
```

### Layer 3: Device Connection (NETCONF/SSH)
```
Juniper → NETCONF (structured, validation included)
Cisco/Arista → SSH CLI (unstructured, raw CLI)
```

### Layer 4: Verification (Unified Parser)
```
Device output → Multi-vendor parser → OpenConfig model → Diff against intent
```

**Result:** Same workflow for all vendors. Vendor-specific logic isolated to templates and parser.

---

## Change Control Gates

Every change must pass these gates before reaching devices:

### Gate 1: Config Validation
- Syntax validation (template rendering succeeds)
- Logical validation (Batfish analysis)
- Policy compliance checks
- **Failure:** Change rejected with explanation

### Gate 2: ServiceNow Integration
- Engineer selects existing CHG ticket
- System verifies:
  - Ticket is approved
  - Change window is open
  - Ticket owner matches engineer
- **Failure:** Change blocked until approved in ServiceNow

### Gate 3: Post-Deployment Verification
- Rendered config matches device running config
- No config drift post-change
- **Failure:** Automatic rollback

### Gate 4: Role-Based Authorization
- 4 role tiers determine who can deploy where
  - **NOC Tier 1:** Read-only (monitor only)
  - **NOC Tier 2:** Can view configs, approve changes
  - **Engineer:** Can deploy to production (requires CHG ticket)
  - **Manager:** Admin access (rare)

---

## Data Sources & Integrations

### Source of Truth #1: Git Repository
```
intents/
├── vtx/
│   ├── smke-vtx-pe1-r/
│   │   ├── openconfig-interfaces.yaml
│   │   ├── openconfig-routing.yaml
│   │   └── openconfig-bgp.yaml
│   └── smke-vtx-pe2-r/
├── maglev/
└── xlink/
```
- **Single source of truth for configs**
- Immutable commit history (audit trail)
- Git revert = automatic rollback
- Config diffs show exactly what changed

### Source of Truth #2: Nautobot (Inventory API)
- Devices (hostname, IP, device_type, site, contract)
- Interfaces (name, speed, enabled status)
- Circuits (provider, service type, bandwidth, costs)
- IPAM (prefixes, individual IPs, DNS)
- Real-time sync (updates push to Redis cache)

### Integration #3: ServiceNow (Change Gate)
- Change request lookup (CHG tickets)
- Approval status verification
- Change window validation
- Audit integration (change records)

### Integration #4: AWS X / Ansible AWX (Execution)
- Job queue (submits playbooks)
- Multi-vendor playbook support
- SSH key management
- Job history + output storage
- Status polling (live job tracking)

### Integration #5: Redis (Cache & Queue)
- Device inventory cache (updates every 5 min)
- Job status queue (real-time polling)
- Session tokens (auth)
- Rate limiting (API throttling)

---

## Why This Architecture

### 1. **Separation of Concerns**
- **Git:** Config intent storage
- **Nautobot:** Inventory/IPAM
- **AWX:** Job execution
- **FastAPI:** Orchestration glue
- **Redis:** Caching + session management

Each tool does one thing well. No monolith.

### 2. **Auditability**
Every change produces:
- Git commit (config intent)
- ServiceNow record (approval)
- AWX job output (what was executed)
- Verification log (success/failure)
- Rollback record (if triggered)

**90-day retention** for compliance.

### 3. **Safety**
- **Mandatory change gates** (ServiceNow approval required)
- **Pre-deployment validation** (Batfish checks logic)
- **Post-deployment verification** (confirm change worked)
- **Automatic rollback** (on failure)
- **Device locks** (double-failure protection)

### 4. **Multi-Vendor Abstraction**
- Juniper, Cisco, Arista all use same workflow
- Vendor logic isolated to templates + parser
- Engineer doesn't need to know CLI syntax
- Easy to add new vendors (template + parser module)

### 5. **Scalability**
- Parallel job execution (AWX runs N devices simultaneously)
- Redis caching reduces Nautobot load
- Config rendering is stateless (scales horizontally)
- Batch operations (50+ devices in single change)

---

## Engineer Experience

### What Engineers Can Do (via Dashboard)

✅ **View** — Real-time device status, running configs, interface state  
✅ **Search** — Find devices by hostname, IP, site, contract  
✅ **Modify** — MTU, description, speed, duplex, enable/disable  
✅ **Track** — Real-time job progress, 90-day history  
✅ **Rollback** — One-click rollback (if change fails)  
✅ **Audit** — See who changed what, when, why (ServiceNow ticket)  

### What Engineers Cannot Do

❌ SSH to devices (no CLI access)  
❌ Bypass change gates (ServiceNow required)  
❌ Make untracked changes (all changes audited)  
❌ Manually edit configs (only via dashboard workflows)  
❌ Deploy without approval (role-based gates)  

**Result:** Reduced risk, full visibility, complete audit trail.

---

## System Metrics & Observability

### Operational KPIs

| Metric | Target | Current |
|--------|--------|---------|
| Change Success Rate | 99%+ | 98.7% |
| Time to Deploy (single device) | < 5 min | 2-3 min |
| Time to Deploy (50 devices) | < 15 min | 8-12 min |
| Automatic Rollback Rate | < 1% | 0.3% |
| Manual Rollback Time | < 60 sec | 30-45 sec |
| Job Status Poll Accuracy | 100% | 99.8% |

### Infrastructure Scale

- **Device Inventory:** 500+ devices
- **Daily Changes:** 50-100 config changes
- **Concurrent Jobs:** 20-50 parallel AWX playbooks
- **Job Queue Latency:** < 2 seconds
- **Verification Latency:** < 5 seconds per device

---

## Why This Approach Works for Network Teams

### Before (Manual)
- Engineer SSHes to device
- Edits config in editor
- Applies change manually
- Documents change in separate ticket system
- Rollback is manual copy-paste

**Problems:** Inconsistent, error-prone, slow rollback, weak audit

### After (NetAuto)
- Engineer uses dashboard
- System renders config from intent
- Change validated (Batfish)
- Approval gated (ServiceNow)
- Auto-executed (AWX)
- Auto-verified (NETCONF/SSH)
- Auto-rolled-back (on failure)
- Full audit trail (Git + ServiceNow)

**Benefits:** Consistent, safe, fast, auditable, scalable

---

## Technical Principles

1. **Intent-Driven:** Config intents are source of truth, not running configs
2. **Vendor-Neutral:** OpenConfig abstracts vendor differences
3. **Immutable History:** Git as audit log (config + who + when + why)
4. **Gate Everything:** Validation + approval + verification + rollback
5. **Fail Safe:** Auto-rollback prevents extended outages
6. **Full Visibility:** Real-time job tracking + 90-day audit trail

---

## What This Demonstrates

✅ **Network Automation Architecture** — End-to-end system design  
✅ **Change Management** — Approval gates, verification, rollback  
✅ **Multi-Vendor Support** — Abstraction layer (OpenConfig)  
✅ **Real-Time Operations** — Job tracking, auto-refresh, alerts  
✅ **Audit & Compliance** — 90-day history, change gates, approval tracking  
✅ **Infrastructure as Code** — Git-driven configuration  
✅ **Risk Mitigation** — Validation, verification, automatic rollback  
✅ **Scalability** — Parallel execution, caching, stateless design  

---

**Summary:** A production network automation platform that replaced manual device configuration with a validated, auditable, vendor-neutral change control system. Every change flows through multiple safety gates, executes in parallel, verifies automatically, and rolls back on failure. Designed to scale from 50 to 500+ devices with full audit trail.
