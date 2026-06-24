# Scenario-Based Labs: Cloud Models and Deployment Locations
# KijaniKiosk Platform — Architecture Reasoning

**Author:** Giovess  
**Date:** 2026-06-24  
**Context:** KijaniKiosk is a growing marketplace platform serving small retailers
primarily in Kenya and East Africa.

---

## Lab 1 — Service Model Selection

### Situation
KijaniKiosk currently manages servers manually. The API processes product orders
and payments. The team is considering a managed runtime platform that auto-scales.

### Task Sentence
"The KijaniKiosk API needs to process variable order volumes without manual server
management, while keeping deployment simple enough for a small engineering team."

### Selected Model: PaaS (Platform as a Service)

IaaS gives full control but keeps the operational burden — the team still patches
OS, manages runtimes, configures networking. SaaS is too opinionated; KijaniKiosk
needs to run custom application logic. PaaS is the correct fit: the provider manages
the infrastructure layer (OS, runtime, scaling) while the team owns the application
code and deployment configuration.

**Examples relevant to KijaniKiosk:** Google App Engine, AWS Elastic Beanstalk,
Railway, Render, or Heroku. All accept a Node.js application and handle scaling,
health checks, and zero-downtime deploys.

### Responsibilities that shift to the provider

| Responsibility | IaaS (current) | PaaS (proposed) |
|---|---|---|
| OS patching | Engineering team | Provider |
| Runtime upgrades (Node.js) | Engineering team | Provider |
| Auto-scaling on traffic spikes | Manual / scripted | Provider |
| Load balancer configuration | Engineering team | Provider |
| Application code | Engineering team | Engineering team |
| Environment variables / secrets | Engineering team | Engineering team |
| Database | Engineering team | Engineering team |

### One benefit
The team deploys by pushing code. There are no servers to SSH into, no nginx
configs to maintain per instance, and no manual scaling decisions during a traffic
spike on market day. A team of three can operate a production system that would
otherwise need a dedicated ops engineer.

### One limitation
PaaS platforms abstract the infrastructure, which means debugging performance
problems is harder. When a request is slow, the team cannot inspect the host OS,
tune kernel parameters, or profile network I/O directly. They depend on whatever
observability tooling the platform exposes. For a payments service where latency
directly affects transaction completion rates, this loss of visibility is a real
operational risk.

---

## Lab 2 — Latency and Region Selection

### Situation
Most KijaniKiosk customers are in Kenya and East Africa. The team is choosing
cloud regions.

### Two shortlisted regions

**Primary: Google Cloud `africa-south1` (Johannesburg) or AWS `af-south-1`
(Cape Town)**
The only cloud region on the African continent with major provider presence.
Round-trip latency from Nairobi to Johannesburg is approximately 40–60ms — far
lower than routing to Europe (150–180ms) or the US (220–280ms). For a
marketplace where users are browsing products and completing payments, the
difference between 60ms and 200ms API latency is perceptible.

**Secondary: Google Cloud `europe-west1` (Belgium) or AWS `eu-west-1` (Ireland)**
Used as a failover region and for any integrations with European payment processors.
Latency from East Africa to Western Europe is higher (~150ms) but acceptable for
background jobs, batch reconciliation, and disaster recovery. Not suitable as a
primary serving region for interactive user requests.

### How geographic distance affects response time
Every network hop between a user's device and the server adds latency. The
physical distance determines the minimum round-trip time (speed of light in fiber
is approximately 200,000 km/s). A server in Cape Town is ~3,400km from Nairobi;
a server in Ireland is ~6,800km. The Cape Town server has a lower latency floor
that no amount of optimization can close on the Ireland server.

Beyond raw distance, regional routing also matters. Traffic from Nairobi to
Johannesburg stays within African internet exchange points (KIXP, JINX). Traffic
to Europe often traverses undersea cables with higher contention during peak hours.

### Latency measurement

```bash
# Measuring from the KijaniKiosk development machine
ping -c 5 google.com
traceroute google.com
```

Observed: ~8ms to nearby Google infrastructure (Nairobi PoP).
Compared to a US endpoint: ~220ms average round-trip.

### Principle
**Deploy primary serving infrastructure in the region geographically closest to
the majority of your users, not in the region where your engineering team works.**

---

## Lab 3 — Availability Zone Failure Thinking

### Situation
If the single data center running KijaniKiosk loses power, all customers lose
access until it recovers.

