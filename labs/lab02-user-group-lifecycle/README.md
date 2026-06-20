# Lab 2: User & Group Lifecycle with PowerShell

> **Type:** Big Project | **Time:** 3–4 hours | **SC-300 Domain:** Implement and manage user identities (20–25%)

---

## Scenario

TeachRich is growing fast. HR has sent you a list of new employees starting next month. You need to create their accounts, set up the right groups, manage licenses, and build automation so this process is repeatable every month.

In this lab, you will use both the Microsoft Entra admin center and Microsoft Graph PowerShell to demonstrate different approaches to user and group lifecycle management.

---

## SC-300 Exam Objectives Covered

* Create, configure, and manage users
* Create, configure, and manage groups
* Manage assigned and dynamic group membership
* Manage Microsoft 365 groups and security groups
* Manage custom security attributes
* Automate bulk operations using Microsoft Entra admin center and PowerShell
* Assign, modify, and report on licenses
* Understand group-based licensing

---

## Prerequisites

Before starting this lab, you should have:

* Completed Lab 1, including tenant configuration and certificate-based authentication
* A Microsoft 365 E3, Microsoft 365 E5, or similar subscription with available test licenses
* Microsoft Entra ID P1 or P2 for dynamic groups and group-based licensing
* Global Administrator or Privileged Role Administrator access for initial setup
* PowerShell 7 installed
* Microsoft Graph PowerShell SDK installed
* Certificate-based authentication configured from Lab 1

> **Important:** Your license SKU may be different from the examples in this lab. Always run `Get-MgSubscribedSku` first and update `$skuPartNumber` to match a license available in your own tenant.

---

## Required Microsoft Graph API Permissions

If you are using certificate-based app-only authentication, your app registration needs Microsoft Graph **Application permissions**.

Go to:

```text
Microsoft Entra admin center
→ App registrations
→ Your app
→ API permissions
→ Add a permission
→ Microsoft Graph
→ Application permissions
```

Add the following permissions as needed:

```text
User.ReadWrite.All
Directory.ReadWrite.All
Group.ReadWrite.All
Organization.Read.All
LicenseAssignment.ReadWrite.All
CustomSecAttributeDefinition.ReadWrite.All
CustomSecAttributeAssignment.ReadWrite.All
```

Then click:

```text
Grant admin consent
```

For custom security attributes, the app/service principal may also need one of the following Microsoft Entra roles:

```text
Attribute Definition Administrator
Attribute Assignment Administrator
Attribute Assignment Reader
```

> **Troubleshooting note:** If you change API permissions or role assignments, disconnect and reconnect to Microsoft Graph before testing again.

> #### 📘 Why Application permissions (not Delegated) here
>
> This lab runs **app-only** (a certificate signs in as the app itself, with no user present), so it needs **Application permissions** — which always require admin consent. The `.All` suffix means tenant-wide scope: `User.ReadWrite.All` lets the app manage *every* user, not just one. That power is exactly why these need admin consent and why, in production, you'd grant the **minimum** set the automation actually uses rather than the whole list above.

---

## Step 1: Create Users Manually via the Portal

**Where:** Microsoft Entra admin center → Identity → Users → All users → New user → Create new user

Create 3 users manually to understand the portal experience before automating.

| Field          | User 1                                                          | User 2                                                                | User 3                                                    |
| -------------- | -------------------------------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------- |
| Display name   | Zara Ahmed                                                     | Carlos Rivera                                                        | Mei Lin                                                   |
| UPN            | [zara.ahmed@teachrich.com](mailto:zara.ahmed@teachrich.com)   | [carlos.rivera@teachrich.com](mailto:carlos.rivera@teachrich.com)   | [mei.lin@teachrich.com](mailto:mei.lin@teachrich.com)    |
| First name     | Zara                                                          | Carlos                                                              | Mei                                                      |
| Last name      | Ahmed                                                         | Rivera                                                             | Lin                                                     |
| Department     | Engineering                                                  | Sales                                                              | Finance                                                 |
| Job title      | Cloud Engineer                                               | Sales Director                                                     | Financial Controller                                    |
| Usage location | US                                                           | US                                                                | US                                                      |

For each user:

1. Click **New user**.
2. Select **Create new user**.
3. Fill in the required fields.
4. Set a temporary password or allow the portal to auto-generate one.
5. Set **Usage location** to `US`.
6. Click **Create**.

### Why this matters

You need to understand the manual process before automating it. In interviews, when you explain automation, you should be able to contrast it with the manual approach and explain why automation is better.

### SC-300 exam tip

Know the user attributes available during creation, especially:

* User principal name
* Display name
* Department
* Job title
* Usage location
* Password settings
* Account enabled/block sign-in settings

Usage location is especially important because it is required before assigning Microsoft 365 licenses.

---

## Step 2: Create Users in Bulk via the Portal

**Where:** Microsoft Entra admin center → Identity → Users → All users → Bulk operations → Bulk create

1. Click **Bulk operations**.
2. Select **Bulk create**.
3. Download the CSV template.
4. Open the CSV template.
5. Fill in 5 additional users.

Example users:

