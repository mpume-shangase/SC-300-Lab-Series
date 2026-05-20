# Lab 1: Tenant Configuration & Role Management

> **Type:** Focused Lab | **Time:** 2-3 hours | **SC-300 Domain:** Implement and manage user identities (20-25%)

---

## Scenario

You've just been hired as the IAM administrator for IAM Lab Corp. Your first task is to configure the Microsoft Entra ID tenant from scratch — set up the role structure, organize the directory with administrative units, and establish the foundational settings that everything else will build on.

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
- Access to entra.microsoft.com
- PowerShell 7 with Microsoft Graph SDK installed

---

## Step 1: Configure Tenant Properties

**Where:** Entra admin center → Entra ID → Overview → Properties

1. Change the tenant display name to `IAM Lab Corp`
2. Set the **Technical contact** to your email address
3. Set the **Notification language** to English
4. Click **Save**

**Why this matters:** The tenant name appears on sign-in pages, consent screens, and in reports. In a real organization, this should reflect the company name. The technical contact receives important service notifications from Microsoft.

**SC-300 exam tip:** The exam may ask where tenant properties are configured and what the technical contact field is used for.

---

## Step 2: Configure User Settings

**Where:** Entra ID → Users → User settings

Configure the following:

| Setting | Value | Why |
|---------|-------|-----|
| Users can register applications | **No** | Prevents shadow IT — only admins should create app registrations |
| Restrict non-admin users from creating tenants | **Yes** | Prevents users from creating unmanaged environments |
| Users can create security groups | **Yes** | Allows self-service group management |
| Guest user access restrictions | **Limited access** (middle option) | Guests can see basic directory info but not enumerate all users |
| Restrict access to Microsoft Entra admin center | **Yes** | Follows least privilege — regular users don't need admin portal access |

Click **Save**.

**Why this matters:** These settings define what regular (non-admin) users can do in your tenant. In production, you want to restrict app registration and admin portal access to prevent unauthorized changes.

**SC-300 exam tip:** The exam frequently tests what each user setting controls. Know the difference between restricting app registration vs. restricting admin portal access.

### How to explain this in an interview:

*"I configured tenant-level user settings to follow the principle of least privilege. Regular users can't register applications or access the admin portal. App registration is restricted to admins to prevent shadow IT. Guest users have limited directory visibility to protect internal user information."*

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
| Enable expiration for | **None** (for now) | We'll enable for specific groups later |

Click **Save**.

### 3c: Naming Policy

**Where:** Entra ID → Groups → Naming policy

1. Under **Blocked words**, add: `CEO`, `Payroll`, `Confidential`, `Secret`
2. Click **Save**

**Why this matters:** Naming policies prevent users from creating groups with misleading or sensitive names. Group expiration prevents stale groups from accumulating indefinitely. Both are governance controls.

**SC-300 exam tip:** Know how naming policies work — prefix/suffix rules and blocked words. Know that expiration only applies to Microsoft 365 groups, not security groups.

---

## Step 4: Configure Device Settings

**Where:** Entra ID → Devices → Device settings

| Setting | Value | Why |
|---------|-------|-----|
| Users may join devices to Microsoft Entra ID | **All** | Allows device registration |
| Users may register their devices | **All** | Enables device management |
| Maximum number of devices per user | **5** | Prevents abuse of device registration |
| Require MFA to register or join devices | **Yes** (if available) | Adds security to device enrollment |

Click **Save**.

**Why this matters:** Device settings control how endpoints connect to your identity platform. Limiting devices per user and requiring MFA for registration prevents unauthorized device enrollment.

**SC-300 exam tip:** Understand the difference between Entra ID join (cloud-native devices), Entra ID registered (personal devices), and Hybrid Entra ID join (domain-joined devices synced to cloud).

---

## Step 5: Explore Built-in Entra ID Roles

**Where:** Entra ID → Roles & admins

This is one of the most heavily tested areas on SC-300. Click on **Roles & admins** and explore the following roles:

