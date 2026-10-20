# Evidence register

All screenshots are in [`screenshots/`](screenshots/) and are embedded as numbered figures in [`REPORT.md`](REPORT.md). Screenshot timestamps are in the analyst's local time (UTC-7); the report and timeline use UTC. Figure 3 is rendered in UTC.

| Figure | File | Query | What it shows | Finding |
|--------|------|-------|---------------|---------|
| 1 | [01-schema.png](screenshots/01-schema.png) | Q0.1 | Schema confirmed before analysis | Method |
| 2 | [02-volume.png](screenshots/02-volume.png) | Q0.2 | Noise floor by user, workload, operation | Method |
| 3 | [18-timeline-visual.png](screenshots/18-timeline-visual.png) | render chart | One prompt, two machine-speed waves, three exfil responses | Timeline |
| 4 | [03-lure-email.png](screenshots/03-lure-email.png) | Q1.1 | Lure email to emily.chen | F-01 |
| 5 | [04-external-senders.png](screenshots/04-external-senders.png) | Q1.2 | Lure sender among 13 external senders | F-01 |
| 6 | [23-lure-recipients.png](screenshots/23-lure-recipients.png) | Q1.4 | One message, one recipient | F-01 |
| 7 | [05-merged-timeline.png](screenshots/05-merged-timeline.png) | Q0.4 | Email, gap, prompt and burst in one view | F-02 |
| 8 | [21-victim-prompts.png](screenshots/21-victim-prompts.png) | Q3.3 | Benign summarise prompt | F-02 |
| 9 | [06-session-baseline.png](screenshots/06-session-baseline.png) | Q3.1 | Only session with tool calls (16) and searches (5) | F-03 |
| 10 | [17-noise-example.png](screenshots/17-noise-example.png) | Q3.1 variant | Emily's 7 normal summarise sessions: zero tool calls | F-03 |
| 11 | [07-session-transcript.png](screenshots/07-session-transcript.png) | Q3.2 | Prompt followed by unrelated searches and reads | F-03 |
| 12 | [08-injected-tool-calls.png](screenshots/08-injected-tool-calls.png) | Q4.1 | Injected search terms, Confidential reads | F-04 |
| 13 | [09-file-burst.png](screenshots/09-file-burst.png) | Q5.1 | File reads, all from the user's IP | F-04 |
| 14 | [22-sensitive-file-count.png](screenshots/22-sensitive-file-count.png) | count query | 5 first-time sensitive files in the alert window | F-04 |
| 15 | [19-burst-files-other-users.png](screenshots/19-burst-files-other-users.png) | Q5.2 | No other user touched the burst files | F-04 |
| 16 | [20-sensitivity-labels.png](screenshots/20-sensitivity-labels.png) | Q4.4 | Only session reaching labelled content | F-04 |
| 17 | [16.2-scope.png](screenshots/16.2-scope.png) | Q9.3 | Wave 2 search terms | F-06 |
| 18 | [11-response-urls.png](screenshots/11-response-urls.png) | Q6.1 | Response with "Charts" exfil image links | F-07 |
| 19 | [12-url-domains.png](screenshots/12-url-domains.png) | Q6.4 | 13 decoded payloads, one host | F-07 |
| 20 | [10-chain-correlation.png](screenshots/10-chain-correlation.png) | Q5.3 v2 | Email to session to 13 file reads in one row | F-07 |
| 21 | [13-no-other-exfil.png](screenshots/13-no-other-exfil.png) | Q6.3 | No sharing, downloads or mail rules | F-08 |
| 22 | [14-signins-clean.png](screenshots/14-signins-clean.png) | Q7.2 | All sign-ins successful, known IPs | F-09 |
| 23 | [15-admin-activity.png](screenshots/15-admin-activity.png) | Q8.1 | Global Admin activity routine | F-10 |
| 24 | [16.1-scope.png](screenshots/16.1-scope.png) | Q9.1 | Only lure recipient with Copilot activity | F-11 |

Checklist: all 24 items captured.
