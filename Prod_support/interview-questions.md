# Production Support / SRE Interview Questions

Preparation guide for the production-support and SRE role described in `jd.md`. The role combines application support, site reliability engineering, cloud operations, controlled change, incident management, and risk governance in a regulated banking environment.

> **How to use this guide:** Do not memorise the sample answers word for word. Prepare one real example for incidents, automation, a risky change, a difficult stakeholder conversation, and a service improvement. Explain the decision, the evidence, the trade-off, and the measurable result.

## What the JD Is Testing

| JD theme | What the interviewer is likely assessing |
|---|---|
| Production stability | Can you restore service quickly while protecting data and avoiding unsafe actions? |
| Incident and problem management | Can you separate recovery from root-cause investigation and prevent recurrence? |
| Change and release control | Can you plan dependencies, validation, rollback, and communications before a production change? |
| Monitoring and alerting | Can you design useful signals instead of creating noisy dashboards? |
| Automation | Can you make repetitive operations consistent, auditable, and safe to repeat? |
| AWS, Linux, Kubernetes, Oracle | Can you troubleshoot across application, platform, database, and network boundaries? |
| Risk and governance | Can you identify risk, track mitigation, preserve evidence, and escalate appropriately? |
| Conduct in banking | Can you protect confidentiality, maintain auditability, and make decisions that support fair client outcomes? |

## Answering Strategy

Use a concise structure for experience-based questions:

1. **Situation:** What service, users, or business process was affected?
2. **Impact:** What was the severity, scope, and time sensitivity?
3. **Actions:** What did you check, change, communicate, and record?
4. **Result:** How was service restored, and what evidence proves the outcome?
5. **Learning:** What permanent control, automation, alert, or documentation did you add?

For technical scenarios, use this sequence:

1. Confirm the symptom, timeframe, affected scope, and recent changes.
2. Establish the user and business impact and assign incident severity.
3. Stabilise safely using a known runbook, rollback, failover, or traffic reduction.
4. Collect evidence before making destructive changes.
5. Isolate the failing layer: client, DNS/network, load balancer, application, container, database, or dependency.
6. Communicate facts, actions, risks, and next update time.
7. Validate recovery from both technical and business perspectives.
8. Complete RCA, problem management, and service improvement actions.

# Production Support, SRE, and ITIL

## 1. What does good production support mean to you?

**What a strong answer should cover:**

- Restore service quickly, but do not trade away data integrity, security, or auditability.
- Work from customer and business impact, not only from CPU or error graphs.
- Use monitoring, runbooks, automation, and clear ownership to reduce mean time to detect and mean time to restore.
- Treat recurring incidents as problem-management work and turn learning into measurable service improvements.
- Keep stakeholders informed with accurate updates and preserve a useful timeline.

**Why it matters:** The JD asks for stability, fast response, prevention, monitoring, automation, and continual improvement. A support engineer is expected to operate the service, not merely react to tickets.

**Likely follow-up:** Which reliability metrics would you report? Discuss availability, MTTD, MTTA, MTTR, incident recurrence, change failure rate, alert quality, and recovery-test results. Explain that a metric should lead to an action.

## 2. Explain the difference between an incident, problem, change, and service request.

**Answer details:**

- **Incident:** An unplanned interruption or reduction in service quality. The immediate objective is restoration.
- **Problem:** The underlying cause, or a recurring pattern of incidents. The objective is preventing recurrence.
- **Change:** An addition, modification, or removal that could affect a service. It requires risk assessment, approval, implementation, validation, and a backout plan.
- **Service request:** A standard, pre-defined request such as access, a report, or a routine operational action.

An incident may be resolved by a workaround while its problem record remains open. A change may be raised to implement the permanent fix.

**Why it matters:** The JD explicitly assigns accountability for incident, problem, change, and risk management. Confusing these categories leads to weak governance and recurring failures.

## 3. How would you handle a Severity 1 production incident?

**Answer details:**

