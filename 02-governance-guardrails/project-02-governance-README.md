# Project 02 – Governance Guardrails: Policy, Tags, Locks & Budgets

**Domain:** Manage Azure identities and governance (AZ-104)
**Company:** [Magnolia Bay Legal Group](../../00-company-scenario/README.md)
**Business priorities addressed:** #2 Know who has access to what · #4 Don't surprise us with the bill
**Builds on:** [Project 01](../project-01-identity-foundation/README.md) (resource groups, groups, and the `VM Operator` role)
**Estimated cost:** $0 (Azure Policy, tags, locks, and budgets are free; denied deployments never create resources)

---

## 1. The Ticket

> **From:** Owen Spanish, Billing Coordinator
> **CC:** Diane Fairhope, Managing Partner
> **To:** IT
> **Subject:** Azure costs and "who owns this?"
>
> I've been looking at the Azure bill now that I can see it. A few things:
>
> - I can't tell which department is spending what. Some items have a department listed and some don't.
> - Diane wants a monthly spending limit, and she wants to hear about it **before** we hit it, not after.
> - I saw a charge for a server size I've never heard of. Can we stop people from picking expensive ones?
> - Our malpractice carrier says client data has to stay in the U.S. Can you guarantee nothing gets created overseas?
>
> **Diane added:** Last month at my old firm someone deleted the production file server by accident. I want our important stuff protected from that, even from IT.

---

## 2. Requirements & Success Criteria

| ID | Requirement | Done when... |
|---|---|---|
| R1 | Resources can only be created in approved U.S. regions | Deploying to a non-U.S. region is **denied** |
| R2 | Only approved, cost-effective VM sizes can be deployed | Deploying a large VM size is **denied** |
| R3 | Every resource group must have a `Department` tag | Creating an untagged resource group is **denied** |
| R4 | Resources automatically inherit `Department` from their resource group | A resource created without the tag gets it automatically; existing resources are fixed with remediation |
| R5 | Guardrails are grouped and managed as one baseline | One **initiative** assigned at the subscription; compliance visible in one dashboard |
| R6 | Production resources are protected from accidental deletion | Deleting a locked resource group fails, even for an Owner |
| R7 | Monthly budget with early warnings to Owen and IT | Budget with alerts at 50%, 80%, and 100% (actual) and 100% (forecasted) |
| R8 | A documented, time-limited exception process | A policy **exemption** with an expiration date and a reason |

---

## 3. Environment

| Item | Value |
|---|---|
| Tenant / subscription | Magnolia Bay Legal lab tenant / `magnolia-bay-admin-lab` |
| Policy scope | Subscription (management groups are a stretch goal) |
| Approved regions | East US, East US 2, Central US *(plus `global` for global services)* |
| Approved VM sizes | A short list of small B- and D-series sizes. Check which are actually available to your subscription first (see Task 2) |
| Required tag | `Department` on resource groups, inherited by resources |
| Locked resource groups | `rg-identity-prod-01`, `rg-litigation-prod-01` |

---

## 4. Build Tasks

Each task explains **what** to do and **why**. Click-by-click steps go in your build notes with screenshots.

> **Timing warning:** Azure Policy is not instant. A new assignment can take **up to 30 minutes** to start being enforced and to show compliance results. If a deny policy "doesn't work" right after you assign it, wait before you troubleshoot. (This is ticket T1.)

### Task 1 – Allowed locations (R1)

Assign the built-in policy **Allowed locations** at the subscription scope with your approved regions. Also assign **Allowed locations for resource groups**.

**Why it matters:**
- These are two different things. A resource group's location only stores its *metadata*. The resources inside can still be anywhere. You need both policies to cover both cases. (Classic AZ-104 question.)
- Include **global** in the allowed list. Some services (like Entra-related resources, Traffic Manager, and DNS zones) deploy to "global," and forgetting it blocks them.
- This is the technical answer to the carrier's data-residency requirement. "We have a policy that denies it" is a much stronger statement than "we told people not to."

**Test:** try to create a storage account in **UK South**. It should fail with a `RequestDisallowedByPolicy` error naming your assignment.

### Task 2 – Allowed VM sizes (R2)

First, find out what's actually available (lesson from Project 01):

```bash
az vm list-skus --location eastus --resource-type virtualMachines \
  --query "[?length(restrictions)==\`0\`].name" -o tsv | grep -E "Standard_(B|D2)"
```

Then assign **Allowed virtual machine size SKUs** with a short list of small sizes from that output.

**Why it matters:**
- This directly answers Owen's "expensive server size" complaint. A single large GPU VM can cost more per day than the firm's whole monthly budget.
- **Testing is free.** A denied deployment is rejected during validation, before anything is built. Try to create a VM with a large size like `Standard_D64s_v5` and screenshot the denial. You never pay for it.

