# Lab 2: User & Group Lifecycle with PowerShell

> **Type:** Big Project | **Time:** 3-4 hours | **SC-300 Domain:** Implement and manage user identities (20-25%)

---

## Scenario

IAM Lab Corp is growing fast. HR has sent you a list of 20 new employees starting next month. You need to create their accounts, set up the right groups, manage licenses, and build automation so this process is repeatable every month. You'll use both the Entra portal and PowerShell to demonstrate different approaches to bulk operations.

---

## SC-300 Exam Objectives Covered

- Create, configure, and manage users
- Create, configure, and manage groups (security, M365, dynamic, assigned)
- Manage custom security attributes
- Automate bulk operations using Entra admin center and PowerShell
- Manage device join and device registration in Microsoft Entra ID
- Assign, modify, and report on licenses

---

## Prerequisites

- Completed Lab 1 (tenant configured with user/group/device settings)
- Microsoft 365 E5 subscription with Global Administrator access
- PowerShell 7 with Microsoft Graph SDK installed
- Certificate-based authentication configured (from Lab 1 or previous project)

---

## Step 1: Create Users Manually via the Portal

**Where:** Entra ID → Users → + New user → Create new user

Create 3 users manually to understand the portal experience before automating:

| Field | User 1 | User 2 | User 3 |
|-------|--------|--------|--------|
| Display name | Zara Ahmed | Carlos Rivera | Mei Lin |
| UPN | zara.ahmed@yourdomain | carlos.rivera@yourdomain | mei.lin@yourdomain |
| First name | Zara | Carlos | Mei |
| Last name | Ahmed | Rivera | Lin |
| Department | Engineering | Sales | Finance |
| Job title | Cloud Engineer | Sales Director | Financial Controller |
| Usage location | US | US | US |

For each user:
1. Click **+ New user** → **Create new user**
2. Fill in all fields above
3. Set a temporary password (or auto-generate)
4. Click **Create**

**Why this matters:** You need to understand the manual process before automating it. In interviews, when you explain your automation, you should be able to contrast it with the manual approach and explain why automation is better.

**SC-300 exam tip:** Know all the user attributes available during creation, especially Usage Location (required for license assignment) and password policies.

---

## Step 2: Create Users in Bulk via the Portal (CSV Upload)

**Where:** Entra ID → Users → Bulk operations → Bulk create

1. Click **Bulk operations** → **Bulk create**
2. Click **Download** to get the CSV template
3. Open the CSV and fill in 5 additional users:

| Name [displayName] | User name [userPrincipalName] | Initial password | Block sign in | First name | Last name | Job title | Department | Usage location |
|---|---|---|---|---|---|---|---|---|
| Fatima Hassan | fatima.hassan@yourdomain | TempPass2026! | No | Fatima | Hassan | HR Specialist | Human Resources | US |
| James Okafor | james.okafor@yourdomain | TempPass2026! | No | James | Okafor | IT Support Lead | IT | US |
| Sofia Petrov | sofia.petrov@yourdomain | TempPass2026! | No | Sofia | Petrov | Marketing Manager | Marketing | US |
| Raj Gupta | raj.gupta@yourdomain | TempPass2026! | No | Raj | Gupta | DevOps Engineer | Engineering | US |
| Emma Wilson | emma.wilson@yourdomain | TempPass2026! | No | Emma | Wilson | Account Manager | Sales | US |

4. Save the CSV and upload it
5. Click **Submit**
6. Review the results — verify all 5 users were created

**Why this matters:** Bulk CSV upload is the portal-based approach to mass user creation. It works for one-off situations but isn't repeatable or auditable like PowerShell automation. Know both approaches for the exam.

**SC-300 exam tip:** The exam may ask about bulk operations — know that CSV upload supports create, invite, delete, and download. Know the required CSV columns.

---

## Step 3: Create Users in Bulk via PowerShell

Open PowerShell and connect to Graph:

```powershell
pwsh

$TenantId = "your-tenant ID"
$ClientId = "your-client ID"

$pfxPassword = Read-Host "Enter your PFX password" -AsSecureString

$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2(
    "$HOME/Documents/IAM-Projects/certs/IAM-Lab-GraphApp.pfx",
    $pfxPassword
)

Connect-MgGraph -ClientId $ClientId -TenantId $TenantId -Certificate $cert
```

Create a CSV file with 10 more users:

```powershell
#Create the ./data folder
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

Write the provisioning script:

```powershell
# ================================
# Bulk Create Users from CSV
# Domain: "Your domain"
# ================================

Import-Module Microsoft.Graph.Users

$Domain = "your domain"
$csvPath = "./data/BulkUsers.csv"

# Check if CSV file exists
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

Verify the users:

```powershell
Get-MgUser -All | Select-Object DisplayName, Department, JobTitle, UserPrincipalName | Sort-Object Department | Format-Table
```

