# Zero Trust Network Access Lab

A production-grade Zero Trust Network Access (ZTNA) lab built on AWS, implementing identity-based microsegmentation, OIDC federation, and continuous SIEM monitoring — all using open-source tools that mirror enterprise products.

**Author:** Rahul Rajkumar Kori
**M.S. Cybersecurity Risk Management** — Indiana University Bloomington, May 2026
**Started:** July 2026 | **Status:** Week 1 Complete — Attack Simulation Phase Next

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         AWS VPC (10.0.0.0/16)                    │
│                                                                   │
│   PUBLIC SUBNET (10.0.1.0/24)                                    │
│   ┌──────────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│   │  ztna-controller  │  │ ztna-authentik │  │   ztna-wazuh    │  │
│   │  OpenZiti         │  │ Authentik IdP  │  │   Wazuh SIEM    │  │
│   │  Controller +     │  │ OAuth2/OIDC    │  │   Manager +     │  │
│   │  Router           │  │ Provider       │  │   Dashboard     │  │
│   │  [Wazuh Agent]    │  │ [Wazuh Agent]  │  │                 │  │
│   └──────────────────┘  └───────────────┘  └─────────────────┘  │
│                                                                   │
│   PRIVATE SUBNET (10.0.2.0/24)                                   │
│   ┌──────────────────┐                                           │
│   │  ztna-service     │  ← No public IP                          │
│   │  nginx (protected)│  ← No open ports (dark network)          │
│   │  ziti-edge-tunnel │  ← Only reachable via OpenZiti identity   │
│   │  [Wazuh Agent]    │                                           │
│   └──────────────────┘                                           │
└──────────────────────────────────────────────────────────────────┘

Access Flow:
  User → Authentik (login) → JWT token → OpenZiti (verify) → mTLS tunnel → nginx

Monitoring Flow:
  All VMs → Wazuh agents → Wazuh Manager → alerts + compliance mapping
```

---

## What This Project Proves

| Zero Trust Principle | Implementation | Proof |
|---|---|---|
| **Verify Explicitly** | OpenZiti mTLS + Authentik OIDC federation | JWT token with correct claims issued and validated |
| **Least Privilege** | OpenZiti Dial/Bind policies with role attributes | Only #clients can access #nginx — nothing more |
| **Assume Breach** | Wazuh SIEM with custom detection rules | Failed logins and JWT auth failures detected in real time |

---

## Tech Stack

| Tool | Role | Enterprise Equivalent |
|---|---|---|
| **OpenZiti** | ZTNA Controller + Router | Zscaler Private Access / Cloudflare Access |
| **Authentik** | Identity Provider (OAuth2/OIDC) | Okta / Azure AD (Entra ID) |
| **Wazuh** | SIEM + Continuous Monitoring | Splunk / Microsoft Sentinel |
| **nginx** | Protected Service | Internal Enterprise Application |
| **AWS VPC** | Cloud Infrastructure | Enterprise Private Cloud |

---

## Key Results

### Zero Trust Enforcement (Day 3)
- **Direct access** to nginx → **Connection timed out** (BLOCKED)
- **OpenZiti access** with valid identity → **Page loads** (ALLOWED)
- Service is completely invisible without a verified identity

### Identity Federation (Day 4)
- Authentik issues signed JWT with correct claims (iss, aud, sub, email)
- OpenZiti configured as relying party with ext-jwt-signer pointing to Authentik JWKS
- Full OAuth2 authorization code flow validated end to end

### SIEM Detection (Day 5)
- Failed SSH login detected with source IP, username, and severity
- Custom rule 100200: OpenZiti JWT authentication failure (level 10)
- Custom rule 100201: Multiple failures in 120 seconds — brute force indicator (level 12)
- All events mapped to HIPAA, PCI-DSS, and NIST compliance frameworks

---

## Project Timeline

| Day | Focus | Status |
|---|---|---|
| Day 1 | NIST 800-207, mTLS, OAuth2/OIDC, STRIDE, IGA concepts | ✅ Complete |
| Day 2 | AWS VPC infrastructure + OpenZiti installation | ✅ Complete |
| Day 3 | Protected service + zero trust enforcement proof | ✅ Complete |
| Day 4 | Authentik IdP + OAuth2/OIDC federation with OpenZiti | ✅ Complete |
| Day 5 | Wazuh SIEM + agents + custom detection rules | ✅ Complete |
| Week 2 | Attack simulation (nmap, Metasploit, Hydra, Wireshark) | ⬜ Next |
| Week 3 | Documentation, architecture diagram, walkthrough video | ⬜ Planned |

---

## NIST 800-207 Alignment

1. **Everything is a resource** — every VM and service is protected
2. **All communication secured regardless of location** — mTLS encryption everywhere
3. **Access is per session** — each connection verified independently
4. **Access determined by dynamic policy** — OpenZiti role-based policies
5. **Monitor all assets continuously** — Wazuh agents on every VM
6. **Authentication strictly enforced** — Authentik OIDC + OpenZiti certificate validation
7. **Collect everything, improve continuously** — Wazuh log collection + custom rules

---

## Repository Structure

```
zero-trust-network-lab/
|-- README.md
|-- progress/
|   |-- Day1_Concepts_Foundation.docx
|   |-- Day2_AWS_OpenZiti.docx
|   |-- Day3_Protected_Service_ZT_Enforcement.docx
|   |-- Day4_Authentik_Identity_Provider.docx
|   |-- Day5_Wazuh_SIEM_Monitoring.docx
|-- configs/
|   |-- wazuh-custom-rules.xml
|-- screenshots/
```

---

## Skills Demonstrated

- **Cloud Infrastructure:** AWS VPC, subnets, security groups, NAT Gateway, EC2
- **Zero Trust Architecture:** NIST 800-207, microsegmentation, dark network principle
- **Identity & Access Management:** OAuth2, OIDC, JWT validation, JWKS, claims mapping
- **Network Security:** mTLS, certificate-based authentication, identity-based access
- **SIEM & Monitoring:** Wazuh deployment, agent management, custom detection rules
- **Identity Federation:** Authentik to OpenZiti trust via ext-jwt-signer
- **Compliance:** HIPAA, PCI-DSS, NIST 800-53 mapping via automated SIEM alerts

---

## Contact

**Rahul Rajkumar Kori**
- GitHub: [github.com/Rahul7259](https://github.com/Rahul7259)
- Email: rkori@iu.edu