1. Acknowledge the alert and confirm whether the impact is real.
2. Open or join the incident bridge and appoint an incident commander, technical leads, communications owner, and scribe where appropriate.
3. Establish impact: affected clients, transactions, regions, services, start time, and current business deadline.
4. Check recent deployments, infrastructure events, dependencies, certificates, capacity, and security signals.
5. Choose the lowest-risk recovery action: rollback, failover, restart of an unhealthy workload, traffic shift, queue control, or a feature switch.
6. Give time-bound updates to the SRE Lead, SRE PO, business stakeholders, and CIO counterparts as required.
7. Validate recovery using health checks, error rates, transaction tests, and business confirmation.
8. Preserve logs and command history, then produce the timeline, RCA, corrective actions, and lessons learned.

**What to avoid:** Running unreviewed commands on multiple production nodes, deleting evidence, hiding uncertainty, or declaring recovery because one dashboard turned green.

## 4. How do you distinguish a workaround from a root cause?

**Answer details:** A workaround reduces impact without proving why the failure occurred, for example restarting a service or failing over to another region. Root cause explains the causal chain and is supported by evidence, such as a connection-pool leak that exhausted database sessions after a deployment. Record both separately: the workaround restores service; the permanent corrective action addresses the cause.

A good RCA also identifies contributing factors such as an insufficient alert, an unsafe default, a missing capacity threshold, a dependency failure, or an incomplete runbook. Avoid blaming an individual; focus on controls and system conditions.

## 5. What is an SLI, SLO, and SLA?

**Answer details:**

- **SLI:** A measured indicator, such as successful transaction rate, latency, or freshness of a batch output.
- **SLO:** The reliability target for that indicator over a time window, such as a percentage of successful requests under a latency threshold.
- **SLA:** A formal service commitment, usually with business or contractual consequences.
- **Error budget:** The amount of unreliability permitted by the SLO. It helps balance release velocity against stability.

In a banking service, define the SLI around a meaningful business transaction, not only infrastructure availability. A host can be healthy while payments, authentication, or end-of-day processing is failing.

## 6. How would you prioritise several alerts arriving at once?

**Answer details:** Correlate alerts by time, service, dependency, and symptom. Identify the likely parent failure and prioritise by client impact, transaction risk, regulatory or market deadline, data integrity, blast radius, and time to breach. Suppress duplicate child alerts only after confirming the relationship. Keep monitoring for secondary failures and split work among responders.

**Why it matters:** Alert volume is not the same as incident severity. The role requires effective monitoring and fast response, especially when multiple services depend on each other.

## 7. What should a production runbook contain?

**Answer details:** Purpose and scope, prerequisites, service ownership, severity criteria, dashboards and logs, safe diagnostic commands, decision points, recovery steps, validation checks, rollback or escalation criteria, communication templates, access requirements, and links to dependencies. Include expected outputs and a last-reviewed date. Commands must be safe to repeat and must clearly identify read-only versus state-changing actions.

## 8. How do you perform an RCA or post-mortem?

**Answer details:** Build an objective timeline from alerts, logs, deployment records, commands, and stakeholder updates. State impact, detection gap, technical cause, contributing conditions, recovery, and what went well. Create corrective actions with an owner, due date, priority, and verification method. Track them to closure and report trends in the monthly dashboard.

A useful RCA is blameless but accountable: it avoids personal blame while making ownership and follow-through explicit.

# Monitoring, Observability, and Alerting

## 9. What would you monitor for a production application?

**Answer details:** Cover the four golden signals and the business path:

- **Latency:** Request, database, queue, and end-to-end transaction latency.
- **Traffic:** Request rate, transaction volume, batch throughput, and queue depth.
- **Errors:** HTTP errors, application exceptions, failed business transactions, retries, and rejected messages.
- **Saturation:** CPU, memory, disk, file descriptors, connection pools, thread pools, database sessions, and Kubernetes resource limits.
- **Business health:** Successful transaction rate, reconciliation counts, settlement or EOD completion, stale data, and downstream acknowledgements.
- **Dependencies:** AWS services, DNS, certificates, network paths, Oracle, messaging, and external providers.

