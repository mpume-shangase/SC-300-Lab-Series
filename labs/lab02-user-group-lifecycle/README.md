# Lab 2: User & Group Lifecycle with PowerShell

> **Type:** Big Project | **Time:** 3–4 hours | **SC-300 Domain:** Implement and manage user identities (20–25%)

---

## Scenario

IAM Lab Corp is growing fast. HR has sent you a list of new employees starting next month. You need to create their accounts, set up the right groups, manage licenses, and build automation so this process is repeatable every month.

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

---

## Step 1: Create Users Manually via the Portal

**Where:** Microsoft Entra admin center → Identity → Users → All users → New user → Create new user

Create 3 users manually to understand the portal experience before automating.

| Field          | User 1                                                        | User 2                                                              | User 3                                                  |
| -------------- | ------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------- |
| Display name   | Zara Ahmed                                                    | Carlos Rivera                                                       | Mei Lin                                                 |
| UPN            | [zara.ahmed@yourdomain.com](mailto:zara.ahmed@yourdomain.com) | [carlos.rivera@yourdomain.com](mailto:carlos.rivera@yourdomain.com) | [mei.lin@yourdomain.com](mailto:mei.lin@yourdomain.com) |
| First name     | Zara                                                          | Carlos                                                              | Mei                                                     |
| Last name      | Ahmed                                                         | Rivera                                                              | Lin                                                     |
| Department     | Engineering                                                   | Sales                                                               | Finance                                                 |
| Job title      | Cloud Engineer                                                | Sales Director                                                      | Financial Controller                                    |
| Usage location | US                                                            | US                                                                  | US                                                      |

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

| Name [displayName] | User name [userPrincipalName]                                       | Initial password | Block sign in | First name | Last name | Job title         | Department      | Usage location |
| ------------------ | ------------------------------------------------------------------- | ---------------- | ------------- | ---------- | --------- | ----------------- | --------------- | -------------- |
| Fatima Hassan      | [fatima.hassan@yourdomain.com](mailto:fatima.hassan@yourdomain.com) | TempPass2026!    | No            | Fatima     | Hassan    | HR Specialist     | Human Resources | US             |
| James Okafor       | [james.okafor@yourdomain.com](mailto:james.okafor@yourdomain.com)   | TempPass2026!    | No            | James      | Okafor    | IT Support Lead   | IT              | US             |
| Sofia Petrov       | [sofia.petrov@yourdomain.com](mailto:sofia.petrov@yourdomain.com)   | TempPass2026!    | No            | Sofia      | Petrov    | Marketing Manager | Marketing       | US             |
| Raj Gupta          | [raj.gupta@yourdomain.com](mailto:raj.gupta@yourdomain.com)         | TempPass2026!    | No            | Raj        | Gupta     | DevOps Engineer   | Engineering     | US             |
| Emma Wilson        | [emma.wilson@yourdomain.com](mailto:emma.wilson@yourdomain.com)     | TempPass2026!    | No            | Emma       | Wilson    | Account Manager   | Sales           | US             |

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

---

### 3d: Create users from the CSV

Replace the domain with your verified Microsoft Entra domain.

Example:

```powershell
$Domain = "yourdomain.com"
```

Full script:

```powershell
# ================================
# Bulk Create Users from CSV
# ================================

Import-Module Microsoft.Graph.Users

$Domain = "yourdomain.com" # Replace with your verified Entra ID domain
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

---

### 3e: Verify the users

```powershell
Get-MgUser -All `
    -Property "displayName,department,jobTitle,userPrincipalName" |
Select-Object DisplayName, Department, JobTitle, UserPrincipalName |
Sort-Object Department |
Format-Table -AutoSize
```

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
Update-MgUser -UserId user@yourdomain.com -UsageLocation "US"
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
