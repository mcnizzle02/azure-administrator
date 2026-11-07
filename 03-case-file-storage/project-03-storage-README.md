# Project 03 – Case File Storage: Blob, Files, Data Protection & Secure Sharing

**Domain:** Implement and manage storage (AZ-104)
**Company:** [Magnolia Bay Legal Group](../00-company-scenario/README.md)
**Business priorities addressed:** #1 Client confidentiality · #3 Never lose a client file · #4 Don't surprise us with the bill
**Builds on:** [Project 01](../01-identity-governance/README.md) (groups for data access) and [Project 02](../02-governance/README.md) (every resource here runs under the `MB-Baseline` policy, locks, and budget)
**Estimated cost:** under $1 if you keep test data small and clean up the file share

---

## 1. The Ticket

> **From:** Tasha Grand, Senior Paralegal
> **CC:** Diane Fairhope, Managing Partner
> **To:** IT
> **Subject:** The file server situation
>
> The old file server is almost full again, and honestly I don't trust it. A few things the litigation team needs:
>
> - A place in Azure for case files. Paralegals need to add and update files. Attorneys need to read them. Billing and the front office don't need to see them at all.
> - Last week someone deleted a whole folder of discovery documents and we had to rebuild it from email. That can't happen again.
> - We have to keep closed case files for **7 years**, but nobody needs to open them after the first few months. Can we pay less for old files?
> - We're under a **litigation hold** on the Henderson matter. Nothing in that case can be changed or deleted until the hold is lifted, by anyone.
> - Co-counsel at Whitfield & Ames needs to download **one exhibit** this week. I don't want to give them access to everything.
>
> **Gina (Office Manager) added:** The front office still needs a regular shared drive for forms and templates.

---

## 2. Requirements & Success Criteria

| ID | Requirement | Done when... |
|---|---|---|
| R1 | A secure storage account for case files | HTTPS only, TLS 1.2+, no anonymous public access, Department tag inherited |
| R2 | Access by job: paralegals write, attorneys read, everyone else nothing | Data-plane RBAC on containers, verified by signing in as each person |
| R3 | Existing case files moved in efficiently | Files uploaded with AzCopy using your Entra sign-in, not an account key |
| R4 | Deleted or overwritten files can be recovered | Soft delete and versioning on; a deleted file and an overwritten file both restored |
| R5 | Old files cost less, and files are deleted after 7 years | Lifecycle management rule; archive tier tested |
| R6 | Henderson files can't be changed or deleted during the hold | Legal hold on the container; delete and overwrite both fail, even for you |
| R7 | Co-counsel can download one exhibit, for a limited time | Read-only user delegation SAS for one blob; revoked when done |
| R8 | A shared drive for the front office | Azure Files share with a snapshot; a deleted file restored from it |

---

## 3. Environment

| Item | Value |
|---|---|
| Resource group | `rg-casefiles-prod-01` (East US, `Department=Litigation`) |
| Storage account | `stmbcasefiles<unique>` (lowercase letters and numbers only, globally unique) |
| Containers | `case-files`, `exhibits-henderson` |
| File share | `office-shared` |
| Access groups | `SG-Paralegals-All` (write), `SG-Litigation-Team` (read). Both from Project 01 |
| Guardrails in effect | `MB-Baseline` (regions, tags), budget alerts. Add a Delete lock at the end. |

---

## 4. Build Tasks

### Task 1 – Create a secure storage account (R1)

Create `rg-casefiles-prod-01` and a **StorageV2, Standard** account in it with:
- **Redundancy:** your choice of LRS, ZRS, or GRS. Write down why, with the price difference.
- **Secure transfer required** (HTTPS only), **minimum TLS 1.2**
- **Allow Blob anonymous access: Disabled**
- **Default to Microsoft Entra authorization in the Azure portal: Enabled**

**Why it matters:**
- Redundancy is a cost-versus-risk decision. LRS keeps 3 copies in one datacenter; ZRS spreads them across zones; GRS adds a copy hundreds of miles away. For a law firm, "what happens if East US has a regional outage?" is a fair question from the carrier.
- GRS replicates to **West US**, the paired region. Your location policy doesn't block it, because the secondary copy isn't a resource you deploy. It's still in the U.S., so data residency holds. Know this when someone asks.
- Anonymous public access is how most cloud data leaks happen. Turning it off at the account level means no one can accidentally make a container public.

**Check:** the storage account inherited `Department=Litigation` from Project 02's policy without you adding it.

