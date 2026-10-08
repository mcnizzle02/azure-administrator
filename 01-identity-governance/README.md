# Project 01 – Identity Foundation & Access Lifecycle

**Domain:** Manage Azure identities and governance (AZ-104)
**Company:** [Magnolia Bay Legal Group](../00-company-scenario/README.md)
**Business priorities addressed:** #1 Client confidentiality · #2 Know who has access to what
**Estimated cost:** $0 (Entra ID and Azure RBAC are free; optional VM test is pennies if deleted the same day)

---

## 1. The Ticket

> **From:** Diane Fairhope, Managing Partner
> **To:** IT
> **Subject:** Getting our people and permissions in order before the move
>
> Before we put a single client file in Azure, I want our accounts set up properly. Right now we have people sharing logins and nobody can tell me who has access to what. Our malpractice carrier is asking for that list by the end of the quarter.
>
> A few specific things:
> - Everyone needs their own account, set up by department.
> - Jordan is new in IT. He should be able to help people with passwords and restart servers, but I don't want him having the keys to everything yet.
> - Owen in billing needs to see what Azure is costing us. He should not be able to change anything.
> - We're working a case with co-counsel at another firm. Their attorney needs to *view* the litigation resources, temporarily.
> - Kevin Semmes in Pensacola gave notice. His last day is Friday. Make sure he's locked out the moment he walks out the door.
>
> Send me a one-page summary of who has access to what when you're done.

---

## 2. Requirements & Success Criteria

| ID | Requirement | Done when... |
|---|---|---|
| R1 | Create user accounts for the staff roster with department, job title, and office filled in | All 10 roster users exist with correct attributes |
| R2 | Create security groups, including at least one **dynamic** group | Group membership updates automatically when a user's attributes change |
| R3 | Create resource groups and assign Azure RBAC **to groups, not individuals**, at the narrowest scope | `Check access` shows each user has only what they need |
| R4 | Give Jordan limited helpdesk powers in Entra ID and limited VM powers in Azure | Jordan can reset a standard user's password and start/restart VMs, but cannot create or delete resources |
| R5 | Give Owen read-only visibility into costs | Owen can view Cost Management but cannot modify anything |
| R6 | Invite external co-counsel as a guest with read-only access to litigation resources only | Guest can sign in, sees only `rg-litigation-prod-01` |
| R7 | Offboard Kevin securely, then prove he can be restored if needed | Sign-in blocked, sessions revoked, account deleted, then restored from Deleted users |
| R8 | Produce an access matrix for the managing partner | One table showing every group/user and their access |

---

## 3. Environment

| Item | Value |
|---|---|
| Tenant | Your free-trial tenant (`<tenant>.onmicrosoft.com`) |
| Region | East US |
| Resource groups | `rg-identity-prod-01`, `rg-litigation-prod-01`, `rg-realestate-prod-01`, `rg-servers-prod-01` |
| Tags (all RGs) | `Owner=it-admin`, `Department=<dept>`, `Environment=prod`, `Project=P01-Identity` |

> **Licensing note:** Dynamic groups and assigning Entra roles to groups need **Microsoft Entra ID P1**. If your tenant doesn't have it, activate the free Entra ID P2 trial from the Entra admin center (Billing → Licenses → All products → Try/Buy). P2 includes everything in P1 and unlocks the stretch goals at the end of this project.

---

## 4. Build Tasks

Each task explains **what** to do and **why** it's done this way. Click-by-click steps go in your build notes with screenshots, written in your own words after you do them.

### Task 1 – Create the users (R1)

Create the 10 roster users from the [company scenario](../00-company-scenario/README.md). Do 2–3 manually in the portal, then create the rest with **Bulk operations → Bulk create** (download the CSV template from the portal and fill it in).

Set **Department**, **Job title**, **Office location**, and **Usage location** (US) on every user.

**Why it matters:**
- The attributes aren't decoration. Dynamic groups (Task 2) build membership from them, so a typo in `Department` silently breaks access later. Clean data in = correct access out.
- Usage location is required before you can assign licenses to a user. Admins who skip it hit an error the first time they try.
- Bulk creation is how this is actually done at onboarding time. Nobody hand-builds 40 accounts.

**CLI practice (do one user this way):**
```bash
az ad user create \
  --display-name "Jordan Dauphin" \
  --user-principal-name jordan.dauphin@<tenant>.onmicrosoft.com \
  --password "<temp-password>" \
  --force-change-password-next-sign-in true
```

### Task 2 – Create the groups (R2)