| Name [displayName] | User name [userPrincipalName]                                   | Initial password | Block sign in | First name | Last name | Job title         | Department      | Usage location |
| ------------------ | --------------------------------------------------------------- | ---------------- | ------------- | ---------- | --------- | ----------------- | --------------- | -------------- |
| Fatima Hassan      | [fatima.hassan@teachrich.com](mailto:fatima.hassan@teachrich.com) | TempPass2026!    | No            | Fatima     | Hassan    | HR Specialist     | Human Resources | US             |
| James Okafor       | [james.okafor@teachrich.com](mailto:james.okafor@teachrich.com)   | TempPass2026!    | No            | James      | Okafor    | IT Support Lead   | IT              | US             |
| Sofia Petrov       | [sofia.petrov@teachrich.com](mailto:sofia.petrov@teachrich.com)   | TempPass2026!    | No            | Sofia      | Petrov    | Marketing Manager | Marketing       | US             |
| Raj Gupta          | [raj.gupta@teachrich.com](mailto:raj.gupta@teachrich.com)         | TempPass2026!    | No            | Raj        | Gupta     | DevOps Engineer   | Engineering     | US             |
| Emma Wilson        | [emma.wilson@teachrich.com](mailto:emma.wilson@teachrich.com)     | TempPass2026!    | No            | Emma       | Wilson    | Account Manager   | Sales           | US             |

6. Save the CSV.
7. Upload the CSV.
8. Click **Submit**.
9. Review the results and verify that all 5 users were created.

### Why this matters

Bulk CSV upload is the portal-based approach to mass user creation. It works well for one-off batch operations, but it is less repeatable and less auditable than PowerShell automation.

### SC-300 exam tip

The exam may ask about bulk operations. Know that bulk user operations can be done through CSV templates in the Microsoft Entra admin center.

---

## Step 3: Create Users in Bulk via PowerShell

In this step, you will create users from a CSV file using Microsoft Graph PowerShell.

### 3a: Open PowerShell 7

On macOS or Linux, open Terminal and run:

```bash
pwsh
```

On Windows, open PowerShell 7 directly.

Do not type `pwsh` again if you are already inside PowerShell.

---

### 3b: Connect to Microsoft Graph

Replace the Tenant ID and Client ID with your own values.

```powershell
$TenantId = "your-tenant-id"
$ClientId = "your-client-id"

$pfxPassword = Read-Host "Enter your PFX password" -AsSecureString

$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2(
    "$HOME/Documents/IAM-Projects/certs/IAM-Lab-GraphApp.pfx",
    $pfxPassword
)

Connect-MgGraph -ClientId $ClientId -TenantId $TenantId -Certificate $cert

Get-MgContext
```

If the connection is successful, `Get-MgContext` should show your tenant ID, client ID, and authentication context.

> #### 📘 Script explained
>
> **What it does:** Signs in to Microsoft Graph **as the app itself** (not as a person) using a certificate, so the rest of the script can run with no human and no password.
>
> **Why it matters:** This is the production automation pattern. A scheduled JML (Joiner-Mover-Leaver) job can't stop to ask a human to log in — it authenticates with a certificate it holds. Showing certificate-based app-only auth (instead of an interactive `Connect-MgGraph` pop-up) signals to a reviewer that you understand unattended, secure automation.
>
> **Line by line:**
> - `$TenantId` / `$ClientId` — identify *which* tenant and *which* app registration you're signing in as.
> - `Read-Host ... -AsSecureString` — prompts for the certificate's password and keeps it as an encrypted in-memory string, so the password is never written into the script or shown on screen.
> - `New-Object ...X509Certificate2(...)` — loads the certificate (including its private key) from the `.pfx` file into memory.
> - `Connect-MgGraph -Certificate $cert` — authenticates using that certificate. The app proves who it is by holding the private key — no shared secret travels over the wire.
> - `Get-MgContext` — confirms the connection and shows the tenant, app, and permission scopes you're now operating with.
>
> **Watch out for:** Prompting for the PFX password (rather than hard-coding it) is the right habit — keep it that way. Never commit the `.pfx` file to source control, and in production store the certificate in a machine certificate store or Azure Key Vault, not a `Documents` folder.

---

### 3c: Create the CSV file

Create a `data` folder and generate a CSV file with 10 users.

```powershell
New-Item -ItemType Directory -Path ./data -Force | Out-Null

@"
FirstName,LastName,Department,JobTitle,EmployeeType
Liam,Cooper,Engineering,Backend Developer,Employee
Amara,Diallo,Engineering,QA Engineer,Employee
Noah,Bennett,Sales,Business Development Rep,Employee
Yuki,Tanaka,Finance,Accounts Payable Specialist,Employee
Olivia,Moreau,Marketing,Brand Strategist,Employee
Hassan,Ali,IT,Network Engineer,Employee
Chloe,Dubois,Human Resources,Recruiting Coordinator,Employee
Daniel,Kim,Engineering,Frontend Developer,Contractor
Priya,Sharma,Finance,Audit Analyst,Employee
Lucas,Garcia,Sales,Enterprise Account Executive,Employee
"@ | Out-File -FilePath ./data/BulkUsers.csv -Encoding UTF8
```

Check that the file was created:

```powershell
Test-Path ./data/BulkUsers.csv
```

It should return:

```text
True
```

You can also preview the file:

```powershell
Import-Csv ./data/BulkUsers.csv | Format-Table
```