### Task 2 – Containers and data-plane access (R2)

Create containers `case-files` and `exhibits-henderson`. Assign at the **container** scope:
- `SG-Paralegals-All` → **Storage Blob Data Contributor** on `case-files`
- `SG-Litigation-Team` → **Storage Blob Data Reader** on both containers

**Why it matters:**
- **Management plane vs. data plane.** Marcus already has *Reader* on `rg-litigation-prod-01`. Reader lets him see that a storage account exists, but not read a single file. Reading data needs a **Storage Blob Data** role. This is one of the most common AZ-104 questions, and one of the most common real-world tickets.
- Scoping to the container, not the account, means a future container for another department doesn't automatically inherit litigation access.
- **Owner doesn't include data access** either. Even your admin account needs a data role (or the account key) to read blobs. Test it.

**Test:** sign in as Tasha (upload works), Marcus (read works, upload fails), and Owen (sees nothing).

### Task 3 – Upload case files with AzCopy (R3)

Create a small folder of fake case files (text files are fine) and upload them to `case-files` using **AzCopy** with `azcopy login` (Entra ID), not an account key or SAS.

**Why it matters:**
- AzCopy is how large migrations are actually done: it's resumable, parallel, and scriptable. `azcopy sync` only copies what changed.
- Signing in with Entra means the upload is logged under your identity and governed by RBAC. An account key is a password to the whole account that can't be traced to a person.

### Task 4 – Soft delete and versioning (R4)

Turn on **blob soft delete** (14 days), **container soft delete** (14 days), and **blob versioning**. Then:
1. Delete a file and restore it.
2. Overwrite a file with different content and restore the previous version.

**Why it matters:**
- This is the direct answer to "someone deleted a folder of discovery documents."
- Soft delete protects against **deletion**; versioning protects against **overwrites** (and ransomware encrypting files in place). You need both.
- Neither one is a backup. They live in the same account. If the account itself is deleted, they go with it. That's why Task 8's lock matters.

### Task 5 – Access tiers and lifecycle management (R5)

Create a **lifecycle management** rule for `case-files`:
- Move to **Cool** 30 days after last modification
- Move to **Archive** 180 days after last modification
- **Delete** 2,555 days (7 years) after last modification

You can't wait 30 days, so also **manually** set one test file to **Archive** and try to open it.

**Why it matters:**
- Hot, Cool, Cold, and Archive trade **storage cost** against **access cost**. Archive is the cheapest to store and the most expensive and slowest to read.
- An archived blob is **offline**. It must be **rehydrated** before it can be read, which takes hours. If Tasha needs a 2019 deposition tomorrow morning, that's a real business problem.
- Cool, Cold, and Archive have **minimum storage durations** and early-deletion charges. Moving data around carelessly can cost more than leaving it in Hot.

### Task 6 – Litigation hold (R6)

Apply a **legal hold** to `exhibits-henderson` with the tag `henderson`. Upload a file, then try to **delete** it and **overwrite** it, as your Owner account.

**Why it matters:**
- This is **immutable storage (WORM: write once, read many)**. Under a legal hold, nobody can modify or delete the data, not even an Owner, until the hold is cleared. It's the storage equivalent of a resource lock, but for the data itself.
- Legal hold has **no expiration**. It stays until someone deliberately removes it, which matches how real litigation holds work.
- ⚠️ **Don't create a *locked* time-based retention policy in this lab.** A locked policy can't be shortened or removed, and you won't be able to delete the data or the account until it expires. An *unlocked* policy or a legal hold is safe to practice with.

### Task 7 – Share one exhibit with co-counsel (R7)

Generate a **user delegation SAS** for **one blob** in `exhibits-henderson`: read-only, HTTPS only, expiring in 48 hours. Open it in an InPrivate window. Then **revoke** it before it expires.

**Why it matters:**
- In Project 01, the co-counsel guest got Reader on a resource group. That's management-plane access to a whole RG; a SAS is data access to **one file**, with no account needed. Know when to use each.
- A **user delegation SAS** is signed with your Entra credentials, not the account key. It's logged under your identity, and it can be revoked without rotating keys that every other app depends on.
- A SAS is a bearer token: **anyone with the link has access** until it expires. Short expiry, smallest scope, read-only.

### Task 8 – Azure Files for the front office, and protect the account (R8)

Create a file share `office-shared` (small quota, e.g., 10 GiB). Upload a few "templates," take a **share snapshot**, delete a file, and restore it from the snapshot. Try mounting it from Windows.

