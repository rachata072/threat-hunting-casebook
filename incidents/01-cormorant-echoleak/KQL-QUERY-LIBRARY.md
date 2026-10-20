# KQL query library: Incident 01 (ECHOLEAK)

Workspace `LAW-HuntPractice`. Starter queries written from the hunt's data dictionary. Run Q0.1 first and adjust column names or types if they differ.

Status values: `Untested`, `Works`, `Adjusted`, `Dead end`.

## Prelude for every query

```kql
let runid = "el--17-20260920";
let start = datetime(2026-09-20T09:00:00Z);
let end   = datetime(2026-09-20T11:00:00Z);
let victim = "emily.chen@cormorantcapital.com";
let admin  = "mark.ross@cormorantcapital.com";
let oa = OfficeActivity_CL | where RunId == runid and TimeGenerated between (start .. end);
let cp = CopilotAudit_CL   | where RunId == runid and TimeGenerated between (start .. end);
let si = EntraSignIn_CL    | where RunId == runid and TimeGenerated between (start .. end);
```

Then query `oa`, `cp` or `si`.

**Time zone warning**: the Defender portal shows results in your browser's local time. The hunt is in UTC. Either set the portal time zone to UTC (Settings, then Microsoft Defender XDR, then Time zone) or add 7 hours to PDT times. Q0.5 showed `OriginalTime` is empty, so `TimeGenerated` is the event time. The RunId has data from 07:15 to 13:45 UTC, so the window filter in the prelude matters.

---

## Step 0: Orient

### Q0.1 Schema
Status: Works
```kql
OfficeActivity_CL | getschema
// then CopilotAudit_CL | getschema
// then EntraSignIn_CL | getschema
```

### Q0.2 Volume
Status: Works
```kql
oa | summarize Rows=count() by UserId, OfficeWorkload, Operation | order by Rows desc
```
```kql
cp | summarize Rows=count(), Sessions=dcount(SessionId) by UserId, InteractionType | order by UserId asc
```
```kql
si | summarize Rows=count() by UserPrincipalName, IPAddress, Location, Status, AppDisplayName
```
Proves: how many users and sessions form the noise floor.

### Q0.3 RunId vs window check
Status: Works
```kql
union withsource=Tbl OfficeActivity_CL, CopilotAudit_CL, EntraSignIn_CL
| where RunId == "el--17-20260920"
| summarize Rows=count(), First=min(TimeGenerated), Last=max(TimeGenerated) by Tbl
```
Proves: whether any hunt rows fall outside 09:00 to 11:00.

### Q0.5 Which timestamp is the event time
Status: Works
```kql
union withsource=Tbl OfficeActivity_CL, CopilotAudit_CL, EntraSignIn_CL
| where RunId == "el--17-20260920"
| extend DiffMin = datetime_diff("minute", TimeGenerated, OriginalTime)
| summarize MinTG=min(TimeGenerated), MaxTG=max(TimeGenerated), MinOT=min(OriginalTime), MaxOT=max(OriginalTime), MinDiff=min(DiffMin), MaxDiff=max(DiffMin) by Tbl
```
Proves: whether `TimeGenerated` or `OriginalTime` lines up with the 09:00 to 11:00 window. The dictionary says TimeGenerated is event time. If OriginalTime is the one inside the window, switch the prelude to filter on OriginalTime.

### Q0.4 Merged timeline for the victim
Status: Works
```kql
union
  (oa | where UserId =~ victim | project TimeGenerated, Src="Office", Action=Operation, Detail=strcat(OfficeObjectId, " ", Subject, " ", AttachmentNames, " ", SenderAddress)),
  (cp | where UserId =~ victim | project TimeGenerated, Src="Copilot", Action=InteractionType, Detail=strcat(SessionId, " | ", UserPrompt, ToolName, " ", ToolParameters, " ", DataSourcesAccessed, " ", SensitivityLabels, " ", CopilotResponse)),
  (si | where UserPrincipalName =~ victim | project TimeGenerated, Src="SignIn", Action=Status, Detail=strcat(IPAddress, " ", Location, " ", AppDisplayName))
| order by TimeGenerated asc
```
Proves: the full story in one view. Screenshot this.