### Task 3 – Require a Department tag on resource groups (R3)

Assign **Require a tag on resource groups** with tag name `Department`.

**Why it matters:**
- Owen can only report spending by department if every cost carries a `Department` tag. Requiring it at creation is cheaper than chasing it later.
- Note the scope: this requires the tag on **resource groups only**. Requiring tags on every single resource tends to break deployments (many resources are created automatically by other services). Task 4 handles resources a smarter way.

**Test:** `az group create --name rg-test-notag --location eastus` should be **denied**. Then add `--tags Department=IT` and it should succeed. Delete the test group afterward.

### Task 4 – Inherit the tag from the resource group (R4)

Assign **Inherit a tag from the resource group if missing** with tag name `Department`.

This policy uses the **Modify** effect, so the assignment needs a **managed identity** with permission to write tags. The portal creates one for you on the **Remediation** tab. Check the box to **create a remediation task** so existing resources are fixed too.

**Why it matters:**
- **Effects decide what a policy does:** *Deny* blocks a request, *Audit* only reports it, *Modify* changes the request (adds the tag) before the resource is created, and *DeployIfNotExists* deploys something extra. Knowing which effect to use is a core governance skill.
- **Policies only act on new and updated resources automatically.** Resources that existed before the assignment stay non-compliant until you run a **remediation task**. People forget this constantly.
- Resource group tags are **not** inherited by default. This is the fix promised in Project 01.

**Test:** create a small resource (a storage account is quick and nearly free; delete it after) in `rg-litigation-prod-01` **without** tags. Check its tags afterward: `Department=Litigation` should be there.

### Task 5 – Bundle it into an initiative (R5)

Once Tasks 1–4 work, rebuild them properly: create an **initiative** (policy set) called `Magnolia Bay Baseline` containing all five policies, assign the initiative at the subscription, and remove the individual assignments.

**Why it matters:**
- Five separate assignments become one thing to assign, track, and report on. Compliance shows as one score.
- It's how this scales. A second subscription later just gets the same initiative.
- Use **parameters** in the initiative (allowed locations, allowed sizes, tag name) so you can reuse it without editing the definition.

**Bonus (recommended):** export the initiative definition to JSON and commit it to `policy-definitions/` in this project folder, the same way you did with `vm-operator.json`.

### Task 6 – Resource locks (R6)

Apply a **Delete** (CanNotDelete) lock to `rg-identity-prod-01` and `rg-litigation-prod-01`.

**Test:** as your Owner account, try to delete `rg-litigation-prod-01`. It should fail.

**Why it matters:**
- **Locks override RBAC permissions.** Even an Owner can't delete a locked resource without removing the lock first. That deliberate extra step is the protection Diane asked for.
- **Two lock types:** *Delete* allows changes but blocks deletion. *Read-only* blocks changes **and** deletion, and it has side effects people don't expect (see ticket T3).
- Removing a lock requires `Microsoft.Authorization/locks/delete`, which comes with Owner and User Access Administrator, **not** Contributor. So your admin group can't remove locks either.

### Task 7 – Budget and alerts (R7)

As your admin account, create a **monthly budget** at the subscription scope (start with something small, like $10) with these alert conditions:

| Type | Threshold | Notify |
|---|---|---|
| Actual | 50% | Owen, IT |
| Actual | 80% | Owen, IT |
| Actual | 100% | Owen, IT, Diane |
| Forecasted | 100% | Owen, IT |

Create an **action group** named `ag-cost-alerts` for the notifications, so the recipient list lives in one place.

**Why it matters:**
- **Budgets alert; they don't stop spending.** Azure won't shut anything off when a budget is exceeded. If Diane asks "will this cap our bill?", the honest answer is no, and you should say so. (Spending limits only exist on certain free/credit offers.)
- **Forecasted alerts** give the "before, not after" warning Diane wants. They fire when Azure predicts you'll exceed the budget by month-end.
- Owen has **Cost Management Reader**, so he can see budgets but can't create them. That's why IT builds it and Owen receives the alerts. To let Finance manage budgets themselves later, the role would be Cost Management Contributor, not Contributor.
- Cost data can lag by 8–24 hours, so alerts are not real-time.

### Task 8 – An exemption (R8)

The firm's e-discovery vendor says their appliance must be deployed in **West US 2** for a specific case.

Create a **policy exemption** on a new resource group (`rg-ediscovery-lab-01`) for the allowed-locations policy, with:
- Category: **Waiver**
- An **expiration date** (for example, 30 days)
- A description explaining the business reason and who approved it