### How multi-AZ reduces this risk
A cloud region contains multiple availability zones — physically separate data
centers within the same geographic area, connected by high-bandwidth low-latency
private links. Each zone has independent power, cooling, and networking. A failure
in Zone A (generator fire, flooding, hardware failure) does not affect Zone B.

By deploying the application in two AZs with a load balancer distributing traffic,
the system continues serving requests from Zone B while Zone A recovers. From the
user's perspective, nothing happened.

### Architecture diagram

```
                    ┌─────────────────────────────────┐
                    │         Region: af-south-1       │
                    │                                  │
          ┌─────────┴──────────┐   ┌──────────────────┴──────────┐
          │  Availability      │   │  Availability               │
          │  Zone A            │   │  Zone B                     │
          │                    │   │                             │
          │  ┌──────────────┐  │   │  ┌──────────────┐          │
          │  │  kk-api      │  │   │  │  kk-api      │          │
          │  │  instance    │  │   │  │  instance    │          │
          │  └──────┬───────┘  │   │  └──────┬───────┘          │
          │         │          │   │         │                   │
          └─────────┼──────────┘   └─────────┼───────────────────┘
                    │                        │
                    └──────────┬─────────────┘
                               │
                    ┌──────────┴──────────┐
                    │   Load Balancer     │
                    │  (routes traffic,   │
                    │   health checks)    │
                    └──────────┬──────────┘
                               │
                         User requests
```

### Components that must replicate across zones

| Component | Why it must replicate |
|---|---|
| Application instances (kk-api, kk-payments) | Direct request handlers — if one AZ fails, the other must serve all traffic |
| Database (read replicas minimum) | A primary DB in a failed AZ makes the app read-only at best, down at worst |
| Load balancer | Must be zone-aware; most cloud providers make this automatic |
| Session/cache layer (Redis) | If sessions are stored in one AZ only, users in the healthy AZ get logged out |

Components that do NOT need to replicate: batch jobs, log aggregation, admin
dashboards — these can tolerate brief unavailability.

---

## Lab 4 — Multi-Region Trade-Offs

### Situation
KijaniKiosk founders believe running infrastructure in multiple geographic regions
will eliminate all outages.

### Why multi-region increases operational complexity

A single-region multi-AZ deployment already handles most failure scenarios
(hardware failure, power outage, network partition within a DC). Multi-region adds:

**Data consistency problems.** A payment recorded in Region A must propagate to
Region B before a user in Region B queries it. The speed of light means there is
always a replication lag. This lag creates the possibility of a user seeing
inconsistent data — a transaction that succeeded but appears missing. Resolving
this requires distributed consensus protocols (Paxos, Raft) or accepting eventual
consistency, both of which add significant engineering complexity.

**Deployment coordination.** Deploying a new version of kk-payments now means
coordinating across two regions. A failed deployment in Region B while Region A
is healthy creates a version skew — two different versions of the payments service
handling transactions simultaneously. Schema migrations become dangerous.

**Cost.** Cross-region data transfer is billed by most cloud providers. A system
that constantly replicates database writes across regions can spend more on
transfer costs than on compute.

### Two situations where multi-region is justified

1. **Regulatory data residency requirements.** If KijaniKiosk expands to a country
that requires financial transaction data to remain within national borders, a
separate regional deployment is legally required — not optional.

2. **Catastrophic regional failure (disaster recovery).** If an entire cloud region
becomes unavailable (rare but documented — AWS us-east-1 has had multi-hour
outages), a warm standby in a second region allows traffic failover. The RTO
(recovery time objective) determines whether active-active or active-passive
multi-region is warranted.

### Recommendation to the founders

Start with a single-region, multi-AZ deployment in `af-south-1`. This eliminates
the most common failure modes (single AZ outages) at a fraction of the complexity
and cost of multi-region. Re-evaluate multi-region when either regulatory
requirements mandate it or when monthly revenue makes the engineering investment
in distributed data consistency justifiable.

---

## Summary

| Decision | Choice | Primary Reason |
|---|---|---|
| Service model | PaaS | Reduces ops burden for small team, preserves app ownership |
| Primary region | af-south-1 (Cape Town / Johannesburg) | Lowest latency for East African users |
| Secondary region | eu-west-1 (Ireland) | Failover + European payment processor integration |
| Initial availability | Single-region, multi-AZ | Eliminates common failures without distributed systems complexity |
| Multi-region trigger | Regulatory requirement or proven revenue justification | Not a default starting point |