Metrics tell you that something is wrong; logs provide event detail; traces help locate latency across service boundaries.

## 10. What makes a good alert?

**Answer details:** A good alert is actionable, tied to a user or service objective, based on a sustained condition, severity-aware, deduplicated, and linked to a runbook. Include the current value, threshold, affected entity, start time, dashboard, and likely next action. Use warning alerts for early investigation and critical alerts for conditions requiring immediate response.

Avoid alerting on every transient CPU spike. Alert on symptoms such as sustained error-budget burn, failed transactions, unavailable replicas, exhausted database connections, or a batch deadline at risk.

## 11. How would you investigate high latency with a normal CPU graph?

**Answer details:** Check whether the issue is global or limited to an endpoint, client, region, or dependency. Compare request latency percentiles rather than averages. Inspect traces, load balancer metrics, application thread and connection pools, garbage collection, database wait events and slow SQL, network retransmits, DNS, storage latency, and downstream calls. Check recent changes and traffic patterns. A normal CPU level does not rule out blocking I/O, lock contention, an overloaded dependency, or a queueing problem.

## 12. Prometheus and Grafana: how do they work together?

**Answer details:** Prometheus scrapes or receives time-series metrics, stores them with labels, and evaluates recording and alerting rules. Grafana queries Prometheus and other data sources to present dashboards and correlations. Design labels carefully to avoid high cardinality. Use exporters or application instrumentation for useful metrics. Alert rules should have ownership, severity, annotations, and a runbook link.

## 13. How would you reduce noisy alerts?

**Answer details:** Measure false-positive rate and alert volume by rule. Remove alerts that do not lead to action, add duration and appropriate thresholds, use rates and percentiles, group related alerts, deduplicate replicas, account for maintenance windows, and separate symptom alerts from diagnostic signals. Test alert rules against historical incidents and verify that an alert would have detected important failures.

# AWS and Cloud Operations

## 14. A service in AWS is unavailable. What is your troubleshooting approach?

**Answer details:** Start with the user-facing symptom and affected region/account/environment. Check deployment history, CloudWatch metrics and logs, load balancer target health, DNS, security groups, network ACLs, route tables, IAM failures, autoscaling activity, container or instance health, and AWS service events. Compare healthy and unhealthy instances or availability zones. Confirm whether the failure is application, platform, network, identity, or a provider-side event.

Use least privilege, avoid changing several variables at once, and record commands and timestamps. Recovery may involve rollback, replacing unhealthy instances, scaling, rerouting traffic, or using a tested failover procedure.

## 15. Explain high availability and disaster recovery in AWS.

**Answer details:** High availability reduces interruption during normal component failures, typically by distributing workloads across Availability Zones and removing single points of failure. Disaster recovery addresses larger failures and defines recovery objectives:

- **RTO:** Maximum acceptable time to restore service.
- **RPO:** Maximum acceptable amount of data loss measured in time.

Discuss backup and restore, replication, multi-AZ or multi-region design, dependency readiness, secrets and configuration, DNS or traffic failover, access, communication, and regular tests. A DR document that has never been exercised is an assumption, not evidence.

## 16. How do you secure AWS operations?

**Answer details:** Use least-privilege IAM roles, short-lived credentials, MFA for human access, separation of duties, private networking where appropriate, encryption in transit and at rest, managed secret storage, logging through CloudTrail and service logs, patching, vulnerability management, and controlled break-glass access. Never place credentials in scripts, images, logs, or chat. In banking, access and operational evidence must support audit and regulatory obligations.

## 17. How would you troubleshoot an EC2 or container workload that is running out of memory?

**Answer details:** Confirm whether the signal is host memory, container limit, JVM/Python process memory, cache, or swap pressure. Check working set, OOM-kill events, restart counts, heap or process profiles, traffic, recent releases, memory limits and requests, and node capacity. Stabilise with safe scaling or traffic control, then fix the leak, unbounded cache, batch size, concurrency, or resource configuration. Validate over a representative period rather than relying on one restart.

