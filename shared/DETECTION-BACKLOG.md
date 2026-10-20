# Detection backlog

One row per detection idea collected during a hunt. Full logic and KQL live in each incident's `DETECTIONS.md`.

| ID | Incident | Stage | ATT&CK / ATLAS | Idea | Telemetry | Status |
|----|----------|-------|----------------|------|-----------|--------|
| D-001 | 01 | Exfiltration | AML.T0057 | Copilot response contains Markdown image link to a non-approved external host | CopilotAudit | Proposed |
| D-002 | 01 | Collection | T1213.002 | One Copilot session reads 3+ files labelled Confidential or higher | CopilotAudit | Proposed |
| D-003 | 01 | Initial Access to Collection | T1566.001, AML.T0051.001 | External attachment, then Copilot session grounded on it, then 5+ Copilot file reads (file-identity join) | OfficeActivity + CopilotAudit | Proposed |
| D-004 | 01 | Collection | T1213.002 | 5+ distinct files in 1 minute on sites the user has not used in 30 days | OfficeActivity | Proposed |
| D-005 | 01 | Discovery | T1083 | 3+ Copilot search calls in one session within 5 minutes | CopilotAudit | Proposed |

Detail: [`incidents/01-cormorant-echoleak/DETECTIONS.md`](../incidents/01-cormorant-echoleak/DETECTIONS.md)
