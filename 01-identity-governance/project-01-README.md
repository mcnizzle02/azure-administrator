# Project 01 – Identity Foundation & Access Lifecycle

**Domain:** Manage Azure identities and governance (AZ-104 / SC-300)
**Company:** [Magnolia Bay Legal Group](../00-company-scenario/README.md)
**Business priorities addressed:** #1 Client confidentiality · #2 Know who has access to what
**Tools used:** Azure portal, Microsoft Entra admin center, Azure Cloud Shell (Azure CLI), Microsoft Graph API (`az rest`), Microsoft Graph PowerShell
**Licensing:** Microsoft Entra ID P2 (trial), required for dynamic groups, role-assignable groups, PIM, access reviews, and Conditional Access
**Actual cost:** $____ *(one test VM for under an hour; Entra ID, RBAC, PIM, and Conditional Access add no Azure consumption cost)*

> **Lab note:** Magnolia Bay Legal Group is fictional. This project was built in a dedicated lab tenant and subscription, separate from my personal Azure workloads. Tenant IDs, subscription IDs, object IDs, UPN domains, and email addresses are redacted in all screenshots.
>
> **Second build:** I completed this project once, then tore the environment down to a verified clean slate and rebuilt it from scratch so every step is documented as it happened. Lessons from the first build are noted where they changed my approach.

---

## 1. The Ticket

> **From:** Diane Fairhope, Managing Partner
> **To:** IT
> **Subject:** Getting our people and permissions in order before the move
>
> Before we put a single client file in Azure, I want our accounts set up properly. Right now we have people sharing logins and nobody can tell me who has access to what. Our malpractice carrier is asking for that list by the end of the quarter.
>
> - Everyone needs their own account, set up by department.
> - Jordan is new in IT. He should be able to help people with passwords and restart servers, but I don't want him having the keys to everything yet.
> - Owen in billing needs to see what Azure is costing us. He should not be able to change anything.
> - We're working a case with co-counsel at another firm. Their attorney needs to *view* the litigation resources, temporarily.
> - Kevin Semmes in Pensacola gave notice. His last day is Friday. Make sure he's locked out the moment he walks out the door.
>
> Send me a one-page summary of who has access to what when you're done.

---

## 2. Requirements & Results