## 18. What is the purpose of infrastructure as code with Terraform?

**Answer details:** Terraform defines infrastructure declaratively, creates a plan for review, tracks managed resources in state, and makes changes repeatable. Protect state, use remote locking and encryption, separate environments, review plans, manage secrets safely, pin provider/module versions where appropriate, and handle drift deliberately. Do not apply a plan blindly in production. Include dependency and rollback considerations; infrastructure changes can affect application availability even when the code change is small.

# Linux, Unix, and Networking

## 19. A Linux production host is slow. What do you check first?

**Answer details:** Establish scope and timing, then inspect load average, CPU user/system/iowait, memory and swap, disk space and inode usage, disk latency, processes, open files, network errors, system logs, and recent changes. Useful commands include `uptime`, `top` or `ps`, `free`, `vmstat`, `iostat`, `df -h`, `df -i`, `ss`, and `journalctl`, subject to the environment's approved tooling and access controls.

Interpret the output rather than listing commands. For example, high load with high iowait points toward storage or blocked I/O, while high system CPU may indicate kernel or network processing.

## 20. How would you troubleshoot a service that cannot connect to a database?

**Answer details:** Check whether the failure affects all instances or one host, then verify DNS resolution, route and port connectivity, security controls, TLS/certificate validity, credentials and secret rotation, database listener health, connection limits, pool exhaustion, and recent network or database changes. Distinguish timeout, refusal, authentication failure, and TLS errors because each implies a different layer. Avoid repeatedly retrying in a way that creates a connection storm.

## 21. Explain useful Linux permission and process concepts for support work.

**Answer details:** Explain users, groups, owner/group/other permissions, symbolic versus numeric modes, `sudo`, service accounts, signals, process trees, file descriptors, and systemd service lifecycle. Use the least privilege needed. Do not fix an access problem by making files world-writable or granting broad administrator access. Check the service's actual user, environment, working directory, limits, and unit configuration.

## 22. What happens when a user accesses an HTTPS application?

**Answer details:** At a high level: DNS resolves the name; the client connects through network controls and possibly a load balancer; TCP is established; TLS negotiates identity and encryption; the request reaches the application; the application calls dependencies such as Oracle; the response returns through the same path. Troubleshoot each boundary with appropriate evidence: DNS records, route and security controls, certificate chain and expiry, target health, access logs, application logs, and dependency metrics.

## 23. How do you troubleshoot intermittent network failures?

**Answer details:** Capture timestamps, source and destination, frequency, affected path, packet or request symptoms, and whether failures correlate with a node, zone, deployment, load, DNS response, or timeout. Compare successful and failed requests, inspect load balancer and host network metrics, use approved connectivity tools, and involve the network team with concrete evidence. Avoid declaring the network the root cause merely because the application timed out.

# Kubernetes and Containers

## 24. A Kubernetes deployment has pods in `CrashLoopBackOff`. What do you do?

**Answer details:** Inspect pod status and events, previous-container logs, current logs, deployment configuration, environment variables, secrets/config maps, image and command, probes, resource limits, service account permissions, node health, and recent rollout history. Use `kubectl describe pod`, `kubectl logs --previous`, and rollout history where approved. Determine whether the process exits, is killed, cannot mount configuration, fails a probe, or is denied access. Roll back only when the change is a credible cause and the rollback is approved and safe.

## 25. What is the difference between readiness and liveness probes?

**Answer details:** A readiness probe decides whether a pod should receive traffic. A liveness probe determines whether the container should be restarted. A startup probe can protect slow-starting applications from premature liveness failures. Probes should test the right condition without depending too deeply on fragile downstream services. Poor probe design can remove every healthy pod from service or cause restart loops during a dependency outage.

## 26. A Kubernetes service returns 503 while pods appear healthy. How do you investigate?

**Answer details:** Verify service selectors match pod labels, endpoint or EndpointSlice membership, readiness status, target ports, ingress or load balancer routing, network policies, TLS, and application access logs. Check whether the application is listening on the expected interface and port. Confirm the request reaches the intended namespace and cluster. "Running" only means the container process exists; it does not prove traffic is routed correctly or the business operation works.

