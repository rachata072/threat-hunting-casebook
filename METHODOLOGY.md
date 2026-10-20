# Methodology

How every incident in this casebook is worked. The point is a repeatable process, not a pile of clever queries.

## 1. Principles

- **Baseline first.** Write down what normal looks like for each user and host before hunting. Anything outside the baseline needs an explanation, not an assumption.
- **Use the right clock.** Confirm which time field is the event time and which is ingestion time. Wrong clock means a wrong timeline.
- **Work the chain, but pivot freely.** Stages are a guide. A finding in stage 6 often sends you back to stage 2.
- **Evidence or it did not happen.** Every timeline row cites a query and the key fields from the result.
- **Record negatives.** "Checked for X, found nothing" is a finding. It shows coverage.
- **Do not stop at the loud part.** The obvious event (for example ransomware notes) is rarely the end. Hunt after it.
- **Be honest about help.** Log any hints or outside help used.

## 2. The hunt loop

1. **Orient.** Schema, row counts, hosts, users, time range, telemetry gaps.
2. **Hypothesise.** For the current stage, which behaviours would show up in this telemetry?
3. **Query.** Smallest query that tests the hypothesis. Save it in the query library.
4. **Triage.** Classify every suspicious result (see section 3).
5. **Pivot.** Take a key from the result and widen the search (see section 4).
6. **Record.** Add a timeline row, a finding or a noise-ledger entry, and a screenshot.
7. **Detect.** Write one detection idea that would have caught it earlier.

## 3. Triage: attacker, MSP, or noise

Every suspicious thing gets one label and a reason.

| Label | Meaning | What proves it |
|-------|---------|----------------|
| ATK | Attacker activity | Outside baseline AND linked to the chain by user, host, process lineage, time or destination |
| MSP | Legitimate admin or tooling | Matches a known admin, tool, schedule or session, and nothing links it to the chain |
| NOISE | Background system behaviour | Occurs on all comparable hosts, on a schedule, with no link to the chain |
| UNK | Not yet decided | Needs another pivot. Never leave UNK in the final report |

Questions to ask before labelling:
- Does this user or account normally do this on this host?
- Is there a legitimate admin session or change window behind it?
- Is the binary where it should be, signed, and named as expected?
- Does the same thing appear on a quiet host with no incident?
- What happened one minute before and one minute after?

## 4. Pivot keys

| Key | Use it to find |
|-----|----------------|
| User / account | Where else that identity appeared |
| Host | Everything else on the machine in a time window |
| Process GUID / PID + host | Full lineage: parent, children, network, files |
| Parent process | Siblings and what else it launched |
| Command-line fragment | Same tooling on other hosts |
| File name / hash | Where else the file landed |
| Remote IP / domain | Other hosts talking to the same place |
| Logon ID | Everything done in one session |
| Time window | Quiet gaps and bursts around a known event |

## 5. Confidence

- **Confirmed**: seen in two independent telemetry sources, or one source plus clear lineage.
- **Probable**: one source, consistent with the chain, no contradiction.
- **Possible**: fits, but alternative explanations remain.

## 6. Close-out checklist

- [ ] Every stage has a finding or a documented negative
- [ ] No UNK left in the noise ledger
- [ ] Timeline is in UTC and sorted
- [ ] Every timeline row cites a query and evidence
- [ ] Post-impact activity was hunted explicitly
- [ ] ATT&CK mapping only includes techniques with evidence
- [ ] Detection ideas added to the shared backlog
- [ ] Hint and help log is complete
- [ ] No answer keys or platform text published
