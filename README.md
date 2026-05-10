# Day 12 – SOC Tier 1 Incident Report: Firewall Rules & Network Segmentation

---

## Incident Summary

- **Incident Type:** Network Perimeter Hardening & Ingress Filtering
- **Severity:** Medium (Exposed MySQL Service Discovered & Remediated)
- **Detection Method:** Default-Deny Firewall Implementation + Nmap Validation Scan
- **Tools Used:** UFW (Uncomplicated Firewall), Nmap 7.99
- **Status:** Complete 7 Rules Active, MySQL Exposure Remediated

---

## Executive Summary

A firewall policy was designed and implemented on a Kali Linux VM using UFW. A default deny posture was applied to all incoming traffic, with explicit allow rules for essential services (SSH, HTTP, HTTPS) and explicit deny rules for high risk legacy or remote access ports (Telnet, FTP, RDP, MySQL).

During Nmap validation testing, an exposed MySQL service on port `3306/tcp` was discovered and immediately remediated through a UFW deny rule. The final state delivers a hardened ingress posture aligned with network segmentation best practices.

---

## Affected System

- **Target Host:** Kali Linux VM (localhost)
- **Firewall:** UFW (Uncomplicated Firewall)
- **Validation Tool:** Nmap 7.99
- **Default Inbound Policy:** Deny
- **Allowed Services:** SSH (22/tcp), HTTP (80/tcp), HTTPS (443/tcp)
- **Denied Services:** Telnet (23/tcp), FTP (21/tcp), RDP (3389/tcp), MySQL (3306/tcp)

---

## Investigation Methodology

---

### 1. UFW Enablement

![UFW Enabled](./screenshots/01_ufw_enabled.png)

- Activated UFW on the Kali host
- Verified service status and active state
- Confirmed UFW was managing the underlying iptables ruleset

### SOC Observations:

- Firewall activation is the foundational step in host-based defense
- Verifying active state prevents false sense of security from inactive rules
- UFW provides a clean abstraction over iptables for policy management

---

### 2. Default-Deny Policy Configuration

![Default Policy](./screenshots/02_ufw_default_policy.png)

- Configured default inbound policy as `deny`
- Configured default outbound policy as `allow`
- Confirmed deny-by-default posture before authoring allow rules

### SOC Observations:

- Default-deny is the gold standard for ingress filtering
- Allow rules are only meaningful when applied over a deny baseline
- Outbound defaults can be tightened later via egress filtering

---

### 3. Initial Rule Implementation

![Initial Rules](./screenshots/03_ufw_rules_numbered.png)

- Created allow rules for SSH (22/tcp), HTTP (80/tcp), HTTPS (443/tcp)
- Created explicit deny rules for Telnet (23), FTP (21), RDP (3389)
- Verified rule numbering and order with `ufw status numbered`

### SOC Observations:

- Numbered rule lists support audit and selective rule modification
- Explicit deny rules make policy intent visible during audits
- Cleartext protocols (Telnet, FTP) must be denied even in lab environments

---

### 4. Nmap Validation Scan & Vulnerability Discovery

![Nmap Scan Results](./screenshots/04_nmap_scan_results.png)

- Executed Nmap scan against the host to validate firewall posture
- Confirmed allowed services responded as expected
- **Discovered exposed MySQL service on port `3306/tcp`** not previously denied

### SOC Observations:

- External validation is non-negotiable rule lists and reality must agree
- Database services must never be reachable from untrusted networks
- MySQL on `3306/tcp` is a recurring exposure finding in production audits

---

### 5. Remediation & Final Firewall State

![Final Rules](./screenshots/05_ufw_final_rules.png)

- Added explicit deny rule for MySQL (`3306/tcp`)
- Re-validated with Nmap to confirm remediation
- Documented final 7 rule policy for handoff and audit retention

### SOC Observations:

- Discovery → remediation cycles must be documented end-to-end
- Re-validation closes the loop on remediation effectiveness
- Final rule state should be exported for version control and audit

---

## Firewall Rules Policy

| Port      | Service | Action     | Reason                                |
|-----------|---------|------------|---------------------------------------|
| 22/tcp    | SSH     | ✅ ALLOW   | Secure remote administration          |
| 80/tcp    | HTTP    | ✅ ALLOW   | Web server traffic                    |
| 443/tcp   | HTTPS   | ✅ ALLOW   | Encrypted web traffic                 |
| 23/tcp    | Telnet  | ❌ DENY    | Cleartext credentials — attack vector |
| 21/tcp    | FTP     | ❌ DENY    | Cleartext credentials — attack vector |
| 3389/tcp  | RDP     | ❌ DENY    | Ransomware ingress — blocked          |
| 3306/tcp  | MySQL   | ❌ DENY    | Database exposed — remediated         |