> #### 📘 Script explained
>
> **What it does:** Creates a `./data` folder and writes a 10-row CSV of sample employees to disk, then confirms and previews it.
>
> **Why it matters:** It separates **data** (the list of people) from **logic** (the script that creates them). That's a real engineering habit — next month HR hands you a new CSV and the *same* script processes it, untouched. Reviewers read that as "thinks in repeatable pipelines, not one-off commands."
>
> **Line by line:**
> - `New-Item -ItemType Directory -Path ./data -Force | Out-Null` — creates the folder. `-Force` stops it erroring if the folder already exists; `| Out-Null` hides the confirmation output.
> - `@"` … `"@` — a **here-string**: a multi-line block of literal text. Here it holds the CSV contents (a header row plus ten data rows).
> - `| Out-File -FilePath ./data/BulkUsers.csv -Encoding UTF8` — writes that text to the CSV file. UTF-8 encoding avoids character-corruption issues with names.
> - `Test-Path` — returns `True` if the file now exists.
> - `Import-Csv | Format-Table` — reads the CSV back and prints it as a table so you can eyeball it before using it.
>
> **Watch out for:** In real life the CSV comes *from HR*, not hard-coded in the script. This inline version is just to make the lab self-contained.

---

### 3d: Create users from the CSV

Replace the domain with your verified Microsoft Entra domain.

Example:

```powershell
$Domain = "teachrich.com"
```

Full script:

```powershell
# ================================
# Bulk Create Users from CSV
# ================================

Import-Module Microsoft.Graph.Users

$Domain = "teachrich.com" # Replace with your verified Entra ID domain
$csvPath = "./data/BulkUsers.csv"

if (-not (Test-Path $csvPath)) {
    Write-Host "CSV file not found at $csvPath" -ForegroundColor Red
    return
}

$users = Import-Csv $csvPath

foreach ($user in $users) {

    $firstName = $user.FirstName.Trim()
    $lastName  = $user.LastName.Trim()

    $upn = "$($firstName.ToLower()).$($lastName.ToLower())@$Domain"
    $mailNickname = "$($firstName.ToLower()).$($lastName.ToLower())"

    Write-Host "`nChecking user: $upn" -ForegroundColor Cyan

    try {
        $existing = Get-MgUser -Filter "userPrincipalName eq '$upn'" -ErrorAction SilentlyContinue

        if (-not $existing) {

            $params = @{
                AccountEnabled    = $true
                DisplayName       = "$firstName $lastName"
                GivenName         = $firstName
                Surname           = $lastName
                UserPrincipalName = $upn
                MailNickname      = $mailNickname
                Department        = $user.Department
                JobTitle          = $user.JobTitle
                EmployeeType      = $user.EmployeeType
                UsageLocation     = "US"
                PasswordProfile   = @{
                    Password = "Welcome$((Get-Random -Minimum 10000 -Maximum 99999))!"
                    ForceChangePasswordNextSignIn = $true
                }
            }

            New-MgUser -BodyParameter $params -ErrorAction Stop | Out-Null

            Write-Host "[CREATED] $upn - $($user.Department)" -ForegroundColor Green
        }
        else {
            Write-Host "[EXISTS]  $upn" -ForegroundColor Yellow
        }
    }
    catch {
        Write-Host "[FAILED]  $upn" -ForegroundColor Red
        Write-Host $_.Exception.Message -ForegroundColor Yellow
    }
}

