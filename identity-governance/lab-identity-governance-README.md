<img width="1672" height="941" alt="ChatGPT Image May 12, 2026 at 11_23_12 AM" src="https://github.com/user-attachments/assets/2cedeac8-3d47-4b77-afbe-28aeb962bda6" />

# Azure Identity & Governance — Contoso Financial Group

> **AZ-104 Lab Series · Day 1** — Built from scratch, no tutorials open.

---

## The Business Problem

Contoso Financial Group needed a production-ready identity and access architecture in Azure. Three teams with different access requirements, one external contractor with temporary access, and two compliance mandates from the CISO — all had to be configured correctly before end of day.

This is not a tutorial walkthrough. I was given the requirements and built the solution.

---

## What I Built

### Identity Structure

| Identity | Type | Group | Purpose |
|---|---|---|---|
| Marquell Proctor | Member user | — | Administrator / Owner |
| Landon Hayes | Member user | DevOps Team | Internal developer |
| Roman Proctor | Member user | Security Team | Security analyst |
| Alex Chen | Guest user (B2B) | — | External contractor, temporary access |

**Why guest for Alex Chen:** External contractors should never be created as member users in your tenant. B2B invitation keeps their identity managed by their home organization and makes it immediately visible (via #EXT# in UPN) that they are external. Mixing contractors into internal team groups violates least privilege — their access is scoped directly at the resource level instead.

**Why Security groups, not M365 groups:** RBAC role assignments require Security groups. M365 groups are for collaboration (Teams, SharePoint, shared mailboxes). Using the wrong group type means your RBAC assignments won't apply.

---

### RBAC Architecture

| Principal | Role | Scope | Reasoning |
|---|---|---|---|
| Security Team (group) | Reader | Subscription | Monitors all resources across the subscription — inherited downward to every RG and resource |
| DevOps Team (group) | Contributor | CFGlab Resource Group | Full resource control scoped to their project only — cannot touch resources outside this RG, cannot grant access to others |
| Alex Chen (user) | Reader | CFGlab Resource Group | Minimum access at narrowest possible scope — time-bound to expire automatically |

**Key decisions:**

- **DevOps Team is Contributor, not Owner** — Contributor has full resource control but cannot grant access to others. Owner could elevate their own permissions or grant access to unauthorized users. For a team that should only be managing resources, Contributor is the correct boundary.

- **DevOps Team is scoped at Resource Group, not Subscription** — Subscription-level Contributor would give them access to the Security team's resources and all other resource groups. Least privilege means narrowest scope that still allows the job to be done.

- **Alex Chen is time-bound** — Permanent access for a temporary contractor is a security risk. Azure supports time-bound role assignments natively. Alex's access automatically expires without requiring manual cleanup.

---

### Conditional Access Policy

**Policy:** Require MFA for all users signing in from outside the corporate network

| Setting | Value | Why |
|---|---|---|
| Users | All users | Applies org-wide |
| Locations condition | Any location, trusted locations excluded | Office network is safe — MFA only required outside |
| Named location | 10.10.10.10/16 — marked as trusted | Simulates corporate office IP range |
| Grant control | Require MFA | Forces second factor for non-trusted sign-ins |
| State | Report-only (lab) / On (production) | Report-only for testing — enables logging without enforcement |

**Why "Mark as trusted" matters:** The Conditional Access exclusion of "trusted locations" is meaningless if no named location is defined and marked as trusted. The policy logic is correct — the named location configuration is what makes it actually function.

**License requirement:** Conditional Access requires **Entra ID P1 minimum**. Without P1, this feature is unavailable regardless of how it's configured.

---

## What I'd Change in Production

1. **Enable the CA policy** — Report-only is for testing only. In production this goes to On immediately.
2. **Add a second CA policy** to block access entirely for users on unknown devices from unknown locations (the CISO's second requirement — compliant device check).
3. **Scope the DevOps Contributor assignment more tightly** — in a real environment I'd create separate resource groups per project rather than one shared CFGlab RG.
4. **PIM for the Owner role** — Marquell's permanent Owner assignment should be converted to just-in-time access via Privileged Identity Management so Owner rights are only active when explicitly needed.
5. **Dynamic group for Security Team** — if the organization uses a consistent department attribute in HR, a dynamic group rule would auto-populate the Security Team group without manual maintenance.

---

## Files

```
identity-governance/
├── README.md                          # This file
├── screenshots/
│   ├── users-list.png
│   ├── groups-devops-members.png
│   ├── groups-security-members.png
│   ├── rbac-assignments.png
│   ├── conditional-access-policies.png
│   ├── ca-policy-conditions.png
│   ├── ca-policy-grant.png
│   └── named-locations.png
```

---

## Lab Series

| Lab | Domain | Status |
|---|---|---|
| **Identity & Governance** | Entra ID · RBAC · Conditional Access · Azure Policy | ✅ Complete |
| Networking | VNet · NSG · Bastion · DNS · Load Balancers | 🔄 In progress |
| Storage | Blob · File Sync · SAS · Private Endpoints | ⬜ Upcoming |
| Compute | VMs · VMSS · App Service · ARM Templates | ⬜ Upcoming |
| Monitor & Backup | Log Analytics · KQL · Recovery Vault · ASR | ⬜ Upcoming |

---

*Part of my AZ-104 preparation series. Every lab is a real business scenario built from a blank page.*