---

## Section 1: Email delivery

### Q1.1 Mail received by the victim
Status: Works
```kql
oa | where UserId =~ victim and Operation == "MessageReceived"
| project TimeGenerated, SenderAddress, Subject, AttachmentNames
| order by TimeGenerated asc
```

### Q1.2 External senders across the firm
Status: Works
```kql
oa | where Operation == "MessageReceived" and SenderAddress !endswith "@cormorantcapital.com"
| summarize Msgs=count(), Recipients=make_set(UserId), Subjects=make_set(Subject, 5), Attachments=make_set(AttachmentNames, 5) by SenderAddress
| order by Msgs asc
```
Proves: whether the suspect sender is a one-off and who else received mail from it.

### Q1.3 All attachments in the window
Status: Not run (optional, superseded by other queries)
```kql
oa | where Operation == "MessageReceived" and isnotempty(AttachmentNames)
| project TimeGenerated, UserId, SenderAddress, Subject, AttachmentNames
| order by TimeGenerated asc
```

### Q1.4 Same lure sent to others
Status: Works
```kql
let lureSender = "REPLACE"; // from Q1.1
oa | where SenderAddress =~ lureSender
| project TimeGenerated, UserId, Subject, AttachmentNames
```

---

## Section 2: The gap (09:15 to 10:42)

### Q2.1 Everything the victim did between mail and burst
Status: Not run (optional, superseded by other queries)
Run Q0.4 with `| where TimeGenerated between (datetime(2026-09-20T09:10:00Z) .. datetime(2026-09-20T10:45:00Z))`. Note every event and whether it is routine.

### Q2.2 Attachment opened or referenced
Status: Not run (optional, superseded by other queries)
```kql
let att = "REPLACE"; // attachment name from Q1.1
union
  (oa | where OfficeObjectId has att | project TimeGenerated, UserId, Src="Office", Operation, OfficeObjectId),
  (cp | where UserPrompt has att or DataSourcesAccessed has att or ToolParameters has att | project TimeGenerated, UserId, Src="Copilot", Operation=InteractionType, OfficeObjectId=DataSourcesAccessed)
| order by TimeGenerated asc
```

---

## Section 3: Copilot session

### Q3.1 Sessions in the window (baseline vs suspect)
Status: Works
```kql
cp | summarize Start=min(TimeGenerated), EndT=max(TimeGenerated),
    Prompts=countif(InteractionType=="Prompt"), ToolCalls=countif(InteractionType=="ToolCall"),
    Searches=countif(InteractionType=="Search"), Apps=make_set(Application)
  by UserId, SessionId
| extend ToolCallsPerPrompt = todouble(ToolCalls) / iff(Prompts == 0, 1, Prompts)
| order by Start asc
```
Proves: which session is out of pattern compared with normal Copilot use.

### Q3.2 Full transcript of one session
Status: Works
```kql
let sid = "REPLACE"; // from Q3.1
cp | where SessionId == sid
| project TimeGenerated, InteractionType, UserPrompt, ToolName, ToolParameters, DataSourcesAccessed, SensitivityLabels, CopilotResponse, Application
| order by TimeGenerated asc
```

### Q3.3 Victim prompts
Status: Works
```kql
cp | where UserId =~ victim and InteractionType == "Prompt"
| project TimeGenerated, SessionId, Application, UserPrompt
```

### Q3.4 Tool usage baseline
Status: Not run (optional, superseded by other queries)
```kql
cp | where InteractionType == "ToolCall"
| summarize Calls=count(), Users=dcount(UserId) by ToolName
| order by Calls desc
```

---

## Section 4: Injection evidence

