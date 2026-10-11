01 - SOC FUNDAMENTALS
=====================


CORE IDEA
---------
A Security Operations Center (SOC) is the function that continuously watches
an organization's environment, detects suspicious activity, investigates it,
and drives the response. It works through three things together: people
(analysts, responders, engineers), process (playbooks, escalation paths), and
technology (SIEM, EDR, SOAR and others).

The L1 analyst sits at the front of that pipeline. Every alert reaches them
first, so the quality of their triage decides how fast real threats are
caught and how much noise the rest of the team has to absorb.


SOC MODELS
----------
In-house
  Organization builds and runs its own team.
  Trade-off: full control and context, but needs sustained budget and hiring.

Virtual
  No fixed facility, team works remotely.
  Trade-off: flexible and cheaper, but depends on strong tooling and
  communication.

Co-managed
  Internal staff paired with an MSSP.
  Trade-off: extends coverage (e.g., 24/7), but hand-offs and clear ownership
  are critical.

Command
  Central SOC overseeing several smaller SOCs across a region.
  Trade-off: consistent standards at scale. Found in large telecoms and
  government/defense.


ROLES
-----
SOC Analyst (L1 / L2 / L3)
  Triages alerts, investigates, recommends remediation.
  L1 classifies and escalates. L2 investigates deeper. L3 handles advanced
  cases and often hunting or detection input.

Incident Responder
  Takes over AFTER an incident is confirmed: scope, contain, eradicate,
  recover, root cause.
  Not a detection role. Detection is the analyst's and detection engineer's
  job.

Threat Hunter
  Proactively searches for threats that existing detections missed.
  Hypothesis-driven, not alert-driven ("if an attacker did X, what would I
  see?").

Security Engineer
  Builds and maintains the tooling: SIEM, log pipelines, SIEM-to-SOAR
  integrations, detection rules. Owns the platform, not the alert queue.

SOC Manager
  Budget, staffing, strategy, metrics, coordination. Operational leadership
  rather than hands-on technical work.


MENTAL MODEL: THE TOOL STACK AS ONE INVESTIGATION
-------------------------------------------------
Each tool answers a different question. I think of an alert as a thread I
pull through all of them.

1. SIEM tells me SOMETHING HAPPENED.
   It collects events and correlates them against rules (e.g., a threshold of
   failed logins in a short window). Its real value is correlation and
   alerting.

2. LOG MANAGEMENT tells me WHO ELSE IS INVOLVED.
   It centralizes web, OS, firewall, proxy and EDR logs so I can search one
   place instead of querying each device. Its value is collection and search.
   Modern SIEMs often do both, but the two jobs are different.

3. EDR tells me WHAT HAPPENED ON THE HOST.
   Processes, network connections, browser history. It also lets me contain
   the machine and hunt an indicator across all endpoints.

4. THREAT INTEL tells me WHETHER THIS ARTIFACT HAS BEEN SEEN BEFORE.
   Hash, IP, domain. It is supporting evidence, never a verdict.

5. SOAR DOES THE REPETITIVE WORK FOR ME.
   Automatic reputation lookups, enrichment, sandbox submission, and
   playbooks that make sure every analyst follows the same steps.


ANALYST WORKFLOW FOR A SINGLE ALERT
-----------------------------------
1. Read the alert: what rule fired, on which host/user, and what the rule is
   actually trying to detect.
2. Enrich: reputation checks, asset context, user context.
3. Investigate: EDR for host activity, log management for network and related
   events.
4. Decide: true positive or false positive, with the evidence written down.
5. Scope: search logs and EDR for the same indicator elsewhere. The alert may
   only name one host.
6. Act or escalate: contain if authorized, escalate per playbook, document
   everything.
7. Feed back: tell the detection team about noisy or broken rules.


CONTAINMENT AND WHAT COMES AFTER
--------------------------------
Containment is one step, not the finish line. The sequence is:
contain -> scope -> eradicate -> recover -> validate before the host returns
to the network.

- Why isolate instead of powering off: shutting down destroys volatile
  evidence in memory (running processes, network connections, injected code).
  EDR network containment cuts the host off from everything except the EDR
  console, so I can keep investigating.

- Why contain at all: to stop the attacker from reaching the internal network
  or moving laterally, and to stop data leaving.

- Why scoping matters: one contained host does not mean the incident is
  contained. If a host talked to a malicious IP, I search for every other
  host that talked to it.


THREAT INTELLIGENCE: HOW TO READ THE RESULTS
--------------------------------------------
- No result does not mean clean. A hash with no hits only means THAT EXACT
  FILE has not been reported. A repacked or recompiled variant has a different
  hash. Static and dynamic analysis still apply.

- IOCs go stale. IP addresses get reassigned (for example, cloud
  infrastructure used by an attacker may later host something unrelated). A
  past malicious listing is not proof of present malice. Always check the
  date of the listing.

- Feeds differ in quality. Confidence, source, and age vary, so corroborate
  across sources.

- Cached results can mislead. A reputation page may show an old scan. Re-scan
  so I am judging the current state, not a snapshot from before the content
  changed.


COMMON FALSE POSITIVES (MY OWN EXAMPLE)
---------------------------------------
An authorized vulnerability scanner running across the network can trigger
port-scan or brute-force alerts. The traffic looks malicious, but the source
is a known, scheduled internal tool.

Triage: confirm the source IP against the asset inventory and scan schedule,
and verify the activity matches the change window.

Feedback to the SIEM team: whitelist or tag the scanner's IPs and schedule so
the rule keeps firing on real scans but stays quiet on approved ones.

A good analyst does not just close the ticket. Reporting the noisy rule is
part of the job and improves the whole SOC.


COMMON ANALYST MISTAKES AND MY GUARDRAILS
-----------------------------------------
Over-relying on VirusTotal
  Problem: AV-evasive malware can show 0 detections.
  Guardrail: Treat VT as one supporting source, never the verdict. Add
  static/dynamic analysis, behavior, and context.

Rushing sandbox analysis
  Problem: Malware may detect the sandbox or delay execution by many minutes.
  Guardrail: Run long enough, watch for delayed behavior, cross-check with
  host evidence.

Inadequate log analysis
  Problem: Missing other affected hosts leaves the attacker in place.
  Guardrail: Always search for the indicator across all logs and endpoints,
  not just the alerted host.

Ignoring result dates
  Problem: Stale or cached data gives a false sense of safety.
  Guardrail: Check timestamps and re-scan.


TOOLS AND DATA SOURCES
----------------------
SIEM:          Splunk, IBM QRadar, ArcSight, FortiSIEM
EDR:           CrowdStrike, SentinelOne, Carbon Black, Trellix (formerly
               FireEye HX)
SOAR:          Splunk SOAR (formerly Phantom), Cortex XSOAR (formerly
               Demisto), IBM QRadar SOAR (formerly Resilient)
Threat intel:  VirusTotal, Cisco Talos Intelligence

Note: vendor names change through acquisitions and rebrands. Verify before
relying on them.


KEY TAKEAWAY
------------
The tools only give me evidence. My job is to turn that evidence into a
defensible decision, and to check how far the problem goes before I call it
contained.