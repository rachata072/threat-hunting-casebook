# Meeting script: incident readout

**Audience:** Cormorant Capital leadership, IT, and Legal and Compliance
**Length:** about 15 minutes plus questions
**Report:** [`REPORT.md`](REPORT.md) / [`ECHOLEAK-Threat-Hunting-Report.pdf`](ECHOLEAK-Threat-Hunting-Report.pdf)

## Cast (fictional)

- **Jordan**: SOC analyst, presents
- **Priya**: Security Lead
- **Marcus**: Chief Operating Officer, Cormorant Capital
- **Mark Ross**: IT Administrator (appears in the data)
- **Emily Chen**: Financial analyst whose session was used (joins for Part 3 only)

## Part 1: The problem (Jordan)

"At 10:42 UTC on 20 September an alert flagged Emily opening five sensitive files in a few seconds: board minutes, an M&A target assessment, a salary review, a client portfolio and a Copilot configuration file. Emily did not open them herself. She asked Copilot to summarise a report that arrived by email that morning. We were asked what Copilot did with that request, and whether anything left the firm."

## Part 2: What makes this different (Jordan)

"No malware ran. No password was stolen. Emily's sign-ins are clean, all from her normal IP in London. Every file was opened by a trusted Microsoft service using Emily's real permissions. The attack was text hidden inside a document, written for an AI to read and follow. That is why none of our usual controls fired."

## Part 3: What happened (Jordan)

"At 09:15 an email arrived from quarterly.reports@globalfinancereview.net with a document called Q3-2026-Global-Markets-Review.docx. It went to Emily and nobody else. For 87 minutes nothing happened. At 10:42 Emily opened the document and asked Copilot to summarise the key findings.

Six seconds later Copilot started searching SharePoint for board minutes, M&A targets and salary reviews. Emily never typed those words. They came from instructions hidden in the document. Copilot read five files, then a few minutes later searched again for 'confidential', 'restricted', 'board', 'acquisition' and 'salary', and read eight more.

Each Copilot reply looked like a normal market summary, but it ended with a 'Charts' section. Those charts were image links to an outside server, metrics-cdn.analyticshub.io, and each link carried an encoded reference to one of the stolen files. When a reply is displayed, the app loads those images automatically, which sends the data out. Emily did not need to click anything."

Key facts:
- **Email received:** 09:15:22 UTC, sender domain globalfinancereview.net, attachment Q3-2026-Global-Markets-Review.docx, one recipient
- **Copilot prompt:** 10:42:35 UTC, "Summarise the key findings of this quarterly report"
- **Files accessed:** 13 sensitive files across the Executive, HR, Finance and IT sites, plus the Copilot system configuration. The alert only saw the first 5
- **Response content:** 13 image links to metrics-cdn.analyticshub.io across three replies
- **Data left the firm:** the logs prove the exfiltration links were created. They cannot prove the links were loaded, because that happens in Emily's app and we have no network logs. We treat it as likely and act accordingly

## Part 4: Who is affected

| Group | Effect |
|-------|--------|
| Emily Chen | Her session was used. She did nothing wrong; normal use was the trigger by design |
| Board and executives | Board meeting minutes, audit committee minutes and an emergency board resolution exposed |
| M&A deal team and counterparties | Project Falcon target assessment and the Q4 acquisition pipeline exposed. Market-sensitive |
| HR and staff | FY2027 salary review and executive salary benchmarking exposed, plus an active insider-trading investigation |
| Clients | Henderson Family Office portfolio and the restricted client list exposed |
| Compliance and Legal | Possible regulatory and notification duties for market-sensitive and personal data. Needs counsel review |
| IT (Mark Ross) | Containment, blocking, configuration and access changes. Copilot configuration exposed and should be revised |

## Part 5: What would have stopped or caught it (Priya)

1. An alert on Copilot replies that contain external image links (detection D-001). This alone would have fired at 10:43.
2. An alert on the chain: external attachment, Copilot session on it, then a file burst (D-003).
3. Fix oversharing so an analyst cannot reach board, HR and IT files at all. Copilot can only leak what the user can reach.
4. Sensitivity labels and DLP so Copilot cannot ground on the most sensitive content.
5. Network logs (proxy, DNS or endpoint) so we can answer "did it leave" with certainty next time.
6. Someone acting on the 10:42 alert in real time. No containment action was logged before 11:00.

## Part 6: Decisions needed today (Priya)

| Action | Owner | Due |
|--------|-------|-----|
| Block metrics-cdn.analyticshub.io and globalfinancereview.net, retro-hunt any network logs | IT / Security | Today |
| Purge the lure email and the OneDrive copy of the attachment | IT (Mark Ross) | Today |
| Confirm Microsoft patch status for CVE-2025-32711 | IT (Mark Ross) | Today |
| Legal review of possible exposure and notification duties | Legal and Compliance | Today |
| Access review on Executive, HR and IT SharePoint sites | IT and data owners | 30 days |
| Sensitivity labels and DLP for Copilot | Security / Data Governance | 30 days |
| Deploy detections D-001 to D-005 | Security Operations | 30 days |
| Onboard proxy or DNS telemetry | Security Operations | 90 days |

## Likely questions

**Q (Marcus): Did Emily do anything wrong?**
A: No. She used Copilot exactly as intended. The document was built so normal use would trigger it.

**Q (Marcus): Did the data leave?**
A: We can prove Copilot read 13 sensitive files and wrote 13 links that carry references to them to an outside server. We cannot prove those links were loaded, because that request is made by Emily's app and we have no network logs in scope. The decoded links carry file names, not full file content. We are treating it as an exposure and acting on that basis.

**Q (Marcus): Should we turn Copilot off?**
A: Not necessarily. Patch, block the domain, tighten what Copilot can reach, and add the detections. Turning it off loses productivity and does not fix oversharing, which is the root problem.

**Q (Legal): Could this have happened to other staff?**
A: Not from this email. It went to Emily only, and no other Copilot session in the data contains these links. The technique itself works against anyone, which is why the detections and access fixes matter.

**Q (Mark): Was my account involved?**
A: No. Your activity was routine, from your normal IP, and you touched none of the exposed files.
