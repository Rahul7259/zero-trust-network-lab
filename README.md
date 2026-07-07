# Zero Trust Network Access (ZTNA) Lab

![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)
![AWS](https://img.shields.io/badge/Cloud-AWS-orange)
![OpenZiti](https://img.shields.io/badge/ZTNA-OpenZiti-blue)
![NIST](https://img.shields.io/badge/Framework-NIST%20SP%20800--207-green)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## Table of Contents

1. [Overview](#overview)
2. [Motivation](#motivation)
3. [Architecture](#architecture)
4. [How It Works](#how-it-works)
5. [Tech Stack](#tech-stack)
6. [Protocols](#protocols)
7. [NIST SP 800-207 Mapping](#nist-sp-800-207-mapping)
8. [IGA Layer](#iga-layer)
9. [Enterprise Mapping](#enterprise-mapping)
10. [AWS Infrastructure](#aws-infrastructure)
11. [Attack Simulations](#attack-simulations)
12. [Threat Model](#threat-model)
13. [Key Concepts](#key-concepts)
14. [Project Progress](#project-progress)
15. [Repository Structure](#repository-structure)
16. [Background](#background)
17. [References](#references)

---

## Overview

A hands-on, production-grade implementation of Zero Trust Network Access (ZTNA)
built entirely on AWS — demonstrating identity-based microsegmentation, dark network
principles, insider threat simulation, and full SIEM-backed attack documentation.

Built on open-source tooling that mirrors enterprise products like Zscaler,
Cloudflare Access, Okta, and Microsoft Sentinel — at zero cost.

**What this project does:**
- Builds a complete ZTNA fabric using OpenZiti on AWS VPC
- Enforces identity-based access using Authentik as an OAuth2/OIDC identity provider
- Protects services with zero open ports — completely invisible to unauthorized scanners
- Attacks the setup using Kali Linux and documents every blocked attempt
- Captures all evidence in Wazuh SIEM with real-time alerting

**What this project proves:**
- Zero trust is not a product — it is an architecture you enforce
- Identity is the new perimeter — network location means nothing
- Every attack can be blocked AND documented with evidence
- Open-source tooling maps directly to enterprise-grade implementations

> *"Security is not a feature you bolt on after the network is built.*
> *It is the foundation you design around from day one."*

---

## Motivation

Networks have evolved — security has not kept up.

Coming from an IAM and network engineering background, I have seen firsthand
how organizations enforce strong identity governance at the application layer
while the underlying network still operates on perimeter trust.

Once inside the network, users and devices are largely trusted by default.
East-west traffic between internal workloads goes uninspected.
VPNs hand out broad network access instead of least-privilege access
to specific resources.

The 2020 SolarWinds breach was not a failure of perimeter security.
It was proof that perimeter security is the wrong model entirely.
Attackers moved laterally across trusted internal networks for months —
undetected — because nothing inside was verifying anything.

**Zero Trust flips this entirely:**
- Every connection is untrusted by default
- Every identity is verified continuously — not just at login
- Every resource is protected individually — not hidden behind a shared wall
- Every access attempt is logged — whether it succeeds or fails

This project is my hands-on answer to that problem.
Not just understanding zero trust conceptually —
but building it from scratch, attacking it with real adversary techniques,
and proving it holds with SIEM evidence.

---

## Architecture

```
                        CONTROL PLANE
                   "Who are you? Should you get in?"

          ┌─────────────────────────────────────────┐
          │              Authentik (IdP)             │
          │         OAuth2 / OIDC Identity Gate      │
          │    Username + Password + MFA verified    │
          └──────────────────┬──────────────────────┘
                             │ confirmed human identity
                             ▼
          ┌─────────────────────────────────────────┐
          │         OpenZiti Controller              │
          │              (Policy Engine)             │
          │   Issues X.509 certificates to verified  │
          │   identities. Manages all access policies│
          │   Revokes access instantly when needed   │
          └──────────────────┬──────────────────────┘
                             │ certificate + policy decision
                             ▼
          ┌─────────────────────────────────────────┐
          │          OpenZiti Router                 │
          │       (Policy Enforcement Point)         │
          │   Verifies certificate on every request  │
          │   Opens encrypted mTLS tunnel if valid   │
          │   Silently drops everything else         │
          └──────────────────┬──────────────────────┘
                             │
                        DATA PLANE
                   "Actual traffic flows here"
                   "Only after control plane approves"
                             │
          ┌──────────────────┴──────────────────────┐
          │                                          │
          ▼                                          ▼

┌──────────────────────┐              ┌──────────────────────┐
│   Protected Service  │              │    Kali Attacker     │
│      (nginx VM)      │              │       (VM5)          │
│                      │              │                      │
│  ZERO OPEN PORTS     │              │  Tries everything:   │
│  Invisible to scans  │              │  - Port scanning     │
│  Only reachable via  │              │  - Lateral movement  │
│  OpenZiti tunnel     │              │  - Credential attack │
│  with valid cert     │              │  - mTLS intercept    │
└──────────────────────┘              └──────────────────────┘

          ┌─────────────────────────────────────────┐
          │               Wazuh SIEM                │
          │   Agent on every VM — logs everything   │
          │   Real-time alerts on attack attempts   │
          │   Full evidence trail for every session │
          └─────────────────────────────────────────┘
```

### AWS VPC Layout

```
┌─────────────────────────────────────────────────────────┐
│                     AWS VPC                             │
│                  10.0.0.0/16                            │
│                                                         │
│  PUBLIC SUBNET — 10.0.1.0/24                           │
│  (Control plane — internet accessible)                  │
│  ┌─────────────────┐      ┌─────────────────┐         │
│  │   Authentik     │      │    OpenZiti     │         │
│  │   VM1           │      │   Controller    │         │
│  │   t2.micro      │      │   VM2           │         │
│  │                 │      │   t2.micro      │         │
│  │   [Elastic IP]  │      │   [Elastic IP]  │         │
│  └─────────────────┘      └─────────────────┘         │
│                                                         │
│  ┌─────────────────┐                                   │
│  │    OpenZiti     │                                   │
│  │    Router       │                                   │
│  │    VM3          │                                   │
│  │    t2.micro     │                                   │
│  └─────────────────┘                                   │
│                                                         │
│  PRIVATE SUBNET — 10.0.2.0/24                          │
│  (Data plane — no internet access)                      │
│  ┌─────────────────┐      ┌─────────────────┐         │
│  │    Protected    │      │  Kali Linux     │         │
│  │    Service      │      │  Attacker       │         │
│  │    VM4          │      │  VM5            │         │
│  │    t2.micro     │      │  t2.micro       │         │
│  │  NO PUBLIC IP   │      │  NO PUBLIC IP   │         │
│  └─────────────────┘      └─────────────────┘         │
│                                                         │
│  ┌─────────────────┐                                   │
│  │     Wazuh       │                                   │
│  │     SIEM        │                                   │
│  │     VM6         │                                   │
│  │     t2.micro    │                                   │
│  └─────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
```

---

## How It Works

### Legitimate User — Step by Step

```
Step 1   User opens OpenZiti client on their device

Step 2   Client checks: do I have a valid certificate?
         No → redirect to Authentik

Step 3   Authentik: enter username + password + MFA
         Verified → issues OAuth2 token to OpenZiti Controller

Step 4   OpenZiti Controller receives OAuth2 token
         Confirms identity is real and registered
         Issues X.509 certificate to user device
         Certificate saved locally — enrollment complete

Step 5   On every subsequent connection:
         Client presents certificate to OpenZiti Router
         Router checks Controller — is cert valid?
         Does this identity have a policy for this service?

Step 6   Policy says yes → Router opens encrypted mTLS tunnel
         User reaches protected service
         Never sees the IP address directly
         Never touches the network without identity verification

Step 7   Wazuh logs the entire session with full timestamp
```

### Attacker — What They See

```
Step 1   Kali runs nmap on private subnet
         Result → zero open ports found
         Protected service is completely invisible

Step 2   Kali tries direct TCP connection to service IP
         Result → connection refused
         No route exists without going through Router

Step 3   Kali tries to request a certificate from Controller
         Result → blocked, no registered identity exists

Step 4   Kali tries to intercept mTLS tunnel traffic
         Result → mutual TLS authentication fails
         Kali does not have the matching private key

Step 5   Kali brute forces Authentik login
         Result → rate limiting triggered, account locked
         Wazuh fires alert with source IP and timestamp

Step 6   Every single attempt captured in Wazuh
         Full timestamped evidence trail generated
```

### Certificate Compromise Scenario

The most sophisticated attack against ZTNA is certificate theft.
Here is why it is extremely difficult and how the architecture defends:

```
Why theft is hard:
  mTLS binds the certificate to the private key on the original device
  The private key never leaves the device in a correctly configured setup
  Stealing the certificate file without the private key is completely useless

Why the window is narrow even if theft succeeds:
  Controller re-verifies every identity continuously
  Anomalous behavior (unusual location, time, volume) triggers Wazuh alert
  Admin can revoke the certificate in seconds — access dropped everywhere instantly

VPN vs ZTNA on credential theft:
  VPN  → stolen credentials = full network access, session continues until expiry
  ZTNA → stolen certificate = only permitted resources, revoked in seconds,
          behavioral detection running continuously
```

---

## Tech Stack

| Layer | Tool | Version | Purpose |
|---|---|---|---|
| Identity Provider | Authentik | Latest | OAuth2/OIDC human identity + MFA |
| ZTNA Core | OpenZiti | Latest | Controller, Router, certificate mgmt |
| Cloud | AWS VPC + EC2 | — | Network isolation and compute |
| Protected Resource | nginx on Ubuntu 22.04 | — | Target service, zero open ports |
| Attack Simulation | Kali Linux | Latest | Adversary techniques simulation |
| SIEM | Wazuh | 4.x | Log collection, detection, alerting |
| Diagramming | draw.io | — | Architecture documentation |
| Version Control | Git + GitHub | — | Project documentation and progress |

---

## Protocols

| Protocol | Where Used | Purpose |
|---|---|---|
| **mTLS** | OpenZiti tunnels | Mutual authentication — both sides verified |
| **X.509** | All OpenZiti identities | Cryptographic identity certificates |
| **OAuth2** | Authentik → OpenZiti | Human identity token exchange |
| **OIDC** | Authentik → OpenZiti | Identity layer on top of OAuth2 |
| **TLS 1.3** | All encrypted traffic | Transport layer encryption |
| **DTLS** | UDP tunnel connections | Encrypted UDP transport |
| **SSH** | Admin VM access | Secure management channel |
| **HTTPS** | All web UIs | Authentik, Wazuh, OpenZiti console |
| **REST API** | OpenZiti management | Controller configuration |
| **LDAP** | Authentik ↔ AD | Enterprise directory integration (optional) |

---

## NIST SP 800-207 Mapping

This project is built directly on the NIST Zero Trust Architecture standard.

| NIST Component | Definition | This Lab |
|---|---|---|
| Policy Engine (PE) | Makes the access decision | OpenZiti Controller brain |
| Policy Administrator (PA) | Executes the decision | OpenZiti Controller — opens/closes paths |
| Policy Enforcement Point (PEP) | The actual gate | OpenZiti Router |
| Subject | The entity requesting access | User with Authentik-verified identity |
| Enterprise Resource | What is being protected | nginx service on private VM |
| Control Plane | Where decisions are made | Controller ↔ Router communication |
| Data Plane | Where traffic flows | Encrypted mTLS tunnels |
| Continuous Monitoring | Always watching | Wazuh SIEM agents on all VMs |
| Dark Network | Zero exposed ports | Protected service has no listening ports |
| Microsegmentation | Isolated network zones | AWS private subnet |

### The 7 NIST Zero Trust Tenets — Applied

| Tenet | How This Lab Applies It |
|---|---|
| 1. Everything is a resource | Every VM, service, and identity treated as a protected resource |
| 2. All communication secured | All traffic encrypted via mTLS regardless of network location |
| 3. Access per session | OpenZiti grants access per connection — not persistent |
| 4. Dynamic policy | Controller evaluates identity, device, and policy on every request |
| 5. Monitor all assets | Wazuh agents on every VM — continuous posture assessment |
| 6. Auth is dynamic | Controller re-verifies every 15 minutes — not just at login |
| 7. Collect everything | Wazuh collects all logs — feeds back to improve policies |

---

## IGA Layer

Identity Governance and Administration (IGA) sits above the entire stack
as the governance brain that decides what access should exist before
any authentication or enforcement happens.

```
┌─────────────────────────────────────────────────────┐
│                      IGA Layer                      │
│         SailPoint / Saviynt / One Identity          │
│                                                     │
│  Who SHOULD have access? → Governs Authentik        │
│  Joiners  → Auto-provision access in Authentik      │
│  Movers   → Modify roles and permissions            │
│  Leavers  → Auto-deprovision → OpenZiti cert revoked│
│  Reviews  → Periodic access certification           │
│  Audit    → Full entitlement trail for compliance   │
└──────────────────────┬──────────────────────────────┘
                       │ governs and provisions
                       ▼
              Authentik (IdP Layer)
              OpenZiti (Enforcement Layer)
              Wazuh (Monitoring Layer)
```

In this lab, IGA lifecycle is managed manually.
In an enterprise deployment, SailPoint or Saviynt would automate
all provisioning and deprovisioning through Authentik.

---

## Enterprise Mapping

Every tool in this lab has a direct enterprise equivalent.
Swapping tools does not change the zero trust architecture.

| This Lab | Enterprise Equivalent | Notes |
|---|---|---|
| Authentik | Okta / Azure AD / Google Workspace | OAuth2/OIDC compatible — drop-in swap |
| OpenZiti Controller | Zscaler Controller / Cloudflare Access | Same PE + PA architecture |
| OpenZiti Router | Zscaler Connector / Cloudflare Tunnel | Same PEP pattern |
| Wazuh | Splunk / Microsoft Sentinel / IBM QRadar | Agent-based SIEM |
| AWS VPC | Enterprise private cloud / on-prem datacenter | Same subnet isolation |
| Kali simulation | Red team / penetration testing engagement | Same attack techniques |
| Manual IGA | SailPoint / Saviynt | Same JML lifecycle |

The identity layer is fully modular. Authentik can be replaced with any
OAuth2/OIDC compliant provider without changing the zero trust policy model.

---

## AWS Infrastructure

### VPC Configuration

| Resource | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Public Subnet | 10.0.1.0/24 |
| Private Subnet | 10.0.2.0/24 |
| Internet Gateway | Attached to public subnet |
| NAT Gateway | Optional — for private subnet outbound |

### EC2 Instances

| VM | Name | Subnet | OS | Purpose |
|---|---|---|---|---|
| VM1 | ztna-authentik | Public | Ubuntu 22.04 | Identity Provider |
| VM2 | ztna-controller | Public | Ubuntu 22.04 | OpenZiti Controller |
| VM3 | ztna-router | Public | Ubuntu 22.04 | OpenZiti Router |
| VM4 | ztna-service | Private | Ubuntu 22.04 | Protected Service |
| VM5 | ztna-attacker | Private | Kali Linux | Attack Simulation |
| VM6 | ztna-wazuh | Private | Ubuntu 22.04 | SIEM |

### Security Groups

**ztna-controller-sg (Public VMs)**

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Admin IP only | SSH management |
| 8440 | TCP | 0.0.0.0/0 | OpenZiti Controller |
| 8441 | TCP | 0.0.0.0/0 | Controller management |
| 8442 | TCP | 0.0.0.0/0 | OpenZiti console UI |
| 8443 | TCP | 0.0.0.0/0 | Edge router |
| 443 | TCP | 0.0.0.0/0 | Authentik HTTPS |

**ztna-private-sg (Private VMs)**

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | 10.0.1.0/24 | SSH from public subnet only |
| All | All | 10.0.0.0/16 | Internal VPC traffic only |

### Cost

| Resource | Cost |
|---|---|
| VPC, subnets, security groups | Free always |
| EC2 t2.micro (free tier) | 750 hrs/month — free for 12 months |
| Elastic IPs (attached) | Free while instance running |
| Data transfer (under 15GB) | Free |
| **Total estimated cost** | **$0 — $5/month** |

---

## Attack Simulations

All attacks simulated from Kali Linux (VM5) — insider threat position.
Every result documented with Wazuh SIEM evidence.

| # | Attack | Tool | What Was Tried | Result | Wazuh Alert |
|---|---|---|---|---|---|
| 1 | Network port scan | nmap | Full port scan of private subnet | Zero open ports found | Yes |
| 2 | Direct service access | curl | TCP connection to service IP | Connection refused | Yes |
| 3 | Lateral movement | Metasploit | Pivot from Kali to service VM | Blocked by microsegmentation | Yes |
| 4 | Credential brute force | Hydra | Password attack on Authentik | Rate limited + account locked | Yes |
| 5 | mTLS interception | Wireshark | Capture and replay tunnel traffic | Mutual auth fails | Yes |
| 6 | Certificate spoofing | Custom script | Present self-signed cert to Router | Controller rejects | Yes |
| 7 | Token replay | Burp Suite | Replay captured OAuth2 token | Token expiry enforced | Yes |

---

## Threat Model

Built using the STRIDE framework aligned to NIST SP 800-207 threat analysis.

| Threat | STRIDE Category | Attack Vector | Mitigation |
|---|---|---|---|
| Identity spoofing | Spoofing | Fake certificate | mTLS — private key never leaves device |
| Traffic interception | Tampering | MitM on tunnel | TLS 1.3 + mutual authentication |
| Deny access logging | Repudiation | Disable Wazuh agent | Agent monitoring + integrity checks |
| Flood Controller | Denial of Service | Controller DDoS | AWS security groups + rate limiting |
| Excess privilege | Elevation of Privilege | Over-permissioned identity | Least privilege per identity policy |
| Credential theft | Spoofing | Stolen OAuth2 token | Token expiry + continuous re-verification |
| Insider threat | All categories | Compromised internal VM | Microsegmentation + Wazuh behavioral detection |

---

## Key Concepts

### Dark Network
The protected service has zero listening ports.
OpenZiti client dials outward to the Router — never inbound.
An attacker scanning the network finds nothing to target.

### Least Privilege
Each identity has a policy permitting access only to specific services.
No identity has broad network access.
Compromise of one identity exposes only what that identity was permitted to reach.

### Continuous Verification
OpenZiti does not trust at login and forget.
The Controller re-verifies every identity on every session.
A revoked certificate drops access everywhere within seconds.

### Blast Radius Containment
If an identity is compromised, the attacker can only reach what that identity
was permitted to reach — not the entire network.
This is the core failure of VPN that ZTNA solves.

### Control Plane vs Data Plane Separation
Access decisions happen on the control plane — Controller and Router talking.
Actual traffic flows on the data plane — encrypted tunnels only.
An attacker intercepting data plane traffic cannot affect control plane decisions.

### Instant Revocation
VPN: revoke credentials — session continues until it expires.
ZTNA: revoke certificate in Controller — access dropped everywhere in seconds.
This is the operational advantage that makes ZTNA superior for incident response.

---

## Project Progress

### Week 1 — Build

- [x] Day 1 — Concepts: NIST 800-207, mTLS, OAuth2, OpenZiti architecture
- [ ] Day 2 — AWS VPC, subnets, security groups, 6 EC2 instances
- [ ] Day 3 — OpenZiti Controller installed, configured, console accessible
- [ ] Day 4 — OpenZiti Router connected, first identity created and tested
- [ ] Day 5 — Authentik installed and integrated as OAuth2/OIDC provider
- [ ] Day 6 — Protected service deployed, end-to-end access verified
- [ ] Day 7 — Wazuh installed, agents on all VMs, dashboards live

### Week 2 — Attack

- [ ] Day 8  — STRIDE threat model written
- [ ] Day 9  — Port scanning and network enumeration (nmap)
- [ ] Day 10 — Lateral movement simulation (Metasploit)
- [ ] Day 11 — Credential attacks on Authentik (Hydra)
- [ ] Day 12 — mTLS interception attempt (Wireshark)
- [ ] Day 13 — Certificate spoofing and token replay
- [ ] Day 14 — Full Wazuh alert review and evidence collection

### Week 3 — Document

- [ ] Day 15 — Mitigation playbook (attack by attack)
- [ ] Day 16 — Final architecture diagram (draw.io)
- [ ] Day 17 — Full GitHub documentation
- [ ] Day 18 — Walkthrough video (10 min demo)
- [ ] Day 19 — LinkedIn article
- [ ] Day 20 — Hardening and policy automation
- [ ] Day 21 — Final polish and portfolio review

---

## Repository Structure

```
zero-trust-network-lab/
│
├── README.md                          ← This file
│
├── architecture/
│   ├── network-diagram.png            ← Full AWS + ZTNA diagram
│   ├── zero-trust-design.md           ← Design decisions and rationale
│   └── nist-mapping.md                ← NIST 800-207 component mapping
│
├── setup/
│   ├── 01-aws-vpc-setup.md            ← VPC, subnets, security groups
│   ├── 02-openziti-controller.md      ← Controller installation guide
│   ├── 03-openziti-router.md          ← Router installation guide
│   ├── 04-authentik-setup.md          ← IdP installation and integration
│   ├── 05-protected-service.md        ← nginx setup on private VM
│   └── 06-wazuh-siem.md               ← SIEM installation and agents
│
├── attacks/
│   ├── 01-port-scanning.md            ← nmap results and analysis
│   ├── 02-lateral-movement.md         ← Metasploit simulation
│   ├── 03-credential-attacks.md       ← Hydra brute force attempt
│   ├── 04-mtls-interception.md        ← Wireshark capture analysis
│   ├── 05-certificate-spoofing.md     ← Spoof attempt documentation
│   └── 06-token-replay.md             ← OAuth2 token replay attempt
│
├── docs/
│   ├── threat-model.md                ← STRIDE threat model
│   ├── mitigation-playbook.md         ← Attack by attack mitigations
│   └── lessons-learned.md             ← What worked, what didn't
│
└── evidence/
    ├── wazuh-alerts/                  ← Screenshots of Wazuh alerts
    └── attack-logs/                   ← Raw log captures
```

---

## Background

This project connects IAM governance experience with network-level
zero trust enforcement — a combination that is rare at the mid-level.

**IAM Experience:**
- 3+ years — Active Directory, RBAC, least privilege, JML, access reviews
- NIST CSF, ISO 27001, HIPAA compliance and audit support
- User provisioning, account lifecycle, permission reporting
- Azure AD / Entra ID, AWS IAM, OAuth2, SAML

**Network Engineering Experience:**
- Backbone and datacenter networks — spine-leaf, MPLS, hybrid WAN
- Cisco IOS — routing, switching, VLANs, end-to-end connectivity
- Enterprise lab — Windows + Linux + Active Directory + VLANs

**Education:**
- M.S. Cybersecurity Risk Management — Indiana University (May 2026)
- CompTIA Security+ | AWS Cloud Practitioner | CCNA
- CompTIA CySA+ (In Progress)

The architecture is fully modular — Authentik can be swapped for any
OAuth2/OIDC compliant provider without changing the zero trust policy model.
This maps directly to enterprise deployments using Okta, Azure AD,
or Google Workspace.

---

## References

- [NIST SP 800-207 — Zero Trust Architecture](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf)
- [Google BeyondCorp — A New Approach to Enterprise Security](https://research.google/pubs/beyondcorp-a-new-approach-to-enterprise-security/)
- [OpenZiti Documentation](https://openziti.io/docs/learn/introduction/)
- [Authentik Documentation](https://goauthentik.io/docs/)
- [Wazuh Documentation](https://documentation.wazuh.com/)
- [CISA Zero Trust Maturity Model](https://www.cisa.gov/zero-trust-maturity-model)
- [NSA Zero Trust Guidance](https://media.defense.gov/2021/Feb/25/2002588480/-1/-1/0/CSI_EMBRACING_ZT_SECURITY_MODEL_UOO115131-21.PDF)

---

## License

MIT License — free to use for learning and reference.

---

*Built by Rahul Rajkumar Kori*
*IAM + Cybersecurity Engineer | Indiana University M.S. Cybersecurity Risk Management*
*[LinkedIn](https://linkedin.com/in/rahul-kori) | [GitHub](https://github.com/Rahul7259)*
