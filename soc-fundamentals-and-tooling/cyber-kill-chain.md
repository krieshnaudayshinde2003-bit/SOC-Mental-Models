# 02 - Cyber Kill Chain

## Core idea

The Cyber Kill Chain, from Lockheed Martin, breaks an intrusion into seven stages: Reconnaissance, Weaponization, Delivery, Exploitation, Installation, Command and Control, and Actions on Objectives. The idea is that an attacker has to complete the stages to succeed, so a defender who breaks the chain at one point stops that attempt. The attacker can still retry by a different route, so breaking the chain is not the same as ending the threat.

As an analyst I don't use it as a memorization list. I use it to answer three practical questions when an alert fires: **where in the attack is this, what must have happened before it, and what is likely to happen next.**

## Mental model: look backward and forward

When an alert fires, I place it on the chain, then work in both directions.

- **Look backward (how did they get here?).** If I confirm Command and Control on a host, earlier stages very likely happened, and I need to find where. That means the original entry point: the email, the download, the vulnerable service. It identifies patient zero and the initial vector, and tells me who else may have received the same payload. I treat this as "probably, go and find it", not "certainly". An in-memory implant may leave no persistence, and stolen credentials with a legitimate remote tool may involve no delivered payload at all.
- **Look forward (what comes next?).** If I confirm Exploitation, I check for persistence and outbound communication. If I see C2, I check for credential theft, lateral movement, and data staging.
- **Earlier is cheaper.** A phishing email caught at Delivery can cost as little as a deleted message, provided I also confirm nobody interacted with it and no other copies got through. The same attack found at Actions on Objectives costs an incident. This is why triaging "low severity" Delivery alerts well matters.

## The seven stages through a SOC lens

| Stage | What the attacker does | Do I see it? | Where I would see it |
|---|---|---|---|
| 1. Reconnaissance | Gathers information on targets: staff, email formats, exposed services | Rarely | Web logs, firewall logs (scanning), threat intel on the company's exposure |
| 2. Weaponization | Builds the payload, e.g., a malicious document or exploit | Almost never | Happens on attacker infrastructure. Only threat intel gives visibility |
| 3. Delivery | Sends the payload: phishing email, malicious link, drive-by download, USB | Yes | Email gateway, proxy, web filter, EDR (downloaded file) |
| 4. Exploitation | Triggers a vulnerability or the user's action to run code | Yes | EDR (process execution), endpoint logs, IDS/IPS, application logs |
| 5. Installation | Establishes persistence: scheduled task, registry run key, service, web shell | Yes | EDR, Windows event logs, file integrity monitoring |
| 6. Command and Control | The compromised host connects back to the attacker | Yes | Proxy, DNS, firewall, NetFlow, EDR network connections |
| 7. Actions on Objectives | Data theft, credential harvesting, lateral movement, ransomware, destruction | Yes, but late | EDR, authentication logs, file server and DLP logs, SIEM correlation |

**What this tells me:** stages 1 and 2 are mostly invisible to a SOC. My real detection window starts at Delivery, so that is where I concentrate tooling and attention.

## What to look for (alert-to-stage examples)

| Alert type | Stage | What I check next |
|---|---|---|
| Phishing email flagged by gateway | Delivery | Was it delivered or quarantined? Did anyone click or open an attachment? |
| Suspicious Office process spawning a shell | Exploitation | Parent/child process chain, command line, network activity after |
| New scheduled task or registry run key from a suspicious process | Installation | What created it, and what does it launch? |
| Periodic outbound connections to a rare domain | Command and Control | Beaconing interval, destination reputation and age, other hosts contacting it |
| Large outbound transfer or mass file access | Actions on Objectives | What data, to where, which account, how much already left |

> **Homework (not done):** for each row above, add the benign explanations I would rule out first and how I tell them apart from malicious activity. Starting points to research: legitimate macros or admin scripts spawning shells from Office, installers creating scheduled tasks, update checks and telemetry producing periodic outbound traffic.

## Triage questions

1. Which stage does this alert represent?
2. What evidence do I have that earlier stages occurred? (Did the user click? Did the file run?)
3. Did the delivery succeed, or was it blocked or never opened?
4. Is there evidence of the next stage? (Persistence, outbound connection, new processes)
5. Who else received or touched the same artifact? I search on several things, not just the hash: sender, subject, URL and domain, and behavior (the same process chain). A hash only matches the identical file, so no hash hit does not mean clean.
6. How much time has passed between stages? Fast progression means urgency.

## Where it breaks down (and why analysts also use MITRE ATT&CK)