**Why this matters:** PowerShell automation is repeatable, auditable, and scalable. The portal CSV upload works once — a PowerShell script works every Monday morning for the rest of the year. This is the key differentiator for the exam and for interviews.

### How to explain this in an interview:

*"I demonstrated three approaches to user creation: manual portal for individual accounts, CSV bulk upload for one-off batch operations, and PowerShell with Graph API for repeatable automation. The PowerShell approach is what I'd use in production because it's scriptable, includes error handling, supports dry-run mode, and generates audit logs."*

---

## Step 4: Create and Manage Different Group Types

**Where:** Entra ID → Groups → + New group

Create each type of group to understand the differences:

### 4a: Security Group (Assigned Membership)

1. **Group type:** Security
2. **Name:** `SG-Project-Alpha`
3. **Description:** `Members of Project Alpha cross-functional team`
4. **Membership type:** Assigned
5. Add 3-4 users manually as members
6. Click **Create**

### 4b: Security Group (Dynamic Membership)

1. **Group type:** Security
2. **Name:** `DG-Engineering-All`
3. **Description:** `All Engineering department employees — auto-populated`
4. **Membership type:** Dynamic User
5. Click **Add dynamic query**
6. **Rule:** `user.department -eq "Engineering"`
7. Click **Validate Rules** — select a known Engineering user and verify they match
8. Click **Save** → **Create**

Create additional dynamic groups:

| Group Name | Rule | Purpose |
|---|---|---|
| `DG-Contractors-All` | `user.employeeType -eq "Contractor"` | All contractors |
| `DG-Finance-All` | `user.department -eq "Finance"` | All Finance employees |
| `DG-Senior-Staff` | `user.jobTitle -contains "Senior" -or user.jobTitle -contains "Director" -or user.jobTitle -contains "Manager"` | All senior-level staff |

### 4c: Microsoft 365 Group

1. **Group type:** Microsoft 365
2. **Name:** `M365-Marketing-Team`
3. **Description:** `Marketing team collaboration group`
4. **Membership type:** Assigned
5. Set an **Owner** (your admin account)
6. Add Marketing department users as members
7. Click **Create**

This creates a shared mailbox, SharePoint site, and Teams channel automatically.

### 4d: Compare Group Types via PowerShell

```powershell
# List all groups with their type
Get-MgGroup -All | Select-Object DisplayName, GroupTypes, SecurityEnabled, MailEnabled, MembershipRule | Format-Table -AutoSize
```

**Why this matters:** Understanding when to use each group type is critical for SC-300. Security groups control access. M365 groups enable collaboration. Dynamic groups reduce manual management. Assigned groups give you direct control.

**SC-300 exam tip:** This is heavily tested. Know that:
- Security groups can be assigned OR dynamic
- M365 groups can be assigned OR dynamic
- Dynamic groups require Entra ID P1/P2
- M365 groups create shared resources (mailbox, SharePoint, Teams)
- Only M365 groups support expiration policies

---

## Step 5: Configure and Use Custom Security Attributes

**Where:** Entra ID → Custom security attributes

Custom security attributes let you add business-specific metadata to users that you can then use for filtering, reporting, and access control.

### 5a: Create an Attribute Set

1. Click **+ Add attribute set**
2. **Name:** `IAMAttributes`
3. **Description:** `Custom attributes for IAM governance`
4. Click **Add**

### 5b: Create Attributes

Click into `IAMAttributes` → **+ Add attribute**

| Attribute name | Type | Allow multiple values | Predefined values |
|---|---|---|---|
| `ClearanceLevel` | String | No | Public, Internal, Confidential, Restricted |
| `ProjectCode` | String | Yes | ALPHA, BETA, GAMMA, DELTA |
| `RiskRating` | String | No | Low, Medium, High |

### 5c: Assign Attributes to Users

1. Go to **Users** → select a user → **Custom security attributes**
2. Click **+ Add assignment**
3. Select `IAMAttributes` → `ClearanceLevel` → set value to `Confidential`
4. Repeat for several users with different values

### 5d: Query Users by Attribute with PowerShell
Before we can query users by attribute with PowerShell, we need to give Ms Graph permissions for custom attributes. 

1. Go to **API permissions** → select → **Add permissions**
2. Add the following permissions 'CustomSecAttributeAssignment.Read.All'
'CustomSecAttributeAssignment.ReadWrite.All'

```powershell
# Find all users with Confidential clearance
Get-MgUser -Filter "customSecurityAttributes/IAMAttributes/ClearanceLevel eq 'Confidential'" `
    -ConsistencyLevel eventual -CountVariable count `
    -Property DisplayName,Department,CustomSecurityAttributes | 
    Select-Object DisplayName, Department | Format-Table
```

**Why this matters:** Custom security attributes let you tag users with business-relevant data (clearance levels, project assignments, risk ratings) that isn't covered by standard directory attributes. This is useful for fine-grained access control and compliance reporting.

