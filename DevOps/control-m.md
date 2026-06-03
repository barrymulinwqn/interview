# CONTROL-M — Interview Fundamentals

## What is CONTROL-M?
CONTROL-M (by BMC Software) is an enterprise workload automation (WA) platform for scheduling, managing, and monitoring batch jobs across heterogeneous environments — on-premises, cloud, containers, and mainframe. Widely used in banking and financial services for STP (Straight-Through Processing), EOD (End-of-Day) batch processing, reporting, and data pipelines.

---

## Architecture

```
CONTROL-M/Enterprise Manager (EM)
  │  Central management, GUI, reporting, API
  │
  ├── CONTROL-M/Server
  │     │  Scheduling engine, job submission, dependency management
  │     │
  │     └── CONTROL-M/Agents
  │           (installed on each managed host — executes jobs)
  │
  └── Self Service (web interface for business users)
```

### Key Components
| Component | Role |
|-----------|------|
| **Enterprise Manager (EM)** | Central management & monitoring console |
| **CONTROL-M/Server** | Scheduling engine; manages job ordering |
| **CONTROL-M/Agent** | Runs on target hosts; executes jobs locally |
| **Forecast** | Predict future job runs and dependencies |
| **Self Service** | Business user interface for job management |
| **Workload Change Manager** | Change control for job definitions |
| **Helix Control-M** | SaaS/cloud-native version |

---

## Core Concepts

### Job
A unit of work — can be:
- OS command / shell script
- Application (SAP, Oracle, PeopleSoft)
- File transfer (SFTP, FTP, AS2)
- Database stored procedure
- Hadoop / Spark job
- Containerized workload
- REST API call

### Folder (formerly Table/SMART Folder)
A logical container grouping related jobs. Jobs within a folder can share variables, scheduling criteria, and dependencies. Folders are ordered as a unit.

### Order Date (ODATE)
The logical date used for job ordering. Usually the business day. Critical for EOD processing — jobs ordered on ODATE even though they may run past midnight.

### New Day Processing
At a configured time (e.g., 18:00), CONTROL-M advances the ODATE and orders jobs for the new business day. Outstanding jobs retain their ODATE.

---

## Job Definition

### Key Job Properties
| Property | Description |
|----------|-------------|
| **Job Name** | Unique identifier within folder |
| **Application / Sub-application** | Logical grouping for reporting |
| **Host** | Agent host or host group to run on |
| **Command / Script** | What to execute |
| **Run As** | OS user for execution |
| **Max Reruns** | Auto-retry on failure |
| **Priority** | Scheduling priority (0-99) |
| **Time Zone** | For time-based scheduling |

### Scheduling
```
# Run every weekday
Days of Week: MON, TUE, WED, THU, FRI

# Run on specific calendar dates
Calendar: BUSINESS_DAYS_ONLY

# Run on day N of month
Monthly: 1, 15          # 1st and 15th

# Time window
Earliest Start: 06:00
Latest Start:   08:00   # job won't start after this
Due Out:        09:00   # SLA — alert if not complete by this time

# Cyclic (repeating within a day)
Cyclic: every 30 minutes, from 06:00 to 20:00
```

---

## Dependencies

### Prerequisite Conditions (In/Out Conditions)
The primary dependency mechanism — string-based flags set/deleted by jobs:
```
Job A sets condition: "PRICES-LOADED-YYYYMMDD"
Job B waits for:      "PRICES-LOADED-YYYYMMDD" to exist

# Condition naming convention (common):
<SYSTEM>-<STATUS>-ODATE
MARKET-DATA-LOADED-ODATE
RISK-CALC-COMPLETE-ODATE
```

| Type | Description |
|------|-------------|
| **In Condition** | Job waits until this condition exists |
| **Out Condition - ADD** | Job adds this condition on completion |
| **Out Condition - DELETE** | Job deletes this condition on completion |
| **Out Condition - ADD on NOTOK** | Condition added only on failure |

### Quantitative Resources
Limit concurrent execution using named resources with a defined quantity:
```
Resource: DB_CONNECTIONS  Quantity: 10
# Prevents more than 10 jobs from using DB connections simultaneously
```

