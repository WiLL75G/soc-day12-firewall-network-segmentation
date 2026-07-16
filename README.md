# Firewall Hardening and Service Exposure Remediation

Building a default deny UFW policy, validating it with Nmap, finding an exposed MySQL service the firewall rule did not actually close, and fixing it properly.

## At a Glance

| Field | Detail |
| --- | --- |
| Task Type | Host firewall hardening and service exposure remediation |
| Tools Used | UFW, Nmap 7.99, systemctl |
| Host | Kali Linux VM, localhost |
| Default Inbound Policy | Deny |
| Allowed | SSH 22, HTTP 80, HTTPS 443 |
| Denied | Telnet 23, FTP 21, RDP 3389, MySQL 3306 |
| Outcome | 7 rules active, MySQL exposure found during validation and closed with layered remediation |

## What Happened

A default deny firewall policy was built on a Kali host, with explicit allows for the services that are needed and explicit denies for the ones that are not.

Then it was tested, and the test is where the lab earned its value. Nmap found MySQL listening on 3306, which the policy had not accounted for. A UFW deny rule was applied. The rescan showed it still reachable.

That failure is the whole project. Writing the rule felt like fixing it. The rescan proved it was not fixed. Everything worth learning here happened in the gap between those two moments.

## UFW Enablement

![UFW Enabled](./screenshots/01_ufw_enabled.png)

Initial state verified as inactive, then activated with ufw enable and confirmed active and persistent across startup.

Checking the state first matters. An inactive firewall with a beautiful ruleset is a wall with no bricks in it.

## Default Deny Policy

![Default Policy](./screenshots/02_ufw_default_policy.png)

Verified via ufw status verbose: deny incoming, allow outgoing, deny routed. Logging confirmed active.

Default deny is the only posture that makes allow rules mean anything. Under default allow, an allow rule is a comment. Under default deny, it is a decision.

Logging is on because a firewall that blocks silently tells you nothing about who is knocking.

## Initial Rule Set

![Initial Rules](./screenshots/03_ufw_rules_numbered.png)

Allow rules for SSH 22, HTTP 80, HTTPS 443. Explicit deny rules for Telnet 23, FTP 21, RDP 3389. Rule numbering verified with ufw status numbered.

The explicit denies are technically redundant under default deny. They are there anyway, because a rule list is also a document. An auditor reading it should see that Telnet was considered and rejected, not left out by accident.

Telnet and FTP are denied because they carry credentials in cleartext. RDP is denied because it is the single most common ransomware ingress path there is.

IPv6 rules are auto generated alongside IPv4. Both get reviewed. A policy that only covers v4 is half a policy.

## Validation Scan and Discovery

![Nmap Scan Results](./screenshots/04_nmap_scan_results.png)

```bash
sudo nmap -sT localhost
```

Allowed services responded as expected.

MySQL was found listening on 3306/tcp. It was not in the policy at all.

This is why validation exists. The ruleset described a host that did not exist. Nmap described the one that did.

## First Remediation Attempt

![UFW Deny MySQL](./screenshots/05_ufw_deny_mysql.png)

```bash
sudo ufw deny 3306/tcp
```

Rule applied for both IPv4 and IPv6, confirmation captured.

At this point the ticket looks closeable. The finding has a rule against it. This is exactly where a lot of remediation stops.

## Rule Verification

![Final Rules](./screenshots/06_ufw_final_rules.png)

ufw status numbered re run, confirming 3306/tcp DENY IN present in the policy. Seven unique rules, fourteen entries counting IPv6.

The rule is there. That confirms the rule exists. It does not confirm the port is closed, and those are different claims.

## Re Validation and Root Cause

![Nmap Post Remediation](./screenshots/07_nmap_post_remediation.png)

The rescan showed 3306/tcp still reachable.

Root cause: UFW does not filter traffic on the loopback interface by default. The rule was correct and the rule was irrelevant, because the traffic never passed through the chain the rule sits in.

Second layer applied:

```bash
sudo systemctl stop mysql
```

Final scan returned 80/tcp open http and nothing else. Closed.

The lesson is not about UFW. It is that a control only works where it sits in the path. A service that binds to loopback is not protected by a firewall that does not filter loopback, and no amount of correct rule syntax changes that. The only thing that caught this was scanning again after saying it was fixed.