**SC-300 exam tip:** Know how to create attribute sets, define attributes with predefined values, assign them to users, and filter users by attribute values.

---

## Step 6: Manage Licenses with PowerShell

### 6a: View Available Licenses

```powershell
Get-MgSubscribedSku | Select-Object SkuPartNumber, ConsumedUnits, 
    @{N='TotalUnits';E={$_.PrepaidUnits.Enabled}} | Format-Table
```

### 6b: Assign Licenses to Users

```powershell
$sku = Get-MgSubscribedSku | Where-Object { $_.SkuPartNumber -eq "SPE_E5" }

# Assign E5 license to all users without a license
$unlicensedUsers = Get-MgUser -All -Property Id,DisplayName,AssignedLicenses | 
    Where-Object { $_.AssignedLicenses.Count -eq 0 -and $_.UserPrincipalName -notlike "*#EXT#*" }

foreach ($user in $unlicensedUsers) {
    try {
        Set-MgUserLicense -UserId $user.Id -AddLicenses @(@{SkuId = $sku.SkuId}) -RemoveLicenses @()
        Write-Host "[LICENSED] $($user.DisplayName)" -ForegroundColor Green
    } catch {
        Write-Host "[FAILED]  $($user.DisplayName): $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

### 6c: Generate a License Report

```powershell
Get-MgUser -All -Property DisplayName,Department,AssignedLicenses,UserPrincipalName | 
    Where-Object { $_.UserPrincipalName -notlike "*#EXT#*" } |
    Select-Object DisplayName, Department, 
        @{N='Licensed';E={if($_.AssignedLicenses.Count -gt 0){"Yes"}else{"No"}}},
        @{N='LicenseCount';E={$_.AssignedLicenses.Count}} |
    Sort-Object Licensed, Department | Format-Table

# Export to CSV
Get-MgUser -All -Property DisplayName,Department,AssignedLicenses,UserPrincipalName |
    Where-Object { $_.UserPrincipalName -notlike "*#EXT#*" } |
    Select-Object DisplayName, Department,
        @{N='Licensed';E={if($_.AssignedLicenses.Count -gt 0){"Yes"}else{"No"}}} |
    Export-Csv -Path "./reports/license-report.csv" -NoTypeInformation

Write-Host "License report exported" -ForegroundColor Green
```

**SC-300 exam tip:** Know how to assign licenses via the portal (Users → Licenses) and via PowerShell. Know that Usage Location must be set before assigning licenses. Know how group-based licensing works (assign licenses to a group instead of individual users).

---

## Step 7: Implement Group-Based Licensing

Instead of assigning licenses to individual users, assign them to a group:

1. Go to **Entra ID** → **Groups** → select `DG-Engineering-All`
2. Click **Licenses** → **+ Assignments**
3. Select **Microsoft 365 E5**
4. Click **Save**

Now every user who is dynamically added to the Engineering group automatically gets an E5 license. When they leave Engineering, the license is automatically removed.

**Why this matters:** Group-based licensing is the production standard. It ensures licensing follows organizational structure and removes the manual step of assigning licenses per user.

**SC-300 exam tip:** Group-based licensing requires Entra ID P1 or higher. Know how license conflicts are handled when a user is in multiple groups with different licenses.

---

## Step 8: Verify Everything

Checklist:

- [ ] 3 users created manually via portal
- [ ] 5 users created via CSV bulk upload
- [ ] 10 users created via PowerShell script
- [ ] Assigned security group created (SG-Project-Alpha)
- [ ] Dynamic security groups created and membership verified (DG-Engineering-All, DG-Contractors-All, DG-Finance-All, DG-Senior-Staff)
- [ ] Microsoft 365 group created (M365-Marketing-Team)
- [ ] Custom security attribute set created (IAMAttributes)
- [ ] Attributes defined (ClearanceLevel, ProjectCode, RiskRating)
- [ ] Attributes assigned to users and queried via PowerShell
- [ ] Licenses assigned via PowerShell
- [ ] License report generated and exported
- [ ] Group-based licensing configured for Engineering

Take screenshots of dynamic group membership, custom attributes, and license assignments.

---

## Key Takeaways for SC-300

1. **Three ways to create users:** Portal (individual), CSV bulk upload (batch), PowerShell (automation) — know when to use each
2. **Four group types:** Assigned security, dynamic security, assigned M365, dynamic M365 — know the differences and use cases
3. **Dynamic groups** auto-manage membership based on user attributes — require P1/P2
4. **Custom security attributes** extend the directory schema for business-specific metadata
5. **Group-based licensing** automates license assignment and is the production standard
6. **Usage Location** must be set before licenses can be assigned

---

## What's Next

➡️ **Lab 3:** [External Identities & Cross-Tenant Access](../lab03-external-identities/) — B2B collaboration, guest users, cross-tenant policies, and external identity providers

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
