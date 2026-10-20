# Data dictionary

Restated from the hunt's reference tables. Confirm real column names and types with `getschema` (Q0.1) before trusting any query. Custom tables often store everything as strings.

## OfficeActivity_CL (M365 unified audit log)

| Field | Use |
|-------|-----|
| TimeGenerated | Event time |
| UserId | User principal name |
| Operation | Activity, e.g. MessageReceived, FileAccessed |
| OfficeWorkload | Exchange, OneDrive, SharePoint |
| OfficeObjectId | Target object (file path, mailbox) |
| SenderAddress | Sender (Exchange events) |
| Subject | Email subject (Exchange events) |
| AttachmentNames | Attachment filenames (Exchange events) |
| RunId | Filter `el--17-20260920` |

## CopilotAudit_CL (Copilot interaction audit)

| Field | Use |
|-------|-----|
| TimeGenerated | Event time |
| UserId | User principal name |
| InteractionType | Prompt, Response, ToolCall, Search |
| UserPrompt | User input (Prompt) |
| CopilotResponse | Copilot output (Response) |
| ToolName | Graph tool invoked (ToolCall) |
| ToolParameters | Tool parameters (ToolCall) |
| DataSourcesAccessed | Files or sources referenced (ToolCall) |
| Application | M365 app context |
| SessionId | Copilot session ID |
| RunId | Filter `el--17-20260920` |

## EntraSignIn_CL (Entra ID sign-in log)

| Field | Use |
|-------|-----|
| TimeGenerated | Event time |
| UserPrincipalName | User principal name |
| IPAddress | Client IP |
| Location | Geolocation |
| Status | Sign-in result |
| ConditionalAccessStatus | CA result |
| AppDisplayName | Application |
| DeviceDetail | Device info |
| RunId | Filter `el--17-20260920` |

## Join keys

| From | To | Key |
|------|----|-----|
| OfficeActivity_CL.UserId | CopilotAudit_CL.UserId | UPN |
| OfficeActivity_CL.UserId | EntraSignIn_CL.UserPrincipalName | UPN |
| OfficeActivity_CL.AttachmentNames | CopilotAudit_CL.DataSourcesAccessed / UserPrompt | Attachment filename |
| CopilotAudit_CL.DataSourcesAccessed | OfficeActivity_CL.OfficeObjectId | File path or name |
| CopilotAudit_CL rows | each other | SessionId |

## Observed schema notes

Confirmed from Q0.1 on all three tables. Every field is a string except `TimeGenerated` and `OriginalTime` (datetime). The documented fields all exist, plus these extras:

| Table | Extra column | Why it matters |
|-------|--------------|----------------|
| OfficeActivity_CL | `ClientIP` | Populated. All of Emily's rows, including Copilot-driven reads, show 203.0.113.45 (Q5.4), so it cannot separate user from agent |
| OfficeActivity_CL | `ResultStatus` | Success or failure of the operation |
| OfficeActivity_CL | `ItemType` | File, mail item, folder and so on |
| OfficeActivity_CL | `SiteUrl` | Empty for the victim's rows in this dataset. Parse the site from `OfficeObjectId` instead (Q5.5) |
| OfficeActivity_CL | `SourceFileName` | Clean filename, easier to match against Copilot data sources than the full path |
| CopilotAudit_CL | `SensitivityLabels` | Labels on the data Copilot touched. Strong signal for a session reaching Confidential content |
| EntraSignIn_CL | `MfaDetail` | How MFA was satisfied. Supports the "identity was not compromised" finding |
| All | `OriginalTime` | A second timestamp. Check against TimeGenerated before relying on either (Q0.5) |
| All | `HuntName`, `Campaign` | Lab metadata. Do not use as evidence: a finding must stand on the telemetry, not on a label the lab added |

