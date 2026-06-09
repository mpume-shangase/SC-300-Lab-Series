# Lab 1: Tenant Configuration & Role Management

> **Type:** Focused Lab | **Time:** 3-4 hours | **SC-300 Domain:** Implement and manage user identities (20-25%)

---

## Scenario

You've just been hired as the IAM administrator for **Teach Rich**. Your first task is to configure the Microsoft Entra ID tenant from scratch — set up the role structure, organize the directory with administrative units, establish foundational settings, and document everything with PowerShell.

---

## SC-300 Exam Objectives Covered

- Configure and manage built-in and custom Microsoft Entra roles
- Recommend when to use administrative units
- Configure and manage administrative units
- Evaluate effective permissions for Microsoft Entra roles
- Configure and manage domains in Microsoft Entra ID and Microsoft 365
- Configure company branding settings
- Configure tenant properties, user settings, group settings, and device settings

---

## Prerequisites

- Microsoft 365 E5 subscription with Global Administrator access
- Access to [entra.microsoft.com](https://entra.microsoft.com)
- PowerShell 7 with Microsoft Graph SDK installed
  ```powershell
  Install-Module Microsoft.Graph -Scope CurrentUser
  ```

---

## Step 1: Configure Tenant Properties

**Where:** Entra admin center → Entra ID → Overview → Properties

1. Change the tenant display name to `Teach Rich`
2. Set the **Technical contact** to your admin email address
3. Set the **Notification language** to English
4. Click **Save**

**Why this matters:** The tenant name appears on sign-in pages, consent screens, and in reports. The technical contact receives important service notifications from Microsoft — licensing alerts, security advisories, tenant health issues.

> **SC-300 exam tip:** The exam asks where tenant properties are configured and what the technical contact field is used for. Answer: Entra ID → Overview → Properties. Technical contact receives service notifications from Microsoft.

> **Real-world context:** When you join an org as an IAM analyst, one of your first tasks is a tenant configuration audit. Check these settings to verify they reflect the current organization and that the technical contact is someone who actually responds.

---

## Step 2: Configure User Settings

**Where:** Entra ID → Users → User settings

| Setting | Value | Why |
|---------|-------|-----|
| Users can register applications | **No** | Prevents shadow IT — only admins should create app registrations |
| Restrict non-admin users from creating tenants | **Yes** | Prevents users from creating unmanaged environments outside your security boundary |
| Users can create security groups | **Yes** | Allows self-service group management — reduces admin tickets |
| Guest user access restrictions | **Limited access** (middle option) | Guests can see basic directory info but cannot enumerate all users |
| Restrict access to Microsoft Entra admin center | **Yes** | Follows least privilege — regular users don't need admin portal access |

Click **Save**.

**Why this matters:** These settings define what regular (non-admin) users can do in your tenant. Restricting app registration prevents shadow IT. Blocking tenant creation keeps your security boundary intact.

> **SC-300 exam tip:** The exam frequently tests what each user setting controls. Know the difference between restricting app registration vs. restricting admin portal access — these are separate toggles that do different things.

> **Interview answer:** *"I configured tenant-level user settings following the principle of least privilege. App registration is restricted to admins to prevent shadow IT. Non-admin tenant creation is blocked to maintain our security boundary. Guest access is limited to protect internal directory information. Admin portal access is restricted to administrators only."*

---

## Step 3: Configure Group Settings

### 3a: General Settings

**Where:** Entra ID → Groups → General (under Settings)

| Setting | Value | Why |
|---------|-------|-----|
| Owners can manage group membership in My Groups | **Yes** | Enables self-service group management |
| Restrict user ability to access groups in My Groups | **No** | Allows users to discover and request group membership |

Click **Save**.

### 3b: Expiration Policy

**Where:** Entra ID → Groups → Expiration

| Setting | Value | Why |
|---------|-------|-----|
| Group lifetime (days) | **365** | Forces annual review of group necessity |
| Email contact for groups with no owners | Your admin email | Gets notified when ownerless groups are expiring |
| Enable expiration for | **None** (for now) | Enable for specific groups in production |

Click **Save**.

> **SC-300 exam tip:** Group expiration only applies to **Microsoft 365 groups** — not security groups or distribution lists. When a group expires it is soft-deleted, not immediately gone — there is a 30-day recovery window.

### 3c: Naming Policy

**Where:** Entra ID → Groups → Naming policy

1. Under **Blocked words**, add: `CEO`, `Payroll`, `Confidential`, `Secret`
2. Click **Save**

**Why this matters:** Naming policies prevent users from creating groups with misleading or sensitive names. Admins are exempt — only regular users are restricted.

> **SC-300 exam tip:** Naming policies apply to Microsoft 365 groups and security groups. Prefix/suffix rules can include user profile attributes like department or country.

---

## Step 4: Configure Device Settings

**Where:** Entra ID → Devices → Device settings

| Setting | Value | Why |
|---------|-------|-----|
| Users may join devices to Microsoft Entra ID | **All** | Allows cloud-native device enrollment |
| Users may register their devices | **All** | Enables BYOD access |
| Maximum number of devices per user | **5** | Prevents inventory noise and limits blast radius |
| Require MFA to register or join devices | **Yes** (if available) | Prevents unauthorized device enrollment |

Click **Save**.

> **SC-300 exam tip — know these three device join types:**
> - **Entra ID Join** — fully cloud-native. Corporate laptop, no on-prem AD dependency. Managed by Intune.
> - **Entra ID Registered** — personal devices, BYOD. Device shows up in tenant but is not fully managed.
> - **Hybrid Entra ID Join** — domain-joined machines synced to the cloud via Entra Connect. Transition state for enterprises with on-prem AD.

---

## Step 5: Create Test Users

**Where:** Entra ID → Users → + New user → Create new user

Create the following users before proceeding. These users are needed for testing the custom role (Step 6), populating administrative units (Step 7), and verifying scoped permissions.

> **Real-world context:** In production you would never create users manually at scale. You would provision from an HR system or sync from on-premises Active Directory. We are doing this manually here to understand the data model. Bulk creation with PowerShell is covered in Lab 2.

### User 1 — IAM Operator (custom role testing)

| Field | Value |
|-------|-------|
| User principal name | `iam.operator@yourdomain.onmicrosoft.com` |
| Display name | IAM Operator |
| Password | Auto-generate — copy it |
| Department | IT |
| Job title | IAM Operator |

### Users 2–4 — Engineering team (AU-Engineering)

| UPN | Display Name | Department | Job Title |
|-----|-------------|------------|-----------|
| `jessica.thompson@yourdomain.onmicrosoft.com` | Jessica Thompson | Engineering | Software Engineer |
| `marcus.johnson@yourdomain.onmicrosoft.com` | Marcus Johnson | Engineering | DevOps Engineer |
| `alex.chen@yourdomain.onmicrosoft.com` | Alex Chen | Engineering | QA Engineer |

### User 5 — Helpdesk AU Test (scoped role testing)

| Field | Value |
|-------|-------|
| User principal name | `helpdesk.au@yourdomain.onmicrosoft.com` |
| Display name | Helpdesk AU Test |
| Department | IT |
| Job title | Helpdesk Analyst |

### User 6 — Standard User (access restriction verification)

| Field | Value |
|-------|-------|
| User principal name | `standard.user@yourdomain.onmicrosoft.com` |
| Display name | Standard User |
| Department | Sales |
| Job title | Sales Associate |

> **SC-300 exam tip:** Usage location is required before you can assign a Microsoft 365 license to a user. Department and Job Title fields feed into dynamic group rules — a group can automatically include all users where Department equals Engineering.

---

## Step 6: Explore Built-in Entra ID Roles

**Where:** Entra ID → Roles & admins

Click on each role below, read the **Description**, and check **Assignments** to see who currently holds it.

| Role | What It Can Do | When To Use |
|------|---------------|-------------|
| **Global Administrator** | Everything — full access to all features | Emergency only. Maximum 2–3 accounts. Never for day-to-day tasks. |
| **Privileged Role Administrator** | Manage role assignments and PIM | IAM team leads — the person who decides who gets what access |
| **User Administrator** | Create/manage users, reset passwords, manage groups | Day-to-day user lifecycle management |
| **Helpdesk Administrator** | Reset passwords for non-admins, invalidate refresh tokens | Tier 1 support — cannot reset passwords for other admins |
| **Authentication Administrator** | Manage authentication methods, reset MFA | Tier 2 escalation for authentication problems |
| **Security Administrator** | Manage security features, Conditional Access, security reports | Security team — intentionally separate from user management |
| **Conditional Access Administrator** | Manage CA policies only | When someone needs CA access without full Security Admin |
| **Application Administrator** | Manage enterprise apps and app registrations | Application integration team |
| **Groups Administrator** | Create/manage all groups and group settings | Delegated group management |
| **License Administrator** | Manage license assignments only | Finance or operations team member handling licenses |

> **SC-300 exam tip:** Helpdesk Administrator **cannot** reset passwords for other admins. If a User Administrator is locked out, a Helpdesk Administrator cannot help — that requires a User Administrator or Global Admin. Global Administrator is never the right answer for a scoped task in an exam scenario.

> **Interview answer:** *"When I evaluate a role request I ask three questions. One — what does this person actually need to do? Two — what is the most scoped role that covers those tasks? Three — does this need to be permanent or should it go through PIM with time-limited activation? Global Administrator is reserved for break-glass accounts — maximum two or three, protected with MFA and PIM activation requirements."*

---

## Step 7: Create a Custom Role

**Where:** Entra ID → Roles & admins → + New custom role

Build a custom role called **IAM Operator** that can manage users and groups but cannot delete them.

1. Click **+ New custom role**
2. **Name:** `IAM Operator`
3. **Description:** `Can create and update users and groups but cannot delete them. Designed for day-to-day IAM operations without destructive permissions.`
4. Click **Next** to go to Permissions
5. Search for and add these permissions:

| Permission | What It Does |
|------------|-------------|
| `microsoft.directory/users/basic/update` | Update basic user properties |
| `microsoft.directory/users/create` | Create users |
| `microsoft.directory/users/password/update` | Reset passwords |
| `microsoft.directory/groups/basic/update` | Update basic group properties |
| `microsoft.directory/groups/create` | Create groups |
| `microsoft.directory/groups/members/update` | Manage group membership |

6. **Do NOT add any `delete` permissions** — this is intentional
7. Click **Next** → **Create**

### Test the role

1. Assign the **IAM Operator** role to `iam.operator@yourdomain.onmicrosoft.com`
2. Sign in as that user in an incognito window
3. Verify the following:

| Test | Expected Result |
|------|----------------|
| Create a user | ✅ Allowed |
| Reset a password | ✅ Allowed |
| Update user properties | ✅ Allowed |
| Delete a user | ❌ Greyed out or 403 error |

**Why this matters:** Custom roles implement fine-grained access control. The IAM Operator can do everything their job requires and nothing beyond it — limited blast radius if the account is compromised.

> **SC-300 exam tip:** Custom roles require **Entra ID P1 or P2** licensing. Custom roles can be assigned at tenant scope or administrative unit scope.

---

## Step 8: Create Administrative Units

**Where:** Entra ID → Administrative units

Administrative units let you scope role assignments to a specific part of the organization. Instead of giving a regional IT team User Administrator over the entire tenant, you give them User Administrator over just their region.

### Create the administrative units

1. Click **+ Add**
2. Fill in the Name and Description below
3. Set membership type to **Assigned**
4. Click **Create**

| Name | Description |
|------|-------------|
| `AU-Engineering` | Administrative unit for Engineering department users |
| `AU-Sales` | Administrative unit for Sales department users |
| `AU-HR` | Administrative unit for Human Resources department users |
| `AU-Finance` | Administrative unit for Finance department users |

### Add users to AU-Engineering

1. Click on **AU-Engineering**
2. Click **Members** → **+ Add member**
3. Add: **Jessica Thompson, Marcus Johnson, Alex Chen**

### Assign a scoped role

1. Click on **AU-Engineering** → **Roles and administrators**
2. Click **+ Add assignments**
3. Select **Helpdesk Administrator**
4. Assign it to `helpdesk.au@yourdomain.onmicrosoft.com`

### Test the scoped role

Sign in as `helpdesk.au@yourdomain.onmicrosoft.com` in an incognito window and verify:

| Test | Expected Result |
|------|----------------|
| Reset password for Jessica Thompson (in AU-Engineering) | ✅ Allowed |
| Reset password for Standard User (not in AU-Engineering) | ❌ Insufficient privileges |

**Why this matters:** Same role, same user, completely different access based on scope. The AU boundary defines who they can manage without any tenant-wide permissions.

> **SC-300 exam tips — AUs are heavily tested:**
> - AUs can contain **users, groups, and devices** — not just users
> - Membership can be **Assigned** (manual) or **Dynamic** (attribute-based rules, requires P1/P2)
> - **Not all roles support AU scoping** — Helpdesk Administrator, User Administrator, Authentication Administrator, and Groups Administrator do. **Global Administrator does not.**
> - A user can be a member of multiple AUs

> **Interview answer:** *"I used administrative units to implement delegated administration. Regional IT teams can manage users in their own offices without accessing accounts in other regions. By scoping Helpdesk Administrator roles to department AUs, each team manages only their own users — no team outside the central IAM function holds tenant-wide privileged roles."*

---

## Step 9: Configure Company Branding

**Where:** Entra ID → Company branding

1. Click **Configure** (or **Edit** if already configured)
2. Upload a **Sign-in page background image**
3. Upload a **Banner logo** for Teach Rich
4. Set **Sign-in page text:**
   ```
   Welcome to Teach Rich. For IT support contact admin@yourdomain.onmicrosoft.com
   ```
5. Click **Save**
6. Test: Open an incognito window → navigate to `login.microsoftonline.com` → verify branding appears

**Why this matters:** Branding is a security control as much as a UX feature. Users trained to look for company branding on the sign-in page will notice when a phishing page lacks it.

> **SC-300 exam tip:** Know the four customizable elements: **background image, banner logo, sign-in page text, and username hint text**. Branding can be configured per locale — default branding applies to all locales unless you create a locale-specific override.

---

## Step 10: Set Up Certificate Authentication

Before running PowerShell against your tenant, you need to authenticate. We use certificate authentication — not a client secret — because certificates are cryptographically stronger, the private key never leaves your machine, and there is nothing to accidentally commit to GitHub.

### Part 1 — Create the App Registration

**Where:** Entra ID → App registrations → + New registration

| Field | Value |
|-------|-------|
| Name | `IAM-Lab-GraphApp` |
| Supported account types | This organizational directory only |
| Redirect URI | Leave blank |

Click **Register**. Copy and save the **Application (client) ID** and **Directory (tenant) ID** — you will need these for every PowerShell connection.

### Part 2 — Assign API Permissions

**Where:** App registrations → IAM-Lab-GraphApp → API permissions → + Add a permission → Microsoft Graph → Application permissions

Add these permissions:

| Permission | Purpose |
|------------|---------|
| `RoleManagement.Read.Directory` | Read role assignments |
| `Directory.Read.All` | Read users, groups, and directory objects |

**Critical:** Click **Grant admin consent for [tenant name]** and confirm. Verify green checkmarks appear next to both permissions. Without this step, every Graph call returns 403 Forbidden.

> **SC-300 exam tip:** **Delegated permissions** act on behalf of a signed-in user. **Application permissions** act as the application itself with admin-consented access. For background automation with no user sign-in, you need application permissions.

### Part 3 — Generate the Certificate

```powershell
# Create directory structure
New-Item -ItemType Directory -Force -Path "$HOME/Documents/IAM-Projects/certs"
New-Item -ItemType Directory -Force -Path "$HOME/Documents/IAM-Projects/reports"

# Generate self-signed certificate - Windows
$cert = New-SelfSignedCertificate `
    -Subject "CN=IAM-Lab-GraphApp" `
    -CertStoreLocation "Cert:\CurrentUser\My" `
    -KeyExportPolicy Exportable `
    -KeySpec Signature `
    -KeyLength 2048 `
    -HashAlgorithm SHA256 `
    -NotAfter (Get-Date).AddYears(2)


# Generate self-signed certificate - Mac
mkdir -p ~/Documents/IAM-Projects/certs
cd ~/Documents/IAM-Projects/certs

openssl req -x509 \
  -newkey rsa:2048 \
  -sha256 \
  -days 730 \
  -nodes \
  -keyout IAM-Lab-GraphApp.key \
  -out IAM-Lab-GraphApp.pem \
  -subj "/CN=IAM-Lab-GraphApp"
```

| Parameter | Purpose |
|-----------|---------|
| `-Subject` | Label embedded in the certificate — matches app registration name |
| `-CertStoreLocation` | Places cert in your personal Windows certificate store |
| `-KeyExportPolicy Exportable` | Allows private key export to create PFX |
| `-KeyLength 2048 / -HashAlgorithm SHA256` | Current cryptographic standard |
| `-NotAfter (Get-Date).AddYears(2)` | Expires in two years — **set a calendar reminder now** |

### Part 4 — Export Certificate Files

```powershell
# Public certificate (.cer) — safe to upload to Entra ID - Windows
Export-Certificate `
    -Cert $cert `
    -FilePath "$HOME/Documents/IAM-Projects/certs/IAM-Lab-GraphApp.cer"

# PFX with private key — stays on your machine only
$pfxPassword = ConvertTo-SecureString -String "ReplaceWithStrongPassword!" -Force -AsPlainText

Export-PfxCertificate `
    -Cert $cert `
    -FilePath "$HOME/Documents/IAM-Projects/certs/IAM-Lab-GraphApp.pfx" `
    -Password $pfxPassword


# Navigate to the correct directory - Mac
cd ~/Documents/IAM-Projects/certs

# Public certificate (.cer) — safe to upload to Entra ID - Mac
openssl x509 \
  -outform der \
  -in IAM-Lab-GraphApp.pem \
  -out IAM-Lab-GraphApp.cer

  # PFX with private key — stays on your machine only
openssl pkcs12 -export \
  -out IAM-Lab-GraphApp.pfx \
  -inkey IAM-Lab-GraphApp.key \
  -in IAM-Lab-GraphApp.pem
```

> ⚠️ The `.pfx` file contains your private key. Treat it like a password — never share it, never commit it to a repository.

### Part 5 — Upload Certificate to Entra ID

**Where:** App registrations → IAM-Lab-GraphApp → Certificates & secrets → Certificates → Upload certificate

Browse to `IAM-Lab-GraphApp.cer` and upload. Verify the thumbprint appears and the expiry shows two years from today.

### Part 6 — Test the Connection

```powershell
# Load values from environment or paste your own for local testing only
# Never hardcode Tenant ID or Client ID in files committed to a repository

$TenantId = $env:GRAPH_TENANT_ID      # or "paste your tenant ID here" for local testing
$ClientId = $env:GRAPH_CLIENT_ID      # or "paste your client ID here" for local testing

$pfxPassword = Read-Host "Enter your PFX password" -AsSecureString

$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2(
    "$HOME/Documents/IAM-Projects/certs/IAM-Lab-GraphApp.pfx",
    $pfxPassword
)

Connect-MgGraph -ClientId $ClientId -TenantId $TenantId -Certificate $cert
```

Successful connection returns: `Welcome To Microsoft Graph!`

**Troubleshooting:**

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Certificate not uploaded correctly | Re-upload the .cer file to Entra ID |
| 403 Forbidden | Admin consent not granted | Go to API permissions → Grant admin consent |
| Module error | Microsoft Graph SDK not installed | Run `Install-Module Microsoft.Graph -Scope CurrentUser` |

---

## Step 11: Document Role Assignments with PowerShell

**Why this matters in a real IAM role:**

As an IAM analyst or engineer you will be asked to prove things. An auditor will ask you to prove the principle of least privilege is being followed. A security team will ask you to prove no unauthorized accounts have Global Administrator. A manager will ask you to prove access was removed when an employee left. The Entra portal shows current state only — it does not give you timestamped evidence.

PowerShell with the Microsoft Graph SDK lets you take a dated snapshot of your access state at any point. This is a differentiator in interviews — a lot of candidates can describe RBAC, fewer can demonstrate they can automate the governance layer on top of it.

> **Full scripts are available in [`lab01/role-assignments-report.ps1`](./role-assignments-report.ps1)**
> Microsoft Graph PowerShell SDK documentation: [aka.ms/graphpowershell](https://aka.ms/graphpowershell)

### Verify connection

```powershell
# Verify active session
Get-MgContext

# Create reports directory
New-Item -ItemType Directory -Force -Path "./reports"
```

### Pull and display all active role assignments

```powershell
Get-MgRoleManagementDirectoryRoleAssignment -All | ForEach-Object {
    $assignment = $_

    $role = Get-MgRoleManagementDirectoryRoleDefinition `
        -UnifiedRoleDefinitionId $assignment.RoleDefinitionId

    $principal = Get-MgDirectoryObject `
        -DirectoryObjectId $assignment.PrincipalId

    [PSCustomObject]@{
        Role       = $role.DisplayName
        AssignedTo = $principal.AdditionalProperties.displayName
        Scope      = $assignment.DirectoryScopeId
    }
} | Format-Table -AutoSize
```

**What each line does:**

| Line | Purpose |
|------|---------|
| `Get-MgDirectoryRoleAssignment -All` | Retrieves all active role assignments. `-All` handles pagination — Graph returns 100 at a time. |
| `Get-MgDirectoryRoleDefinition` | Resolves the role GUID to a human-readable name |
| `Get-MgDirectoryObject` | Resolves the principal GUID to the actual user, group, or service principal |
| `[PSCustomObject]@{}` | Builds a clean three-field output object |
| `Scope` | Shows `/` for tenant-wide assignments, AU object ID for scoped assignments |
| `Format-Table -AutoSize` | Renders as a table with auto-sized columns |

### Export full audit report to CSV

```powershell

#Before running the export code, make sure these modules are loaded
Import-Module Microsoft.Graph.Identity.Governance
Import-Module Microsoft.Graph.Identity.DirectoryManagement

# Make sure the reports folder exists
New-Item -ItemType Directory -Path "./reports" -Force | Out-Null

Get-MgRoleManagementDirectoryRoleAssignment -All | ForEach-Object {
    $assignment = $_

    $role = Get-MgRoleManagementDirectoryRoleDefinition `
        -UnifiedRoleDefinitionId $assignment.RoleDefinitionId

    $principal = Get-MgDirectoryObject `
        -DirectoryObjectId $assignment.PrincipalId

    [PSCustomObject]@{
        Role          = $role.DisplayName
        AssignedTo    = $principal.AdditionalProperties.displayName
        PrincipalType = $principal.AdditionalProperties.'@odata.type'
        Scope         = $assignment.DirectoryScopeId
        Timestamp     = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    }
} | Export-Csv -Path "./reports/role-assignments.csv" -NoTypeInformation

Write-Host "Report saved to ./reports/role-assignments.csv" -ForegroundColor Green

```

**Additional fields explained:**

| Field | Purpose |
|-------|---------|
| `PrincipalType` | Shows `#microsoft.graph.user` or `#microsoft.graph.servicePrincipal` — filter in Excel to find all service principals with privileged roles |
| `Timestamp` | Converts a data dump into evidence — dated, reproducible, auditable |
| `-NoTypeInformation` | Removes the PowerShell type header line that breaks Excel and most reporting tools |

### Verify lab-specific configurations

```powershell
# Verify custom role assignments
Write-Host "`n=== Custom Role Assignments ===" -ForegroundColor Cyan

Get-MgRoleManagementDirectoryRoleAssignment -All | ForEach-Object {
    $assignment = $_

    $role = Get-MgRoleManagementDirectoryRoleDefinition `
        -UnifiedRoleDefinitionId $assignment.RoleDefinitionId

    $principal = Get-MgDirectoryObject `
        -DirectoryObjectId $assignment.PrincipalId

    if ($role.DisplayName -eq "IAM Operator") {
        [PSCustomObject]@{
            Role       = $role.DisplayName
            AssignedTo = $principal.AdditionalProperties.displayName
            Scope      = $assignment.DirectoryScopeId
        }
    }
} | Format-Table -AutoSize


# Verify AU-scoped assignments
Write-Host "`n=== AU-Scoped Role Assignments ===" -ForegroundColor Cyan

Get-MgRoleManagementDirectoryRoleAssignment -All | ForEach-Object {
    $assignment = $_

    $role = Get-MgRoleManagementDirectoryRoleDefinition `
        -UnifiedRoleDefinitionId $assignment.RoleDefinitionId

    $principal = Get-MgDirectoryObject `
        -DirectoryObjectId $assignment.PrincipalId

    if ($assignment.DirectoryScopeId -ne "/") {
        [PSCustomObject]@{
            Role       = $role.DisplayName
            AssignedTo = $principal.AdditionalProperties.displayName
            Scope      = $assignment.DirectoryScopeId
        }
    }
} | Format-Table -AutoSize
```

> **Real-world context:** Most mature organizations run access reviews on a quarterly cadence for standard roles and monthly for highly privileged roles. This script is step one of that process — pull the report, send to role owners for certification, action the removals, document that the review happened. If you join an org that does not have this automated, building something like this is a meaningful contribution on day one.

---

## Step 12: Verify Everything

Go through this checklist and confirm each item. Take screenshots for your `screenshots/` folder.

### Tenant & Settings
- [ ] Tenant name changed to `Teach Rich`
- [ ] Technical contact email set
- [ ] Users can register applications: **No**
- [ ] Restrict non-admin users from creating tenants: **Yes**
- [ ] Restrict access to Entra admin center: **Yes**
- [ ] Guest user access: **Limited**
- [ ] Group expiration: **365 days**
- [ ] Group naming policy: Blocked words configured
- [ ] Device limit: **5 per user**
- [ ] Require MFA for device enrollment: **Yes**

### Users Created
- [ ] `iam.operator` created
- [ ] `jessica.thompson` created
- [ ] `marcus.johnson` created
- [ ] `alex.chen` created
- [ ] `helpdesk.au` created
- [ ] `standard.user` created

### Roles & Access
- [ ] Custom role (IAM Operator) created with correct permissions
- [ ] Custom role tested — create ✅ reset ✅ delete ❌
- [ ] Administrative units created (AU-Engineering, AU-Sales, AU-HR, AU-Finance)
- [ ] Engineering users assigned to AU-Engineering
- [ ] Helpdesk AU Test user assigned scoped Helpdesk Administrator on AU-Engineering
- [ ] Scoped role tested — Engineering user ✅ Standard User ❌

### Branding & Auth
- [ ] Company branding configured and visible on sign-in page
- [ ] App registration (IAM-Lab-GraphApp) created
- [ ] API permissions granted with admin consent
- [ ] Certificate generated, exported, and uploaded
- [ ] Graph connection tested successfully

### Documentation
- [ ] Role assignments displayed in terminal
- [ ] CSV report exported to `./reports/role-assignments.csv`
- [ ] Custom role and AU-scoped assignments verified in PowerShell

---

## Key Takeaways for SC-300

1. **User settings** control what non-admins can do — restrict app registration and admin portal access
2. **Administrative units** are for delegated administration — scope roles to specific parts of the organization
3. **Custom roles** implement fine-grained access — only grant the exact permissions needed
4. **Built-in roles** follow least privilege — use User Administrator instead of Global Administrator for user management
5. **Company branding** is a security control as much as a UX feature — helps users identify legitimate sign-in pages
6. **Group policies** (naming, expiration) are governance controls that prevent sprawl
7. **Certificate authentication** is the professional standard for automated PowerShell access to Graph API
8. **PowerShell documentation** is how you produce evidence for access reviews and audits — not a bonus step

---

## What's Next

➡️ **Lab 2:** [User & Group Lifecycle with PowerShell](../lab02-user-group-lifecycle/) — Bulk user creation, dynamic groups, custom security attributes, and license management

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
