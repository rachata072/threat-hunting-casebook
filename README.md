# Threat Hunting Casebook

A growing series of threat hunting investigations. Each incident is reconstructed from raw telemetry with KQL (Microsoft Sentinel / Log Analytics), written up the way a SOC or IR team would hand it to a client: timeline, evidence, MITRE ATT&CK mapping, noise versus attacker judgement, and detections that would have caught it.

One repo, one folder per incident. Every incident uses the same file set so the series reads consistently and each new case is quick to start. Incidents cover both endpoint telemetry (Sysmon, Windows Security) and cloud and AI audit logs (M365, Copilot, Entra ID).

## Incident index

| # | Case ID | Scenario | Tier | Hosts | Status |
|---|---------|----------|------|-------|--------|
| 01 | [Operation ECHOLEAK](incidents/01-cormorant-echoleak/) | Cormorant Capital Partners: M365 Copilot indirect prompt injection and data exfiltration (CVE-2025-32711) | Practice | 3 cloud tables | Complete: [report](incidents/01-cormorant-echoleak/REPORT.md) / [PDF](incidents/01-cormorant-echoleak/ECHOLEAK-Threat-Hunting-Report.pdf) |
| 02 | | | | | Not started |

### Parked

| Slot | Case ID | Scenario | Why parked |
|------|---------|----------|------------|
| P1 | HL-INC-2026-0204 | Harborlight Dental & Insurance: ransomware, full reconstruction | Past due date on the platform. Kept locally and published only when complete |

Open the incident folder and start at its `README.md`, or go straight to the final report.

## Featured: Incident 01, Operation ECHOLEAK

An analyst asked Microsoft 365 Copilot to summarise an emailed report. Hidden instructions in the document hijacked the session: Copilot searched for and read 13 sensitive files plus its own system configuration, then hid references to them in image links pointing to an attacker domain. The alert saw only 5 of the files. Sign-ins were clean throughout.

- 37-page formal report with 24 evidence figures, MITRE ATT&CK and ATLAS mapping, IOCs and 11 recommendations
- Cross-table KQL correlation joining email, Copilot session and file access on file identity
- Five proposed detections, the first of which would have fired one minute into the attack
- A precise statement of what the logs cannot prove, and the telemetry that would close each gap

## What every incident contains

| File | Purpose |
|------|---------|
| `README.md` | Case summary, scope, status, headline result |
| `BRIEFING.md` | The client brief and baseline, restated in my own words |
| `HUNT-PLAN.md` | Stage-by-stage hypotheses and what to look for |
| `KQL-QUERY-LIBRARY.md` | Every query used, grouped by stage, with what it proved |
| `TIMELINE.md` | The reconstructed attack timeline (UTC) |
| `FINDINGS.md` | Evidence-backed findings, noise ledger, reasoning |
| `REPORT.md` + PDF | The final formal report |
| `MITRE-MAPPING.md` | Techniques observed, with the evidence for each |
| `DETECTIONS.md` | Rules that would have caught it earlier |
| `MEETING-SCRIPT.md` | Client readout: problem, fix, who is affected |
| `EVIDENCE.md` + `screenshots/` | Screenshot checklist and result placeholders |

Interview STAR answers and the blog post live one level above the repo, per incident.

## Series-wide files

- `METHODOLOGY.md`: how I run a hunt, how I classify attacker versus MSP versus noise, how I grade confidence.
- `shared/KQL-COMMON.md`: reusable query patterns by tactic.
- `shared/ATTACK-COVERAGE.md`: ATT&CK coverage across all incidents.
- `shared/DETECTION-BACKLOG.md`: detection ideas collected from every case.
- `templates/`: copy these to start a new incident.
- `NEW-INCIDENT-CHECKLIST.md`: the steps to open incident N+1.

## Ground rules for this repo

1. Scenarios come from a hunt training platform. Each incident README names the source and what I added on top.
2. No flag values, answer keys or verbatim platform hint text are published. Everything here is my own reasoning and wording.
3. Every claim in a finding points to a query and a row of evidence.
4. Negative results are recorded, not hidden.
5. Cost disclosure per incident is stated in that incident's README.

## License

MIT. See `LICENSE`.