### Control Resources (Semaphores)
```
Shared: multiple jobs can hold simultaneously
Exclusive: only one job at a time (mutex)
```

### File Watcher / Event-Based Triggers
```
Wait for file:     /data/feed/prices_YYYYMMDD.csv
File created/modified/deleted
Wildcards supported
```

---

## Variables & Scheduling Parameters

### System Variables
| Variable | Value |
|----------|-------|
| `%%ODATE` | Order date (YYYYMMDD) |
| `%%ORDERID` | Unique job run ID |
| `%%JOBNAME` | Current job name |
| `%%FOLDER` | Folder name |
| `%%RUNCOUNT` | Number of times job has run today |
| `%%$DATE` | Current date |
| `%%$TIME` | Current time |
| `%%$YEAR`, `%%$MONTH`, `%%$DAY` | Date parts |

### User-Defined Variables
```
Variable: %%ENVIRONMENT   Default: PROD
Variable: %%DB_HOST       Default: db01.example.com
Variable: %%BATCH_SIZE    Default: 5000
```

Usage in scripts:
```bash
#!/bin/bash
echo "Processing for ODATE: %%ODATE"
python /opt/batch/run.py --date %%ODATE --env %%ENVIRONMENT
```

---

## Calendars

### Calendar Types
| Type | Purpose |
|------|---------|
| **Regular** | Specific dates or patterns (monthly, yearly) |
| **Rule-based** | Dynamic rules (first Monday, last business day) |
| **Periodic** | Every N days |

Calendars are referenced in scheduling criteria to define business-day-aware scheduling:
```
Schedule: Run only on TRADING_DAYS calendar
Skip: Run when MARKET_HOLIDAY calendar does NOT apply
```

---

## Alerts & Notifications

### Alert Conditions
- Job ended **NOTOK** (non-zero exit code)
- Job didn't start by **Due In** time
- Job didn't complete by **Due Out** time (SLA breach)
- Job is still running after expected duration
- Job is **Late Start** / **Late End**

### Notification Methods
```
Email: ops-team@bank.com
SNMP Trap: monitoring system
Remedy / ServiceNow: auto-create incident ticket
PagerDuty / Opsgenie: on-call alerting
Custom script: call internal alerting API
```

### Do/Mail/Sysout on Action
```
On OK:    Send report email with output
On NOTOK: Page on-call, create ServiceNow incident
On Late:  Notify SLA manager
```

---

## Job Handling Operations

### Manual Actions
| Action | Description |
|--------|-------------|
| **Order** | Submit job for execution now |
| **Force OK** | Mark failed job as OK (allows dependents to proceed) |
| **Rerun** | Re-execute a job |
| **Hold** | Prevent job from starting |
| **Free** | Release a held job |
| **Kill** | Terminate a running job |
| **Confirm** | Manual approval before job proceeds |
| **Delete** | Remove job from active jobs |
| **Bypass** | Set Out Conditions without running |

### When to Force OK?
When a job fails but its output is acceptable, or a dependency condition needs to be unblocked. Common in production operations for EOD processing deadlines.

---

## CONTROL-M Automation API

### REST API (Helix Control-M / v9+)
```bash
# Login
curl -X POST "https://ctm-server:8443/automation-api/session/login" \
     -H "Content-Type: application/json" \
     -d '{"username":"apiuser","password":"***"}'

# Deploy job definitions
ctm deploy jobs.json

# Run job
ctm run job::order -s "folder=MyFolder&jobname=MyJob&ctm=prod-ctm"

# Get run status
ctm run status <runId>

# Get job output
ctm run job::output <runId> <jobId>
```