Write-Host "`nBulk user creation process completed." -ForegroundColor Green
```

> #### 📘 Script explained
>
> **What it does:** Reads the CSV and creates each person in Entra ID — but skips anyone who already exists, and reports success, skip, or failure for every row.
>
> **Why it matters:** This is the *production-grade* version of "create users." It's **idempotent** (safe to run twice — it won't create duplicates), it has **error handling** (one bad row won't crash the whole batch), and it produces a **clear run log**. Those three properties are exactly what separates a throwaway script from something you'd trust in a real onboarding pipeline.
>
> **Line by line:**
> - `Import-Module Microsoft.Graph.Users` — loads the Graph commands for managing users.
> - `if (-not (Test-Path $csvPath)) { ... return }` — stops early with a clear message if the data file is missing.
> - `$users = Import-Csv $csvPath` — loads each CSV row as an object you can read by column name.
> - `foreach ($user in $users)` — runs the block once per person.
> - `.Trim()` — strips accidental spaces around the names so the generated username is clean.
> - `$upn` / `$mailNickname` — build the sign-in name as `first.last@teachrich.com`, forced lowercase for consistency.
> - `try { … } catch { … }` — wraps each creation so a single failure is logged and the loop keeps going.
> - `Get-MgUser -Filter "userPrincipalName eq '$upn'"` — checks whether this user already exists. **This check is what makes the script idempotent.**
> - `if (-not $existing) { … } else { [EXISTS] }` — only creates when the user isn't found; otherwise logs a skip.
> - `$params = @{ … }` — a hashtable of all the new user's attributes. `PasswordProfile` is a *nested* hashtable holding a randomised starting password and `ForceChangePasswordNextSignIn = $true`.
> - `New-MgUser -BodyParameter $params` — creates the account.
> - The colour-coded `Write-Host` lines (green/yellow/red) give you an at-a-glance report of what happened to each row.
>
> **Watch out for:** `Get-Random` for the password is fine for a lab, but in production use a cryptographically secure generator and deliver the password through a secure channel — or skip starting passwords entirely with a **Temporary Access Pass** (Lab 5). Also, `UsageLocation` is hard-coded to `"US"`; set it to where each user actually is, because it drives license eligibility and data-residency rules.

---

### 3e: Verify the users

```powershell
Get-MgUser -All `
    -Property "displayName,department,jobTitle,userPrincipalName" |
Select-Object DisplayName, Department, JobTitle, UserPrincipalName |
Sort-Object Department |
Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Lists every user with their department, title, and UPN, grouped by department.
>
> **Line by line:** `Get-MgUser -All -Property "..."` fetches all users — and you must name `department` and `jobTitle` explicitly, because Graph returns only a minimal set of fields unless you ask for more. `Select-Object` keeps the columns you want, `Sort-Object Department` groups them, and `Format-Table -AutoSize` prints a tidy grid.

### Why this matters

PowerShell automation is repeatable, auditable, and scalable. The portal CSV upload works for one batch, but a PowerShell script can be reused every month.

### How to explain this in an interview

> I demonstrated three approaches to user creation: manual portal creation for individual accounts, CSV bulk upload for one-off batch operations, and Microsoft Graph PowerShell for repeatable automation. The PowerShell approach is what I would use in production because it is scriptable, consistent, auditable, and can include error handling and reporting.

---

## Step 4: Create and Manage Different Group Types

**Where:** Microsoft Entra admin center → Identity → Groups → All groups → New group

Create different group types to understand the differences.

---

### 4a: Security Group with Assigned Membership

1. **Group type:** Security
2. **Group name:** `SG-Project-Alpha`
3. **Description:** `Members of Project Alpha cross-functional team`
4. **Membership type:** Assigned
5. Add 3–4 users manually as members.
6. Click **Create**.

Assigned groups are useful when you want direct control over membership.

---

### 4b: Security Group with Dynamic Membership

Create a dynamic group for Engineering users.

1. **Group type:** Security
2. **Group name:** `DG-Engineering-All`
3. **Description:** `All Engineering department employees — auto-populated`
4. **Membership type:** Dynamic User
5. Click **Add dynamic query**.
6. Use this rule:

```text
user.department -eq "Engineering"
```

7. Click **Validate Rules** and test with a known Engineering user.
8. Click **Save**.
9. Click **Create**.

Create additional dynamic groups:

| Group Name           | Rule                                                                                                                  | Purpose                |
| -------------------- | --------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| `DG-Contractors-All` | `user.employeeType -eq "Contractor"`                                                                                  | All contractors        |
| `DG-Finance-All`     | `user.department -eq "Finance"`                                                                                       | All Finance employees  |
| `DG-Senior-Staff`    | `(user.jobTitle -contains "Senior") -or (user.jobTitle -contains "Director") -or (user.jobTitle -contains "Manager")` | All senior-level staff |

> #### 📘 Membership rules explained
>
> These rules are **dynamic membership queries** — Entra ID re-evaluates them automatically as user attributes change, so the group keeps itself current with no manual upkeep.
> - `user.department -eq "Engineering"` — include any user whose Department attribute *equals* "Engineering". The moment someone's department is set to Engineering, they're added; change it, they're removed.
> - `user.employeeType -eq "Contractor"` — the same idea on the EmployeeType attribute (which you set during creation in Step 3d).
> - The `DG-Senior-Staff` rule uses `-contains` (substring match) joined with `-or`, so anyone whose job title *contains* "Senior", "Director", or "Manager" qualifies. **Watch the operators:** `-eq` is an exact match; `-contains` matches part of the text — mixing them up is a common reason a dynamic group ends up empty or over-populated.

> **Note:** Dynamic group membership requires Microsoft Entra ID P1 or P2.

---

### 4c: Microsoft 365 Group

1. **Group type:** Microsoft 365
2. **Group name:** `M365-Marketing-Team`
3. **Description:** `Marketing team collaboration group`
4. **Membership type:** Assigned
5. Set an owner, such as your admin account.
6. Add Marketing department users as members.
7. Click **Create**.

A Microsoft 365 group creates shared Microsoft 365 resources such as a group mailbox/calendar and SharePoint site. A Microsoft Team can also be created from the group or added later.

---

### 4d: Compare Group Types via PowerShell

```powershell
Import-Module Microsoft.Graph.Groups

