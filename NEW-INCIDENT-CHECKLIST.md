# Opening a new incident

1. Pick the next number N and a short slug, e.g. `02-short-name`.
2. Copy the incident folder structure from `incidents/01-cormorant-echoleak/` (files only, then clear the content) or start from `templates/`.
3. Fill `README.md` header: case ID, source, tier, hosts, table name, time window, cost note.
4. Write `BRIEFING.md` in your own words. Capture the baseline per user and host.
5. Run the orient queries (schema, counts, gaps) before anything else.
6. Hunt. Save each query to `KQL-QUERY-LIBRARY.md` as you go, with what it proved.
7. Update `TIMELINE.md`, `FINDINGS.md` and the noise ledger after each stage, not at the end.
8. Take screenshots into `screenshots/` using the naming rule in `screenshots/README.md`.
9. Finish with `MITRE-MAPPING.md`, `MEETING-SCRIPT.md`, then the STAR file and blog post one level up.
10. Update the incident index in the root `README.md`, `shared/ATTACK-COVERAGE.md` and `shared/DETECTION-BACKLOG.md`.
11. Run the close-out checklist in `METHODOLOGY.md`.