### Job Definition (JSON format for API)
```json
{
  "Folder_Name": {
    "Type": "Folder",
    "ControlmServer": "prod-ctm",
    "OrderMethod": "Manual",
    "Job_Name": {
      "Type": "Job:Script",
      "Host": "app-server-01",
      "RunAs": "appuser",
      "FilePath": "/opt/batch/scripts",
      "FileName": "process_trades.sh",
      "Variables": [
        { "ENVIRONMENT": "PROD" },
        { "BATCH_DATE": "%%ODATE" }
      ],
      "When": {
        "Weekdays": ["MON", "TUE", "WED", "THU", "FRI"],
        "FromTime": "0600",
        "ToTime": "0800"
      },
      "InConditions": [
        { "Name": "MARKET-DATA-LOADED-%%ODATE", "AND_OR": "AND" }
      ],
      "OutConditions": [
        { "Name": "TRADES-PROCESSED-%%ODATE", "When": "OK" },
        { "Name": "TRADES-FAILED-%%ODATE", "When": "NOTOK" }
      ],
      "Notify": {
        "MailMessage": {
          "To": "ops@bank.com",
          "Subject": "Trade Processing Failed - %%ODATE"
        },
        "When": "NOTOK"
      },
      "MaxRerun": 3,
      "RerunEvery": "5"
    }
  }
}
```

---

## EOD Batch Processing (Financial Context)

### Typical EOD Flow
```
Market Close (16:30)
  └── MARKET-CLOSE-ODATE (condition set by feed)
        └── Position Capture
              └── Risk Calculation (parallel: Equity, FX, Rates)
                    └── P&L Calculation
                          └── Regulatory Reporting
                                └── Client Reports
                                      └── EOD-COMPLETE-ODATE
```

### Best Practices
1. Use **ODATE** for date variables — handles midnight crossover correctly
2. Define **SLA (Due Out)** times for critical jobs
3. Use **host groups** for failover — jobs run on any available agent
4. **Cyclic jobs** for intraday position updates (every 15 min)
5. Use **resource pools** to prevent DB overload during peak batch
6. Keep **job logs** for at least 30 days for audit purposes
7. Test dependency chains in **non-production** with simulated conditions
8. Use **Workload Change Manager** for all production changes (change control)

---

## Common Interview Questions

### Q1: What is the difference between a Folder and a Sub-folder in CONTROL-M?
A **Folder** is the top-level container ordered as a unit on the ODATE. A **Sub-folder** is a nested grouping within a folder, allowing hierarchical organisation. Dependencies can cross folder boundaries.

### Q2: What is ODATE and why is it important?
**Order Date** — the logical business date associated with a job run. Essential in finance because batch runs often span midnight. Using ODATE ensures that a job processing "Monday's data" has consistent date references even if it runs at 01:00 Tuesday.

### Q3: What are Prerequisite Conditions?
String flags set/deleted by jobs to signal state. Jobs declare "In Conditions" they wait for and "Out Conditions" they produce. This is the primary dependency mechanism — more flexible than direct parent-child links because conditions can be set/deleted manually.

### Q4: How would you handle a failed EOD job that is blocking downstream jobs?
1. Investigate the root cause (check SYSOUT/job log)
2. If the failure is non-critical or data is acceptable: **Force OK** the job
3. This adds the Out Condition and releases downstream dependencies
4. Alternatively: manually add the missing In Condition to unblock
5. Raise a production incident for root-cause analysis

### Q5: What is New Day Processing?
The process by which CONTROL-M advances the ODATE at a configured time (the "New Day" time). Jobs scheduled for the new business day are ordered. Outstanding jobs from the previous day retain their old ODATE.

### Q6: How do you prevent too many jobs from hitting a database at once?
Use **Quantitative Resources** — define a resource (e.g., `DB_CONNECTIONS`) with a max count. Jobs declare how many units they need; CONTROL-M ensures the total in-use count never exceeds the limit.

### Q7: Difference between Hold and Wait for time?
- **Hold**: Manually suspended; requires manual Free to release
- **Wait for time** (Earliest Start): Job is held by the scheduler until the configured start time — automatically released

### Q8: How does CONTROL-M integrate with CI/CD pipelines?
Using the **Automation API** (REST/CLI), deployment pipelines can: deploy job definitions from source control (`ctm deploy`), trigger job runs (`ctm run`), check status, and validate definitions before production deployment.

### Q9: What is Forecast used for?
Simulates future job scheduling based on current definitions and calendars. Used to predict the timing and sequence of jobs for upcoming business days, validate schedule changes, and plan resource capacity.

### Q10: How do you handle daylight saving time transitions?
Define jobs in the correct **time zone**. CONTROL-M handles DST transitions per time zone settings. For jobs with absolute times, verify they behave correctly during the transition (especially cyclic jobs crossing the DST change).