## 27. Explain Kubernetes requests, limits, and autoscaling.

**Answer details:** Requests influence scheduling and represent expected resource reservation; limits cap container usage. CPU throttling and memory limits have different failure modes, including latency and OOM kills. Horizontal Pod Autoscaling can scale replicas from metrics, but it does not solve a database bottleneck or an incorrect resource limit by itself. Validate scaling behavior, node capacity, startup time, disruption budgets, and dependency capacity.

## 28. How would you perform a safe Kubernetes release?

**Answer details:** Use an immutable, scanned image and a reviewed manifest. Confirm compatibility, configuration, probes, resource settings, migrations, dependencies, and observability. Deploy progressively using rolling, canary, or blue-green strategy as appropriate. Watch error rate, latency, saturation, restarts, business transactions, and logs. Define abort thresholds and rollback steps before starting. Database changes should be backward-compatible when old and new application versions may run simultaneously.

# Programming, Scripting, and Automation

## 29. What makes a production automation script safe?

**Answer details:** Validate inputs and environment, fail fast on unexpected errors, use explicit timeouts, produce structured logs, avoid leaking secrets, use least-privilege credentials, make operations idempotent, support dry-run where possible, handle partial failure, and return meaningful exit codes. Add locking or concurrency control when duplicate execution could cause damage. Include pre-checks, post-checks, and a rollback or operator escalation path.

## 30. Write or describe a Python health check for a dependent service.

**Answer details:** The check should use a short timeout, verify the expected response rather than only TCP connectivity, distinguish timeout from authentication or server errors, emit a clear result, and exit non-zero on failure. It should not mutate production data or create an alert storm. For a database or message broker, use a low-cost read-only operation and close resources correctly. Configuration and credentials should come from approved secret mechanisms, not source code.

**Likely follow-up:** How would you test it? Cover success, timeout, DNS failure, malformed response, authentication failure, slow response, retry behavior, and secret absence.

## 31. How would you automate a deployment with Ansible?

**Answer details:** Use inventories or dynamic inventory, roles, variables, handlers, templates, vault or an external secret store, check mode, tags, and clear task failure behavior. Make tasks idempotent: running the playbook twice should converge to the same state. Use a controlled pipeline with peer review, an approval gate for production, limited concurrency, pre-checks, health validation, and a documented backout plan.

## 32. What should a CI/CD pipeline do before production deployment?

**Answer details:** Build an immutable artifact, run unit and integration tests, perform static analysis and dependency or image scanning, validate configuration and infrastructure plans, publish versioned evidence, deploy to a representative environment, run smoke and business checks, and require appropriate approval. Production promotion should use the same artifact tested earlier. Include deployment health gates, monitoring, rollback, and audit records.

## 33. Describe an automation improvement you would look for in a support team.

**Answer details:** Start with a high-volume, error-prone, low-judgement task such as log collection, evidence gathering, health checks, deployment validation, or standard restart orchestration. Measure the current effort and failure rate, build guardrails and dry-run behavior, test it away from production, add approval and audit logging, document it, and measure the result. Automation should reduce operational risk, not just reduce keystrokes.

# Oracle and Database Support

## 34. An application reports slow Oracle queries. What evidence do you collect?

**Answer details:** Identify the affected SQL and time window. Check response-time percentiles, execution plans, indexes and statistics, wait events, locks and blocking sessions, CPU and I/O, connection pool behavior, data volume, recent schema or code changes, and concurrent workload. Compare a healthy period with the incident period. Coordinate with the DBA team and avoid making an index, parameter, or query change directly in production without review and a rollback plan.

## 35. What is a database connection pool, and how can it cause an outage?

**Answer details:** A pool reuses a bounded number of database connections. Too small a pool creates application queuing; too large a pool can exhaust database sessions and increase contention. Leaked connections, long-running transactions, network partitions, stale connections, or a slow database can keep the pool occupied. Monitor active, idle, pending, and timed-out connections, database session limits, query duration, and pool wait time. Recovery may require traffic control or safe service recovery, followed by correcting the leak or timeout configuration.

