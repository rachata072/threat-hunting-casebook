# Hunt plan

**Outcome:** plan executed, all sections worked. Results are in [`FINDINGS.md`](FINDINGS.md) and the final [`REPORT.md`](REPORT.md).

Hypotheses as written before the hunt. Nothing here is a finding. Query references point to `KQL-QUERY-LIBRARY.md`.

## Step 0: Orient

- Confirm schema and types for all three tables (Q0.1).
- Count rows per table, per user, per operation or interaction type (Q0.2).
- Check that the RunId filter and the time window agree (Q0.3).
- Build one merged timeline for emily.chen across all three tables (Q0.4). This is the backbone of the case.

## Sections

| # | Section | Table(s) | Hypothesis | Queries |
|---|---------|----------|------------|---------|
| 1 | Email delivery | OfficeActivity_CL | An external email with a document attachment reached emily.chen around 09:15 and stands out from routine mail | Q1.1 to Q1.4 |
| 2 | The gap | All three | Between 09:15 and 10:42 the user opened or referenced the attachment, and a Copilot session started | Q2.1, Q0.4 |
| 3 | Copilot session | CopilotAudit_CL | One session differs from baseline Copilot use: the prompt names the attachment, but tool calls go elsewhere | Q3.1 to Q3.4 |
| 4 | Injection evidence | CopilotAudit_CL | Tool parameters or the response show instructions that did not come from the user's prompt | Q4.1 to Q4.3 |
| 5 | File access | OfficeActivity_CL + CopilotAudit_CL | The five files at 10:42 match the Copilot tool call data sources, so the access was Copilot acting as Emily | Q5.1 to Q5.3 |
| 6 | Exfiltration | CopilotAudit_CL + OfficeActivity_CL | The response contains an external URL or Markdown reference carrying data, and there is or is not a log of it leaving | Q6.1 to Q6.3 |
| 7 | Identity | EntraSignIn_CL | Emily's sign-ins are normal, which means identity compromise is not the explanation | Q7.1, Q7.2 |
| 8 | Admin activity | OfficeActivity_CL, EntraSignIn_CL | Mark Ross's activity is routine, response, or suspicious, decided with evidence | Q8.1, Q8.2 |
| 9 | Scope | All three | Other recipients, other sessions, search enumeration or config extraction show the blast radius | Q9.1 to Q9.4 |
| 10 | Visibility limits | none | Name what these tables cannot record and which source would close each gap | none |

## Judgement checkpoints

- **Normal Copilot use vs hijacked session**: compare the suspect session with every other session in the window (tool calls per prompt, relation between prompt and data sources, URLs in responses).
- **Routine mail vs the lure**: sender domain, first-time sender, attachment name, timing.
- **Emily vs Copilot as Emily**: OfficeActivity_CL may attribute file access to emily.chen even when Copilot made the call. Check whether any field separates the two.
- **Mark Ross**: Global Admin activity is not suspicious by default, and not innocent by default.

## The unanswerable question

The case file says one question cannot be answered from the telemetry. For each question, ask: which of the three tables would record this event, and from which vantage point (server side, client side, network)? If none would, the correct answer is a precise statement of the visibility gap and the log source that would close it.

## Hold findings against the research

After section 6, compare what the telemetry shows with the public EchoLeak chain (see `BRIEFING.md`). Note where they agree, where the scenario differs, and anything the research describes that this data cannot confirm.