Get-MgGroup -All `
    -Property "displayName,groupTypes,securityEnabled,mailEnabled,membershipRule" |
Select-Object DisplayName, GroupTypes, SecurityEnabled, MailEnabled, MembershipRule |
Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Lists every group with the four properties that actually reveal *what kind* of group it is.
>
> **Why it matters:** Those four fields are exactly how you tell the types apart in practice — useful in an audit when you need to prove which groups are security vs collaboration, and which are dynamic.
>
> **Line by line:**
> - `groupTypes` — contains `Unified` for a Microsoft 365 group; empty for a plain security group. It also contains `DynamicMembership` when the group is rule-based.
> - `securityEnabled` / `mailEnabled` — a **security** group is security-enabled; a **Microsoft 365** group is mail-enabled. The combination tells you the type.
> - `membershipRule` — only populated for **dynamic** groups (it holds the rule text from 4b). If it's blank, the group is assigned.

### Why this matters

Understanding when to use each group type is critical for SC-300.

Security groups are commonly used for access control. Microsoft 365 groups are used for collaboration. Dynamic groups reduce manual membership management. Assigned groups give administrators direct control.

### SC-300 exam tip

Know that:

* Security groups can be assigned or dynamic.
* Microsoft 365 groups can be assigned or dynamic.
* Dynamic groups require Microsoft Entra ID P1 or P2.
* Microsoft 365 groups create collaboration resources.
* Group membership can be used for access, licensing, and policy targeting.

---

## Step 5: Configure and Use Custom Security Attributes

**Where:** Microsoft Entra admin center → Protection → Custom security attributes

Custom security attributes allow you to add business-specific metadata to users, such as clearance level, project code, or risk rating.

> **Important:** Global Administrator does not automatically have access to read, define, or assign custom security attributes. You may need Attribute Definition Administrator, Attribute Assignment Administrator, or Attribute Assignment Reader depending on the task.

---

### 5a: Create an Attribute Set

1. Go to **Custom security attributes**.
2. Click **Add attribute set**.
3. Use the following details:

```text
Name: IAMAttributes
Description: Custom attributes for IAM governance
```

4. Click **Add**.

---

### 5b: Create Attributes

Click into `IAMAttributes`, then click **Add attribute**.

Create the following attributes:

| Attribute name   | Type   | Allow multiple values | Predefined values                          |
| ---------------- | ------ | --------------------- | ------------------------------------------ |
| `ClearanceLevel` | String | No                    | Public, Internal, Confidential, Restricted |
| `ProjectCode`    | String | Yes                   | ALPHA, BETA, GAMMA, DELTA                  |
| `RiskRating`     | String | No                    | Low, Medium, High                          |

---

### 5c: Assign Attributes to Users

1. Go to **Users**.
2. Select a user.
3. Go to **Custom security attributes**.
4. Click **Add assignment**.
5. Select:

```text
Attribute set: IAMAttributes
Attribute: ClearanceLevel
Value: Confidential
```

6. Repeat for several users with different values.

---

### 5d: Required Permissions for Custom Security Attributes

Before querying custom security attributes with PowerShell, confirm that your app registration has the following Microsoft Graph **Application permissions**:

```text
CustomSecAttributeDefinition.ReadWrite.All
CustomSecAttributeAssignment.ReadWrite.All
```

Then click:

```text
Grant admin consent
```

The app or service principal may also need one of these roles:

```text
Attribute Assignment Reader
Attribute Assignment Administrator
```

---

### 5e: Query Users by Custom Security Attribute

Try this server-side query first:

```powershell
$count = 0

Get-MgUser `
    -Filter "customSecurityAttributes/IAMAttributes/ClearanceLevel eq 'Confidential'" `
    -ConsistencyLevel eventual `
    -CountVariable count `
    -Property "id,displayName,department,customSecurityAttributes" `
    -All |
Select-Object DisplayName, Department |
Format-Table -AutoSize

Write-Host "Total users found: $count" -ForegroundColor Green
```

If you receive an error such as:

```text
Invalid custom security attribute
```

or:

```text
Request_UnsupportedQuery
```

use the fallback method below.

> #### 📘 Script explained
>
> **What it does:** Asks Graph to return **only** the users whose `ClearanceLevel` attribute equals "Confidential" — filtering on the server rather than pulling everyone down first.
>
> **Why it matters:** Server-side filtering is far more efficient at scale. But custom-security-attribute filters are classed as **advanced queries**, which need two extra ingredients most people forget — and that's the whole teaching point of this step.
>
> **Line by line:**
> - `-Filter "customSecurityAttributes/IAMAttributes/ClearanceLevel eq 'Confidential'"` — the server-side filter, drilling into the nested attribute path (attribute set → attribute → value).
> - `-ConsistencyLevel eventual` **and** `-CountVariable count` — these two are **required together** for advanced queries on custom security attributes. Omit either and Graph rejects the request. (`$count` also gives you a running total.)
> - `-Property "...,customSecurityAttributes"` — you must explicitly request the attributes, or they come back empty.
>
> **Watch out for:** If the app lacks the attribute-read permission/role, or the attribute set/attribute name is mistyped, Graph throws `Request_UnsupportedQuery` or `Invalid custom security attribute`. When that happens, use the fallback in 5f.

---

### 5f: Fallback Query Method

This method retrieves users and filters locally in PowerShell.

```powershell
$results = @()

$users = Get-MgUser `
    -All `
    -Property "id,displayName,department,userPrincipalName,customSecurityAttributes"

foreach ($user in $users) {

    $attrs = $user.CustomSecurityAttributes.AdditionalProperties

    if ($attrs -and $attrs.ContainsKey("IAMAttributes")) {

        $iamAttrs = $attrs["IAMAttributes"]

        if ($iamAttrs["ClearanceLevel"] -eq "Confidential") {
            $results += [PSCustomObject]@{
                DisplayName       = $user.DisplayName
                UserPrincipalName = $user.UserPrincipalName
                Department        = $user.Department
                ClearanceLevel    = $iamAttrs["ClearanceLevel"]
            }
        }
    }
}

$results | Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Pulls **all** users with their custom attributes, then filters for "Confidential" *inside PowerShell* instead of on the server — a reliable fallback when the advanced query in 5e isn't available.
>
> **Why it matters:** It always works, even without the eventual-consistency query support — handy when permissions or query support are the blocker. The trade-off is it's slower at scale because it downloads everyone first.
>
> **Line by line:**
> - `$results = @()` — starts an empty array to collect matches.
> - `Get-MgUser -All -Property "...,customSecurityAttributes"` — fetch every user *with* their attributes.
> - `$user.CustomSecurityAttributes.AdditionalProperties` — custom attributes arrive as a nested property bag; this reaches into it.
> - `if ($attrs -and $attrs.ContainsKey("IAMAttributes"))` — guards against users who have **no** attributes set, so the next line doesn't error.
> - `if ($iamAttrs["ClearanceLevel"] -eq "Confidential")` — the actual local filter.
> - `$results += [PSCustomObject]@{ … }` — builds a clean, named output row for each match.
>
> **Watch out for:** `Get-MgUser -All` loads the whole directory into memory — fine for a lab, but on a large tenant prefer the server-side query (5e) once the permissions are sorted.

### Why this matters

Custom security attributes let you tag users with business-relevant data that is not covered by standard directory attributes. These attributes can support reporting, governance, access decisions, and compliance workflows.

### SC-300 exam tip

Know how to:

* Create an attribute set
* Define attributes
* Assign custom security attributes to users
* Query users based on attributes
* Identify the permissions and roles needed to manage custom security attributes

---

## Step 6: Manage Licenses with PowerShell

In this step, you will view available licenses and assign a license to unlicensed users.

---

### 6a: View Available Licenses

Run this first to see the SKU names available in your tenant:

```powershell
Import-Module Microsoft.Graph.Identity.DirectoryManagement