## 36. How do you protect data integrity during a production fix?

**Answer details:** Prefer reversible, reviewed, and idempotent operations. Confirm backups and recovery capability, understand transaction boundaries, use a constrained scope, validate counts and business totals before and after, reconcile dependent systems, and preserve an audit trail. Do not run an untested data correction merely because it appears faster than restoring service. Escalate when the fix may affect client balances, regulatory reporting, settlement, or historical records.

## 37. Explain a safe approach to a database schema change.

**Answer details:** Assess dependencies and backward compatibility. Separate expand and contract where possible: add compatible structures, deploy code that can use both versions, backfill in controlled batches, validate performance and data, then remove obsolete structures in a later approved change. Plan locks, runtime, capacity, monitoring, rollback limitations, and recovery before implementation. A database rollback is not always a simple reverse SQL statement, so the plan must be explicit.

# Change, Release, Risk, and Resilience

## 38. What belongs in a production change plan?

**Answer details:** Business purpose, scope, affected services and dependencies, implementation steps, owners, prerequisites, risk assessment, test evidence, maintenance window, communication plan, monitoring, validation criteria, fallback or rollback, data conversion or migration details, and post-change review. Include commands or links to approved runbooks where needed. Confirm that surrounding systems, infrastructure, networking, certificates, schedules, and access have been considered.

This directly reflects the JD's requirement for implementation and fallback plans and review of dependent changes.

## 39. How do you decide whether to roll back a release?

**Answer details:** Define the decision before deployment: error-rate, latency, failed-transaction, health-check, or business-impact thresholds. Compare rollback risk with continued exposure. Confirm that the previous version is available, compatible with data and dependencies, and that rollback will not duplicate or lose work. Communicate the decision, stop further rollout, preserve evidence, validate the restored version, and create follow-up problem work.

## 40. What would you do if a senior stakeholder asks you to bypass change control?

**Answer details:** Explain the operational, client, audit, and regulatory risks calmly. Use the emergency-change process if the situation qualifies, obtain the required approval, document the reason and scope, restrict the action to the minimum necessary, and plan a review afterward. If the request remains unsafe or conflicts with policy, escalate through the appropriate SRE Lead, manager, risk, or incident channel. Do not conceal an unauthorised production change.

**Why it matters:** The JD explicitly includes governance, risk, compliance, and Code of Conduct. Good support judgment includes knowing when urgency requires a controlled emergency path, not when controls can be ignored.

## 41. How would you manage a production risk until it is closed?

**Answer details:** Describe the risk, affected service and business process, likelihood, impact, existing controls, owner, mitigation, target date, residual risk, and escalation path. Track the action in the required system, such as Riskwise for information-security risks or M7 for operational risks as named in the JD. Verify that the mitigation works and obtain formal acceptance or closure evidence. Do not mark a risk closed merely because an action was started.

## 42. What should a DR or BCP exercise prove?

**Answer details:** It should demonstrate that the service can meet agreed RTO and RPO under a defined failure scenario. Test application deployment or startup, data recovery or replication, secrets and access, DNS or routing, external dependencies, operational staffing, monitoring, communications, reconciliation, and return to normal operation. Record actual recovery time, data loss, defects, owners, and due dates. Feed the results into contingency documentation and service-improvement actions.

## 43. What would you include in a monthly production-support dashboard?

**Answer details:** Incident count and severity, client or business impact, MTTD and MTTR, recurring incidents, problem backlog and aging, change volume and success rate, change-related incidents, alert volume and noise, availability and SLO performance, open risks, RCA and SIP action status, overdue actions, DR-test findings, and training or knowledge-transfer progress. Add trend commentary and decisions needed from management; a dashboard should support action, not just display numbers.

# Banking Context

## 44. Why do you want a production-support role in a bank?

