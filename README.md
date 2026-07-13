# Red vs Blue: Building a SOC for DDoS Defense & Threat Detection

A team-built Security Operations Center exercise defending a live network against real DDoS, web application, and Wi-Fi attacks — using FortiGate as the enforcement point and Splunk as the detection/analysis layer. I led the FortiGate firewall configuration and network infrastructure, with additional work on the Splunk side.

## Overview

This was a live, adversarial Red vs Blue exercise run through AAST's College of Computing and Information Technology. A Red Team executed real attacks — DDoS floods, web exploitation, Wi-Fi deauthentication — against a segmented VMware network, while a Blue Team defended in real time using a FortiGate virtual firewall (DoS policy, IPS, WAF, Web Filter, GeoIP blocking) with all logs forwarded to Splunk for detection and analysis. The exercise produced a full lab report, defense effectiveness analysis, and a SOC incident-response playbook.

**Team:** Ali Elkhouly (Penetration Testing Lead), Mazen Elfaham, Ahmed Hady (Blue Team / Purple Team / Log Analysis) — supervised by Eng. Ahmed Nour Eldin Noaman, AAST CCIT, May 2026.

**My role:** FortiGate firewall configuration and network infrastructure setup — building and tuning the DoS policy, IPS/WAF profiles, Web Filter rules, and firewall policies that stood between the Red Team and the target application — plus supporting work on the Splunk log ingestion side.

## Architecture & Tech Stack

- **Firewall/enforcement point:** FortiGate (virtual appliance on VMware) — DoS Policy, IPS, WAF, Web Filter, GeoIP blocking
- **SIEM:** Splunk Enterprise, receiving FortiGate logs via UDP syslog (port 514)
- **Target:** Ubuntu-hosted Flask app with intentionally vulnerable endpoints (XSS, LFI, SSRF, unrestricted file upload, HTML injection), exposed both on the LAN and via ngrok
- **Attacker:** Kali Linux — hping3, LOIC, Slowloris, Ettercap, MITMf, aircrack-ng suite
- **Network:** Fully segmented VMware LAN (192.168.1.0/24)

```
[Kali Linux Red Team — 192.168.1.64]
        |  DDoS floods, web attacks, Wi-Fi deauth
        v
[FortiGate — DoS Policy / IPS / WAF / Web Filter / GeoIP]
        |
        +--> [Ubuntu Vulnerable App — 192.168.1.50:5000]
        |
        +--syslog UDP:514--> [Splunk SIEM — 192.168.1.117:8000]
```

## What I Did

- Configured the FortiGate DoS policy with rate-limiting thresholds for ICMP flood, TCP SYN flood, and UDP flood detection
- Built firewall policies including a dedicated `Block-RedTeam` rule for rapid containment once the attacker IP was identified, and a `GeoIP-Block` policy restricting inbound traffic by country
- Configured the IPS profile (`Block-Web-Attacks`) with signatures for SQL injection, XSS, and malicious PHP upload detection
- Configured the WAF profile (`WAF-Vulnerable-App`) enforcing rules against generic attack patterns, XSS, SQL injection, LFI, oversized requests, and illegal content types
- Built the Web Filter policy (`Block-SSRF-LFI`) explicitly blocking the `/lfi` and `/ssrf` endpoints
- Set up syslog forwarding from FortiGate to Splunk (UDP 514) and supported Splunk-side configuration for ingesting and indexing the incoming firewall/traffic logs

## Key Findings / Results

Defense effectiveness measured across every attack vector the Red Team executed:

| Attack Vector | Primary Defense | Effectiveness | Residual Risk |
|---|---|---|---|
| LOIC HTTP/TCP/UDP Flood | DoS Policy + IP Block | 92–100% | Minimal |
| hping3 ICMP Flood | DoS Policy | 88% | Low |
| hping3 TCP SYN Flood | DoS Policy | 85% | Low |
| Slowloris HTTP Exhaustion | IPS (partial) | 75% | Medium |
| XSS Injection | WAF + IPS | 95% | Very Low |
| HTML Injection | WAF | 90% | Low |
| LFI (`/etc/passwd`, access logs) | Web Filter | 98% | Negligible |
| SSRF (localhost, link-local) | Web Filter | 98% | Negligible |
| **SSRF → LFI chained attack** | Web Filter (both endpoints) | 98% | Negligible |
| Malicious `.php` file upload | IPS | 90% | Low |
| **Wi-Fi Deauthentication** | **None — confirmed gap** | **0%** | **Critical** |

**Chained attack, caught:** The Red Team attempted an SSRF → LFI chain — using SSRF to reach the internal service at `127.0.0.1:5000`, then LFI to read `/etc/passwd` and Apache's access log. The Web Filter blocked both legs of the chain independently, and Splunk log correlation confirmed the SSRF origin via the `Python-urllib` user-agent in the Apache access log — tying both attack stages together in the timeline.

**The honest finding:** the Wi-Fi deauthentication attack (`aireplay-ng --deauth`) forcibly disconnected 8 client devices, captured a WPA2 4-way handshake (offline crack risk), and knocked out both Splunk connectivity and the FortiGate management interface — creating a monitoring blind spot during the exact window an attacker would want one. Current defenses caught 0% of it, because WPA2 without 802.11w Management Frame Protection has no defense against deauth frames by design. This became the single most important recommendation in the report: enable MFP, move toward WPA3, deploy WIDS.

## Challenges & What I'd Improve

- **Threshold tuning was a real trade-off**: rate-limiting thresholds for ICMP/SYN flood detection let some packets through before triggering (85–88% effectiveness) — lowering the threshold catches more but risks false positives on legitimate traffic. This needed live tuning during the exercise rather than a "correct" value set once.
- **Slowloris exposed a gap in generic IPS signatures**: because it's low-volume by design, it partially evaded volumetric DoS thresholds (75% effectiveness) — the fix identified was deploying protocol-specific slow-HTTP IPS signatures rather than relying on rate-based detection alone.
- **The Wi-Fi deauth gap was the biggest lesson**: it's easy to over-invest in web/DDoS defenses (which we did well) while leaving an unauthenticated Layer 2 attack completely open. Given more time, I'd push for WPA2-Enterprise (802.1X) or WPA3 and MFP enforcement as a first-class part of the network design, not an afterthought.

## Skills Demonstrated

`FortiGate` `Firewall Policy Configuration` `DoS/DDoS Mitigation` `IPS/IDS` `WAF` `GeoIP Filtering` `Splunk SIEM` `SPL` `Syslog Forwarding` `Network Segmentation` `SSRF/LFI Analysis` `Incident Response Playbook Design` `Blue Team Defense`

## Reports

Full documentation in files
- `comprehensive-cybersecurity-report.md` — full lab documentation, defense report, and SOC playbook (network topology, tool configs, attack log summaries, escalation matrix)

## Exercise Context

Red vs Blue: Building a SOC for DDoS Defense and Threat Detection — Arab Academy for Science, Technology & Maritime Transport (AASTMT), College of Computing and Information Technology (CCIT), May 2026. Classification note: original report is marked "Internal – Lab Environment Only" 