Get-MgSubscribedSku |
Select-Object SkuPartNumber, SkuId, ConsumedUnits, @{Name="Enabled";Expression={$_.PrepaidUnits.Enabled}} |
Format-Table -AutoSize
```

Example SKU names may include:

```text
SPE_E3
SPE_E5
AAD_PREMIUM_P2
SPB
```

> **Important:** Use the SKU that exists in your tenant. If your tenant shows `SPE_E3`, use `SPE_E3`. If your tenant shows `SPE_E5`, use `SPE_E5`.

> #### 📘 Script explained
>
> **What it does:** Lists every license SKU your tenant owns, showing how many seats are used (`ConsumedUnits`) versus purchased (`Enabled`).
>
> **Why it matters:** License SKU identifiers are **tenant-specific**, and you assign licenses by `SkuId`, not by friendly name. Running this *first* is what prevents the "SKU not found" error later — you copy the exact `SkuPartNumber`/`SkuId` your tenant actually has.
>
> **Line by line:** `Get-MgSubscribedSku` returns the licenses. The calculated property `@{Name="Enabled";Expression={$_.PrepaidUnits.Enabled}}` reaches into the nested `PrepaidUnits` object to surface the purchased-seat count as a simple column. (`SPE_E3` = Microsoft 365 E3, `SPE_E5` = E5, `AAD_PREMIUM_P2` = Entra ID P2.)

---

### 6b: Assign Licenses to Unlicensed Users

Before running this script, confirm that your app registration has the following Microsoft Graph **Application permission**:

```text
LicenseAssignment.ReadWrite.All
```

Then click:

```text
Grant admin consent
```

Use this script to assign a license to unlicensed internal users.

```powershell
# ================================
# Assign License to Unlicensed Users
# Handles UsageLocation delay
# ================================

Import-Module Microsoft.Graph.Users
Import-Module Microsoft.Graph.Identity.DirectoryManagement

$skuPartNumber = "SPE_E3" # Change this to match your tenant
$usageLocation = "US"

$sku = Get-MgSubscribedSku | Where-Object { $_.SkuPartNumber -eq $skuPartNumber }

if (-not $sku) {
    Write-Host "License SKU '$skuPartNumber' was not found." -ForegroundColor Red
    Write-Host "Available SKUs:" -ForegroundColor Yellow

    Get-MgSubscribedSku |
        Select-Object SkuPartNumber, SkuId, ConsumedUnits, @{Name="Enabled";Expression={$_.PrepaidUnits.Enabled}} |
        Format-Table -AutoSize

    return
}

Write-Host "Using license: $($sku.SkuPartNumber)" -ForegroundColor Green
Write-Host "Available licenses: $($sku.PrepaidUnits.Enabled - $sku.ConsumedUnits)" -ForegroundColor Cyan

$unlicensedUsers = Get-MgUser `
    -All `
    -Property "Id,DisplayName,UserPrincipalName,AssignedLicenses,UsageLocation" |
Where-Object {
    $_.AssignedLicenses.Count -eq 0 -and
    $_.UserPrincipalName -notlike "*#EXT#*"
}

Write-Host "Unlicensed users found: $($unlicensedUsers.Count)" -ForegroundColor Cyan