**Answer details:** Connect your answer to reliable services, disciplined operations, meaningful client impact, resilience, and learning across technology and business domains. Explain that banking support requires both technical speed and careful judgment because outages can affect client access, financial transactions, market confidence, data protection, and regulatory obligations. Mention your interest in automation and continual improvement, not only firefighting.

Avoid generic statements about the bank. Base your answer on the role's stated responsibilities, the listed stakeholders, and your own experience.

## 45. How do you balance fast incident recovery with regulatory and conduct obligations?

**Answer details:** Protect clients and data first, use approved access and emergency procedures, keep an accurate timeline, communicate facts without speculation, preserve evidence, and escalate potential conduct, security, privacy, or financial-crime concerns. Recovery actions must be proportionate and reversible where possible. After recovery, ensure reporting, RCA, risk tracking, and control improvements are completed.

Use the JD's conduct themes in your answer: fair outcomes for clients, effective financial markets, financial-crime compliance, the right environment, and collaborative escalation of risk and compliance matters.

## 46. How would you communicate with business stakeholders during an outage?

**Answer details:** Use plain language and state what is known, what is affected, when it started, what users should do, what the team is doing, the current risk, and the next update time. Do not expose sensitive technical details unnecessarily or promise an unverified resolution time. Distinguish technical recovery from business recovery, such as transaction reconciliation or backlog clearance.

## 47. Tell me about a time you disagreed with another team during an incident.

**Answer details:** Use a factual timeline and shared impact statement. Explain how you tested competing hypotheses, used evidence, made the least-risk decision, and kept communication respectful. Show that escalation is about reducing client and service impact, not proving which team was at fault. End with what changed in ownership, monitoring, documentation, or collaboration.

## 48. How do you handle sensitive production data and privileged access?

**Answer details:** Follow least privilege, approved access paths, segregation of duties, MFA, time-bound elevation, masking or redaction, secure transfer, and retention rules. Avoid copying customer data into tickets, personal files, or chat. Use synthetic or masked data for testing. Record access and operational actions so they are auditable, and report suspected exposure immediately through the required process.

# Scenario Exercises

## 49. A deployment completed successfully, but payment failures increased ten minutes later. Walk us through it.

**Strong response outline:**

- Confirm the failure rate, transaction type, affected clients or regions, and comparison with the pre-deployment baseline.
- Correlate deployment timing with application logs, traces, database behavior, queues, downstream responses, certificates, and feature flags.
- Stop further rollout and engage the incident process.
- Decide on rollback or feature disablement using predefined thresholds and compatibility checks.
- Check for partially processed transactions and reconcile them safely.
- Communicate technical and business impact separately.
- Validate recovery with real or controlled business transactions.
- Create RCA actions for test coverage, deployment gates, observability, and rollback readiness.

**What this tests:** Incident response, change management, dependency analysis, data integrity, communication, and learning from failure.

## 50. The Kubernetes pods are healthy, but the end-of-day batch is late. What do you investigate?

**Strong response outline:** Check the scheduler or Control-M-style orchestration if applicable, job dependencies, input arrival, queue depth, worker capacity, database locks, downstream acknowledgements, time zones and calendars, retries, and business completion criteria. Compare job duration and throughput with historical baselines. Avoid restarting workers blindly if that could duplicate work. Establish a safe recovery or catch-up plan and confirm reconciliation before declaring completion.

**What this tests:** Business-aware monitoring, batch operations, dependency reasoning, and data integrity.

## 51. Oracle is responding slowly and the application connection pool is exhausted. What is your immediate plan?

**Strong response outline:** Declare or assess incident severity, protect the database from a connection storm, measure affected transactions, inspect pool waiters and database sessions, identify blocking or expensive SQL, and coordinate with the DBA and application teams. Apply a documented traffic-control, read-only, failover, or rollback action if appropriate. Validate whether work was committed, retryable, or duplicated. Then address query plans, indexes, pool settings, transaction timeouts, and capacity as problem-management work.

**What this tests:** Cross-layer troubleshooting, safe stabilisation, Oracle knowledge, and transaction awareness.

