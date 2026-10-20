# Framework mapping: Operation ECHOLEAK

MITRE ATT&CK covers the enterprise side (email, cloud data access, exfiltration). MITRE ATLAS covers the AI-specific behaviour (prompt injection, model-driven data leakage). Only techniques with evidence are listed.

## Final mapping

| Tactic | Technique | ID | Evidence |
|--------|-----------|----|----------|
| Initial Access | Phishing: Spearphishing Attachment | T1566.001 | Single targeted email with poisoned attachment (F-01, Q1.1, Q1.4) |
| Execution | User Execution: Malicious File | T1204.002 | User opened the file and invoked Copilot on it; nothing ran on the device (F-02) |
| AI Attack (ATLAS) | LLM Prompt Injection: Indirect | AML.T0051.001 | Search terms not present in the prompt appear 6 s after it (F-03, F-04) |
| Discovery | File and Directory Discovery | T1083 | 8 Graph.SearchFiles queries enumerating SharePoint (F-04, F-06) |
| Discovery | System Information Discovery | T1082 | Copilot system configuration read and leaked (F-05) |
| Collection | Data from Information Repositories: SharePoint | T1213.002 | 13 files read from Executive, HR, Finance and IT sites (F-04, F-06) |
| Exfiltration | Exfiltration Over C2 Channel (closest fit) | T1041 | Data carried in image URLs to metrics-cdn.analyticshub.io (F-07) |
| AI Attack (ATLAS) | LLM Data Leakage | AML.T0057 | 13 items encoded into Copilot responses (F-07, Q6.4) |

## Platform mapping vs my mapping

The hunt platform maps 8 techniques. I agree with most and disagree with two, argued from the evidence.

| Platform technique | My view | Reason |
|--------------------|---------|--------|
| T1566.001 Phishing: Spearphishing Attachment | Agree | External first-time sender, single targeted recipient |
| T1204.002 User Execution: Malicious File | Agree, with a note | The "execution" is Copilot reading the file, not code running on the device |
| T1059 Command and Scripting Interpreter | Disagree | No interpreter or script ran. The LLM carried out the instructions through its own tools, which AML.T0051.001 describes precisely |
| T1213 Data from Information Repositories | Agree, prefer T1213.002 | All files came from SharePoint |
| AML.T0051.001 LLM Prompt Injection: Indirect | Agree | Core technique |
| T1041 Exfiltration Over C2 Channel | Partly disagree | No persistent C2 channel existed; data left in image URLs fetched on render. Kept only as the closest ATT&CK fit, alongside AML.T0057 |
| T1082 System Information Discovery | Agree in spirit | Discovery of the AI system's configuration. ATLAS has a closer LLM-specific discovery technique; verify the current ID |
| T1083 File and Directory Discovery | Agree | Search calls enumerated files before reading them |

## What does not fit cleanly

- **The exfiltration channel.** ATT&CK has no technique for "an AI assistant writes a link that the victim's client loads." T1041 and T1567 both assume a channel the attacker operates or a web service the attacker uploads to. ATLAS LLM Data Leakage describes the outcome better than any ATT&CK exfiltration technique.
- **Execution.** Nothing executed on an endpoint. The closest concept is that the model "executed" natural-language instructions, which is why ATLAS is needed alongside ATT&CK for AI incidents.
- **Identity.** No credential access technique applies: the attacker never held Emily's credentials.

ATLAS identifiers should be re-verified at atlas.mitre.org before external publication, as ATLAS renames and renumbers techniques.