| Group | Type | Membership |
|---|---|---|
| `SG-IT-AzureAdmins` | Security, Assigned | You (your admin account) |
| `SG-IT-Support` | Security, Assigned | Jordan |
| `SG-Finance-CostReaders` | Security, Assigned | Owen |
| `SG-Paralegals-All` | Security, **Dynamic User** | Rule: `(user.department -eq "Paralegals")` |
| `SG-Litigation-Team` | Security, **Dynamic User** | Rule: `(user.jobTitle -contains "Litigation")` |
| `SG-External-CoCounsel` | Security, Assigned | The guest (added in Task 5) |

**Why it matters:**
- **Permissions go to groups, never directly to people.** When someone joins or leaves, you change one group membership instead of hunting down a dozen individual assignments. This is also what makes the access matrix (R8) possible to produce and keep accurate.
- **Dynamic groups remove the human error from onboarding/offboarding.** When HR changes someone's department, their access follows automatically.
- Dynamic membership is not instant. Entra ID processes rules in the background, which is the root cause of a common ticket (see Troubleshooting T1).

### Task 3 – Resource groups and Azure RBAC (R3, R5)

Create the four resource groups with the required tags. Then assign roles:

| Who | Role | Scope | Why this role and scope |
|---|---|---|---|
| `SG-IT-AzureAdmins` | Contributor | Subscription | Can build and manage everything, but **cannot grant access to others** (that requires Owner or User Access Administrator). Keep Owner on your break-glass account only. |
| `SG-Litigation-Team` | Reader | `rg-litigation-prod-01` | Can see their resources, can't change them. |
| `SG-Finance-CostReaders` | Cost Management Reader | Subscription | Sees spending, can't touch resources. Cost data is at subscription level, so that's the narrowest scope that works here. |

**Why it matters:**
- **Scope is inherited downward:** management group → subscription → resource group → resource. A role at the subscription applies to every resource group under it. Always ask "what's the smallest scope that still lets them do the job?"
- **Azure RBAC and Entra ID roles are two separate systems.** Azure RBAC controls *Azure resources* (VMs, storage, networks). Entra ID roles control *the directory* (users, groups, passwords). This distinction is heavily tested on AZ-104 and SC-300, and it trips up a lot of people. Task 4 uses both on purpose.

**CLI practice:**
```bash
az role assignment create \
  --assignee-object-id <SG-Litigation-Team-objectId> \
  --assignee-principal-type Group \
  --role "Reader" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-litigation-prod-01
```

### Task 4 – Limited powers for Jordan (R4)