| Role | What It Can Do | When To Use |
|------|---------------|-------------|
| **Global Administrator** | Everything — full access to all features | Emergency only. Should have maximum 2 accounts with this role |
| **User Administrator** | Create/manage users, reset passwords, manage groups | Day-to-day user management |
| **Groups Administrator** | Create/manage all groups, manage group settings | Delegated group management |
| **Security Administrator** | Manage security features, read security reports, manage CA policies | Security team members |
| **Privileged Role Administrator** | Manage role assignments in Entra ID and PIM | IAM team leads |
| **Authentication Administrator** | Manage authentication methods, reset passwords for non-admins | Help desk escalation |
| **Helpdesk Administrator** | Reset passwords for non-admins, invalidate refresh tokens | Tier 1 support |
| **Application Administrator** | Manage enterprise apps and app registrations | Application management |
| **Conditional Access Administrator** | Manage Conditional Access policies | Security policy management |
| **License Administrator** | Manage license assignments | License management |

**Action:** Click on each role, then click **Description** to read what permissions it has. Click **Assignments** to see who currently holds the role.

### How to explain this in an interview:

*"I follow the principle of least privilege for role assignments. Instead of giving someone Global Administrator just because they need to manage users, I'd assign User Administrator — it gives them exactly what they need without the ability to change tenant settings, manage billing, or modify security policies. Global Admin should be limited to 2-3 break-glass accounts."*

---

## Step 6: Create a Custom Role

**Where:** Entra ID → Roles & admins → + New custom role

Build a custom role called **IAM Operator** that can manage users and groups but cannot delete them:

1. Click **+ New custom role**
2. **Name:** `IAM Operator`
3. **Description:** `Can create and update users and groups but cannot delete them. Designed for day-to-day IAM operations.`
4. Click **Next** to go to Permissions
5. Search for and add these permissions:
   - `microsoft.directory/users/basic/update` — Update basic user properties
   - `microsoft.directory/users/create` — Create users
   - `microsoft.directory/users/password/update` — Reset passwords
   - `microsoft.directory/groups/basic/update` — Update basic group properties
   - `microsoft.directory/groups/create` — Create groups
   - `microsoft.directory/groups/members/update` — Manage group membership
6. Do NOT add any `delete` permissions
7. Click **Next** → **Create**

**Test the role:**
1. Create a test user called `iam.operator@yourdomain.onmicrosoft.com`
2. Assign the **IAM Operator** custom role to this test user
3. Sign in as the test user in an incognito window
4. Verify: Can they create users? ✅ Can they reset passwords? ✅ Can they delete users? ❌

**Why this matters:** Custom roles let you implement fine-grained access control. Instead of choosing between overpowered built-in roles, you create exactly the permissions someone needs.

**SC-300 exam tip:** Know how to create custom roles, what permissions are available, and the difference between built-in and custom roles. Custom roles require Entra ID P1 or P2.

---

## Step 7: Create Administrative Units

**Where:** Entra ID → Administrative units

Administrative units let you scope role assignments to a specific part of the organization. Instead of giving someone User Administrator over the entire tenant, you give them User Administrator over just the Engineering department.

### Create the administrative units:

1. Click **+ Add**
2. **Name:** `AU-Engineering`
3. **Description:** `Administrative unit for Engineering department`
4. Click **Next** → leave membership type as **Assigned** → **Create**

Repeat for:
- `AU-Sales` — Administrative unit for Sales department
- `AU-HR` — Administrative unit for Human Resources department
- `AU-Finance` — Administrative unit for Finance department

### Add users to administrative units:

1. Click on **AU-Engineering**
2. Click **Members** → **+ Add member**
3. Add the Engineering users (Jessica Thompson, Marcus Johnson, Alex Chen)
4. Repeat for each AU with the appropriate department users

### Assign a scoped role:

1. Click on **AU-Engineering**
2. Click **Roles and administrators**
3. Click **+ Add assignments**
4. Select **Helpdesk Administrator**
5. Assign it to a test user

Now that test user can only reset passwords for users inside AU-Engineering — not for anyone else in the tenant.

**Why this matters:** Administrative units are how large organizations delegate management without giving away too much power. A regional IT team should only manage users in their region, not the entire company.

**SC-300 exam tip:** Administrative units are heavily tested. Know how to create them, add members (assigned vs. dynamic), and scope role assignments to them. Know that AUs can contain users, groups, and devices.

### How to explain this in an interview:

*"I used administrative units to implement delegated administration. Our regional IT teams needed to manage users in their own offices, but we didn't want them accessing user accounts in other regions. By creating administrative units per department and scoping Helpdesk Administrator roles to those units, each team can only manage their own users."*

---

## Step 8: Configure Company Branding

**Where:** Entra ID → Company branding

Company branding customizes the sign-in page that users see when they log in.

