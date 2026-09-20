# Odoo Production Stack

> A production-oriented, containerized and multi-tenant deployment architecture for **Odoo Enterprise**, built around Docker, PostgreSQL and Nginx Proxy Manager.

This project provides a reusable infrastructure stack for deploying multiple isolated Odoo environments on a shared server while keeping the application, database, persistent storage and reverse-proxy layers containerized and independently manageable.

The goal is not to modify Odoo itself, but to provide a clean and reproducible infrastructure layer around it.

---

## Architecture

```text
                         Internet
                            │
                            ▼
                ┌──────────────────────┐
                │  Nginx Proxy Manager  │
                │    Reverse Proxy      │
                │   TLS / Routing       │
                └──────────┬───────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
     ┌─────────────────┐       ┌─────────────────┐
     │   Odoo Tenant 1 │       │   Odoo Tenant 2 │
     │    Container    │       │    Container    │
     └────────┬────────┘       └────────┬────────┘
              │                         │
              └────────────┬────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │    PostgreSQL   │
                  │    Container    │
                  └─────────────────┘
                           │
                           ▼
                    Persistent Data
```

The architecture is designed around the following principles:

* Containerized application workloads
* Shared PostgreSQL infrastructure
* Tenant isolation at the Odoo/database level
* Centralized reverse proxy
* Persistent storage outside the application lifecycle
* Reproducible deployments
* Environment-based configuration
* Easy tenant provisioning
* Clear separation between application and infrastructure concerns

---

# Features

### Containerized Odoo Enterprise

Odoo Enterprise is packaged into a dedicated Docker image using a custom Dockerfile.

The containerized application includes:

* Odoo Enterprise source
* Python runtime
* Required system dependencies
* Python dependencies
* Custom configuration
* Addons support

The application image is designed to be reproducible and deployable across environments.

---

### Multi-Tenant Ready

The stack is designed with multiple Odoo tenants in mind.

A typical deployment can look like:

```text
odoo.example.com
        │
        ▼
    Tenant A
        │
        └── PostgreSQL database: tenant_a

odoo2.example.com
        │
        ▼
    Tenant B
        │
        └── PostgreSQL database: tenant_b

odoo3.example.com
        │
        ▼
    Tenant C
        │
        └── PostgreSQL database: tenant_c
```

Each tenant can have:

* Independent Odoo configuration
* Independent PostgreSQL database
* Independent filestore
* Independent domain
* Independent lifecycle
* Independent backup/restore strategy

This allows multiple business environments to operate on the same underlying infrastructure without requiring a separate physical server for every deployment.

---

# Infrastructure Components

| Component                 | Purpose                          |
| ------------------------- | -------------------------------- |
| Odoo Enterprise           | ERP application layer            |
| PostgreSQL                | Relational database              |
| Docker                    | Application/container runtime    |
| Nginx Proxy Manager       | Reverse proxy and TLS management |
| Docker Compose            | Service orchestration            |
| Persistent Volumes        | Database and application data    |
| Environment Configuration | Deployment-specific settings     |

---

# Project Structure

```text
odoo-production-stack/
│
├── docker/
│   └── odoo/
│       └── Dockerfile
│
├── config/
│   └── odoo.conf
│
├── addons/
│
├── compose/
│   └── docker-compose.yml
│
├── tenants/
│   ├── tenant-a/
│   ├── tenant-b/
│   └── ...
│
├── scripts/
│   ├── backup.sh
│   ├── restore.sh
│   └── tenant-create.sh
│
├── .env.example
├── .gitignore
└── README.md
```

> The exact structure may evolve as the provisioning and automation layers are expanded.

---

# Deployment Model

The stack follows a layered architecture:

```text
┌─────────────────────────────────────────────┐
│                  Clients                    │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│            Nginx Proxy Manager              │
│        TLS / Reverse Proxy / Routing        │
└──────────────────────┬──────────────────────┘
                       │
              Docker Network
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
┌──────────────────┐       ┌──────────────────┐
│   Odoo Tenant A  │       │   Odoo Tenant B  │
└────────┬─────────┘       └────────┬─────────┘
         │                          │
         └────────────┬─────────────┘
                      ▼
              ┌───────────────┐
              │  PostgreSQL   │
              └───────────────┘
```

This separation makes individual layers independently replaceable and easier to operate.

---

# Getting Started

## Requirements

Before deploying the stack, make sure the host provides:

* Linux
* Docker Engine
* Docker Compose Plugin
* Sufficient CPU and RAM for the expected tenant count
* Persistent storage
* A DNS provider capable of pointing tenant domains to the host
* Odoo Enterprise source and appropriate licensing

---

## Configuration

Create the environment file:

```bash
cp .env.example .env
```

Configure deployment-specific values:

```env
POSTGRES_DB=postgres
POSTGRES_USER=odoo
POSTGRES_PASSWORD=change-me

ODOO_VERSION=18
ODOO_ADMIN_PASSWORD=change-me

TZ=UTC
```

Secrets should never be committed to the repository.

---

# Build the Odoo Image

Build the custom Odoo Enterprise image:

```bash
docker compose build odoo
```

The Dockerfile is responsible for preparing the runtime environment and installing the required Odoo dependencies.