---

## Vulnerability Discovered During Testing

| Finding                | Detail                                                 |
|------------------------|--------------------------------------------------------|
| **Port**               | `3306/tcp`                                             |
| **Service**            | MySQL Database                                         |
| **Risk**               | Database directly accessible data exfiltration risk  |
| **Discovery Method**   | Nmap validation scan                                   |
| **Action Taken**       | UFW deny rule applied immediately                      |
| **Re-Validation**      | Confirmed via post-remediation Nmap scan               |
| **Status**             | Remediated ✅                                          |

---

## Indicators of Compromise / Exposure (IOCs)

| Type                | Indicator                                  | Source           |
|---------------------|--------------------------------------------|------------------|
| Exposed Service     | MySQL on `3306/tcp`                        | Nmap Scan        |
| Cleartext Protocol  | Telnet (`23/tcp`) — denied                 | Policy Audit     |
| Cleartext Protocol  | FTP (`21/tcp`) — denied                    | Policy Audit     |
| Remote Access Risk  | RDP (`3389/tcp`) — denied                  | Policy Audit     |
| Allowed Surface     | SSH (`22/tcp`), HTTP (`80/tcp`), HTTPS (`443/tcp`) | Policy Definition |

---

## MITRE ATT&CK Mapping

| Behavior                              | Technique ID | Description                                                |
|---------------------------------------|--------------|------------------------------------------------------------|
| Network Service Discovery             | T1046        | Nmap simulates adversary port enumeration                  |
| External Remote Services              | T1133        | Allowed services hardened via firewall                     |
| Remote Services: SSH                  | T1021.004    | SSH allowed but restricted to authorized ingress           |
| Remote Services: RDP                  | T1021.001    | RDP denied to prevent ransomware ingress                   |
| Exploit Public-Facing Application     | T1190        | Reduced attack surface mitigates this technique            |
| Valid Accounts                        | T1078        | Cleartext-credential protocols (FTP, Telnet) denied        |
| Impair Defenses: Disable/Modify Tools | T1562.004    | UFW protects against unauthorized firewall modification    |

---

## SOC Analyst Findings

- Default deny ingress policy successfully implemented on the host
- Three essential services (SSH, HTTP, HTTPS) are explicitly allowed
- Four high-risk services (Telnet, FTP, RDP, MySQL) are explicitly denied
- MySQL service was found exposed during Nmap validation and remediated within the same session
- Final firewall policy contains 7 rules, all active and validated
- No additional unexpected services detected on post remediation scan

---

## SOC Analyst Response

- Maintain default deny ingress posture as a baseline standard
- Validate firewall rules with external scans after every policy change
- Treat all database services (MySQL, PostgreSQL, MSSQL, Redis) as deny-by-default
- Require justification and access controls for any new allow rule
- Schedule recurring Nmap audits to detect configuration drift
- Forward UFW logs to a SIEM (Splunk) for ingress denial alerting
- Implement equivalent egress filtering as a follow up hardening step

---

## Analyst Insight

Firewall hardening is one of the highest leverage controls available to a SOC. A default deny ingress policy converts the host's attack surface from "everything not explicitly blocked" to "nothing not explicitly allowed" eliminating an entire class of exposure findings before they ever appear in a vulnerability scan. The MySQL exposure caught during validation is exactly the kind of finding that quietly persists in production environments for years; rule lists alone are insufficient, and external validation must be a permanent step in the firewall lifecycle.

---

## Learning Outcome

This investigation demonstrates the ability to:

- Configure UFW with a default deny ingress posture
- Author and number firewall rules for SOC audit and handoff
- Validate firewall policy with Nmap port scanning
- Detect and remediate exposed services in real time
- Distinguish between essential and high risk network services
- Apply network segmentation principles to host-level firewalls
- Map firewall controls to MITRE ATT&CK adversary techniques
- Document the discovery → remediation → re-validation cycle

---

## Repository Structure

```
firewall-rules-network-segmentation-lab/
├── README.md
└── screenshots/
    ├── 01_ufw_enabled.png
    ├── 02_ufw_default_policy.png
    ├── 03_ufw_rules_numbered.png
    ├── 04_nmap_scan_results.png
    └── 05_ufw_final_rules.png
```

---

## Conclusion

This lab demonstrates a real world firewall configuration and network segmentation workflow. Using UFW, a strict default deny ingress policy was implemented with only essential services allowed. Nmap validation revealed an exposed MySQL port that was remediated within the same session and re-validated for closure. The output mirrors the exact process a SOC or network security engineer follows when hardening a Linux host policy authoring, validation, discovery, and remediation in a closed loop.
