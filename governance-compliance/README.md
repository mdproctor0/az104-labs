<img width="1672" height="941" alt="ChatGPT Image May 12, 2026 at 11_23_12 AM" src="https://github.com/user-attachments/assets/a58a07fa-f809-4e90-9ab9-5abd73701a32" />

# Azure Governance & Compliance — Contoso Financial Group

> **AZ-104 Lab Series · Day 2** — Built from scratch, no tutorials open.

---

## The Business Problem

Compliance auditors are arriving in 30 days. A pre-audit review identified three critical gaps in Contoso Financial Group's Azure environment:

1. No governance structure — no way to enforce company-wide policy consistently
2. Resources being deployed in unauthorized regions outside Contoso's operating areas
3. Critical production resources unprotected against accidental deletion
4. Administrators holding permanent privileged access — a direct security risk

All four problems had to be resolved before end of day.

---

## What I Built

### 1. Management Group Hierarchy

```
Tenant Root Group
    └── Contoso Financial Group (CFG_MG)
            ├── Development (Contoso_Development)
            │       └── CFG_Development [Subscription]
            └── Production (Contoso_Production)
                    └── CFG_Production [Subscription]
```

**Why this structure:**

- **Contoso Financial Group** is the parent management group. Any company-wide baseline policy assigned here inherits automatically to every subscription underneath — current and future. This is the correct scope for the region restriction policy.
- **Separate Development and Production management groups** allow environment-specific policy and RBAC without affecting each other. Dev can have looser controls. Production is locked down.
- **Separate subscriptions per environment** means billing is isolated, RBAC assignments cannot bleed across environments, and a misconfiguration in Development cannot touch Production resources.

**Why not assign subscriptions to the same management group:** Putting both subscriptions under one MG with no child structure removes the ability to govern them differently. The hierarchy exists specifically to enable differentiated governance at scale.

---

### 2. Region Restriction Policy

**Policy:** Allowed locations — Restrict deployments to East US and West US only

| Setting | Value | Why |
|---|---|---|
| Policy definition | Allowed locations (built-in) | Deny effect — blocks non-compliant deployments entirely |
| Scope | Contoso Financial Group (parent MG) | Inherits to all current and future subscriptions |
| Allowed locations | eastus, westus | Contoso's approved operating regions |
| Non-compliance message | Custom — explains why the deployment was blocked | Reduces support tickets, tells engineers exactly what happened |
| Remediation task | None | Deny effect stops future violations — remediation applies to DeployIfNotExists/Modify effects only |

**Why assigned at the parent Management Group:**
Assigning at the subscription level would require re-assigning the policy every time a new subscription is added. Assigning at the Management Group means any new subscription added to Contoso Financial Group inherits the restriction automatically — no additional configuration required.

**What happens when a developer tries to deploy to North Europe:**
The deployment is blocked immediately with a policy violation error showing the custom non-compliance message. The resource is never created. This is a Deny effect — not an audit, not a warning. A hard block.

---

### 3. Resource Lock — CFG_ID Production Resource Group

**Lock applied:** Delete lock on CFG_ID resource group

| Setting | Value | Why |
|---|---|---|
| Lock name | rg-delete-lock | Descriptive naming convention |
| Lock type | Delete | Prevents deletion, preserves full operational capability |
| Scope | CFG_ID Resource Group | Protects all resources inside the RG |
| Notes | Prevents accidental deletion | Documents intent for other administrators |

**Why Delete and not ReadOnly:**

The scenario required protecting against accidental deletion *without* removing anyone's ability to manage resources day to day. These two lock types produce completely different outcomes:

- **Delete lock** — administrators can still create, modify, start, stop, and reconfigure resources. They simply cannot delete the resource group or anything inside it. Operations continue normally.
- **ReadOnly lock** — all write operations are blocked. No one can modify resources, add new VMs, change configurations, or deploy anything into the resource group. This would have stopped the DevOps team from doing their job entirely.

**The exam trap this addresses:** Many candidates apply ReadOnly thinking it's "more secure." In an active environment it breaks operations. Match the lock type to what the business actually needs.

**Who can remove this lock:** Only users with Owner or User Access Administrator role. Contributor cannot manage locks — this is intentional. Even a Contributor cannot circumvent the lock by removing it.

---

### 4. PIM — Privileged Identity Management

**Configured:** PIM role settings for Owner — just-in-time access instead of permanent assignment