**Why it matters:**
- Real environments always need exceptions. The wrong way is weakening or deleting the policy; the right way is a narrowly scoped, documented, **expiring** exemption.
- *Waiver* means "we accept the risk." *Mitigated* means "the requirement is met another way." Auditors care about the difference.
- The expiration date means the exception closes itself if everyone forgets about it.

---

## 5. Validation

| Test | Expected result | Pass/Fail |
|---|---|---|
| Create a storage account in UK South | Denied (`RequestDisallowedByPolicy`) | |
| Create a VM with a large size | Denied during validation, no resources created | |
| Create a resource group with no `Department` tag | Denied | |
| Create a resource with no tags in `rg-litigation-prod-01` | Created with `Department=Litigation` added | |
| Existing untagged resources after remediation | Tagged; compliant | |
| Initiative compliance dashboard | One assignment, one compliance score | |
| Owner deletes `rg-litigation-prod-01` | Denied by the lock | |
| Owen opens Budgets | Sees the budget, can't edit it | |
| Deploy to West US 2 in `rg-ediscovery-lab-01` | Allowed (exemption); elsewhere still denied | |

---

## 6. Troubleshooting Tickets

Work these after the build. Diagnose before opening the hints, and record your actual investigation in the Troubleshooting Log.

**T1 – "You said overseas regions were blocked, but I just created something in UK South."**
<details><summary>Hints</summary>

- When was the policy assigned, and when was the resource created?
- Is the assignment's **enforcement mode** set to *Enabled* or *Disabled*?
- Is the effect *Deny* or *Audit*? What does each one do with a non-compliant request?
- What does `az policy state trigger-scan` do, and what doesn't it do?
</details>

**T2 – "The compliance dashboard says 40% of our resources are non-compliant, but nothing is broken."**
<details><summary>Hints</summary>

- Were those resources created before or after the policy was assigned?
- What does a remediation task do, and which effects support it?
- Is non-compliance always a problem, or sometimes just a report?
</details>

**T3 – "Jordan can't restart the file server anymore."** *(after someone applies a Read-only lock to `rg-servers-prod-01`)*
<details><summary>Hints</summary>

- Did Jordan's role change? Check his access first.
- What kinds of operations does a Read-only lock block? Is restarting a VM a read?
- Hint inside the hint: start and restart are **POST** operations. How does Azure treat them under a Read-only lock?
- Which lock type would protect the server from deletion without breaking Jordan's job?
</details>

**T4 – "Owen says he never got the budget alert email."**
<details><summary>Hints</summary>

- Has spending actually reached a threshold? How delayed is cost data?
- Check the action group's email address, and whether the message went to spam. (Microsoft recommends allow-listing `azure-noreply@microsoft.com`.)
- Can you send a test notification from the action group?
</details>

---

## 7. Cleanup & Cost

- [ ] Delete any test storage accounts and resource groups (`rg-test-notag`, test resources)
- [ ] Remove the West US 2 test resource; let the exemption expire or delete it
- [ ] **Keep** the initiative, locks, and budget. Every later project runs under them
- [ ] Before deleting a locked resource group in the future, remove the lock first, and document that you did
- [ ] Record the actual cost: $____

> **Heads-up for future projects:** these guardrails now apply to everything you build. If a later deployment fails, check Policy and locks early in your troubleshooting. That's the point of them, and it's realistic.

---

## 8. My Build Notes *(fill in as you go)*

### What I built
<!-- Screenshots in ./images/. Redact subscription IDs, tenant IDs, object IDs, and emails. -->

### Why I made these choices
<!-- In your own words. -->

### Troubleshooting Log
| Issue | What I checked | Root cause | Fix |
|---|---|---|---|
| | | | |

### Lessons Learned

---

## Stretch Goals

- **Management groups:** create `mg-magnolia` → `mg-magnolia-prod` / `mg-magnolia-lab`, move the subscription under it, and assign the initiative at the management group instead. Explain why governance at the MG level scales better.
- **Custom policy:** write a policy that denies storage accounts with public blob access enabled, and commit the JSON.
- **Audit first, then deny:** reassign one policy with *Audit*, review the compliance report, then switch to *Deny*. This is how policies are rolled out safely in production.
- **Microsoft Defender for Cloud:** open the regulatory compliance view and compare it with your own initiative.
- **Budget action:** connect the budget's action group to an Automation runbook or Logic App that deallocates lab VMs at 100%.

## Interview Talking Points

- The difference between RBAC (*who* can do something) and Policy (*what* can be done, by anyone)
- Policy effects: Deny vs. Audit vs. Modify vs. DeployIfNotExists, and when I used each
- Why existing resources need a remediation task
- Why locks protect against Owners, and why I chose Delete over Read-only
- Why a budget doesn't cap spending, and how I'd explain that to a managing partner
- How I handled an exception without weakening the policy