The Kill Chain has real limits, and I should be able to say so.

- **It is linear, but real attacks are not.** Attackers loop, skip stages, and repeat steps. After landing on one host they may repeat Exploitation and Installation on others.
- **It is built around malware and perimeter intrusion.** It fits phishing-to-malware well. It fits poorly for stolen credentials, insider threats, cloud and SaaS compromise, or attackers who never drop malware.
- **It is thin after the foothold.** Privilege escalation, credential access, discovery, and lateral movement are crammed into "Actions on Objectives", and that is where much of the real intrusion happens.
- **No technique-level detail.** It tells me the phase, not the specific behavior to hunt for.

MITRE ATT&CK covers these gaps with tactics and specific techniques. In practice: the Kill Chain gives me the story and the phase, and ATT&CK gives me the exact behavior to detect and report on.

## MITRE ATT&CK mapping

Rough, tactic-level mapping. It is approximate, because ATT&CK is not a strict extension of the Kill Chain.

| Kill Chain stage | ATT&CK tactic (ID) |
|---|---|
| Reconnaissance | Reconnaissance (TA0043) |
| Weaponization | Resource Development (TA0042) |
| Delivery / Exploitation | Initial Access (TA0001), Execution (TA0002) |
| Installation | Persistence (TA0003) |
| Command and Control | Command and Control (TA0011) |
| Actions on Objectives | Credential Access (TA0006), Lateral Movement (TA0008), Collection (TA0009), Exfiltration (TA0010), Impact (TA0040) |
| No clean Kill Chain equivalent (happens after the foothold) | Privilege Escalation (TA0004), Defense Evasion (TA0005), Discovery (TA0007) |

> **Homework (not done):** verify every tactic ID against the current ATT&CK version, and add technique IDs to the five alert examples above, only for the ones I can explain live.

## How defenders break the chain

For each stage there are defensive actions: detect, deny, disrupt, degrade, deceive, and contain. Examples:

- **Delivery:** email filtering, blocking known-bad domains and senders (deny)
- **Exploitation:** patching, application allow-listing, EDR blocking execution (deny/disrupt)
- **Installation:** endpoint hardening, alerting on new persistence (detect)
- **Command and Control:** DNS sinkholing, egress filtering, blocking destinations (deny/disrupt)
- **Actions on Objectives:** network segmentation, DLP, host isolation (contain)

As a SOC analyst, my part is mainly **detect** and **contain**, plus feeding findings back so other teams can deny and disrupt earlier.

> **Homework (not done):** check the original Lockheed Martin paper for the exact list of courses of action and use the wording from the version I cite.

## Common false positives and pitfalls

- **Internet-wide scanning looks like Reconnaissance.** It is constant background noise. On its own it is not an incident, but a scan from an IP that later shows up in Delivery or C2 alerts matters.
- **Assuming the stages happen in order.** Finding C2 first does not mean I missed nothing earlier. It means I must go back and find it.
- **Stopping at the alerted host.** One confirmed stage on one machine says nothing about scope. I search for the same indicators elsewhere.
- **Treating "blocked" as "safe".** A blocked delivery means that attempt failed. I still check whether other copies of the message or payload got through.
- **Using the framework as a checklist.** A real incident will not fill in all seven boxes neatly.

## Escalation criteria

- **Delivery only**, with no sign of user interaction or execution: handle at L1 (block, purge, report) and document. If many users received the same campaign but nobody interacted, that is still L1 plus a report, not an escalation.
- **Exploitation or beyond** (code executed, persistence, or C2) is treated as a potential incident and escalated. At L1 I isolate the host if I am authorized to, rather than powering it off, which preserves volatile memory, then hand off.
- **Multiple hosts or users showing Exploitation or beyond:** escalate, since scope has widened.
- **Any evidence of Actions on Objectives** (credential theft, lateral movement, data leaving): escalate immediately.

## Tools / data sources

- **Delivery:** email gateway, proxy, web filter
- **Exploitation and Installation:** EDR, Windows event logs, Sysmon where deployed
- **C2:** DNS logs, proxy, firewall, NetFlow, EDR network telemetry
- **Actions on Objectives:** authentication logs, file server logs, DLP
- **Correlation and scoping:** SIEM, log management
- **Context on attackers and infrastructure:** threat intelligence

## Key takeaway

The Kill Chain is not something to recite. It is a way to position an alert in the attack, trace backward to the likely entry point, and forward to what the attacker will do next. I use ATT&CK alongside it for the technique-level detail it lacks.