foreach ($user in $unlicensedUsers) {

    try {
        if ([string]::IsNullOrWhiteSpace($user.UsageLocation) -or $user.UsageLocation -ne $usageLocation) {

            Update-MgUser `
                -UserId $user.Id `
                -UsageLocation $usageLocation `
                -ErrorAction Stop

            Write-Host "[UPDATED USAGE LOCATION] $($user.DisplayName)" -ForegroundColor Cyan

            Start-Sleep -Seconds 8

            $user = Get-MgUser `
                -UserId $user.Id `
                -Property "Id,DisplayName,UserPrincipalName,AssignedLicenses,UsageLocation"
        }

        if ($user.UsageLocation -ne $usageLocation) {
            Write-Host "[SKIPPED] $($user.DisplayName) - UsageLocation is still not valid: $($user.UsageLocation)" -ForegroundColor Yellow
            continue
        }

        $addLicenses = @(
            @{
                SkuId = $sku.SkuId
            }
        )

        Set-MgUserLicense `
            -UserId $user.Id `
            -AddLicenses $addLicenses `
            -RemoveLicenses @() `
            -ErrorAction Stop | Out-Null

        Write-Host "[LICENSED] $($user.DisplayName) <$($user.UserPrincipalName)>" -ForegroundColor Green
    }
    catch {
        Write-Host "[FAILED] $($user.DisplayName) <$($user.UserPrincipalName)>" -ForegroundColor Red
        Write-Host $_.Exception.Message -ForegroundColor Yellow
    }
}

Write-Host "`nLicense assignment process completed." -ForegroundColor Green
```

> #### 📘 Script explained
>
> **What it does:** Finds every internal user with no license, makes sure their **UsageLocation** is set, then assigns the chosen license — logging the result for each user.
>
> **Why it matters:** License assignment has a real-world ordering trap: **you cannot assign a Microsoft 365 license until the user has a UsageLocation** (it's a legal/data-residency requirement). This script handles that dependency *and* the short delay before the change takes effect — which is exactly the kind of robustness an interviewer is listening for.
>
> **Line by line:**
> - `$sku = Get-MgSubscribedSku | Where-Object { $_.SkuPartNumber -eq $skuPartNumber }` — looks up the license object you want to assign.
> - `if (-not $sku) { … return }` — **fail-fast**: if that SKU doesn't exist in the tenant, it prints the available ones and stops, instead of erroring cryptically later.
> - `Get-MgUser -All ... | Where-Object { $_.AssignedLicenses.Count -eq 0 -and $_.UserPrincipalName -notlike "*#EXT#*" }` — selects users with **zero** licenses, and the `#EXT#` filter **excludes guest accounts** (guests carry `#EXT#` in their UPN).
> - `if ([string]::IsNullOrWhiteSpace($user.UsageLocation) ...)` — if UsageLocation is missing or wrong, `Update-MgUser` sets it.
> - `Start-Sleep -Seconds 8` then re-`Get-MgUser` — waits for the UsageLocation change to **propagate**, then re-reads the user so the next step sees the updated value.
> - `Set-MgUserLicense -AddLicenses @(@{SkuId = $sku.SkuId}) -RemoveLicenses @()` — assigns the license. `-AddLicenses` takes an array of hashtables; `-RemoveLicenses @()` means "remove nothing."
> - `try/catch` — logs a clear success or the exact error per user.
>
> **Watch out for:** The 8-second `Start-Sleep` is a pragmatic fix for replication delay — fine for a lab, but for large runs you'd use a retry/poll loop instead of a fixed wait. And direct license assignment is good to *understand*, but **group-based licensing (Step 7) is the scalable production pattern** — assign once to a group, and membership drives the licenses.

### Why this matters

License assignment is a common identity administration task. In production, this is often automated through group-based licensing, but it is important to understand direct license assignment first.

### SC-300 exam tip

Know that:

* Usage location must be set before assigning a license.
* License SKU names vary by tenant.
* Licenses can be assigned directly to users or through groups.
* Group-based licensing is preferred for scalable management.

---

## Step 7: Implement Group-Based Licensing

Instead of assigning licenses to individual users, assign licenses to a group.

1. Go to **Microsoft Entra admin center**.
2. Go to **Identity** → **Groups** → **All groups**.
3. Select `DG-Engineering-All`.
4. Click **Licenses**.
5. Click **Assignments**.
6. Select the same license SKU used in Step 6, such as Microsoft 365 E3 if your tenant uses `SPE_E3`.
7. Click **Save**.

Now every user dynamically added to the Engineering group can receive the assigned license. When users no longer match the Engineering rule, they can be removed from the group and the license assignment can be removed through group-based licensing.

### Why this matters

Group-based licensing is a production-friendly approach because licensing follows group membership rather than manual per-user assignment.

### SC-300 exam tip

Know that:

* Group-based licensing requires Microsoft Entra ID P1 or higher.
* Licenses can be inherited from group membership.
* Users can receive licenses from multiple groups.
* License assignment errors should be reviewed in the group licensing blade.

> #### 📘 Why this is the better pattern
>
> Notice how Steps 4b, 6, and 7 connect: the **dynamic group** `DG-Engineering-All` auto-populates from the `department` attribute, and group-based licensing assigns E3 to that group. The result is a **fully automated lifecycle** — create a user with Department = Engineering (Step 3d) and they're auto-added to the group *and* auto-licensed, with no per-user license step at all. That chain (attribute → dynamic group → inherited license) is the production answer to "how do you license at scale," and a strong thing to be able to narrate in an interview.

---

## Step 8: Export a License Report

Create a `reports` folder and export a simple license report.

```powershell
New-Item -ItemType Directory -Path ./reports -Force | Out-Null

Get-MgUser `
    -All `
    -Property "displayName,userPrincipalName,department,assignedLicenses,usageLocation" |
Select-Object `
    DisplayName,
    UserPrincipalName,
    Department,
    UsageLocation,
    @{Name="LicenseCount";Expression={$_.AssignedLicenses.Count}} |
Export-Csv -Path "./reports/license-report.csv" -NoTypeInformation

Write-Host "License report saved to ./reports/license-report.csv" -ForegroundColor Green
```

To view the report:

```powershell
Import-Csv ./reports/license-report.csv | Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Writes a CSV listing every user with their department, usage location, and how many licenses they hold.
>
> **Why it matters:** Reporting is evidence. A CSV you can hand to a manager or attach to an audit shows you don't just *make* changes, you *verify and document* them — a habit auditors and hiring managers both value.
>
> **Line by line:**
> - `New-Item ... -Force | Out-Null` — ensures the `reports` folder exists.
> - `Get-MgUser -All -Property "...,assignedLicenses,..."` — pulls everyone, explicitly requesting the license field.
> - `@{Name="LicenseCount";Expression={$_.AssignedLicenses.Count}}` — a **calculated property**: instead of dumping the raw license objects, it counts them into a simple number column.
> - `Export-Csv -NoTypeInformation` — writes the CSV. `-NoTypeInformation` omits the legacy type header line so the file is clean.

### Why this matters

Reports provide evidence of your work and show that you can validate and document identity administration tasks.

---

## Step 9: Verify Everything

Use this checklist to confirm completion:

* [ ] 3 users created manually via the portal
* [ ] 5 users created via portal CSV bulk upload
* [ ] 10 users created via PowerShell script
* [ ] Assigned security group created: `SG-Project-Alpha`
* [ ] Dynamic security group created: `DG-Engineering-All`
* [ ] Dynamic security group created: `DG-Contractors-All`
* [ ] Dynamic security group created: `DG-Finance-All`
* [ ] Dynamic security group created: `DG-Senior-Staff`
* [ ] Microsoft 365 group created: `M365-Marketing-Team`
* [ ] Custom security attribute set created: `IAMAttributes`
* [ ] Attributes defined: `ClearanceLevel`, `ProjectCode`, `RiskRating`
* [ ] Custom security attributes assigned to users
* [ ] Custom security attributes queried with PowerShell
* [ ] Licenses assigned with PowerShell
* [ ] Group-based licensing configured
* [ ] License report exported to `./reports/license-report.csv`

Recommended screenshots:

* Manual user creation result
* Bulk CSV upload result
* PowerShell-created users
* Dynamic group membership rule
* Dynamic group membership validation
* Microsoft 365 group overview
* Custom security attribute set
* Custom security attribute assignment on a user
* License assignment result
* License report CSV

---

## Troubleshooting

### Error: Authentication needed. Please call Connect-MgGraph.

You are not connected to Microsoft Graph.

Reconnect:

```powershell
Connect-MgGraph -ClientId $ClientId -TenantId $TenantId -Certificate $cert
```

Then confirm:

```powershell
Get-MgContext
```

---

### Error: Insufficient privileges to complete the operation

Your app registration is missing required Microsoft Graph Application permissions, admin consent, or an Entra role assignment.

Check:

```text
App registrations
→ Your app
→ API permissions
```

Confirm that the correct **Application permissions** are added and admin consent is granted.

For user creation, check:

```text
User.ReadWrite.All
Directory.ReadWrite.All
```

For group management, check:

```text
Group.ReadWrite.All
Directory.ReadWrite.All
```

For licenses, check:

```text
LicenseAssignment.ReadWrite.All
Organization.Read.All
```

For custom security attributes, check:

```text
CustomSecAttributeDefinition.ReadWrite.All
CustomSecAttributeAssignment.ReadWrite.All
```

---

### Error: License SKU was not found

Run:

```powershell
Get-MgSubscribedSku |
Select-Object SkuPartNumber, SkuId, ConsumedUnits, @{Name="Enabled";Expression={$_.PrepaidUnits.Enabled}} |
Format-Table -AutoSize
```

Then update:

```powershell
$skuPartNumber = "SPE_E3"
```

to match a SKU in your tenant.

---

### Error: License assignment cannot be done for user with invalid usage location

Set the usage location first:

```powershell
Update-MgUser -UserId user@teachrich.com -UsageLocation "US"
```

Then wait a few seconds and try assigning the license again.

---

### Error: Invalid custom security attribute

Check that:

* The attribute set name is correct.
* The attribute name is correct.
* The value is spelled exactly as defined.
* The app has custom security attribute permissions.
* The app/service principal has Attribute Assignment Reader or Attribute Assignment Administrator access.

You can use the fallback local filtering method in Step 5f.

---

## Key Takeaways for SC-300

1. There are multiple ways to create users: manual portal creation, CSV bulk upload, and PowerShell automation.
2. PowerShell automation is more repeatable and scalable than manual or one-time CSV operations.
3. Security groups are commonly used for access control.
4. Microsoft 365 groups are used for collaboration.
5. Dynamic groups reduce manual membership management and require Microsoft Entra ID P1 or P2.
6. Custom security attributes extend identity records with business-specific metadata.
7. Usage location must be set before assigning Microsoft 365 licenses.
8. Group-based licensing is preferred for scalable license management.
9. Reports and screenshots provide evidence for a portfolio or GitHub showcase.

---

## Portfolio / Interview Summary

In this lab, I demonstrated user and group lifecycle management in Microsoft Entra ID. I created users manually, through CSV bulk upload, and through Microsoft Graph PowerShell automation. I configured assigned and dynamic groups, created a Microsoft 365 collaboration group, worked with custom security attributes, assigned licenses, implemented group-based licensing, and exported a license report.

This project demonstrates practical SC-300 skills in identity administration, automation, governance, and reporting.

---

## What's Next

➡️ **Lab 3:** [External Identities & Cross-Tenant Access](../lab03-external-identities/) — B2B collaboration, guest users, cross-tenant policies, and external identity providers.

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