1. Click **Configure** (or **Edit** if already configured)
2. Upload a **Sign-in page background image** (use any professional-looking image — you can use a free stock photo)
3. Upload a **Banner logo** (use a simple logo or text image for "IAM Lab Corp")
4. Set **Sign-in page text:** `Welcome to IAM Lab Corp. Contact IT support at admin@thecyberkhroniclesgmail.onmicrosoft.com for assistance.`
5. Click **Save**
6. Test it: Open an incognito window and go to `login.microsoftonline.com` — you should see your custom branding

**Why this matters:** Branding builds user trust — employees know they're signing into the right portal. It also helps prevent phishing — if users are trained to look for company branding, a phishing page without it looks suspicious.

**SC-300 exam tip:** Know where company branding is configured and what elements can be customized (background image, banner logo, sign-in page text, and username hint).

---

## Step 9: Document Your Role Assignments with PowerShell

Open your terminal and run PowerShell:

```bash
pwsh
```

Connect to Graph:

```powershell
$TenantId = "9d0e259d-f64a-4fee-b472-719ca3682f8a"
$ClientId = "52327820-2397-4ba8-994e-0a4df9b84549"
$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2("$HOME/Documents/IAM-Projects/certs/IAM-Lifecycle-Cert.pfx")
Connect-MgGraph -ClientId $ClientId -TenantId $TenantId -Certificate $cert
```

List all active role assignments:

```powershell
Get-MgDirectoryRoleAssignment -All | ForEach-Object {
    $role = Get-MgDirectoryRoleDefinition -UnifiedRoleDefinitionId $_.RoleDefinitionId
    $principal = Get-MgDirectoryObject -DirectoryObjectId $_.PrincipalId
    [PSCustomObject]@{
        Role = $role.DisplayName
        AssignedTo = $principal.AdditionalProperties.displayName
        Scope = $_.DirectoryScopeId
    }
} | Format-Table -AutoSize
```

Export to CSV for documentation:

```powershell
# Save the role assignments report
Get-MgDirectoryRoleAssignment -All | ForEach-Object {
    $role = Get-MgDirectoryRoleDefinition -UnifiedRoleDefinitionId $_.RoleDefinitionId
    $principal = Get-MgDirectoryObject -DirectoryObjectId $_.PrincipalId
    [PSCustomObject]@{
        Role = $role.DisplayName
        AssignedTo = $principal.AdditionalProperties.displayName
        PrincipalType = $principal.AdditionalProperties.'@odata.type'
        Scope = $_.DirectoryScopeId
        Timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    }
} | Export-Csv -Path "./reports/role-assignments.csv" -NoTypeInformation

Write-Host "Role assignments exported to ./reports/role-assignments.csv" -ForegroundColor Green
```

**Why this matters:** Documentation is governance. Knowing who has what role, and being able to prove it to auditors, is a core IAM responsibility.

---

## Step 10: Verify Everything

Go through this checklist and confirm each item:

- [ ] Tenant name changed to IAM Lab Corp
- [ ] Technical contact email set
- [ ] Users can register applications: No
- [ ] Restrict non-admin users from creating tenants: Yes
- [ ] Restrict access to Entra admin center: Yes
- [ ] Guest user access: Limited
- [ ] Group expiration: 365 days
- [ ] Group naming policy: Blocked words configured
- [ ] Device limit: 5 per user
- [ ] Custom role (IAM Operator) created and tested
- [ ] Administrative units created (AU-Engineering, AU-Sales, AU-HR, AU-Finance)
- [ ] Users assigned to correct administrative units
- [ ] Scoped role assignment tested on an administrative unit
- [ ] Company branding configured and visible on sign-in page
- [ ] Role assignments documented with PowerShell

Take screenshots of each completed configuration for your `screenshots/` folder.

---

## Key Takeaways for SC-300

1. **User settings** control what non-admins can do — restrict app registration and admin portal access
2. **Administrative units** are for delegated administration — scope roles to specific parts of the organization
3. **Custom roles** implement fine-grained access — only grant the exact permissions needed
4. **Built-in roles** follow least privilege — use User Administrator instead of Global Administrator for user management
5. **Company branding** is a security control as much as a UX feature — helps users identify legitimate sign-in pages
6. **Group policies** (naming, expiration) are governance controls that prevent sprawl

---

## What's Next

➡️ **Lab 2:** [User & Group Lifecycle with PowerShell](../lab02-user-group-lifecycle/) — Bulk user creation, dynamic groups, custom security attributes, and license management

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