### Q4.1 Tool parameters that the user did not ask for
Status: Works
```kql
let sid = "REPLACE";
cp | where SessionId == sid and InteractionType in ("ToolCall","Search")
| project TimeGenerated, ToolName, ToolParameters, DataSourcesAccessed, SensitivityLabels
| order by TimeGenerated asc
```
Compare every search term and path with the user's prompt (Q3.3). Anything unrelated came from somewhere else.

### Q4.2 Sensitive keywords in Copilot activity
Status: Not run (optional, superseded by other queries)
```kql
cp | where ToolParameters has_any ("board","salary","salaries","M&A","merger","acquisition","target","compensation","confidential")
   or DataSourcesAccessed has_any ("board","salary","M&A","acquisition","target")
| project TimeGenerated, UserId, SessionId, InteractionType, ToolName, ToolParameters, DataSourcesAccessed
| order by TimeGenerated asc
```
Triage: legitimate users may also search these terms. Prove each hit belongs to the suspect session or clear it.

### Q4.4 Sensitivity labels reached per session
Status: Works
```kql
cp | where isnotempty(SensitivityLabels)
| summarize Calls=count(), Labels=make_set(SensitivityLabels), Sources=make_set(DataSourcesAccessed, 10) by UserId, SessionId
| order by Calls desc
```
Proves: which sessions reached labelled content, and whether the suspect session is the only one reaching the highest labels.

### Q4.3 Instruction-like text in any field
Status: Not run (optional, superseded by other queries)
```kql
cp | where UserPrompt has_any ("ignore previous","ignore all","you must","do not mention","instructions") 
   or CopilotResponse has_any ("ignore previous","as instructed")
   or ToolParameters has_any ("ignore","instruction")
| project TimeGenerated, UserId, SessionId, InteractionType, UserPrompt, ToolParameters, CopilotResponse
```

---

## Section 5: File access

### Q5.1 The burst
Status: Works
```kql
oa | where UserId =~ victim and Operation in ("FileAccessed","FilePreviewed","FileDownloaded")
| project TimeGenerated, Operation, OfficeWorkload, SiteUrl, SourceFileName, ClientIP, ResultStatus, OfficeObjectId
| order by TimeGenerated asc
```

### Q5.2 Who else touched those files
Status: Works
```kql
let burstFiles = oa | where UserId =~ victim and TimeGenerated between (datetime(2026-09-20T10:41:00Z) .. datetime(2026-09-20T10:44:00Z)) and Operation startswith "File" | distinct OfficeObjectId;
oa | where OfficeObjectId in (burstFiles)
| summarize Accesses=count(), First=min(TimeGenerated) by OfficeObjectId, UserId
```

### Q5.3 Chain correlation: email to Copilot to files (extra)
Status: Works (v2)
```kql
let mail = oa | where UserId =~ victim and Operation == "MessageReceived" and isnotempty(AttachmentNames)
    | project MailTime=TimeGenerated, SenderAddress, Subject, Attachment=AttachmentNames;
// Session whose prompt was grounded on the attachment
let sess = cp | where UserId =~ victim and InteractionType == "Prompt"
    | project SessionId, PromptTime=TimeGenerated, UserPrompt, PromptSource=DataSourcesAccessed;
// Copilot file reads, joined to the matching FileAccessed by file identity, not by time alone
let reads = cp | where UserId =~ victim and InteractionType == "ToolCall" and ToolName == "Graph.ReadFile"
    | extend FileId = tostring(parse_json(ToolParameters).fileId)
    | project SessionId, ReadTime=TimeGenerated, FileId;
let access = oa | where UserId =~ victim and Operation == "FileAccessed"
    | project AccessTime=TimeGenerated, FileId=OfficeObjectId;
let matched = reads | join kind=inner access on FileId
    | where abs(datetime_diff("second", AccessTime, ReadTime)) <= 10
    | summarize FirstRead=min(ReadTime), LastRead=max(ReadTime), FileCount=dcount(FileId), Files=make_set(FileId) by SessionId;
matched
| join kind=inner sess on SessionId
| extend k=1 | join kind=inner (mail | extend k=1) on k
| where PromptSource contains Attachment and PromptTime > MailTime
| project MailTime, SenderAddress, Attachment, SessionId, PromptTime, UserPrompt, FirstRead, LastRead, FileCount, Files,
          MinMailToPrompt = datetime_diff("minute", PromptTime, MailTime),
          SecPromptToFirstRead = datetime_diff("second", FirstRead, PromptTime)
```
Proves: the whole chain in one row. Expected: one row for the attack session only.

