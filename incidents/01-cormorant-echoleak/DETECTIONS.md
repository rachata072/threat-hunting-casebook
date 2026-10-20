# Detections

Rules that would have caught this hijacked Copilot session earlier, using audit data an E5 tenant already has. Logic is derived from the evidence in this hunt. **Status: proposed, not yet validated in production.** Tune against normal Copilot use before enabling.

| ID | Rule (plain words) | Tables | Noise risk | Response | Would have fired at |
|----|--------------------|--------|------------|----------|---------------------|
| D-001 | Copilot response contains Markdown image syntax pointing to a host outside approved corporate domains | CopilotAudit_CL | Low to medium | Alert, analyst review | 10:43:08 |
| D-002 | One Copilot session reads 3 or more files labelled Confidential or higher | CopilotAudit_CL | Medium | Alert | 10:42:51 |
| D-003 | External email with attachment, then a Copilot session grounded on that attachment within 3 hours, then 5 or more Copilot file reads | OfficeActivity_CL + CopilotAudit_CL | Low | High-priority alert | 10:42:55 |
| D-004 | User accesses 5 or more distinct files within 1 minute on sites not accessed in the previous 30 days | OfficeActivity_CL | Medium | Alert | 10:42:55 |
| D-005 | 3 or more Copilot search calls in one session within 5 minutes | CopilotAudit_CL | Medium | Alert | 10:42:45 |

## D-001: external image links in Copilot responses

```kql
CopilotAudit_CL
| where TimeGenerated > ago(1h) and InteractionType == "Response"
| extend ImgHost = extract(@"!\[[^\]]*\]\(https?://([^/\)\s]+)", 1, CopilotResponse)
| where isnotempty(ImgHost)
| where not(ImgHost endswith "cormorantcapital.sharepoint.com" or ImgHost endswith ".microsoft.com")
| project TimeGenerated, UserId, SessionId, ImgHost
```
False positives: legitimate responses that embed public images. Maintain an allow list of approved hosts.

## D-002: labelled content reached by one session

```kql
CopilotAudit_CL
| where TimeGenerated > ago(1h) and InteractionType == "ToolCall" and ToolName == "Graph.ReadFile"
| where SensitivityLabels has_any ("Confidential", "Highly Confidential", "Restricted")
| summarize LabelledReads=count(), Files=make_set(DataSourcesAccessed, 20) by UserId, SessionId
| where LabelledReads >= 3
```
False positives: executives and analysts who legitimately research labelled content. Scope by role or site.

## D-003: email to Copilot to file burst chain

Based on Q5.3 in [`KQL-QUERY-LIBRARY.md`](KQL-QUERY-LIBRARY.md), scheduled hourly with a 3-hour lookback. Correlates on file identity (Copilot `Graph.ReadFile` fileId = `FileAccessed` object within 10 seconds), not on time alone. The first version of this query joined on time alone and produced false matches; the file-identity join removed them.

## D-004: burst of first-time file access

```kql
let lookback = OfficeActivity_CL
    | where TimeGenerated between (ago(30d) .. ago(1h)) and Operation == "FileAccessed"
    | extend Site = extract(@"/sites/([^/]+)/", 1, OfficeObjectId)
    | distinct UserId, Site;
OfficeActivity_CL
| where TimeGenerated > ago(1h) and Operation == "FileAccessed"
| extend Site = extract(@"/sites/([^/]+)/", 1, OfficeObjectId)
| join kind=leftanti lookback on UserId, Site
| summarize Files=dcount(OfficeObjectId), Sites=make_set(Site) by UserId, bin(TimeGenerated, 1m)
| where Files >= 5
```
Note: in this dataset Copilot's reads are recorded as the user, from the user's IP, so this rule alone cannot tell a person from an agent. Pair it with D-002 or D-003.

## D-005: Copilot search burst

```kql
CopilotAudit_CL
| where TimeGenerated > ago(1h) and InteractionType == "Search"
| summarize Searches=count(), Terms=make_set(ToolParameters, 20) by UserId, SessionId, bin(TimeGenerated, 5m)
| where Searches >= 3
```

## Controls that reduce the risk (beyond detection)

1. Sensitivity labels and DLP policies that stop Copilot grounding on board, HR, investigation and M&A content.
2. Remove oversharing: analysts should not be able to reach Executive, HR and IT sites at all. Copilot can only leak what the user can reach.
3. Restrict rendering of external images in Copilot responses where the platform allows it.
4. Treat external attachments as untrusted input for AI tools (mail filtering and detonation).
5. Network telemetry (proxy, DNS or endpoint) so exfiltration delivery can be confirmed.

All five rules are copied to [`../../shared/DETECTION-BACKLOG.md`](../../shared/DETECTION-BACKLOG.md).
