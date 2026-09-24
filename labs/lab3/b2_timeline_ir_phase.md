# B2 Timeline: Incident Response Phase (09:29:xx — 09:36:12 UTC)

Use this as the **eval reference** — test your timelining skill against this section.

## Context

After AWS notified Frothly at 09:16:53, the IR team began containment ~13 minutes later. The goal: disable the compromised access key and verify no unauthorized resources were created.

## Events

| Timestamp (UTC) | Actor | Event | Source | Details |
|-----------------|-------|-------|--------|---------|
| ~09:29:xx | IR Team | Notification received | Unknown | IR team alerted to compromised credentials |
| 09:30:53 | IR Team (mars host) | aws_ir tool execution (1st) | osquery:results | First containment attempt — key disablement initiated |
| 09:33:59 | IR Team | aws_ir tool execution (2nd) | osquery:results | Verification run |
| 09:35:01 | IR Team | aws_ir tool execution (3rd) | osquery:results | Final confirmation |
| 09:35:58 | bstoll | ListUsers via AWS Console | CloudTrail | Developer manually verifies IAM state |
| 09:36:12 | bstoll | UpdateAccessKey — DISABLED | CloudTrail | Access key AKIAJOGCDXJ5NW5PXUPA disabled |

## Key Entities

| Entity | Type | Role |
|--------|------|------|
| mars (host) | Endpoint | IR team workstation running aws_ir |
| bstoll | User | Developer who caused the exposure; also participated in containment |
| aws_ir | Tool | Automated IR tool for AWS key disablement |
| AKIAJOGCDXJ5NW5PXUPA | AWS Access Key | Disabled at 09:36:12 |

## Detection Metrics

| Metric | Value |
|--------|-------|
| Time from compromise to AWS detection | 47 seconds |
| Time from AWS notification to IR start | ~13 minutes |
| Time from IR start to containment | ~6 minutes |
| Total time to containment | ~20 minutes |

## Critical Gap

The STS token (ASIAZB6TMXZ7LL6JBJQA) generated at 09:16:12 was **not explicitly revoked**. Disabling the access key does not invalidate existing STS sessions. This is a common mistake in AWS incident response.

## Useful SPL Queries

```spl
# IR tool executions
index=botsv3 sourcetype="osquery:results" aws_ir
| sort _time

# Key disablement event
index=botsv3 sourcetype="aws:cloudtrail" eventName="UpdateAccessKey"
userIdentity.accessKeyId="AKIAJOGCDXJ5NW5PXUPA"
| table _time, sourceIPAddress, requestParameters.status
```
