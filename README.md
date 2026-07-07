# Zero Trust Network Access (ZTNA) Lab

![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)
![AWS](https://img.shields.io/badge/Cloud-AWS-orange)
![OpenZiti](https://img.shields.io/badge/ZTNA-OpenZiti-blue)
![NIST](https://img.shields.io/badge/Framework-NIST%20SP%20800--207-green)

---

## Overview

A hands-on implementation of Zero Trust Network Access (ZTNA) built on AWS,
designed to demonstrate identity-based microsegmentation, dark network principles,
and attack resilience — aligned to NIST SP 800-207 zero trust architecture.

This project bridges IAM governance with network-level identity enforcement.
Every connection is authenticated, authorized, and continuously verified.
No implicit trust. No open ports. No perimeter assumptions.

> *"Security isn't a feature you bolt on after the network is built.
> It's the foundation you design around from day one."*

---

## Motivation

Networks have evolved — security hasn't kept up.

Coming from an IAM and network engineering background, I have seen firsthand
how organizations enforce strong identity governance at the application layer
while the network layer still operates on perimeter trust — if you are inside
the network, you are trusted.

That model is broken.

East-west traffic between datacenter workloads is largely uninspected.
A single compromised account can move laterally across internal segments.
VPNs hand users broad network access instead of least-privilege access
to specific resources.

The 2020 SolarWinds breach was not a failure of perimeter security —
it was proof that perimeter security is the wrong model entirely.

Zero Trust flips this. Every connection is untrusted by default.
Every identity is verified continuously. Every resource is protected
individually — not hidden behind a wall that assumes everyone inside is safe.

This project is my hands-on answer to that problem. Not just understanding
zero trust conceptually, but building it, attacking it, and proving it holds.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      AWS VPC                            │
│                                                         │
│  PUBLIC SUBNET (10.0.1.0/24)                           │
│  ┌─────────────────┐      ┌─────────────────┐         │
│  │    Authentik    │      │    OpenZiti     │         │
│  │   Identity      │─────►│   Controller    │         │
│  │   Provider      │      │   (Brain)       │         │
│  │   OAuth2/OIDC   │      │   Issues Certs  │         │
│  └─────────────────┘      └────────┬────────┘         │
│                                    │                   │
│                           ┌────────▼────────┐         │
│                           │    OpenZiti     │         │
│                           │    Router       │         │
│                           │   (Enforcer)    │         │
│                           │   PEP Layer     │         │
│                           └────────┬────────┘         │
│                                    │                   │
│  PRIVATE SUBNET (10.0.2.0/24)      │                  │
│         ┌──────────────────────────┘                  │
│         │                                             │
│  ┌──────▼──────────┐      ┌─────────────────┐        │
│  │    Protected    │      │  Kali Linux     │        │
│  │    Service      │      │  Attacker VM    │        │
│  │    (nginx)      │      │  Simulation     │        │
│  │  ZERO OPEN PORTS│      │                 │        │
│  └─────────────────┘      └─────────────────┘        │
│                                                        │
│  ┌─────────────────┐                                  │
│  │     Wazuh       │                                  │
│  │     SIEM        │                                  │
│  │  Monitors All   │                                  │
│  └─────────────────┘                                  │
└────────────────────────────────────────────────────────┘
```

---

## How It Works

### Legitimate User Flow

```
1. User opens OpenZiti client
2. No certificate → redirected to Authentik
3. Authentik verifies identity (username + password + MFA)
4. OpenZiti Controller issues X.509 certificate
5. Certificate presented to OpenZiti Router
6. Router checks policy → access granted
7. Encrypted mTLS tunnel opened to protected service
8. Wazuh logs the entire session
```

### Attack Flow (What Kali Sees)

```
1. Port scan → zero open ports found (dark network)
2. Direct connection to service IP → refused
3. Request certificate without identity → blocked
4. Intercept mTLS tunnel → mutual auth fails
5. Credential brute force → rate limited + locked
6. Every attempt logged and alerted in Wazuh
```

---

## Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| Identity Provider | Authentik | OAuth2/OIDC human identity verification + MFA |
| ZTNA Core | OpenZiti | Controller, Router, certificate management |
| Cloud Infrastructure | AWS VPC | Network isolation, subnets, security groups |
| Protected Resource | nginx (Ubuntu EC2) | Target service with zero open ports |
| Attack Simulation | Kali Linux | Lateral movement, scanning, credential attacks |
| SIEM | Wazuh | Log collection, threat detection, alerting |

---

## Protocols Used

| Protocol | Where | Purpose |
|---|---|---|
| mTLS | OpenZiti tunnels | Mutual authentication — both sides verified |
| X.509 | All identities | Cryptographic identity certificates |
| OAuth2 / OIDC | Authentik → OpenZiti | Human identity token exchange |
| TLS 1.3 | All encrypted traffic | Transport layer encryption |
| SSH | Admin access | Secure VM management |
| HTTPS | All web interfaces | Authentik, Wazuh, OpenZiti console |

---

## NIST SP 800-207 Mapping

| NIST Component | This Lab |
|---|---|
| Policy Engine | OpenZiti Controller — makes access decisions |
| Policy Administrator | OpenZiti Controller — executes decisions |
| Policy Enforcement Point | OpenZiti Router — the actual gate |
| Identity Provider | Authentik — verifies human identity |
| Continuous Monitoring | Wazuh SIEM — watches everything |
| Microsegmentation | AWS private subnet — isolated segments |
| Dark Network | Protected service has zero open ports |

---

## Enterprise Mapping

| This Lab | Enterprise Equivalent |
|---|---|
| Authentik | Okta / Azure AD / Google Workspace |
| OpenZiti Controller | Zscaler Controller / Cloudflare Access |
| OpenZiti Router | Zscaler Connector / Cloudflare Tunnel |
| Wazuh | Splunk / Microsoft Sentinel |
| AWS VPC | Enterprise private cloud / on-prem datacenter |

---

## Attack Simulations (Week 2)

| Attack | Tool | Expected Result | Logged in Wazuh |
|---|---|---|---|
| Network port scan | nmap | Zero ports found | Yes |
| Direct service access | curl | Connection refused | Yes |
| Lateral movement | Metasploit | Blocked by microsegmentation | Yes |
| Credential brute force | Hydra | Rate limited + locked | Yes |
| mTLS interception | Wireshark | Mutual auth fails | Yes |
| Certificate spoofing | Custom | Controller rejects | Yes |

---

## Project Progress

- [x] Day 1 — Concepts: NIST 800-207, mTLS, OAuth2, OpenZiti architecture
- [ ] Day 2 — AWS VPC, subnets, security groups, EC2 instances
- [ ] Day 3 — OpenZiti Controller installed and configured
- [ ] Day 4 — OpenZiti Router connected, first identity created
- [ ] Day 5 — Authentik integrated as OAuth2/OIDC identity provider
- [ ] Day 6 — Protected service deployed, end-to-end access verified
- [ ] Day 7 — Wazuh installed, agents deployed, dashboards configured
- [ ] Week 2 — Attack simulations and threat model documentation
- [ ] Week 3 — Mitigation playbook, architecture diagrams, writeup

---

## Threat Model

Built using the STRIDE framework:

| Threat | Category | Mitigation |
|---|---|---|
| Stolen credentials | Spoofing | mTLS binds cert to device private key |
| Traffic interception | Tampering | TLS 1.3 encryption on all tunnels |
| Identity repudiation | Repudiation | Wazuh logs every access with timestamp |
| Service unavailable | Denial of Service | Controller redundancy, rate limiting |
| Unauthorized access | Elevation of Privilege | Least privilege policies per identity |

---

## Key Concepts Demonstrated

- **Dark Network** — Protected service has zero listening ports. Invisible to scanners.
- **Least Privilege** — Each identity accesses only what policy explicitly permits.
- **Continuous Verification** — Controller re-verifies every identity every session.
- **Instant Revocation** — Certificate revoked in Controller = access dropped everywhere in seconds.
- **Blast Radius Containment** — Compromised identity reaches only its permitted resources.
- **Identity as Perimeter** — Network location means nothing. Identity means everything.

---

## Background

This project connects my IAM governance experience with network-level
zero trust enforcement:

- 3+ years IAM — Active Directory, RBAC, least privilege, JML, access reviews
- NIST CSF, ISO 27001, HIPAA compliance experience
- Network engineering background — VLANs, Cisco IOS, backbone and datacenter networks
- M.S. Cybersecurity Risk Management, Indiana University (May 2026)

The architecture is modular — Authentik can be swapped for any OAuth2/OIDC
compliant provider (Okta, Azure AD, Google Workspace) without changing the
zero trust policy model.

---

## References

- [NIST SP 800-207 — Zero Trust Architecture](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf)
- [Google BeyondCorp Research Paper](https://research.google/pubs/beyondcorp-a-new-approach-to-enterprise-security/)
- [OpenZiti Documentation](https://openziti.io/docs/learn/introduction/)
- [CISA Zero Trust Maturity Model](https://www.cisa.gov/zero-trust-maturity-model)

---

## License

MIT License — feel free to use this for your own learning.

---

*Built by Rahul Rajkumar Kori — IAM + Network Security Engineer*
*[LinkedIn](https://linkedin.com/in/rahul-kori) | [GitHub](https://github.com/Rahul7259)*