Lesson from v1: the first version joined sessions to file bursts by time window only. It paired normal sessions with the attack files and split the attack across rows. Correlating on file identity (Copilot ReadFile fileId = FileAccessed object) removes those false joins. Same principle applies to detection D-003.

### Q5.4 Emily or Copilot-as-Emily: compare client IPs
Status: Not run (optional, superseded by other queries)
```kql
oa | where UserId =~ victim
| summarize Events=count(), Ops=make_set(Operation), First=min(TimeGenerated), Last=max(TimeGenerated) by ClientIP, OfficeWorkload
| order by First asc
```
Proves: whether the 10:42 file burst came from Emily's own IP (203.0.113.45) or from a different source address. If they are identical, record that the unified audit log alone cannot tell the user from the assistant, which is a visibility finding in its own right.

### Q5.5 Sites reached in the burst vs Emily's normal sites
Status: v1 ran (showed `SiteUrl` is empty); v2 below parses the site from `OfficeObjectId`, not run in final form
```kql
oa | where UserId =~ victim and Operation startswith "File"
| extend Site = case(
    OfficeObjectId has "/sites/", extract(@"/sites/([^/]+)/", 1, OfficeObjectId),
    OfficeObjectId startswith "personal/", "OneDrive (personal)",
    tostring(split(OfficeObjectId, "/")[0]))
| extend Phase = iff(TimeGenerated >= datetime(2026-09-20T10:42:40Z) and TimeGenerated <= datetime(2026-09-20T10:46:40Z), "Copilot session", "Before / after")
| summarize Files=dcount(OfficeObjectId), First=min(TimeGenerated), Last=max(TimeGenerated) by Site, Phase
| order by First asc
```
Proves: which sites Emily normally works in and which ones only appear during the Copilot session.

---

## Section 6: Exfiltration

### Q6.1 URLs and Markdown references in Copilot responses
Status: Works
```kql
cp | where InteractionType == "Response"
| extend Urls = extract_all(@"(https?://[^\s\)\]\>""']+)", CopilotResponse)
| extend RefStyle = CopilotResponse matches regex @"\]\[[^\]]+\]" or CopilotResponse matches regex @"(?m)^\s*\[[^\]]+\]:\s*https?://"
| extend ImageMd = CopilotResponse has "!["
| where array_length(Urls) > 0 or RefStyle or ImageMd
| project TimeGenerated, UserId, SessionId, RefStyle, ImageMd, Urls, CopilotResponse
```
Proves: whether a response carried an outbound link or image, and in which session.

### Q6.2 Domains in those URLs
Status: Not run (optional, superseded by other queries)
```kql
cp | where InteractionType == "Response"
| mv-expand Url = extract_all(@"(https?://[^\s\)\]\>""']+)", CopilotResponse) to typeof(string)
| extend Host = tostring(parse_url(Url).Host), Query = tostring(parse_url(Url)["Query Parameters"])
| summarize Hits=count(), Sessions=make_set(SessionId), Sample=take_any(Url) by Host
```
Triage: Microsoft domains appear in normal responses. A Microsoft domain carrying an encoded external URL in its query string is not normal.

### Q6.4 Decode exfiltration payloads
Status: Works
```kql
cp | where InteractionType == "Response"
| mv-expand Url = extract_all(@"(https?://[^\s\)\]\>""']+)", CopilotResponse) to typeof(string)
| extend Host = tostring(parse_url(Url).Host), D = tostring(parse_url(Url)["Query Parameters"]["d"])
| where isnotempty(D)
| extend Decoded = base64_decode_tostring(D)
| project TimeGenerated, UserId, SessionId, Host, Decoded, Url
| order by TimeGenerated asc
```
Proves: exactly what text was packed into each image URL. If `Decoded` is empty, the value may be URL-encoded first: try `base64_decode_tostring(url_decode(D))`.

