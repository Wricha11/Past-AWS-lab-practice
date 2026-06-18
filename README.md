# AWS Cloud Security Monitoring Lab

A self-built AWS lab environment for practicing **cloud-native security monitoring, detection, and RMF-aligned continuous monitoring**. The lab deploys a small multi-instance environment and instruments it with **CloudWatch**, then I map the resulting monitoring and logging coverage back to **NIST SP 800-53** control families.

> Built by William Richardson (ISSE / CISSP / AWS Cloud Practitioner) as a hands-on portfolio project bridging cloud engineering and Risk Management Framework (RMF) practice.

---

## Why this project

DoD and federal systems are increasingly migrating from on-premises to cloud, and the **continuous monitoring** step of RMF looks very different in a cloud authorization boundary than on physical infrastructure. This lab is a working reference for:

- Standing up a monitored AWS environment from scratch
- Demonstrating where cloud-native services satisfy (or inherit) NIST 800-53 controls
- Practicing detection, alerting, and triage in a cloud context before doing it on a production ATO package

---

## Architecture

```
                          ┌─────────────────────────────┐
                          │        AWS Account          │
                          │                             │
  ┌──────────┐            │   VPC (10.0.0.0/16)         │
  │ CloudTrail│◀──────────│   ├─ Public subnet          │
  │ (API audit)│          │   │   └─ EC2: bastion/web    │
  └──────────┘            │   ├─ Private subnet          │
  ┌──────────┐            │   │   ├─ EC2: app            │
  │ GuardDuty │◀──────────│   │   └─ EC2: monitoring     │
  │ (threat   │           │   ├─ Security Groups         │
  │  detection)│          │   └─ IAM roles / least-priv  │
  └──────────┘            │                             │
  ┌──────────┐            │                             │
  │CloudWatch │◀──────────│   Logs + metrics + alarms    │
  │(logs/alarms)│         └─────────────────────────────┘
  └──────────┘
```

**Core components**

| Component | Purpose |
|-----------|---------|
| **VPC + subnets** | Isolated network boundary with public/private segmentation |
| **3x EC2 instances** | Workload + monitoring host to generate and observe activity |
| **Security Groups** | Host-level network access control (least privilege) |
| **IAM roles/policies** | Scoped permissions; no long-lived root usage |
| **CloudWatch** | Centralized logs, metrics, and alarms |

---

## Monitoring & detection design

The lab focuses on getting **signal** out of the environment, not just standing it up.

- **CloudWatch Logs** — EC2 system/auth logs shipped via the CloudWatch agent; metric filters created on key patterns (e.g., failed SSH/console logins, root usage, IAM policy changes).
- **CloudWatch Alarms** — alarms on the metric filters above, notifying an SNS topic.

### Example detection use cases

| Use case | Source | Signal |
|----------|--------|--------|
| Repeated failed SSH logins | CloudWatch Logs metric filter | Brute-force / credential attack |

> Snippets below are illustrative starting points — adapt names, ARNs, and thresholds to your environment.

**Example: metric filter for root usage (CloudWatch Logs)**

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/Logs" \
  --filter-name "RootAccountUsage" \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations metricName=RootAccountUsageCount,metricNamespace=SecurityMetrics,metricValue=1
```

**Example: alarm on that metric**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "RootAccountUsageAlarm" \
  --metric-name RootAccountUsageCount \
  --namespace SecurityMetrics \
  --statistic Sum --period 300 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions <SNS_TOPIC_ARN>
```

---

## RMF / NIST 800-53 control mapping

A key goal of the lab is showing **how cloud-native services support continuous monitoring controls**. This is a demonstration mapping, not an authorization artifact.

| Control family | Example controls | How the lab addresses it |
|----------------|------------------|--------------------------|
| **AU – Audit & Accountability** | AU-2, AU-6, AU-12 | CloudTrail captures management events; CloudWatch centralizes and retains logs; metric filters support review/analysis |
| **SI – System & Information Integrity** | SI-4 | GuardDuty + CloudWatch alarms provide monitoring and intrusion-detection signal |
| **CA – Assessment, Authorization & Monitoring** | CA-7 | Continuous monitoring strategy realized through alarms, GuardDuty findings, and periodic review |
| **AC – Access Control** | AC-2, AC-6 | IAM least-privilege roles; CloudTrail alerts on IAM changes |
| **CM – Configuration Management** | CM-6 | Security Groups and baseline configuration tracked; drift surfaced via CloudTrail |

> In a real authorization boundary, many of these controls would be **inherited** from the Cloud Service Provider (CSP) and documented in the security package. This lab mirrors that pattern at small scale.

---

## Setup (high level)

> ⚠️ Costs money if left running. Tear down when finished. Never commit credentials, account IDs, or keys.

1. **Prereqs** — AWS account, AWS CLI configured, an IAM user/role with appropriate permissions, an SSH key pair.
2. **Network** — create the VPC, subnets, route tables, internet gateway, and Security Groups.
3. **Compute** — launch the EC2 instances; attach scoped IAM instance profiles; install the CloudWatch agent.
4. **Logging** — enable a multi-region CloudTrail to a dedicated S3 bucket; configure CloudWatch log groups.
5. **Detection** — create metric filters + alarms; create the SNS topic and subscribe an endpoint; enable GuardDuty.
6. **Validate** — generate test activity (failed logins, IAM changes) and confirm alarms/findings fire.
7. **Teardown** — destroy resources to avoid charges.

---

## Skills demonstrated

- AWS core services: VPC, EC2, IAM, S3, Security Groups
- Cloud-native logging/monitoring: CloudWatch (agent, logs, metric filters, alarms)
- Translating cloud telemetry into NIST 800-53 continuous-monitoring coverage
- RMF concepts: control inheritance, authorization boundaries, continuous monitoring

---

## Roadmap

- [ ] Add Infrastructure-as-Code (Terraform or CloudFormation) for repeatable deploys
- [ ] Add AWS Config rules for configuration-drift detection (CM family)
- [ ] Document a sample POA&M workflow from a simulated finding
- [ ] Add an incident-response runbook for the brute-force use case

---

## Disclaimer

This is a personal learning lab. It is **not** a production-ready reference architecture, contains no real or sensitive data, and is unaffiliated with any employer or government program. Example configurations are starting points and should be reviewed and hardened before any real use.
