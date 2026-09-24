# B2 Timeline: Attack Phase (09:16:12 — 09:28:54 UTC)

Use this as the **reference timeline** when building your timelining skill.

## Context

After Bud (bstoll) pushed AWS credentials to a public GitHub repo at 09:16:06, attackers began exploiting the key `AKIAJOGCDXJ5NW5PXUPA` within 6 seconds.

## Events

| Timestamp (UTC) | Actor | Event | Source | Details |
|-----------------|-------|-------|--------|---------|
| 09:16:12 | 35.153.154.221 | GetCallerIdentity SUCCESS | CloudTrail | Validates credentials; identifies account 622676721278, IAM user "web_admin" |
| 09:16:12 | 35.153.154.221 | GetSessionToken SUCCESS | CloudTrail | Creates STS temp credentials (ASIAZB6TMXZ7LL6JBJQA) for persistence |
| 09:16:12 | 35.153.154.221 | CreateUser DENIED | CloudTrail | Privilege escalation attempt blocked by IAM policy |
| 09:16:12 | 35.153.154.221 | CreateAccessKey DENIED | CloudTrail | Attempted to create new credentials |
| 09:16:14 | 139.198.18.205 | First RunInstances attempt | CloudTrail | Cryptomining campaign begins |
| 09:16:18 | 209.107.196.112 | ListAccessKeys attempt | CloudTrail | Probing for additional credentials |
| 09:16:22 | 139.198.18.205 | Mass RunInstances across 15 regions | CloudTrail | 576 total attempts, all blocked by service limits |
| 09:16:53 | AWS | Notification email sent | stream:smtp | Case 5244329601 — "Your AWS account is compromised" |
| ~09:21:xx | 139.198.18.205 | RunInstances attempts end | CloudTrail | All 576 attempts failed |
| 09:27:06 | 82.102.18.111 | ElasticWolf console exploration | CloudTrail | Manual AWS access using derived credentials |
| 09:28:54 | 139.198.18.205 | Last STS token activity | CloudTrail | Final API call; 12m48s total attack duration |

## Key Entities

| Entity | Type | Role |
|--------|------|------|
| AKIAJOGCDXJ5NW5PXUPA | AWS Access Key | Compromised credential |
| ASIAZB6TMXZ7LL6JBJQA | STS Token | Derived temp credential |
| web_admin | IAM User | Compromised account |
| 35.153.154.221 | IP | Initial scanner, STS generator |
| 139.198.18.205 | IP | Cryptominer (637 API calls) |
| 209.107.196.112 | IP | Credential enumerator |
| 82.102.18.111 | IP | ElasticWolf manual exploration |

## Useful SPL Queries

```spl
# All activity from the compromised key
index=botsv3 sourcetype="aws:cloudtrail" userIdentity.accessKeyId="AKIAJOGCDXJ5NW5PXUPA"
| stats count by _time, eventName, sourceIPAddress, errorCode
| sort _time

# RunInstances attempts by region
index=botsv3 sourcetype="aws:cloudtrail" eventName="RunInstances"
| stats count by awsRegion, sourceIPAddress, errorCode
| sort -count
```
