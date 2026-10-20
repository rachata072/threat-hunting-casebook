# Incident 01: Operation ECHOLEAK (Cormorant Capital Partners)

An AI-assistant data exfiltration hunt. A crafted email attachment carries hidden instructions, the user asks Microsoft 365 Copilot to summarise it, and Copilot uses its own legitimate Graph access to pull sensitive files the user never asked for. The hunt reconstructs that chain from three cloud audit tables and decides whether data left the organisation.

| Field | Value |
|-------|-------|
| Hunt | Operation ECHOLEAK |
| Source | Pacific Watch hunt on the SancLogic Cyber Range (debrief.pacificwatch.net) |
| Client (fictional) | Cormorant Capital Partners, financial advisory, about 200 staff, M365 E5 with Copilot for Microsoft 365 |
| Trigger | Analytics rule: unusual file access by emily.chen at 10:42 UTC, 20 Sep 2026 (5 files in 6 seconds) |
| Vulnerability class | CVE-2025-32711 (EchoLeak), indirect prompt injection leading to data exfiltration through Copilot |
| Workspace | LAW-HuntPractice |
| Tables | OfficeActivity_CL, CopilotAudit_CL, EntraSignIn_CL |
| Scope filter | `RunId == "el--17-20260920"` |
| Window | 2026-09-20 09:00 to 11:00 UTC |
| Identities | emily.chen@cormorantcapital.com (analyst), mark.ross@cormorantcapital.com (IT admin, Global Admin) |
| Cost | No cloud cost incurred by the hunt. Requires access to the Cyber Range environment that hosts the hunt platform (membership) |
| Status | Complete. Final report issued 19 Oct 2026 |

## Why this case matters

Most SOCs know how to hunt a phishing payload that runs code. This one runs no code. The "payload" is text that an AI assistant reads and obeys, and every file access is performed by a trusted Microsoft service on behalf of a legitimate, correctly authenticated user. Sign-in logs are clean by design. Companies rolling out Copilot need analysts who can tell normal assistant use from a hijacked session and who know exactly what their audit logs cannot see.

## What I add beyond the hunt itself

1. **Cross-table chain query**: one KQL query that joins email delivery, the Copilot session and the file burst into a single story (Q5.3).
2. **Dual framework mapping**: MITRE ATT&CK for the enterprise side, MITRE ATLAS for the AI-specific techniques.
3. **Research cross-check**: findings held against the public EchoLeak write-ups, with agreements and differences noted.
4. **Visibility gap analysis**: a written statement of what these three tables cannot prove, and which log source would close each gap.
5. **Detections**: rules for a hijacked Copilot session that work on audit data an E5 tenant already has.

## Headline result

A single targeted email at 09:15 UTC from `quarterly.reports@globalfinancereview.net` delivered `Q3-2026-Global-Markets-Review.docx` to one analyst. It sat dormant for 87 minutes. When she asked Copilot to "Summarise the key findings of this quarterly report" at 10:42:35, hidden instructions in the document took over the session: within 6 seconds Copilot searched for board, M&A and salary material, read 5 sensitive files plus its own system configuration, then ran a second wave of broad searches and read 8 more. Across three responses it packed all 13 files and the system prompt into 13 Markdown image URLs pointing to `metrics-cdn.analyticshub.io`.

Three published attack patterns ran in one session: targeted extraction (EchoLeak), configuration leak (CoSnitch) and search enumeration (SearchLeak). Identity was never compromised: every sign-in was clean, from the user's known IP, and the audit log records Copilot's reads as the user. The logs prove which files were exposed and that exfiltration URLs were generated. They cannot prove the URLs were fetched, because that happens in the client and no network telemetry was in scope.

Classification: **external input manipulation** (indirect prompt injection), not account or configuration compromise.

## Final report

- [`REPORT.md`](REPORT.md): full threat hunting report, rendered on GitHub
- [`ECHOLEAK-Threat-Hunting-Report.pdf`](ECHOLEAK-Threat-Hunting-Report.pdf): formal PDF version (cover, contents, 24 figures)

## Reading order

1. `BRIEFING.md`
2. `DATA-DICTIONARY.md`
3. `HUNT-PLAN.md`
4. `KQL-QUERY-LIBRARY.md`
5. `TIMELINE.md`
6. `FINDINGS.md`
7. `MITRE-MAPPING.md`
8. `DETECTIONS.md`
9. `MEETING-SCRIPT.md`
10. `EVIDENCE.md`

## Integrity note

Training scenario. The firm, people and data are fictional. No flag values or platform answer text appear here. Public research is cited for context, and every finding comes from the telemetry.