| Setting | Configured Value | Production Best Practice | Why It Matters |
|---|---|---|---|
| Activation maximum duration | 8 hours | 4 hours or less | Limits the window of exposure per activation |
| Require justification | Yes | Yes | Creates an audit trail for every Owner activation |
| Require MFA on activation | None ⚠️ | Required | **Not changed in lab** — see note below |
| Require approval | No ⚠️ | Yes — second administrator | **Not changed in lab** — see note below |
| Allow permanent eligible | No | No | Eligible assignments must expire and be reviewed |
| Expire eligible after | 1 year | 1 year | Forces annual access review |
| Allow permanent active | No | No | No one holds Owner permanently |
| Expire active after | 6 months | 6 months | Even direct active assignments expire |

---

#### ⚠️ Lab Note — Settings Not Changed and Why They Should Be in Production

**MFA on activation is set to None — should be Required in production**

In this lab environment MFA on activation was left at None to avoid disrupting access during testing. In a production financial services environment this must be set to **Require Azure MFA.**

Why it matters: When someone activates Owner through PIM, they are elevating to the highest privilege level in Azure. Without requiring MFA at activation, an attacker who has already compromised a user's session can activate Owner without any additional verification. MFA at activation means even a stolen session cannot escalate to Owner without the physical second factor. This is not optional in a regulated environment.

**Require approval is set to No — should be Yes in production**

In this lab approval was not configured because there is only one administrator account and approving your own request defeats the purpose. In a production environment with multiple administrators this must be set to **Yes** with a designated approver who is not the requestor.

Why it matters: Without approval, any eligible user can activate Owner for themselves at any time with zero oversight. With approval, every Owner activation requires a second person to consciously authorize it. This creates accountability, provides a real-time alert to another administrator that elevated access is being used, and creates an approval audit trail that satisfies compliance requirements in financial and government environments.

**The security difference between PIM and permanent Owner:**

| Scenario | Permanent Owner | PIM Eligible Owner |
|---|---|---|
| Account compromised at 2 AM | Attacker has Owner immediately and permanently | Attacker must activate Owner — triggers MFA, requires approval, limited to 8 hours |
| Malicious insider | Has Owner every second of every day | Owner only active during explicit activation windows with audit trail |
| Account left logged in | Full Owner access indefinitely | Owner access expires automatically — next session starts with no elevated privileges |
| Compliance audit | "User has had Owner for 3 years" | Full activation log — every time Owner was used, why, and for how long |

---

## What I'd Change in Production

1. **PIM MFA on activation → Required** — non-negotiable for Owner in any regulated environment
2. **PIM require approval → Yes** — second administrator must authorize every Owner activation
3. **Reduce activation duration** — 8 hours covers a full work day; 4 hours is more appropriate for Owner-level access
4. **Enable Conditional Access policy** — currently Report-only from Day 1; production requires On
5. **Add access reviews** — PIM supports scheduled access reviews so eligible assignments are periodically confirmed rather than silently expiring and being renewed without scrutiny

---

## Screenshots

### Management Group Hierarchy
![Management Group Hierarchy](screenshots/Management%20group%20hierarchy.png)

### Policy Assignment — Review
![Policy Assignment](screenshots/Policy%20assignment%20review.png)

### Resource Lock
![Resource Lock](screenshots/Resource%20lock.png)

### PIM Settings
![PIM Settings](screenshots/PIM%20settings.png)

---

## Labs

| Lab | Domain | Status |
|---|---|---|
| [Identity & Governance](./identity-governance) | Entra ID · RBAC · Conditional Access · Azure Policy | ✅ Complete |
| [Governance Compliance](./governance-compliance) | Entra ID · RBAC · Conditional Access · Azure Policy | ✅ Complete |
| [Networking](./networking) | VNet · NSG · Bastion · DNS · Load Balancers · VPN Gateway | 🔄 In progress |
| [Storage](./storage) | Blob · File Sync · SAS Tokens · Private Endpoints | ⬜ Upcoming |
| [Compute](./compute) | VMs · VMSS · App Service · ARM Templates · Containers | ⬜ Upcoming |
| [Monitor & Backup](./monitoring) | Log Analytics · KQL · Azure Monitor · Recovery Vault · ASR | ⬜ Upcoming |

---

*Part of my AZ-104 preparation series. Every lab is a real business scenario built from a blank page.*
