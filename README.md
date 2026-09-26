# ☁️ Cloud Cost vs Reliability Trade-off

> **Design infrastructure for the problem you actually have—not the problem you might have someday.**

This project explores how to design a small AWS environment for a web application with **limited traffic, limited budget, and a defined tolerance for downtime**.

Instead of automatically reaching for Kubernetes, load balancers, auto-scaling, multi-AZ deployments, or multi-region architecture, the project asks a simpler question:

> **What level of infrastructure does this application actually need right now?**

The result is a deliberately simple architecture that prioritizes **cost efficiency, data safety, observability, and operational simplicity**.

---

## 🎯 Scenario

Imagine a small web application with:

* ~400 daily active users
* Low traffic and concurrency
* Limited infrastructure budget
* Short outages (~10 minutes) are acceptable
* The application is not mission-critical
* **User data must not be lost**

The goal is not maximum availability.

The goal is to build an infrastructure design that is **appropriate for the workload and constraints**.

---

# 🏗️ Architecture

The initial architecture intentionally uses a small number of components:

```text
                         Internet
                            │
                            ▼
                    ┌───────────────┐
                    │     Nginx     │
                    │ Reverse Proxy │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │    Node.js    │
                    │      App      │
                    │     PM2       │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   Managed DB  │
                    │   RDS / DB    │
                    └───────────────┘
                            
                            ▲
                            │
                    ┌───────┴────────┐
                    │    Monitoring  │
                    │   CloudWatch   │
                    └────────────────┘
```

### Infrastructure components

| Layer           | Technology                   | Purpose                                     |
| --------------- | ---------------------------- | ------------------------------------------- |
| Compute         | EC2 / Ubuntu                 | Application host                            |
| Web server      | Nginx                        | Reverse proxy / static files                |
| Runtime         | Node.js                      | Application                                 |
| Process manager | PM2                          | Application restart/recovery                |
| Database        | Managed database             | Persistent application data                 |
| Security        | Security Groups              | Network access control                      |
| Monitoring      | CloudWatch                   | Metrics, alarms and availability monitoring |
| Backup          | Database backups / snapshots | Data protection                             |

---

# 💰 The Core Decision

The central architectural decision was:

> **Use one application VM instead of immediately building a highly available architecture.**

This introduces a **single point of failure**.

That is intentional.

```text
                    ┌─────────────────┐
                    │    Internet     │
                    └────────┬────────┘
                             │
                             ▼
                     ┌──────────────┐
                     │    EC2 #1    │
                     │              │
                     │ Nginx        │
                     │ Node.js      │
                     │ PM2          │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   Managed DB │
                     └──────────────┘
```

If the EC2 instance fails, the application may become unavailable until the instance is recovered.

For this scenario, that risk is acceptable because:

* Traffic is low
* The application is not mission-critical
* Short downtime is acceptable
* Infrastructure cost matters
* Operational simplicity has value

The important distinction is:

> **A single-server architecture is not inherently wrong. It is a trade-off.**

---

# ⚖️ Cost vs Reliability

The project compares two possible architectures.

### Current architecture

```text
        Internet
           │
           ▼
      ┌─────────┐
      │  EC2    │
      │  Nginx  │
      │ Node.js │
      └────┬────┘
           │
           ▼
      ┌─────────┐
      │   RDS   │
      └─────────┘
```

### Future high-availability architecture

```text
                         Internet
                            │
                            ▼
                    ┌───────────────┐
                    │ Load Balancer │
                    └───────┬───────┘
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
             ┌─────────┐         ┌─────────┐
             │ EC2 #1  │         │ EC2 #2  │
             │   AZ-A  │         │   AZ-B  │
             └────┬────┘         └────┬────┘
                  │                   │
                  └─────────┬─────────┘
                            ▼
                     ┌─────────────┐
                     │ Multi-AZ DB │
                     └─────────────┘
```

The second architecture provides more resilience, but also introduces additional infrastructure, configuration, monitoring, and cost.

---

# 📊 Trade-off Matrix

| Area                 | Current Design              | Future HA Design           | Trade-off                         |
| -------------------- | --------------------------- | -------------------------- | --------------------------------- |
| Compute              | Single EC2                  | Multiple EC2 + ASG         | Lower cost vs higher availability |
| Traffic distribution | Direct access / single host | Load Balancer              | Simplicity vs redundancy          |
| Database             | Single managed DB           | Multi-AZ DB                | Lower cost vs automatic failover  |
| Recovery             | Manual recovery             | Automated replacement      | Operational effort vs automation  |
| Monitoring           | Basic CloudWatch            | Expanded dashboards/alarms | Simplicity vs visibility          |
| Backup               | Regular backups/snapshots   | Expanded DR strategy       | Lower cost vs faster recovery     |
| Availability         | Lower                       | Higher                     | Cost increases with resilience    |