## Firewall Policy

| Port | Service | Action | Reason |
| --- | --- | --- | --- |
| 22/tcp | SSH | Allow | Remote administration, encrypted |
| 80/tcp | HTTP | Allow | Web server traffic |
| 443/tcp | HTTPS | Allow | Encrypted web traffic |
| 21/tcp | FTP | Deny | Cleartext credentials |
| 23/tcp | Telnet | Deny | Cleartext credentials |
| 3389/tcp | RDP | Deny | Common ransomware ingress path |
| 3306/tcp | MySQL | Deny | Database, no business on an ingress path |

## Exposure Finding

| Field | Detail |
| --- | --- |
| Port | 3306/tcp |
| Service | MySQL |
| Risk | Directly reachable database service |
| Discovery | Nmap validation scan against localhost |
| First action | UFW deny rule applied |
| Result | Still reachable, UFW does not filter loopback by default |
| Second action | Service stopped via systemctl |
| Re validation | Post remediation Nmap scan confirmed closed |
| Status | Remediated, layered |

## Exposure Indicators

| Type | Indicator | Source |
| --- | --- | --- |
| Exposed service | MySQL on 3306/tcp | Nmap scan |
| Allowed surface | SSH 22, HTTP 80, HTTPS 443 | Policy definition |
| Denied surface | Telnet 23, FTP 21, RDP 3389 | Policy audit |
| Control limitation | UFW does not filter the lo interface by default | Validation test |

## MITRE ATT&CK Relevance

| Technique | ID | Why It Applies |
| --- | --- | --- |
| Network service discovery | T1046 | Nmap mirrors how an attacker finds 3306 |
| External remote services | T1133 | The allowed surface is what this policy defines |
| Remote services, RDP | T1021.001 | Denied at the perimeter |
| Remote services, SSH | T1021.004 | Allowed, therefore the ingress path that needs monitoring |

Mapping note: these are the techniques the policy is written against. Nothing was observed or attempted. This is a hardening build, not an intrusion.

## Analyst Findings

Default deny ingress implemented and verified active.

Three services explicitly allowed, four explicitly denied, seven unique rules.

MySQL found exposed on 3306 during validation, absent from the original policy.

The firewall deny rule alone did not close it, because the service was reachable on loopback.

Layered remediation, firewall rule plus service shutdown, closed the exposure.

Closure confirmed by rescan, not by assumption.

## Recommended Response

Keep default deny ingress as the baseline and treat every allow as a decision that needs a reason.

Validate with an external scan after every policy change, because the ruleset and the host disagree more often than anyone expects.

Treat every database service, MySQL, PostgreSQL, MSSQL, Redis, as deny by default.

Audit listening services with ss -tulpn regularly, since the firewall can only filter what it sees.

Stop or rebind services that have no reason to be on the network at all. Not listening beats being blocked.

Forward UFW logs to Splunk so ingress denials become alertable rather than just written to disk.

Schedule recurring Nmap audits to catch drift.

## What This Lab Demonstrates

Building a default deny host firewall policy and understanding why the order matters.

Writing an auditable ruleset where the intent is visible, including the explicit denies.

Validating policy against reality with an external scan rather than trusting the config.

Finding an exposure that the policy design missed entirely.

Diagnosing why a correct rule failed, rather than assuming the rule worked.

Applying layered remediation and proving closure with a confirming rescan.

## Repository Structure

```
firewall-rules-network-segmentation-lab/
├── README.md
└── screenshots/
    ├── 01_ufw_enabled.png
    ├── 02_ufw_default_policy.png
    ├── 03_ufw_rules_numbered.png
    ├── 04_nmap_scan_results.png
    ├── 05_ufw_deny_mysql.png
    ├── 06_ufw_final_rules.png
    └── 07_nmap_post_remediation.png
```

---

[![LinkedIn](https://img.shields.io/badge/LinkedIn-WilliamInCyber-blue?style=flat&logo=linkedin)](https://linkedin.com/in/WilliamInCyber)
[![X](https://img.shields.io/badge/X-WilliamInCyber-black?style=flat&logo=x)](https://x.com/WilliamInCyber)