| ID | Requirement | Result |
|---|---|---|
| R1 | User accounts for the staff roster with department, job title, office, and license | ✅ 10 users (portal, CLI + Graph API, bulk CSV), all P2-licensed |
| R2 | Security groups, including dynamic groups | ✅ 6 groups: 2 dynamic, 1 role-assignable |
| R3 | Azure RBAC assigned to groups at the narrowest workable scope | ✅ Verified with Check access and sign-in tests |
| R4 | Jordan: limited helpdesk powers (Entra) and limited VM powers (Azure) | ✅ Helpdesk Administrator + custom `VM Operator` role |
| R5 | Owen: read-only cost visibility | ✅ Cost Management Reader; budget creation denied |
| R6 | External co-counsel: read-only access to litigation resources only | ✅ B2B guest in a dedicated group, Reader on one resource group |
| R7 | Offboard Kevin securely, then prove he can be restored | ✅ Blocked, sessions revoked, deleted, restored (still disabled) |
| R8 | Access matrix for the managing partner | ✅ See [Section 5](#5-access-matrix) |
| S1 | Just-in-time admin access (PIM) | ✅ Helpdesk Administrator eligible, approval + MFA + justification required |
| S2 | Recurring review of external access | ✅ Quarterly access review, partner as reviewer |
| S3 | Conditional Access replacing security defaults | ✅ 4 policies, rolled out report-only first, break-glass excluded |

---

## 3. Environment

| Item | Value |
|---|---|
| Tenant | Dedicated lab tenant (Magnolia Bay Legal), Microsoft Entra ID P2 trial |
| Subscription | `magnolia-bay-admin-lab` (pay-as-you-go, isolated from personal workloads) |
| Region | East US |
| Resource groups | `rg-identity-prod-01`, `rg-litigation-prod-01`, `rg-realestate-prod-01`, `rg-servers-prod-01` |
| Tags (all RGs) | `Owner=it-admin`, `Department=<dept>`, `Environment=prod`, `Project=P01-Identity` |
| Accounts kept outside the project | My admin account (Global Admin, subscription Owner) and `breakglass01` (emergency Global Admin) |

---

## 4. Build

### Task 0 – Verified clean slate

Before rebuilding, I removed every Project 01 and Project 02 object, purged deleted users and groups from the directory recycle bin, re-enabled security defaults, and verified the result. Only my admin account, `breakglass01`, `SG-CA-Exclusions`, and my own Owner assignment remained.

![Users after teardown](images/00-setup/task0-users-clean.png)
![Groups after teardown](images/00-setup/task0-groups-clean.png)

### Setup – License and resource groups

Confirmed the P2 trial and its expiration date, which set the deadline for the P2-dependent work. Created one resource group in the portal and the rest with the CLI, all tagged to the [company standard](../00-company-scenario/README.md#required-tags).

```bash
az group create --name rg-litigation-prod-01 --location eastus \
  --tags Owner=it-admin Department=Litigation Environment=prod Project=P01-Identity

az group list --query "[?tags.Project=='P01-Identity'].{Name:name, Dept:tags.Department}" -o table
```

![P2 license](images/00-setup/task0-setup-p2-license.png)
![Resource groups with tags](images/00-setup/task0-setup-rg-list-with-tags.png)

---

### Task 1 – Users (R1)

| Method | Users | Why |
|---|---|---|
| Portal | Diane Fairhope, Tasha Grand | See every attribute the directory stores |
| Azure CLI + Graph API | Jordan Dauphin | Scripted onboarding |
| Bulk create (CSV) | Remaining 7 | How onboarding is done at scale |

Every user has **Department**, **Job title**, **Office location**, and **Usage location** set. `az ad user create` only supports basic properties, so I set the rest with a direct Graph API call. I used `az rest` instead of `Connect-MgGraph` because security defaults block the device-code sign-in `Connect-MgGraph` uses from Cloud Shell (a lesson from the first build).

```bash
az ad user create --display-name "Jordan Dauphin" \
  --user-principal-name jordan.dauphin@$DOMAIN \
  --password "<temp-password>" --force-change-password-next-sign-in true

az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/users/jordan.dauphin@$DOMAIN" \
  --headers "Content-Type=application/json" \
  --body '{"department":"IT","jobTitle":"IT Support Technician","officeLocation":"Mobile","usageLocation":"US"}'
```

P2 licenses were assigned to all ten users. The bulk-create CSV (which contains temporary passwords) is not in this repository.

![Manual user properties](images/01-users/task1-manual-users-properties.png)
![Bulk create results](images/01-users/task1-bulk-results.png)
![All users with department, title, and office](images/01-users/task1-all-users-columns.png)
![Licenses assigned](images/01-users/task1-licenses-assigned.png)

---

### Task 2 – Groups (R2)

| Group | Type | Membership |
|---|---|---|
| `SG-IT-AzureAdmins` | Assigned | IT admin account |
| `SG-IT-Support` | Assigned, **role-assignable** | Jordan |
| `SG-Finance-CostReaders` | Assigned | Owen |
| `SG-Paralegals-All` | **Dynamic** | `(user.department -eq "Paralegals")` |
| `SG-Litigation-Team` | **Dynamic** | `(user.jobTitle -contains "Litigation")` |
| `SG-External-CoCounsel` | Assigned | External guest (Task 5) |

I created `SG-IT-Support` in the portal because a role-assignable group can only be created with role-management permission, which the Azure CLI's token doesn't carry. I tested the Paralegals rule with **Validate Rules** before saving it. `SG-Litigation-Team` was created through the Graph API, since `az ad group create` can't create dynamic groups:

```bash
az rest --method POST --url "https://graph.microsoft.com/v1.0/groups" \
  --headers "Content-Type=application/json" \
  --body '{
    "displayName": "SG-Litigation-Team", "mailNickname": "SG-Litigation-Team",
    "securityEnabled": true, "mailEnabled": false,
    "groupTypes": ["DynamicMembership"],
    "membershipRule": "(user.jobTitle -contains \"Litigation\")",
    "membershipRuleProcessingState": "On"
  }'
```

![Role-assignable setting](images/02-groups/task2-role-assignable-setting.png)
![Validating the dynamic rule](images/02-groups/task2-dynamic-rule-validate.png)
![Groups list](images/02-groups/task2-groups-list.png)
![Members of every group, including both dynamic groups](images/02-groups/task2-dynamic-members.png)

---

### Task 3 – Azure RBAC (R3, R5)

| Group | Role | Scope | Method |
|---|---|---|---|
| `SG-IT-AzureAdmins` | Contributor | Subscription | Portal |
| `SG-Litigation-Team` | Reader | `rg-litigation-prod-01` | CLI |
| `SG-Finance-CostReaders` | Cost Management Reader | Subscription | CLI |

```bash
az role assignment create --assignee-object-id $LIT --assignee-principal-type Group \
  --role "Reader" --scope /subscriptions/$SUB/resourceGroups/rg-litigation-prod-01 \
  --description "P01 - litigation team read-only access to case resources"
```

I tested both directions: Tasha has **Reader** on the litigation resource group and **nothing** on the identity resource group, and her portal shows only `rg-litigation-prod-01`. Owen filled out the entire budget form, and the denial came only when the request reached Azure Resource Manager.

![Contributor for SG-IT-AzureAdmins at subscription scope](images/03-rbac/task3-contributor-review.png)
![All role assignments](images/03-rbac/task3-role-assignments-list.png)
![Tasha: Check access](images/03-rbac/task3-tasha-check-access.png)
![Tasha's portal view](images/03-rbac/task3-tasha-portal-view.png)
![Owen's budget creation denied](images/03-rbac/task3-owen-budget-denied.png)

---

### Task 4 – Limited powers for Jordan (R4)

#### Part A – Helpdesk Administrator (Entra ID role, directory scope)

Assigned to the role-assignable `SG-IT-Support` group through the Graph role-management API. In the first build, the portal routed this assignment through PIM and failed with "The role is not found," so I used the API path that had worked. The assignment became PIM-eligible in Stretch 1.

```bash
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignments" \
  --headers "Content-Type=application/json" \
  --body "{\"principalId\":\"$GROUP\",\"roleDefinitionId\":\"$ROLE\",\"directoryScopeId\":\"/\"}"
```

![Helpdesk Administrator assignment](images/04-jordan/task4-helpdesk-assignment.png)
![Jordan resets a standard user's password](images/04-jordan/task4-priya-reset-success.png)
![Jordan blocked from resetting an administrator's password](images/04-jordan/task4-admin-reset-blocked.png)

#### Part B – Custom `VM Operator` role (Azure RBAC)

No built-in role allows "start and restart VMs but nothing else," so I wrote one: [`role-definitions/vm-operator.json`](role-definitions/vm-operator.json)

```json
"Actions": [
  "Microsoft.Compute/virtualMachines/read",
  "Microsoft.Compute/virtualMachines/start/action",
  "Microsoft.Compute/virtualMachines/restart/action",
  "Microsoft.Compute/virtualMachines/deallocate/action",
  "Microsoft.Resources/subscriptions/resourceGroups/read"
]
```

The role is *assignable* across the subscription but *assigned* only on `rg-servers-prod-01`. I pulled the scope from Azure (`az group show --query id`) instead of typing it.

![Jordan has VM Operator through SG-IT-Support](images/04-jordan/task4-vm-operator-check-access.png)

**Test VM:** Ubuntu, `Standard_D2alds_v7`, no public IP, no NSG, auto-shutdown on, disk and NIC set to delete with the VM. Deleted the same day, and the resource group was verified empty.

![Test VM deployed](images/04-jordan/task4-vm-deployed.png)
![Jordan stopped, started, and restarted the VM](images/04-jordan/task4-jordan-power-ops.png)
![Resize denied: missing virtualMachines/write](images/04-jordan/task4-jordan-resize-denied.png)
![Delete denied: missing virtualMachines/delete](images/04-jordan/task4-jordan-delete-denied.png)
![Run command unavailable](images/04-jordan/task4-jordan-runcommand-denied.png)
![Resource group empty after cleanup](images/04-jordan/task4-rg-empty-after-cleanup.png)

---

### Task 5 – External co-counsel as a B2B guest (R6)

Invited **Alex Morgan (Co-counsel – Whitfield & Ames)** as a guest, added the guest to `SG-External-CoCounsel`, and assigned that group **Reader** on `rg-litigation-prod-01`.

The guest first landed in their **home** directory and saw nothing (ticket T3). Using the tenant-specific portal URL put them in ours, where they could see only the litigation resource group and couldn't change it. Default guest restrictions blocked them from browsing the staff directory.

![Guest invitation](images/05-guest/task5-invite.png)
![Guest user access restrictions](images/05-guest/task5-guest-access-settings.png)
![Guest view of rg-litigation-prod-01](images/05-guest/task5-guest-litigation.png)
![Guest blocked from the Users list](images/05-guest/task5-guest-directory-restricted.png)

---

### Task 6 – Offboarding Kevin (R7)

| Step | Time |
|---|---|
| Recorded groups, license, and object ID | 10:11 PM |
| Blocked sign-in | 10:13 PM |
| Revoked sessions | 10:15 PM |
| Sign-in test → blocked | 10:17 PM |
| Department → `Former Employee` | 10:20 PM |
| Removed from `SG-Paralegals-All` automatically | 10:22 PM |
| Deleted | 10:23 PM |
| Restored (managing partner needed his files) | 10:25 PM |

```bash
az ad user update --id $KEVIN --account-enabled false
az rest --method POST --url "https://graph.microsoft.com/v1.0/users/$KEVIN/revokeSignInSessions"
az rest --method PATCH --url "https://graph.microsoft.com/v1.0/users/$KEVIN" \
  --headers "Content-Type=application/json" --body '{"department":"Former Employee"}'
az ad user delete --id $KEVIN
az rest --method POST --url "https://graph.microsoft.com/v1.0/directory/deletedItems/$KEVIN/restore"
```

The restore brought back the **same object** with its license, still **disabled**, and still outside the dynamic group because his department remained `Former Employee`. The firm can retrieve his files, but Kevin can't sign in.

![Kevin's groups before offboarding](images/06-offboarding/task6-kevin-groups-before.png)
![Kevin's license before offboarding](images/06-offboarding/task6-kevin-license-before.png)
![Kevin's sign-in blocked](images/06-offboarding/task6-kevin-signin-blocked.png)
![SG-Paralegals-All after the department change](images/06-offboarding/task6-paralegals-after.png)
![Kevin in Deleted users](images/06-offboarding/task6-kevin-deleted.png)
![Kevin restored: same account, still disabled](images/06-offboarding/task6-kevin-restored.png)

---

### Stretch 1 – Privileged Identity Management (S1)

Converted Jordan's Helpdesk Administrator access from **standing** to **just-in-time**.

**Role settings:** 2-hour maximum activation · MFA required · justification required · **approval required** (approver: IT admin)

**Order:** removed the active assignment first, then added the eligible one. PIM won't let the same group be active and eligible for one role at once. This time the eligible assignment worked in the portal. In the first build it failed with "The role is not found," which supports my earlier conclusion that PIM wasn't fully set up yet in a tenant where P2 had just been activated.

![PIM role settings](images/08-pim/stretch1-role-settings.png)
![SG-IT-Support eligible for Helpdesk Administrator](images/08-pim/stretch1-eligible-assignment.png)
![Reset denied before activation](images/08-pim/stretch1-reset-denied-before.png)
![Activation request with justification](images/08-pim/stretch1-activation-request.png)
![Approval](images/08-pim/stretch1-approval.png)
![Reset succeeds after activation](images/08-pim/stretch1-reset-success-after.png)
![PIM audit: request, approval, activation](images/08-pim/stretch1-audit-log.png)

---

### Stretch 2 – Access review of external access (S2)

**`AR-External-CoCounsel-Quarterly`:** reviews **guests only** in `SG-External-CoCounsel`, quarterly, with **Marcus Theodore** (the litigation partner) as reviewer. Results apply automatically; if the reviewer doesn't respond, access is removed.

**Test:** Marcus denied the guest. The guest was removed from the group, but the account was kept, because co-counsel may work with the firm again. I then added the guest back, and their Reader access returned automatically because it's assigned to the group.

![Access review settings](images/09-access-review/stretch2-review-settings.png)
![Reviewer denies the guest](images/09-access-review/stretch2-reviewer-deny.png)
![Guest removed from the group](images/09-access-review/stretch2-guest-removed.png)

---

### Stretch 3 – Conditional Access (S3)

Replaced security defaults with four Conditional Access policies. Every policy excludes `SG-CA-Exclusions`, which contains only the `breakglass01` emergency account.

| Policy | Users | Condition | Grant / Session |
|---|---|---|---|
| CA01-AllUsers-AllApps-RequireMFA | All users | — | Require MFA |
| CA02-Admins-AllApps-RequireMFA | Global, Privileged Role, Helpdesk, User Administrators | — | Require MFA · 4-hour sign-in frequency |
| CA03-AllUsers-LegacyAuth-Block | All users | Client apps: Exchange ActiveSync, Other clients | Block |
| CA04-AllUsers-AllApps-BlockOutsideUS | All users | Any location except named location `US-Only` | Block |

**Rollout:**
1. Verified `breakglass01` signs in, holds Global Administrator, and is the only member of `SG-CA-Exclusions`
2. Created all four policies in **Report-only**
3. Disabled security defaults at **3:08 PM** and turned on CA01 at **3:09 PM**, a one-minute window without enforced MFA
4. Validated with **What If** (target: Azure Management), then turned on CA02–CA04 one at a time
5. Signed in as `breakglass01` with all four policies on

**What If results:**

| User | Condition | Policies applied |
|---|---|---|
| breakglass01 | — | None (excluded) |
| Tasha | Country: United Kingdom | CA01 + CA04 (block) |
| Tasha | Client app: Other clients | CA01 + CA03 (block) |
| Jordan (Helpdesk Admin activated via PIM) | — | CA01 + CA02 |

![Break-glass exclusion group](images/10-conditional-access/stretch3-breakglass-exclusion-group.png)
![Security defaults disabled](images/10-conditional-access/stretch3-security-defaults-disabled.png)
![All four policies on](images/10-conditional-access/stretch3-policy-list-on.png)
![MFA prompt for a standard user](images/10-conditional-access/stretch3-tasha-mfa-prompt.png)
![What If: break-glass excluded](images/10-conditional-access/stretch3-whatif-breakglass.png)
![What If: UK sign-in blocked](images/10-conditional-access/stretch3-whatif-uk-blocked.png)
![What If: legacy authentication blocked](images/10-conditional-access/stretch3-whatif-legacy-blocked.png)
![What If: admin policy applies while Jordan's role is active](images/10-conditional-access/stretch3-whatif-jordan-admin.png)
![Sign-in log: Conditional Access tab](images/10-conditional-access/stretch3-signin-log-ca-tab.png)
![Break-glass signs in with all policies enforced](images/10-conditional-access/stretch3-breakglass-signin-after.png)

---

## 5. Access Matrix

*Verified with `az role assignment list`, a Graph role-assignment query, group membership queries, Check access, and sign-in tests as each user.*

| Group / Account | Members | Entra ID role | Azure role | Scope |
|---|---|---|---|---|
| `SG-IT-AzureAdmins` | Leamon McNutt | — | Contributor | Subscription |
| `SG-IT-Support` | Jordan Dauphin | Helpdesk Administrator *(PIM-eligible, approval required, 2-hour max)* | VM Operator *(custom)* | `rg-servers-prod-01` |
| `SG-Finance-CostReaders` | Owen Spanish | — | Cost Management Reader | Subscription |
| `SG-Litigation-Team` *(dynamic)* | Tasha Grand, Marcus Theodore, Renee Daphne | — | Reader | `rg-litigation-prod-01` |
| `SG-External-CoCounsel` | Alex Morgan (Co-counsel – Whitfield & Ames) | — | Reader | `rg-litigation-prod-01` |
| `SG-Paralegals-All` *(dynamic)* | Tasha Grand | — | None yet *(reserved for case-file storage, Project 03)* | — |
| `SG-CA-Exclusions` | breakglass01 | — | — | Excluded from all CA policies |
| **Admin account** *(direct)* | — | Global Administrator | Owner | Subscription |
| **breakglass01** *(direct)* | — | Global Administrator *(permanent, by design)* | — | — |
| **Kevin Semmes** | — | — | None. Disabled, restored for file access only | — |
| Diane, Carlos, Priya, Gina | — | — | None *(no business need yet)* | — |

![All role assignments](images/07-access-matrix/task7-role-assignments-all.png)

### Summary for the managing partner

> **Subject: Azure access summary – ready for the carrier**
>
> Diane, every staff member now has their own account, and nobody shares logins. Access is granted by role, not by person, so when someone joins, changes jobs, or leaves, their access updates in one place.
>
> - **IT** manages our Azure environment. Jordan can restart servers but can't create, delete, or change anything. His password-reset access is switched on only when needed, requires approval, and every use is logged. He can't reset an administrator's password.
> - **Owen** can see what Azure costs us and can't change anything.
> - **The litigation team** and **co-counsel at Whitfield & Ames** can view litigation resources only. Co-counsel signs in with their own firm's account, a partner confirms each quarter that they still need access, and access can be removed in one step when the case closes.
> - **Kevin's** account was locked out on his last day. It's been restored only so his files can be retrieved for the case. He can't sign in.
> - **Everyone else**, including you, has no access to our Azure systems, because their jobs don't require it. We grant access by need, not by title.
> - **All accounts require multi-factor authentication**, and sign-ins from outside the United States are blocked.
> - **One emergency administrator account** exists in case our normal sign-in protections ever fail. It's stored securely and isn't used day to day.
>
> Every permission was tested by signing in as each person, not just configured.

---

## 6. Validation

All tests were run in InPrivate windows, signed in as each user.

| Test | Expected | Result |
|---|---|---|
| Tasha opens the Azure portal | Sees only `rg-litigation-prod-01`, read-only | ✅ Pass |
| Check access: Tasha on `rg-identity-prod-01` | No access | ✅ Pass |
| Owen creates a budget | Denied at submission | ✅ Pass |
| Jordan resets Priya's password | Succeeds | ✅ Pass |
| Jordan resets the admin's password | Blocked | ✅ Pass |
| Jordan restarts / stops / starts a VM | Succeeds | ✅ Pass |
| Jordan resizes a VM | Denied (`virtualMachines/write`) | ✅ Pass |
| Jordan deletes a VM | Denied (`virtualMachines/delete`) | ✅ Pass |
| Jordan uses Run command | Denied | ✅ Pass |
| Guest signs in | Sees only `rg-litigation-prod-01`; can't browse users | ✅ Pass |
| Kevin signs in after offboarding | Blocked | ✅ Pass |
| Kevin's department changed | Removed from `SG-Paralegals-All` automatically | ✅ Pass |
| Kevin restored | Same object and license; still disabled; outside dynamic groups | ✅ Pass |
| Jordan resets a password before PIM activation | Denied | ✅ Pass |
| Jordan activates (justification + MFA + approval) | Activated, time-limited | ✅ Pass |
| Jordan resets a password after activation | Succeeds | ✅ Pass |
| PIM audit | Request, approval, and activation logged | ✅ Pass |
| Reviewer denies guest | Removed from group; account retained | ✅ Pass |
| What If: break-glass | No policies apply | ✅ Pass |
| What If: UK sign-in / legacy client | Blocked by CA04 / CA03 | ✅ Pass |
| What If: Jordan with active admin role | CA01 + CA02 | ✅ Pass |
| Break-glass sign-in with all policies on | Succeeds | ✅ Pass |

---

## 7. Troubleshooting Log

| Issue | What I checked | Root cause | Fix |
|---|---|---|---|
| Rebuild teardown: captured IDs produced garbled errors and `Bad Request` | Error text was being overwritten mid-line; `which az` | Windows Azure CLI (`az.exe`) running inside WSL returns `\r\n`, so every captured variable ended in an invisible carriage return | Stripped `\r` with `tr -d '\r'`, then moved to Cloud Shell (Linux CLI) for the rebuild |
| Teardown: `az ad group delete` on `SG-IT-Support` → *Insufficient privileges* | My role (Global Admin) vs. the tool's permissions | Role-assignable groups require role-management permission, which the Azure CLI's token doesn't carry | Deleted the group in the portal |
| Teardown: deleted security groups still present | Graph `directory/deletedItems` | Security groups are soft-deleted for 30 days, like users | Purged them through the Graph API |
| Task 1: bulk-created job titles formatted differently (`Partner-Litigation` vs. `Senior Paralegal - Litigation`) | Verified all user attributes with a Graph query | Inconsistent formatting in the CSV | The Litigation rule uses `-contains`, so membership was unaffected. Documented as a data-quality risk for stricter rules |
| Task 3: `MissingSubscription` when assigning Reader | Printed `$SUB` (correct), re-read the scope | Typo: `/subscription/` instead of `/subscriptions/` | Corrected the path; now pull scopes from Azure (`az group show --query id`) instead of typing them |
| Task 4: Cloud Shell started a new session mid-task | Cloud Shell banner | Cloud Shell without storage is ephemeral; local files like `vm-operator.json` don't persist | Saved the role definition locally; the role itself was unaffected in Azure |
| Stretch 1: PIM portal assignment | Compared with the first build | First build: "The role is not found" right after P2 activation. Second build: worked in the portal | Confirms the first-build failure was PIM not yet being fully set up in a newly activated tenant |

---

## 8. Why I Made These Choices

> ✏️ *Draft. Rewrite in my own words before publishing.*

- **Groups, not individuals.** Every permission goes to a group, so onboarding, job changes, and offboarding are one membership change, and the access matrix stays accurate. The two direct assignments (my admin account and break-glass) are documented, not hidden.
- **Dynamic groups.** Access follows HR attributes. Changing Kevin's department removed his access without anyone touching group membership.
- **Contributor, not Owner, for the admin group.** Contributor manages every resource but can't grant access, so a compromised admin account can't create backdoor role assignments.
- **Two permission systems, used deliberately.** Jordan's password resets are an **Entra role** (directory scope); his server control is **Azure RBAC** (resource-group scope).
- **A custom role instead of a built-in one.** Leaving out `write` blocks resizing and every other change; leaving out `runCommand/action` means Jordan can operate servers but can't run code on them. I accepted that he can't see network or disk details, because his job is restarting servers.
- **A separate group for external access**, so all outside access is visible and removable in one place.
- **Block, then revoke.** Blocking stops new sign-ins; revoking kills existing refresh tokens. The reverse order would leave a window to sign back in.
- **PIM with approval for helpdesk access.** Approval kept on as an example of least-privilege access: the role exists only when it's being used, for a stated reason, with a second person's sign-off.
- **Removing only the guest's membership** after the access review, because co-counsel may work with the firm again.
- **Conditional Access rollout:** a tested break-glass account first, every policy in report-only, What If before enforcement, policies turned on one at a time, and a one-minute gap between disabling security defaults and enforcing MFA.
- **Unknown countries treated as outside the US**, the stricter choice for client data.

---

## 9. Lessons Learned

- I learned a lot about security and the importance of least-privilege access.
- I improved my knowledge of PIM and Conditional Access.
- I learned to double-check my commands when using the CLI.
- ✏️ *Add more in your own words*

---

## 10. Cleanup & Cost

- [x] Test VM, disk, NIC, and VNet deleted the same day; `rg-servers-prod-01` verified empty
- [x] Users, groups, roles, PIM, access review, and Conditional Access **kept** for later projects
- [ ] Remove the guest and the co-counsel group's access when the "case closes"
- [ ] Actual cost from Cost Management: $____

---

## Interview Talking Points

- Why I assign permissions to groups, not users
- Azure RBAC vs. Entra ID roles, and when I used each
- Why Contributor instead of Owner for the admin group
- Why I built a custom role, and the trade-off I accepted
- Why blocking sign-in alone isn't enough to lock out a departing employee
- PIM: matching friction to risk
- How I rolled out Conditional Access without locking myself out
- Why I rebuilt the project from a verified clean slate