---

# Start the Infrastructure

```bash
docker compose up -d
```

Verify running containers:

```bash
docker compose ps
```

Inspect application logs:

```bash
docker compose logs -f odoo
```

Inspect PostgreSQL logs:

```bash
docker compose logs -f postgres
```

---

# Tenant Model

A tenant represents an independent Odoo deployment context.

Example:

```text
Tenant A
├── Domain
│   └── erp.company-a.com
├── Odoo configuration
├── PostgreSQL database
└── Filestore

Tenant B
├── Domain
│   └── erp.company-b.com
├── Odoo configuration
├── PostgreSQL database
└── Filestore
```

The reverse proxy routes incoming traffic based on the requested hostname.

```text
erp.company-a.com ──► Odoo Tenant A
erp.company-b.com ──► Odoo Tenant B
erp.company-c.com ──► Odoo Tenant C
```

---

# Database Strategy

PostgreSQL runs as a dedicated service rather than being embedded into the Odoo application container.

This provides:

* Clear separation of concerns
* Persistent database storage
* Independent database lifecycle
* Easier backup and recovery
* Better operational visibility
* Ability to scale the database layer independently

The architecture is compatible with both:

```text
One PostgreSQL instance
        │
        ├── tenant_a
        ├── tenant_b
        └── tenant_c
```

and, where required:

```text
PostgreSQL instance A
        │
        └── Tenant A

PostgreSQL instance B
        │
        └── Tenant B
```

---

# Reverse Proxy

Nginx Proxy Manager provides the ingress layer for the deployment.

Responsibilities include:

* Reverse proxying
* Domain-based routing
* TLS certificate management
* HTTPS termination
* Host-based tenant routing

Example:

```text
HTTPS
  │
  ▼
Nginx Proxy Manager
  │
  ├── company-a.com ──► tenant-a
  ├── company-b.com ──► tenant-b
  └── company-c.com ──► tenant-c
```

This keeps public-facing traffic separate from the internal application network.

---

# Persistent Storage

Application state must survive container recreation.

The deployment therefore separates ephemeral containers from persistent data:

```text
Container Lifecycle
        │
        ├── Odoo container
        ├── PostgreSQL container
        └── Nginx Proxy Manager container

Persistent State
        │
        ├── PostgreSQL data
        ├── Odoo filestore
        └── Configuration / certificates
```

Removing or recreating an application container should not result in loss of business data.

---

# Backup & Recovery

The infrastructure is designed to support automated backups of:

### PostgreSQL

```bash
pg_dump
```

or database-level backup strategies for tenant databases.

### Odoo Filestore

The Odoo filestore must be backed up together with the corresponding database.

A valid Odoo backup therefore consists of:

```text
Database
    +
Filestore
    +
Configuration
```

A database-only backup is not considered a complete application recovery strategy.

---

# Security Considerations

The deployment follows several basic infrastructure security principles:

* Secrets are provided through environment configuration
* Database ports should not be exposed publicly
* PostgreSQL should communicate through the internal Docker network
* Odoo should not be directly exposed to the Internet
* HTTPS should terminate at the reverse proxy
* Persistent volumes should have controlled filesystem permissions
* Administrative credentials should be rotated from their defaults
* Production secrets must never be committed to Git

Recommended production topology:

```text
Internet
   │
   │ :443
   ▼
Nginx Proxy Manager
   │
   │ Internal Docker Network
   ▼
Odoo
   │
   │ Internal Docker Network
   ▼
PostgreSQL
```

---

# Operational Goals

This project is intended to evolve toward a complete infrastructure automation platform.

Planned capabilities include:

* [ ] Automated tenant provisioning
* [ ] Automated tenant removal
* [ ] Automated database creation
* [ ] Automated domain configuration
* [ ] Automated TLS provisioning
* [ ] Automated backup scheduling
* [ ] Automated restore
* [ ] Health checks
* [ ] Resource limits
* [ ] Centralized logging
* [ ] Prometheus metrics
* [ ] Grafana dashboards
* [ ] Ansible-based host provisioning
* [ ] CI/CD pipeline
* [ ] Disaster recovery workflow
* [ ] Zero/minimal-downtime deployment strategy
* [ ] Tenant lifecycle management

---

# DevOps Scope

This project intentionally focuses on the infrastructure surrounding Odoo rather than Odoo application development.

The main engineering concerns are:

```text
Containerization
       │
       ├── Docker
       │
       ├── Image Build
       │
       └── Runtime Isolation
       
Networking
       │
       ├── Docker Networks
       ├── Reverse Proxy
       ├── DNS
       └── TLS

Data
       │
       ├── PostgreSQL
       ├── Persistent Volumes
       ├── Backup
       └── Recovery

Operations
       │
       ├── Deployment
       ├── Monitoring
       ├── Logging
       └── Troubleshooting

Automation
       │
       ├── Docker Compose
       ├── Shell Automation
       ├── Ansible
       └── CI/CD
```

The long-term objective is to make deployment **repeatable, observable and automatable** rather than dependent on manual server configuration.

---

# Why This Project Exists

Traditional Odoo deployments can quickly become difficult to maintain when multiple custome