**Part A – Entra ID role (directory side):** Assign Jordan the **Helpdesk Administrator** role. (Assign it directly to Jordan, or to `SG-IT-Support` if you create that group as *role-assignable*, which is a setting you must choose when the group is created and can't add later.)

**Part B – Azure custom role (resource side):** None of the built-in roles match "start and restart VMs but nothing else," so build a **custom role** called `VM Operator` with only these actions:
```json
"Actions": [
  "Microsoft.Compute/virtualMachines/read",
  "Microsoft.Compute/virtualMachines/start/action",
  "Microsoft.Compute/virtualMachines/restart/action",
  "Microsoft.Compute/virtualMachines/deallocate/action",
  "Microsoft.Resources/subscriptions/resourceGroups/read"
]
```
Set the assignable scope to your subscription, then assign it to `SG-IT-Support` on `rg-servers-prod-01` only.

*Optional test:* deploy the smallest B-series Linux VM into `rg-servers-prod-01`, sign in as Jordan, confirm he can restart it but cannot delete it or resize it. **Delete the VM the same day.**

**Why it matters:**
- **Helpdesk Administrator instead of Global Administrator** is least privilege in action. Helpdesk Admin can reset passwords for regular users but is blocked from resetting passwords for most admin accounts, so a compromised junior account can't be used to take over a Global Admin.
- **Custom roles** exist for exactly this situation: when "Contributor" is too much and "Reader" is too little. Being able to read and write a role definition is a skill hiring managers look for.

### Task 5 – External co-counsel as a guest (R6)

Invite an external user (use a second personal email you control) via **Users → Invite external user**. Add them to `SG-External-CoCounsel`, and assign that group **Reader** on `rg-litigation-prod-01`.

**Why it matters:**
- **B2B guest access** means the outside attorney signs in with *their own* organization's account. You never create or manage a password for them, and when their firm disables their account, their access to yours stops too.
- Giving them a separate group (instead of adding them to `SG-Litigation-Team`) keeps external access visible and easy to remove when the case closes. Auditors love being able to see "all external access" in one place.

### Task 6 – Offboarding Kevin (R7)

Do these in order and note the reason for the order:

1. **Block sign-in** on Kevin's account
2. **Revoke sessions**
3. Change his `Department` to something else or remove it, and watch him drop out of `SG-Paralegals-All` automatically
4. Remove him from any assigned groups
5. **Delete** the account
6. Plot twist: Diane emails that Kevin's files are needed for a case. **Restore** him from **Deleted users**.

**Why it matters:**
- Blocking sign-in stops *new* logins, but a user who's already signed in keeps working on existing tokens. **Revoking sessions** invalidates those, which is why both steps are needed for an immediate lockout.
- Deleted users stay in a soft-deleted state for **30 days** and can be restored with their group memberships and object ID intact. After that, they're gone permanently. Knowing this window exists has saved many admins from a very bad day.

### Task 7 – The access matrix (R8)

Build a table for Diane showing every group, its members, and its access (Entra roles and Azure RBAC). Verify each row with **Access control (IAM) → Check access** rather than from memory.

**Why it matters:** this is the deliverable the business actually asked for. The technical work only counts if you can explain it to a non-technical managing partner.

---

## 5. Validation

Sign in as each test user in an **InPrivate/Incognito** window so you don't mix up sessions with your admin account.

| Test | Expected result | Pass/Fail |
|---|---|---|
| Tasha (Litigation paralegal) opens the Azure portal | Sees `rg-litigation-prod-01` only, read-only | |
| Kevin's department changed | Removed from `SG-Paralegals-All` automatically | |
| Jordan resets Priya's password | Succeeds | |
| Jordan tries to reset *your* admin password | Blocked | |
| Jordan tries to delete a VM | Denied | |
| Owen opens Cost Management | Sees costs, cannot create a budget | |
| Guest signs in | Sees only `rg-litigation-prod-01` | |
| Kevin tries to sign in after offboarding | Blocked | |
| Kevin restored from Deleted users | Account and group memberships return | |

> **Expect an MFA prompt** when test users first sign in. New tenants have **security defaults** turned on, which requires every user to register for MFA. That's a feature, not a bug. Note it in your writeup.

---

## 6. Troubleshooting Tickets

After the build is complete, work these tickets. Try to diagnose before opening the hints. Document your actual investigation in the Troubleshooting Log.

**T1 – "I'm a litigation paralegal and I can't see anything in Azure."** (Tasha)
<details><summary>Hints</summary>

- Is she actually a member of `SG-Litigation-Team`? Check the group's members, not the user.
- What does her `jobTitle` say exactly? Does it match the rule?
- When was she added? Dynamic membership and role assignments both take time to apply.
- Did she sign out and back in? Her token may predate the assignment.
</details>

**T2 – "Jordan says the password reset button is greyed out for one of the partners."**
<details><summary>Hints</summary>

- Does that partner hold any Entra admin role? What can a Helpdesk Administrator reset, and what can't it?
- Is this a bug, or the security control working as designed? How would you explain that to Jordan?
</details>

**T3 – "Co-counsel accepted the invite but says the Azure portal is empty."**
<details><summary>Hints</summary>

- Which directory is the guest looking at in the portal? Guests land in their *home* directory by default.
- Look at the directory switcher in the portal settings.
</details>

---

## 7. Cleanup & Cost

- [ ] Delete any test VM and its disk, NIC, and public IP (or delete `rg-servers-prod-01` entirely)
- [ ] **Keep** users and groups. Later projects reuse them
- [ ] Remove the guest when the "case closes" (a good habit to document)
- [ ] Record the actual cost from Cost Management: $____

---

## 8. My Build Notes *(fill in as you go)*

### What I built
<!-- Screenshots go in ./images/. Redact tenant IDs, subscription IDs, and emails. -->

### Why I made these choices
<!-- In your own words. This is the section interviewers read. -->

### Troubleshooting Log
| Issue | What I checked | Root cause | Fix |
|---|---|---|---|
| | | | |

### Lessons Learned

---

## Stretch Goals (with Entra ID P2 trial)

- **Privileged Identity Management (PIM):** make Jordan's Helpdesk Administrator role *eligible* instead of permanent, so he activates it only when needed, with a reason and time limit
- **Access reviews:** schedule a review of `SG-External-CoCounsel` so the case attorney confirms each quarter that the guest still needs access
- **Conditional Access:** require MFA for all admin roles and block sign-ins from outside the US
- Rebuild Tasks 1–3 entirely in **PowerShell (Microsoft Graph module)** or **Azure CLI** as a single script

## Interview Talking Points

- Why I assign permissions to groups, not users
- The difference between Azure RBAC and Entra ID roles, and when I used each
- Why Contributor instead of Owner for the admin group
- Why I built a custom role instead of using a built-in one
- Why blocking sign-in alone isn't enough to lock out a departing employee

