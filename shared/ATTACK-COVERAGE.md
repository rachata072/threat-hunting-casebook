# ATT&CK coverage across incidents

A cell holds a technique ID only when the technique was observed with evidence.

| Tactic | 01 ECHOLEAK | 02 | 03 |
|--------|-------------|----|----|
| Initial Access | T1566.001 | | |
| Execution | T1204.002 | | |
| Persistence | | | |
| Privilege Escalation | | | |
| Defense Evasion | | | |
| Credential Access | | | |
| Discovery | T1083, T1082 | | |
| Lateral Movement | | | |
| Collection | T1213.002 | | |
| Command and Control | | | |
| Exfiltration | T1041 (closest fit) | | |
| Impact | | | |

## MITRE ATLAS coverage (AI-specific techniques)

| ATLAS technique | 01 ECHOLEAK | 02 | 03 |
|-----------------|-------------|----|----|
| LLM Prompt Injection: Indirect | AML.T0051.001 | | |
| LLM Data Leakage | AML.T0057 | | |