Then apply a **Delete lock** to `rg-casefiles-prod-01`.

**Why it matters:**
- Azure Files is SMB, a real mapped drive, so the front office doesn't need to learn anything new. Blob storage is better for large volumes of documents accessed by apps or AzCopy.
- **Port 445:** SMB uses TCP 445, and many home ISPs block it. If the mount fails, that's probably why (ticket T5).
- Snapshots are point-in-time, read-only copies of the whole share. They're the Files equivalent of versioning.
- The lock protects the account that soft delete, versioning, and snapshots all live inside. Without it, one deletion takes all the protection with it.

---

## 5. Validation

| Test | Expected result | Pass/Fail |
|---|---|---|
| Storage account tags | `Department=Litigation` inherited | |
| Container set to public access | Not possible | |
| Tasha uploads to `case-files` | Succeeds | |
| Marcus reads / uploads | Read succeeds, upload fails | |
| Owen opens the storage account | No access | |
| Admin (Owner) reads a blob with Entra auth before a data role | Fails | |
| AzCopy upload with `azcopy login` | Succeeds | |
| Deleted file restored | Restored | |
| Overwritten file, previous version restored | Restored | |
| Read an archived blob | Fails until rehydrated | |
| Delete or overwrite a file under legal hold | Fails, even as Owner | |
| SAS link before / after revocation | Works / fails | |
| File restored from share snapshot | Restored | |
| Delete `rg-casefiles-prod-01` | `ScopeLocked` | |

---

## 6. Troubleshooting Tickets

**T1 – "I can see the storage account, but it says I'm not authorized to see the files."**
<details><summary>Hints</summary>

- Which role does the person have, and is it a management-plane or a data-plane role?
- Is the portal using Entra authorization or the access key? (Check "Authentication method" on the container.)
- Role assignments can take a few minutes to apply.
</details>

**T2 – "Co-counsel says the link doesn't work."**
<details><summary>Hints</summary>

- Has it expired? Was the **start time** set to "now," and could clock differences matter?
- Was it revoked? Does it include read permission? Is it for the right blob?
</details>

**T3 – "I can't open the deposition from 2019."**
<details><summary>Hints</summary>

- What tier is it in? What does rehydration cost, and how long does it take?
- Standard vs. High priority rehydration: which would you pick, and why?
</details>

**T4 – "I'm trying to delete an old exhibit and Azure won't let me."**
<details><summary>Hints</summary>

- Is there a legal hold or retention policy on the container? A resource lock on the RG?
- Who is allowed to clear a legal hold, and should IT do it without the attorney's approval?
</details>

**T5 – "The shared drive won't map."**
<details><summary>Hints</summary>

- `Test-NetConnection -ComputerName <account>.file.core.windows.net -Port 445`
- Is the ISP or network blocking 445? What are the alternatives (VPN, Azure File Sync, portal or Storage Explorer)?
</details>

---

## 7. Cleanup & Cost

- [ ] Clear the legal hold before deleting any test data under it
- [ ] Delete rehydration test blobs; keep test data tiny
- [ ] Delete the file share snapshots and share if you aren't keeping them
- [ ] **Keep** the storage account, containers, and lock if you'll reuse them in Project 04/05 (private endpoints)
- [ ] To delete the account later: remove the lock, clear legal holds, then delete
- [ ] Record the actual cost: $____

---

## Stretch Goals

- **Storage firewall:** allow access only from your public IP. Watch what breaks (Cloud Shell, the portal's Storage browser) and explain why.
- **Policy:** assign the built-in "Storage account public access should be disallowed" with Deny, and add it to `Magnolia Bay Baseline` as version 1.1.0.
- **Customer-managed keys:** encrypt the account with a key in Azure Key Vault. Explain what you gain and what you now have to manage.
- **Object replication** to a second account in Central US.
- **Identity-based access for Azure Files** (Microsoft Entra Kerberos), so the front office signs in with their own accounts instead of a key.
- **Microsoft Defender for Storage:** enable it and review what it detects.

## Interview Talking Points

- Management plane vs. data plane, and why Owner can't read blobs by default
- LRS vs. ZRS vs. GRS, and how I chose
- Soft delete vs. versioning vs. snapshots vs. backup
- Access tiers, rehydration, and early-deletion charges
- Legal hold vs. time-based retention, and why I never locked a policy in a lab
- Account key vs. service SAS vs. user delegation SAS
- Blob storage vs. Azure Files, and when I'd use each
