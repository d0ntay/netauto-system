# Netauto

A network automation platform for a multivendor service provider, built around Nautobot as the source of truth and orchestration hub. Unifies network inventory, config management, and workflow automation behind a single API gateway and dashboard.

## Architecture

Nautobot is the core of the platform. The API gateway is a thin auth/routing layer. AWX is the execution engine called by Nautobot Jobs. All workflow logic lives in Nautobot.

```
                    ┌─────────────────────────────┐
                    │        Dashboard (React)     │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │      API Gateway (FastAPI)   │
                    │  SSO auth, routing,          │
                    │  API aggregation             │
                    └──┬────────┬────────┬────┬───┘
                       │        │        │    │
          ┌────────────▼──┐  ┌──▼───┐ ┌──▼──┐ │
          │   Nautobot    │  │ AWX  │ │SNOW │ │
          │  (Core/Brain) │  │      │ │     │ │
          │  DCIM,Circuits│  │Exec  │ │ITSM │ │
          │  IPAM,Golden  │  │Engine│ │     │ │
          │  Config,Jobs  │  │      │ │     │ │
          └──────┬────────┘  └──▲───┘ └──▲──┘ │
                 │              │         │    │
                 │   orchestrates via     │    │
                 │   Nautobot Jobs ───────┘    │
                 │                             │
              ┌──▼─────────────────────────────▼──┐
              │          Git Server                │
              │  Config backups, templates,        │
              │  playbooks, Nautobot Jobs          │
              └───────────────────────────────────┘
```

## Project Structure

```
Nauto/
├── gateway/                # FastAPI API gateway
│   └── src/gateway/        # Auth, routing, aggregation endpoints
├── nautobot-custom/        # Custom Nautobot instance
│   └── src/nautobot_custom/  # Custom models, Jobs, plugins
├── dashboard/              # React frontend (future)
└── docs/                   # Design specs and documentation
```

## Services

### Nautobot (Core)

The brain of the platform. Owns all network data and orchestrates workflows.

- **DCIM**: Devices, interfaces, cables, platforms, manufacturers
- **Circuits**: Circuit records, providers, terminations linked to device interfaces
- **IPAM**: VRFs (one per customer), prefixes, IP addresses, VLANs
- **Tenancy**: Customer modeling via Tenants, scoping all objects per customer
- **Golden Config**: Config backup, intended config generation, and compliance
- **Jobs**: Workflow orchestration across AWX, ServiceNow, and Git

### API Gateway (FastAPI)

Thin routing and auth layer. No business logic.

- SSO authentication and session management
- Per-user Nautobot API token management
- Request routing/proxying to backend services
- API aggregation (composing multi-backend data into single responses)

### Dashboard (React)

Custom frontend for engineers and project managers. Provides a user-friendly interface to Nautobot data, compliance status, and job execution.

### Ansible AWX

Execution engine only. Runs playbooks against network devices, pulls dynamic inventory from Nautobot via the `ansible-nautobot` inventory plugin, and syncs playbooks from Git.

### ServiceNow

Ticket and change management. Nautobot Jobs create/update/validate tickets via ServiceNow REST API.

### Git Server

Version control for config backups, Jinja2 templates, Ansible playbooks, and Nautobot Jobs.

## Tech Stack

- **Nautobot**: Python, Django
- **Gateway**: Python, FastAPI
- **Dashboard**: React, TypeScript
- **Config Management**: Jinja2, Golden Config
- **Device Communication**: NAPALM, Netmiko
- **Automation Execution**: Ansible, AWX
- **Infrastructure**: Docker, Docker Compose