---

<img width="1414" height="2000" alt="Architecture diagram" src="https://github.com/user-attachments/assets/0b2d5732-61b8-4357-b89b-ab530d89b034" />


# 🧠 Why Not Kubernetes?

Kubernetes is powerful, but it would introduce additional operational complexity that this workload does not currently require.

For ~400 daily users with low concurrency and an accepted downtime window, a Kubernetes cluster would solve problems that aren't currently part of the scenario.

That doesn't make Kubernetes unnecessary in general.

It means:

> **The architecture should follow the workload—not the other way around.**

---

# 🚫 Intentionally Not Using

The initial design deliberately avoids:

* ❌ Kubernetes
* ❌ Service mesh
* ❌ Multi-region deployment
* ❌ Complex microservices
* ❌ Auto Scaling Groups
* ❌ Application Load Balancer
* ❌ Multi-AZ database deployment

These technologies may become appropriate later.

They are not automatically justified simply because they are considered "production-grade."

---

# 🔐 Reliability Does Not Mean Only Uptime

One important design decision in this project is that **data safety has a higher priority than application availability**.

For example:

```text
Application temporarily unavailable
              │
              ▼
       Recover the server
              │
              ▼
       Application returns
```

is an acceptable scenario.

But:

```text
Database failure
      │
      ▼
Permanent data loss
```

is not.

This is why the design still uses a **managed database with backups**, even though the application layer is intentionally simple.

---

# 🛡️ Security Baseline

The infrastructure also follows a basic security model.

### Security Group

Expected access pattern:

```text
Internet
   │
   ├── HTTPS :443 ──────► Application
   │
   └── HTTP  :80 ───────► Application
                         
SSH :22
   │
   └──► Restricted to trusted source
```

SSH should not be exposed broadly to the internet when it can be restricted to a known administrative source.

---

# 📡 Monitoring

The initial monitoring strategy focuses on detecting problems that matter for a small deployment.

Examples include:

* CPU utilization
* Disk usage
* Instance availability
* Database connections
* Application availability
* Relevant system/application metrics

CloudWatch alarms can then notify the operator when defined thresholds are exceeded.

### Important limitation

Some host-level metrics, such as detailed memory utilization, require additional configuration such as the **CloudWatch Agent** rather than appearing automatically from basic EC2 monitoring.

That distinction is important when designing observability.

---

# 🔧 Operational Decision Framework

Instead of adding infrastructure immediately, problems are mapped to potential solutions.

```text
                Problem
                   │
                   ▼
           What is actually
              failing?
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
      CPU        Disk       Traffic
        │          │          │
        ▼          ▼          ▼
   Resize VM   Expand disk   Add capacity
                              │
                              ▼
                         Load Balancer
                              +
                         Auto Scaling
```

The principle is:

> **Don't add complexity until the existing architecture demonstrates that it needs it.**

---

# 📈 When Would the Architecture Change?

The architecture should evolve when the application's requirements change.

### Example triggers

**Traffic increases**

```text
400 users
    ↓
Several thousand users
    ↓
Capacity becomes a concern
    ↓
Add additional application instances
```

**Downtime becomes unacceptable**

```text
10-minute outage acceptable
          ↓
Business requirements change
          ↓
Availability becomes critical
          ↓
Introduce redundancy
```

**Manual recovery becomes painful**

```text
Manual restart
     ↓
Frequent failures
     ↓
Operational burden increases
     ↓
Automated recovery becomes valuable
```

**Database availability becomes critical**

```text
Single DB
   ↓
Downtime becomes unacceptable
   ↓
Consider Multi-AZ deployment
```

---

# 🚀 Upgrade Path

The architecture has a defined path for increasing resilience.

### Stage 1 — Current

```text
1 × EC2
1 × Managed DB
CloudWatch
Backups
```

### Stage 2 — Increased capacity

```text
Load Balancer
      │
 ┌────┴────┐
 ▼         ▼
EC2       EC2
```

### Stage 3 — Automated recovery

```text
Load Balancer
      │
      ▼
Auto Scaling Group
      │
 ┌────┴────┐
 ▼         ▼
EC2       EC2
```

### Stage 4 — Database resilience

```text
Application Tier
       │
       ▼
  Multi-AZ DB
```

### Stage 5 — Disaster Recovery

If business requirements justify it:

```text
Primary Region
      │
      │ backups / replication
      ▼
Secondary Region
```

Each stage is introduced because of a **specif**
