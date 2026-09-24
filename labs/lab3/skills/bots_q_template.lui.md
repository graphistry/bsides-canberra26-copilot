# BOTS Q — BOTSv3 Investigation Skill

This skill preloads context for investigating the Splunk BOTSv3 dataset, so the agent doesn't waste time guessing indices, sourcetypes, or time ranges.

## BOTSv3 Environment

- **Index**: `botsv3`
- **Time range**: August 2018 (`earliest=0` captures all data)
- **Dataset**: Splunk Boss of the SOC v3 — simulated enterprise with multi-vector attacks

## Key Sourcetypes

| Category | Sourcetype | Use For |
|----------|-----------|---------|
| AWS CloudTrail | `aws:cloudtrail` | API calls, IAM activity, EC2 operations |
| Email | `stream:smtp`, `ms:o365:reporting:messagetrace` | Email content, phishing, notifications |
| Authentication | `WinEventLog:Security` | Login events (4624=success, 4625=failure) |
| Endpoint | `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | Process execution, file changes |
| Network | `suricata`, `stream:tcp`, `stream:http` | Network connections, HTTP traffic |
| DNS | `stream:dns` | Domain lookups |
| Endpoint monitoring | `code42:security` | File exposure detection |
| OS query | `osquery:results` | Host telemetry, IR tool execution |

## Investigation Approach

1. Always use `index=botsv3` — this is the only relevant index
2. Start with broad queries to scope the activity, then narrow
3. Use `stats`, `timechart`, and `transaction` for correlation
4. Check the `errorCode` field in CloudTrail events — it distinguishes success from failure

```py
# {"v": 1, "name": "bots_query", "form": {"fields": [{"name": "query", "type": "string", "widget": "code", "language": "splunk"}]}, "defaults": {"query": "search index=botsv3 sourcetype=\"aws:cloudtrail\" | stats count by eventName, errorCode | sort -count | head 20"}}
results = Splunk.fetch_data(query=query)
```

[[results]]
