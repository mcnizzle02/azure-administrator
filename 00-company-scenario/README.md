# 00 – Company Scenario: Magnolia Bay Legal Group

> **Note:** Magnolia Bay Legal Group is a fictional company created for this portfolio. Every project in this repository is built as if I were the cloud administrator for this firm. All names, users, and data are invented.

---

## Company Profile

| | |
|---|---|
| **Company** | Magnolia Bay Legal Group, LLP |
| **Industry** | Legal services (civil litigation, real estate, estate planning) |
| **Size** | 42 employees |
| **Headquarters** | Mobile, AL |
| **Branch office** | Pensacola, FL (8 staff) |
| **Remote staff** | ~10 attorneys and paralegals work hybrid |
| **Cloud platform** | Microsoft Azure + Microsoft Entra ID |
| **My role** | Cloud Administrator (one-person IT team with an outside MSP for after-hours support) |

## The Business Problem

Magnolia Bay has run for years on an aging on-premises file server, a single domain controller in a closet at the Mobile office, and shared passwords for several systems. After a near-miss phishing incident and a client audit that asked hard questions about who can access case files, the managing partners approved a move to Azure.

Their priorities, in their own words:

1. **"Client files must stay confidential."** Only the people working a matter should be able to see it. Attorney-client privilege is not negotiable.
2. **"We need to know who has access to what."** Clients and malpractice insurers are now asking for proof.
3. **"Pensacola and remote staff should work like they're in the main office."**
4. **"Don't surprise us with the bill."** The firm is cost-conscious; every resource needs an owner and a budget.
5. **"If something breaks or gets deleted, we need it back."**

Every project in this repo answers one or more of these priorities.

## Organization

| Department | Headcount | Access needs |
|---|---|---|
| Partners | 5 | Everything in their practice area, firm financials (read) |
| Associates (attorneys) | 12 | Case files for their practice area |
| Paralegals | 10 | Case files for their practice area, document templates |
| Legal Assistants | 6 | Calendars, templates, limited case-file access |
| Finance & Billing | 3 | Billing system, financial records |
| Office Administration | 4 | HR files, general office resources |
| IT | 2 | Azure administration (me + junior tech) |

**External parties:** co-counsel firms, expert witnesses, and outside auditors occasionally need **temporary, limited** access.

### Sample staff roster (used across projects)

| Display Name | Department | Job Title | Office |
|---|---|---|---|
| Diane Fairhope | Partners | Managing Partner | Mobile |
| Marcus Theodore | Partners | Partner – Litigation | Mobile |
| Renee Daphne | Associates | Associate Attorney – Litigation | Mobile |
| Carlos Saraland | Associates | Associate Attorney – Real Estate | Pensacola |
| Tasha Grand | Paralegals | Senior Paralegal – Litigation | Mobile |
| Kevin Semmes | Paralegals | Paralegal – Real Estate | Pensacola |
| Priya Bayou | Legal Assistants | Legal Assistant | Mobile |
| Owen Spanish | Finance & Billing | Billing Coordinator | Mobile |
| Gina Loxley | Office Administration | Office Manager | Mobile |
| Jordan Dauphin | IT | IT Support Technician | Mobile |

---

## Azure Standards

These standards apply to every project. Writing them down first is the point: real organizations run on consistent conventions, and following them is part of the job.

### Region

- **Primary region:** East US (closest low-cost region for Alabama/Florida users)
- Deploying outside approved regions is blocked by Azure Policy (see Project 02).

### Naming convention

Format: `<resource-type>-<workload>-<environment>-<instance>`

| Resource | Prefix | Example |
|---|---|---|
| Resource group | `rg` | `rg-identity-prod-01` |
| Virtual network | `vnet` | `vnet-hub-prod-01` |
| Subnet | `snet` | `snet-servers-prod-01` |
| Network security group | `nsg` | `nsg-servers-prod-01` |
| Virtual machine | `vm` | `vm-fileserver-prod-01` |
| Storage account | `st` | `stmbcasefilesprod01` *(no dashes allowed, must be globally unique)* |
| Log Analytics workspace | `log` | `log-ops-prod-01` |
| Key Vault | `kv` | `kv-mbsecrets-prod-01` |

**Groups in Entra ID:** `SG-<Department>-<Purpose>` (e.g., `SG-Paralegals-All`, `SG-IT-AzureAdmins`)

### Required tags

| Tag | Example values | Why |
|---|---|---|
| `Owner` | `it-admin` | Someone is accountable for every resource |
| `Department` | `IT`, `Litigation`, `Finance` | Cost reporting by department |
| `Environment` | `prod`, `dev`, `lab` | Separate real workloads from experiments |
| `Project` | `P01-Identity` | Ties resources back to this repo |

### Cost rules

- Monthly budget with alerts at **50%, 80%, and 100%**
- Every project lives in its own resource group and is **deleted after documentation** unless a later project depends on it
- VMs are **deallocated** when not in use; smallest size that meets the need
- Hourly-billed networking resources (Bastion, VPN Gateway, NAT Gateway, public IPs) are deployed, tested, documented, and removed the same day

### Security baseline

- **Least privilege:** permissions are assigned to groups, not individual users, at the narrowest scope that works
- **No shared accounts.** Every person has their own identity
- **MFA for everyone,** with stronger controls for admins
- **No secrets in this repo** (keys, SAS tokens, connection strings, passwords). Screenshots are redacted

---

## Project Roadmap

Organized by the AZ-104 exam domains.

| # | Domain | Project | Business priority | Status |
|---|---|---|---|---|
| 01 | Identity & Governance | Identity Foundation & Access Lifecycle | 1, 2 | 🟡 In progress |
| 02 | Identity & Governance | Governance Guardrails: Policy, Tags, Locks & Budgets | 2, 4 | ⚪ Planned |
| 03 | Storage | Secure Case-File Storage & Recovery | 1, 5 | ⚪ Planned |
| 04 | Compute | File Server VM: Deploy, Secure, Resize & Template | 3 | ⚪ Planned |
| 05 | Networking | Hub-and-Spoke Network for Mobile & Pensacola | 1, 3 | ⚪ Planned |
| 06 | Monitor & Maintain | Website Uptime Monitor (Azure Functions + App Insights) | 5 | 🟢 Complete |
| 07 | Monitor & Maintain | Log Analytics, KQL & Alerting | 2, 5 | ⚪ Planned |
| 08 | Monitor & Maintain | Backup & Restore Drill | 5 | ⚪ Planned |

## How Each Project Is Documented

Every project README follows the same format:

1. **The Ticket:** the request as it would arrive from the business
2. **Requirements & Success Criteria:** what "done" looks like
3. **Build:** what I deployed, with screenshots
4. **Why I Made These Choices:** the reasoning behind each design decision
5. **Validation:** how I proved it works
6. **Troubleshooting Log:** what broke and how I fixed it
7. **Cleanup & Cost:** what I removed and what it cost
8. **Lessons Learned**

---

## About Me

Tier 2 IT analyst with 13 years in IT support and a U.S. Army veteran, building hands-on Azure administration and cloud security skills. Currently pursuing a B.S. in Computer Science Information Technology at the University of South Alabama.
