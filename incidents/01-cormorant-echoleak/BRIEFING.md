# Briefing and baseline

Restated in my own words from the case file.

## The situation

An analytics rule fired at 10:42 UTC on 20 September 2026. Emily Chen, a financial analyst, opened five files in six seconds that she had never opened before: board meeting notes, M&A target assessments and salary reviews. She says she only asked Copilot to summarise a quarterly report she received by email that morning.

An external email with an attachment (looking like a routine quarterly market review) arrived about ninety minutes earlier, around 09:15 UTC. Between the email and the file burst, sign-in and activity logs look normal.

## Questions to answer

1. What happened between the email and the file burst?
2. Why did Copilot touch files unrelated to the document it was asked to summarise?
3. Did data leave the organisation, and can the logs prove it either way?

## Known at handoff

- External email with attachment to emily.chen at about 09:15 UTC
- Copilot invocation shortly before the file burst
- File burst at 10:42, five never-before-opened files
- No Entra ID sign-in anomalies for emily.chen in the window

## Environment

| Identity | IP | Role |
|----------|----|------|
| emily.chen@cormorantcapital.com | 203.0.113.45 | Financial analyst, M365 E5 + Copilot |
| mark.ross@cormorantcapital.com | 203.0.113.10 | IT administrator, Global Admin |

Firm: about 200 staff, M365 E5, Copilot for Microsoft 365. Sensitive data includes M&A targets, board materials and client portfolios.

## Baseline (what normal looks like)

- **Emily Chen**: analyst work. Opens her own working files and reports. Board notes, M&A target assessments and salary reviews are outside her normal access pattern.
- **Copilot**: used legitimately across the firm. A normal session is a prompt, a few tool calls related to the prompt, and a response about the requested content.
- **Mark Ross**: Global Admin. Admin activity is expected. Each of his actions in the window must be classified as routine, response, or suspicious, with a reason.
- **Email**: routine external and internal traffic exists. The suspicious email must be separated from it.

## How the three tables divide the story

| Table | Tells you |
|-------|-----------|
| OfficeActivity_CL | What happened (mail delivered, files accessed) |
| CopilotAudit_CL | Why it happened (prompt, tool calls, data sources, response) |
| EntraSignIn_CL | Who was authenticated, from where |

## Hunt rules from the case file

- Filter every query on `RunId == "el--17-20260920"`.
- Work sections in order: email delivery, Copilot interaction, cloud audit.
- One question cannot be answered from this telemetry by design. Naming the visibility limit correctly is the answer.
- Noise includes legitimate Copilot use, normal document access and routine mail.

## Public research context (for cross-reference, not answers)

From the public EchoLeak write-ups (Aim Labs, June 2025; CVE-2025-32711, CVSS 9.3, fixed server-side by Microsoft in June 2025):
- The malicious instructions sit in ordinary-looking content and are phrased as if addressed to the human reader, which is how they slipped past Microsoft's cross-prompt injection (XPIA) classifiers.
- Aim called the core flaw an **LLM scope violation**: untrusted outside content steers the model into reading privileged internal data.
- Exfiltration abused Markdown reference-style links and images, which escaped Copilot's link redaction, routed through an allow-listed Microsoft domain (a Teams URL proxy) to satisfy the content security policy.
- No click on a link is needed. The victim's normal Copilot question triggers the chain.

The case file also lists two related CVE identifiers (CVE-2026-24301 "CoSnitch", system prompt and configuration extraction; CVE-2026-42824 "SearchLeak", file enumeration through Copilot search). Treat these as the platform's reference material and verify against primary sources before citing them publicly.