## Qualification follow-ups

### How do you choose between Python, Java, and Go for a support automation task?

**Answer details:** Choose based on the existing service and team capabilities, not personal preference. Python is often effective for operational tooling and API integration; Java is common for enterprise services where JVM behavior, thread pools, garbage collection, and connection pools matter; Go is useful for small, portable, concurrent command-line tools and services. Discuss maintainability, observability, dependency management, security scanning, runtime support, testing, and how the tool will be operated in production. Be ready to explain one language-specific troubleshooting example rather than claiming equal depth in all three.

### What is the difference between a Docker image and a container?

**Answer details:** An image is an immutable, versioned package containing application code, runtime, libraries, and metadata. A container is a running process created from that image with runtime configuration, networking, storage, and resource limits. Discuss small trusted base images, multi-stage builds, non-root execution, image scanning, pinned versions, health checks, logging to standard output, and avoiding secrets in image layers. Connect the answer to Kubernetes, where the image is deployed as part of a pod specification.

### How would you use Nagios, Prometheus, or Grafana in a support environment?

**Answer details:** Nagios is commonly used for plugin-based host and service checks, while Prometheus is well suited to labelled time-series metrics and rule evaluation; Grafana provides dashboards and correlation across data sources. Explain that the tool matters less than signal quality, ownership, severity, runbook linkage, retention, access control, and testing. If the environment uses more than one tool, define which system is authoritative for paging, metrics, logs, and historical analysis.

### Which certifications are useful for this role, and how do they change your work?

**Answer details:** The JD lists AWS, SRE, and ITIL certifications as good to have. Explain the practical knowledge behind each: AWS architecture, security, resilience, and operations; SRE principles such as SLOs, error budgets, automation, and toil reduction; and ITIL practices for incident, problem, change, and service management. A certification is evidence of structured learning, not a substitute for production examples. Describe how you apply the concepts and where you would continue learning.

## 52. An auditor asks for evidence of a high-risk production change. What do you provide?

**Strong response outline:** Provide the approved change record, risk assessment, implementation and backout plans, approvals, test evidence, deployment logs, timestamps, monitoring and validation evidence, incident linkage if relevant, communication records, and post-implementation review. Redact sensitive information according to policy. If evidence is missing, state that clearly, escalate the control gap, and create remediation rather than reconstructing or altering records.

**What this tests:** Governance, honesty, auditability, risk management, and professional conduct.

# Quick Revision Checklist

Before the interview, be ready to explain:

- One Severity 1 or high-impact incident from detection through post-mortem.
- One production change with dependencies, validation, and rollback.
- One automation example with idempotency, safety controls, and measurable benefit.
- One monitoring improvement that reduced detection time or alert noise.
- One Linux, AWS, Kubernetes, or Oracle troubleshooting example with evidence.
- One disagreement or escalation handled professionally.
- One DR/BCP exercise or a realistic design for testing RTO and RPO.
- How you protect client data, follow least privilege, and preserve audit evidence.
- How you would report incident trends, RCA actions, SIP actions, and risks to management.
- The difference between restoring a service now and preventing the next incident.

# Questions to Ask the Interviewer

1. What are the most business-critical services and their current availability or SLO targets?
2. Which parts of the role are primary incident response, release control, automation, and continual service improvement?
3. How are on-call coverage, incident command, and escalation organised across the SRE team and CIO counterparts?
4. Which AWS, Kubernetes, monitoring, automation, and Oracle tools are used in production today?
5. What are the most important reliability or risk improvements expected in the first six months?
6. How are RCA, SIP, change-failure, and risk actions measured and reviewed?
7. How often are DR/BCP procedures exercised, and what were the most recent lessons learned?

## Final Guidance

The strongest answers will show two qualities at the same time: operational urgency and controlled judgment. Explain how you restore service quickly, but also how you protect client outcomes, data integrity, security, compliance, and the audit trail. For every technical answer, name the evidence you would collect and the validation that would prove the service is genuinely recovered.