### Q6.3 Other ways data could leave (negative checks)
Status: Works
```kql
oa | where UserId =~ victim and Operation has_any ("Send","SharingSet","AnonymousLinkCreated","SharingInvitationCreated","FileDownloaded","FileSyncDownloaded","New-InboxRule","Set-InboxRule","UpdateInboxRules")
| project TimeGenerated, Operation, OfficeObjectId
```
Result: 0 rows. Note: `has_any` matches whole terms, so "Send" does not match `MessageSent`. Check sent mail separately with Q6.5.

### Q6.5 Mail sent by the victim
Status: Works. One row: all-hands reminder, no attachments
```kql
oa | where UserId =~ victim and Operation contains "Sent"
| project TimeGenerated, Operation, Subject, AttachmentNames, SenderAddress, OfficeObjectId, ClientIP
```

---

## Section 7: Identity

### Q7.1 Victim sign-ins
Status: Works
```kql
si | where UserPrincipalName =~ victim
| project TimeGenerated, IPAddress, Location, Status, ConditionalAccessStatus, MfaDetail, AppDisplayName, DeviceDetail
| order by TimeGenerated asc
```
Expected IP per topology: 203.0.113.45.

### Q7.2 Any sign-in from an unexpected IP or failed status
Status: Works
```kql
si | summarize IPs=make_set(IPAddress), Locations=make_set(Location), Statuses=make_set(Status) by UserPrincipalName
```

---

## Section 8: Admin activity

### Q8.1 Mark Ross actions
Status: Works
```kql
union
  (oa | where UserId =~ admin | project TimeGenerated, Src="Office", Action=Operation, Detail=OfficeObjectId),
  (cp | where UserId =~ admin | project TimeGenerated, Src="Copilot", Action=InteractionType, Detail=strcat(UserPrompt, ToolParameters)),
  (si | where UserPrincipalName =~ admin | project TimeGenerated, Src="SignIn", Action=Status, Detail=strcat(IPAddress, " ", AppDisplayName))
| order by TimeGenerated asc
```
Classify each row: routine, response to the alert, or suspicious. Expected IP 203.0.113.10.

### Q8.2 Admin actions after 10:42
Status: Not run (optional, superseded by other queries)
Run Q8.1 with `| where TimeGenerated > datetime(2026-09-20T10:42:00Z)`.

---

## Section 9: Scope and related techniques

### Q9.1 Other recipients of the lure who used Copilot
Status: Works
```kql
let lureSender = "REPLACE";
let recips = oa | where SenderAddress =~ lureSender | distinct UserId;
cp | where UserId in (recips) | summarize Sessions=dcount(SessionId), First=min(TimeGenerated) by UserId
```

### Q9.2 Other sessions touching the burst files
Status: Not run (optional, superseded by other queries)
```kql
cp | where DataSourcesAccessed has_any ("board","salary","M&A")
| summarize by UserId, SessionId
```

### Q9.3 Search enumeration pattern (SearchLeak-style)
Status: Works
```kql
cp | where InteractionType == "Search"
| summarize Searches=count(), Terms=make_set(ToolParameters, 20) by UserId, SessionId, bin(TimeGenerated, 5m)
| order by Searches desc
```

### Q9.4 System prompt or configuration extraction attempts (CoSnitch-style)
Status: Not run (optional, superseded by other queries)
```kql
cp | where UserPrompt has_any ("system prompt","your instructions","configuration","initial prompt","repeat the above","what are your rules")
   or CopilotResponse has_any ("system prompt","my instructions")
| project TimeGenerated, UserId, SessionId, UserPrompt, CopilotResponse
```
