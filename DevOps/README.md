# DevOps & Infrastructure — Interview Preparation

Skill fundamentals for a **Revenue Apps / Development & Infrastructure** role in a financial services environment.

---

## Skill Index

| File | Topics Covered |
|------|---------------|
| [shell-scripting.md](shell-scripting.md) | Bash scripting, variables, loops, functions, arrays, error handling, traps, text processing |
| [unix-linux.md](unix-linux.md) | Filesystem, process management, memory, networking, systemd, logging, permissions, performance tuning |
| [ansible.md](ansible.md) | Architecture, playbooks, modules, roles, inventory, Vault, Jinja2 templates, best practices |
| [docker.md](docker.md) | Images, Dockerfile, containers, networking, volumes, Compose, security, multi-stage builds |
| [rdbms-mysql.md](rdbms-mysql.md) | ACID, SQL (DDL/DML), JOINs, CTEs, window functions, indexes, transactions, replication, ILM |
| [elk-stack.md](elk-stack.md) | Elasticsearch (queries, aggregations, ILM), Logstash pipelines, Filebeat, Kibana, cluster ops |
| [git-bitbucket.md](git-bitbucket.md) | Git internals, branching strategies, merging/rebasing, hooks, Bitbucket Pipelines, PR workflow |
| [python-django-web.md](python-django-web.md) | Python OOP/concurrency, Django ORM/views/security, HTML5 APIs, CSS layout, modern JS (ES6+) |
| [control-m.md](control-m.md) | Architecture, job definitions, dependencies, variables, calendars, EOD batch, Automation API |
| [openshift.md](openshift.md) | OCP vs K8s, SCCs, Routes, ImageStreams, BuildConfigs, RBAC, Operators, monitoring, `oc` CLI |

---

## Study Approach

1. **Fundamentals first** — understand the core concept before memorising commands
2. **Draw architectures** — be able to sketch how components connect
3. **Explain tradeoffs** — interviewers want to know *why*, not just *how*
4. **Tie to financial context** — relate answers to EOD batch, trading systems, low-latency, compliance, audit

## Key Finance/Banking Themes
- **Availability & resilience** — HA design, failover, no single point of failure
- **Security & compliance** — audit logs, least-privilege, secrets management, change control
- **SLA / STP** — batch job deadlines, monitoring, alerting on late jobs
- **Change management** — controlled deployments, rollback strategies
- **Data integrity** — ACID, idempotent scripts, reconciliation
