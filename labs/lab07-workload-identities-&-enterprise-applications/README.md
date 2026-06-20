# Lab 7: Workload Identities & Enterprise Applications

> **Type:** Big Project | **Time:** 3–4 hours | **SC-300 Domain:** Plan and implement workload identities (20–25%)

---

## Scenario

The TeachRich development team needs to integrate several applications with Microsoft Entra ID. TeachRich uses SaaS applications for business operations, custom internal applications for resource management, and Azure services that need secure programmatic access to company data.

You have been asked to plan and implement workload identities and enterprise application access. In this lab, you will compare workload identity types, create managed identities, integrate enterprise applications, configure app assignments and app roles, manage user and admin consent, create application collections, and export enterprise application reports.

The TeachRich verified domain used throughout this lab is:

```text
teachrich.com
```

---

## SC-300 Exam Objectives Covered

* Select appropriate identities for applications and Azure workloads
* Create managed identities
* Compare system-assigned and user-assigned managed identities
* Assign and use managed identities to access Azure resources
* Plan and implement settings for enterprise applications
* Assign Microsoft Entra roles for managing enterprise applications
* Design and implement integration for SaaS applications
* Assign users, groups, and app roles to enterprise applications
* Configure and manage user consent
* Configure and manage admin consent workflow
* Create and manage application collections
* Generate enterprise application reports

---

## Prerequisites

Before starting this lab, you should have:

* Completed Labs 1–6
* A Microsoft 365 E3, Microsoft 365 E5, or similar subscription
* Microsoft Entra ID P1 or P2 features available
* An Azure subscription for managed identity exercises
* A verified Microsoft Entra domain, such as `teachrich.com`
* PowerShell 7 installed
* Microsoft Graph PowerShell SDK installed
* Azure PowerShell module installed if you are completing the Azure managed identity steps
* Certificate-based Microsoft Graph authentication configured from Lab 1
* Test users and groups created from earlier labs

Example TeachRich groups used in this lab:

```text
DG-Engineering-All
DG-Finance-All
DG-Senior-Staff
SG-Project-Alpha
```

> **Important:** Some Azure managed identity steps require an Azure subscription. If you do not have one, document the managed identity concepts, screenshots, and expected configuration instead of skipping the learning objective.

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
Application.ReadWrite.All
AppRoleAssignment.ReadWrite.All
Directory.ReadWrite.All
ServicePrincipalEndpoint.ReadWrite.All
DelegatedPermissionGrant.ReadWrite.All
Policy.Read.All
Policy.ReadWrite.PermissionGrant
Group.ReadWrite.All
User.Read.All
```

Then click:

```text
Grant admin consent
```

Use the permissions based on the tasks you plan to complete:

| Task                             | Useful permissions                                           |
| -------------------------------- | ------------------------------------------------------------ |
| Read enterprise applications     | `Application.ReadWrite.All`, `Directory.ReadWrite.All`       |
| Manage service principals        | `Application.ReadWrite.All`, `Directory.ReadWrite.All`       |
| Assign users/groups to apps      | `AppRoleAssignment.ReadWrite.All`, `Directory.ReadWrite.All` |
| Read users and groups            | `User.Read.All`, `Group.ReadWrite.All`                       |
| Manage consent grants            | `DelegatedPermissionGrant.ReadWrite.All`                     |
| Manage permission grant policies | `Policy.ReadWrite.PermissionGrant`                           |
| Read policy settings             | `Policy.Read.All`                                            |

> **Troubleshooting note:** If you add or change API permissions, always click **Grant admin consent**, then disconnect and reconnect to Microsoft Graph.

> #### 📘 Why these permissions matter
>
> Enterprise applications and workload identities are represented in Microsoft Entra ID as **service principals** and **app registrations**. Managing them requires application and directory permissions.
>
> `Application.ReadWrite.All` allows the app to read and manage application objects and service principals. `AppRoleAssignment.ReadWrite.All` is needed when assigning users or groups to enterprise applications. Consent-related permissions are sensitive because they control what applications can access on behalf of users or the organization.
>
> In production, consent and application management should be tightly governed. Do not allow broad application permissions without review.

---

## Before You Run PowerShell: Connect to Microsoft Graph

Open PowerShell 7.

On macOS or Linux, open Terminal and run:

```bash
pwsh
```

On Windows, open PowerShell 7 directly.

Replace the Tenant ID and Client ID with your own values.

```powershell
$TenantId = "your-tenant-id"
$ClientId = "your-client-id"
$TenantDomain = "teachrich.com"

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
> **What it does:** Signs in to Microsoft Graph using the app registration and certificate created in Lab 1.
>
> **Why it matters:** Enterprise application administration is often automated. PowerShell allows you to report on apps, service principals, assignments, and consent settings in a repeatable way.
>
> **Line by line:**
>
> * `$TenantId` identifies the TeachRich tenant.
> * `$ClientId` identifies the app registration that has Microsoft Graph permissions.
> * `$TenantDomain = "teachrich.com"` stores the verified domain for reuse in lab scripts.
> * `Read-Host ... -AsSecureString` prompts for the PFX password without displaying it.
> * `New-Object ... X509Certificate2(...)` loads the certificate and private key.
> * `Connect-MgGraph ... -Certificate $cert` authenticates to Microsoft Graph as the application.
> * `Get-MgContext` confirms the connection.
>
> **Watch out for:** Do not commit your `.pfx` file, certificate password, client secrets, or real app credentials to GitHub.

---

## Step 1: Understand Workload Identity Types

Create a reference document:

```text
docs/workload-identity-guide.md
```

Use this table:

| Identity Type                        | What It Is                                                                                                     | When To Use                                                                                               | Example                                         |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| **System-assigned managed identity** | An identity tied to one Azure resource. It is created and deleted with that resource.                          | Use when one Azure resource needs access to another Azure resource.                                       | Azure VM reads secrets from Key Vault           |
| **User-assigned managed identity**   | A standalone managed identity that can be attached to multiple Azure resources.                                | Use when multiple resources need the same identity or when the identity should survive resource deletion. | Several Azure Functions use one shared identity |
| **Service principal**                | An identity for an application or service in Microsoft Entra ID.                                               | Use for custom apps, automation, CI/CD pipelines, and non-human access.                                   | IAM automation app registration                 |
| **Enterprise application**           | The service principal representation of an application in your tenant, often from the Microsoft Entra gallery. | Use for SaaS app integration and SSO.                                                                     | Salesforce, ServiceNow, Slack, AWS              |
| **User account**                     | A human identity.                                                                                              | Use only for human sign-in, not automation.                                                               | TeachRich employee sign-in                      |

### Why this matters

The SC-300 exam tests whether you can select the correct identity type for a given scenario. A common mistake is using a user account for automation. For automation and application access, use managed identities or service principals instead.

### SC-300 exam tip

Know this distinction:

| Scenario                               | Best Identity Type     |
| -------------------------------------- | ---------------------- |
| Azure VM needs to access Key Vault     | Managed identity       |
| Azure Function needs to access Storage | Managed identity       |
| CI/CD pipeline needs Graph API access  | Service principal      |
| SaaS app needs SSO                     | Enterprise application |
| Employee signs in to Microsoft 365     | User account           |

### How to explain this in an interview

> I choose the identity type based on the workload. For Azure resources that need to access other Azure resources, I use managed identities because they remove the need to manage secrets. For custom applications and automation, I use service principals with certificate authentication. For SaaS integrations, I use enterprise applications from the Microsoft Entra gallery. I avoid using user accounts for automation because they are difficult to govern and can break when a password changes or a user leaves.

---

## Step 2: Create and Use Managed Identities

Managed identities give Azure resources an identity in Microsoft Entra ID without requiring you to store passwords, client secrets, or certificates in code.

---

### 2a: Create an Azure Resource for Testing

You need an Azure resource to attach a managed identity to. For this lab, use a storage account.

**Where:** Azure portal → Storage accounts → Create

Use these values:

| Setting              | Value                                     |
| -------------------- | ----------------------------------------- |
| Resource group       | `RG-TeachRich-IAMLab`                     |
| Storage account name | `teachrichiamstorage` plus random numbers |
| Region               | Choose your nearest region                |
| Performance          | Standard                                  |
| Redundancy           | Locally redundant storage for lab use     |

> **Important:** Storage account names must be globally unique. If `teachrichiamstorage` is taken, add random numbers, such as `teachrichiamstorage4821`.

---

### 2b: Enable a System-Assigned Managed Identity

1. Open the storage account.
2. Go to **Identity**.
3. Select the **System assigned** tab.
4. Change **Status** to:

```text
On
```

5. Click **Save**.
6. Copy the **Object ID** for your notes.

### Why this matters

A system-assigned managed identity belongs to the Azure resource. If the resource is deleted, the identity is deleted too.

---

### 2c: Create a User-Assigned Managed Identity

**Where:** Azure portal → Managed identities → Create

Use these values:

| Setting        | Value                        |
| -------------- | ---------------------------- |
| Resource group | `RG-TeachRich-IAMLab`        |
| Name           | `mi-teachrich-automation`    |
| Region         | Same as your storage account |

Click **Review + create**, then **Create**.

### Why this matters

A user-assigned managed identity is a standalone Azure resource. It can be attached to multiple Azure resources and can continue existing even if one attached resource is deleted.

---

### 2d: Grant Managed Identity Access to a Resource

Give the system-assigned identity access to the storage account.

1. Go to the storage account.
2. Select **Access control (IAM)**.
3. Click **Add** → **Add role assignment**.
4. Select this role:

```text
Storage Blob Data Reader
```

5. Assign access to:

```text
Managed identity
```

6. Select the storage account’s system-assigned managed identity.
7. Click **Review + assign**.

### Why this matters

Managed identities still need permissions. Creating the identity only gives the resource an identity; assigning an Azure RBAC role gives that identity access to a resource.

---

### 2e: List Managed Identities with PowerShell

```powershell
# ================================
# List Managed Identities
# ================================

Import-Module Microsoft.Graph.Applications

Get-MgServicePrincipal `
    -All `
    -Filter "servicePrincipalType eq 'ManagedIdentity'" `
    -Property "displayName,appId,id,servicePrincipalType,alternativeNames" |
Select-Object `
    DisplayName,
    AppId,
    Id,
    ServicePrincipalType,
    @{Name="ManagedIdentityType";Expression={
        if ($_.AlternativeNames -match "isExplicit=True") {
            "User-assigned"
        }
        else {
            "System-assigned"
        }
    }} |
Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Lists managed identities in the tenant and attempts to identify whether each one is user-assigned or system-assigned.
>
> **Why it matters:** Managed identities appear in Microsoft Entra ID as service principals. This report helps you see that Azure resources and managed identities are represented as identity objects in the directory.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Applications` loads the Microsoft Graph application and service principal commands.
> * `Get-MgServicePrincipal -Filter "servicePrincipalType eq 'ManagedIdentity'"` returns only service principals that are managed identities.
> * `-Property "..."` requests the fields needed for reporting.
> * `Select-Object` shapes the output into readable columns.
> * The calculated `ManagedIdentityType` column checks `AlternativeNames` for a sign that the identity is user-assigned.
> * `Format-Table -AutoSize` displays the output in a clean table.
>
> **Watch out for:** The method used to distinguish system-assigned and user-assigned identities may vary depending on how Graph returns the object. For final evidence, also capture screenshots from the Azure portal.

### SC-300 exam tip

Know the difference:

| Type                             | Lifecycle                                   | Reuse                                 |
| -------------------------------- | ------------------------------------------- | ------------------------------------- |
| System-assigned managed identity | Created and deleted with one Azure resource | One-to-one                            |
| User-assigned managed identity   | Created as its own Azure resource           | Can be attached to multiple resources |

---

## Step 3: Add Enterprise Applications from the Gallery

Enterprise applications allow you to integrate SaaS and custom applications with Microsoft Entra ID.

**Where:** Microsoft Entra admin center → Enterprise applications → New application

---

### 3a: Add a SAML Application

1. Click **New application**.
2. Search for a SAML application, such as:

```text
AWS IAM Identity Center
```

or another gallery application available in your tenant.

3. Click **Create**.
4. Open the application.
5. Go to **Single sign-on**.
6. Select:

```text
SAML
```

7. Review the SAML configuration fields:

| Field                   | Meaning                                       |
| ----------------------- | --------------------------------------------- |
| Identifier / Entity ID  | The unique identifier for the application     |
| Reply URL / ACS URL     | Where Entra sends the SAML response           |
| Sign-on URL             | The application login URL                     |
| Federation Metadata XML | Metadata shared with the application          |
| Certificate             | Used by the app to validate the SAML response |

8. Download the **Federation Metadata XML** or **Certificate (Base64)** for documentation.

> **Important:** For a portfolio lab, you can document the configuration screen without completing the full production SSO integration, especially if you do not own the external SaaS tenant.

### Why this matters

SAML is still widely used for enterprise SaaS and legacy applications. You need to understand the configuration fields and metadata exchange process.

---

### 3b: Add an OIDC-Based Application

1. Click **New application**.
2. Search for a modern SaaS application, such as Slack or another available gallery app.
3. Click **Create**.
4. Open the application.
5. Go to **Single sign-on**.
6. Review the SSO method and configuration options.

### Why this matters

OIDC is commonly used by modern web and mobile applications. It uses tokens rather than SAML assertions and is usually easier for modern app developers to integrate.

---

### 3c: Document SSO Protocol Differences

Create this file:

```text
docs/sso-protocol-comparison.md
```

Use this table:

| Feature        | SAML 2.0                               | OpenID Connect                              |
| -------------- | -------------------------------------- | ------------------------------------------- |
| Token format   | XML SAML assertion                     | JSON Web Token                              |
| Primary use    | Enterprise SSO and legacy applications | Modern web, mobile, and API-based apps      |
| Common flow    | Browser redirect and POST binding      | Authorization code flow                     |
| Configuration  | Metadata XML and certificates          | Discovery endpoint, client ID, redirect URI |
| Token size     | Larger                                 | Smaller                                     |
| Mobile support | Limited compared to OIDC               | Strong support                              |
| Best fit       | Older enterprise SaaS and on-prem apps | Modern SaaS, custom apps, mobile apps       |

### SC-300 exam tip

Know the conceptual difference:

* SAML uses XML assertions.
* OIDC uses JSON-based tokens.
* SAML is common in enterprise SSO.
* OIDC is common in modern applications.

---

## Step 4: Assign Users and Groups to Enterprise Applications

Enterprise applications can be restricted so only assigned users or groups can access them.

---

### 4a: Configure App Assignment Requirement

1. Open your enterprise application.
2. Go to **Properties**.
3. Set:

```text
Assignment required?
```

to:

```text
Yes
```

4. Click **Save**.

### Why this matters

When assignment is required, only explicitly assigned users and groups can access the app. This prevents tenant-wide access to applications that should be limited.

---

### 4b: Assign Users and Groups

1. Open the enterprise application.
2. Go to **Users and groups**.
3. Click **Add user/group**.
4. Select a TeachRich group, such as:

```text
DG-Engineering-All
```

5. If the application has roles, select the correct app role.
6. Click **Assign**.

### Why this matters

Assigning groups instead of individual users is easier to manage. If group membership changes, app access changes with it.

---

### 4c: Create Custom App Roles

For custom applications, app roles can represent application-specific permissions.

Create app roles such as:

| Display Name    | Value            | Description                             |
| --------------- | ---------------- | --------------------------------------- |
| Viewer          | `Viewer`         | Can view information in the application |
| Contributor     | `Contributor`    | Can create and edit information         |
| Project Manager | `ProjectManager` | Can manage project-related work         |
| Admin           | `Admin`          | Can manage application settings         |

### Why this matters

App roles are included in tokens sent to the application. The application can then use those roles to decide what the user is allowed to do.

### SC-300 exam tip

Know the difference:

| Role Type       | Scope                                                    |
| --------------- | -------------------------------------------------------- |
| Directory role  | Gives permissions in Microsoft Entra ID or Microsoft 365 |
| App role        | Gives permissions inside a specific application          |
| Azure RBAC role | Gives permissions to Azure resources                     |

---

### 4d: Assign Users to App Roles

1. Go to the enterprise app.
2. Open **Users and groups**.
3. Click **Add user/group**.
4. Select a user or group.
5. Select an app role such as:

```text
Project Manager
```

6. Click **Assign**.

### Why this matters

This allows the app to receive role claims in the token and apply role-based access inside the application.

---

## Step 5: Configure User and Admin Consent

Consent controls which applications can access Microsoft 365 or Microsoft Graph data.

**Where:** Microsoft Entra admin center → Enterprise applications → Consent and permissions

---

### 5a: Configure User Consent Settings

1. Go to **Consent and permissions**.
2. Open **User consent settings**.
3. Select:

```text
Allow user consent for apps from verified publishers, for selected permissions
```

4. Open **Permission classifications**.
5. Add low-impact permissions such as:

```text
openid
profile
email
offline_access
```

6. Click **Save**.

### Why this matters

This allows users to consent to low-risk permissions from verified publishers but prevents users from approving risky permissions without admin review.

---

### 5b: Configure Admin Consent Workflow

1. Go to **Consent and permissions**.
2. Open **Admin consent settings**.
3. Set:

```text
Users can request admin consent to apps they are unable to consent to
```

to:

```text
Yes
```

4. Add your admin account as a reviewer.
5. Enable email notifications.
6. Enable request expiration reminders.
7. Set request expiration to:

```text
30 days
```

8. Click **Save**.

### Why this matters

Admin consent workflow gives users a formal way to request access to an app instead of being blocked with no next step. It also gives the IAM team a chance to review permissions before approval.

---

### 5c: Test the Consent Flow

1. Open an incognito/private browser window.
2. Sign in as a regular TeachRich user.
3. Try to access an app that requests permissions the user cannot consent to.
4. Confirm that the user sees an approval request option.
5. Sign in as an admin.
6. Go to **Enterprise applications** → **Admin consent requests**.
7. Review, approve, or deny the request.

### SC-300 exam tip

Know the user consent options:

| Consent Setting                                                | Meaning                                                        |
| -------------------------------------------------------------- | -------------------------------------------------------------- |
| Do not allow user consent                                      | Users cannot consent to apps                                   |
| Allow consent for verified publishers and selected permissions | Users can approve low-risk permissions from trusted publishers |
| Allow user consent for all apps                                | Least restrictive and usually not recommended                  |

### How to explain this in an interview

> I configured the consent framework for TeachRich so users can consent only to low-impact permissions from verified publishers. Anything beyond that requires admin approval through the admin consent workflow. This balances productivity with security because users can request the apps they need while IAM still reviews high-impact permissions.

---

## Step 6: Create Application Collections

Application collections organize apps in the My Apps portal.

**Where:** Microsoft Entra admin center → Enterprise applications → Application collections

---

### 6a: Create Department Collections

Create the following collections:

| Collection Name      | Target Group                      | Purpose                        |
| -------------------- | --------------------------------- | ------------------------------ |
| Engineering Tools    | `DG-Engineering-All`              | Apps used by engineering users |
| Finance & Accounting | `DG-Finance-All`                  | Apps used by finance users     |
| Project Alpha        | `SG-Project-Alpha`                | Apps used by a project team    |
| Company-Wide         | All users or selected staff group | Common business apps           |

---

### 6b: Assign Collections to Groups

1. Open a collection.
2. Go to **Users and groups**.
3. Add the appropriate group.
4. Save the assignment.

---

### 6c: Test the My Apps Experience

1. Open a browser.
2. Go to:

```text
https://myapplications.microsoft.com
```

3. Sign in as a test user.
4. Confirm that the correct collections appear.

### Why this matters

Application collections improve the user experience by organizing apps by department, project, or business function.

### SC-300 exam tip

Know that application collections are used to organize the My Apps portal experience and can be assigned to users or groups.

---

## Step 7: Generate Enterprise Application Reports

Create a `reports` folder and export a report of enterprise applications and assignments.

```powershell
# ================================
# Export Enterprise Application Report
# ================================

Import-Module Microsoft.Graph.Applications

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$apps = Get-MgServicePrincipal `
    -All `
    -Filter "servicePrincipalType eq 'Application'" `
    -Property "id,appId,displayName,servicePrincipalType,signInAudience,accountEnabled,appOwnerOrganizationId"

$report = foreach ($app in $apps) {

    $assignments = Get-MgServicePrincipalAppRoleAssignedTo `
        -ServicePrincipalId $app.Id `
        -All `
        -ErrorAction SilentlyContinue

    [PSCustomObject]@{
        AppName             = $app.DisplayName
        AppId               = $app.AppId
        ServicePrincipalId  = $app.Id
        AccountEnabled      = $app.AccountEnabled
        SignInAudience      = $app.SignInAudience
        AssignedUsers       = ($assignments | Where-Object { $_.PrincipalType -eq "User" }).Count
        AssignedGroups      = ($assignments | Where-Object { $_.PrincipalType -eq "Group" }).Count
        AssignedServicePrincipals = ($assignments | Where-Object { $_.PrincipalType -eq "ServicePrincipal" }).Count
    }
}

$report |
    Sort-Object AppName |
    Format-Table AppName, AccountEnabled, AssignedUsers, AssignedGroups, SignInAudience -AutoSize

$report |
    Sort-Object AppName |
    Export-Csv -Path "./reports/enterprise-apps-report.csv" -NoTypeInformation

Write-Host "Enterprise application report exported to ./reports/enterprise-apps-report.csv" -ForegroundColor Green
Write-Host "Total applications reported: $($report.Count)" -ForegroundColor Cyan
```

> #### 📘 Script explained
>
> **What it does:** Lists enterprise applications in the tenant, counts user and group assignments, previews the report, and exports it to CSV.
>
> **Why it matters:** Enterprise application reporting helps identify which apps exist in the tenant and whether access is assigned directly to users, groups, or service principals.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Applications` loads application and service principal commands.
> * `New-Item -ItemType Directory -Path ./reports -Force` creates the reports folder if it does not exist.
> * `Get-MgServicePrincipal -Filter "servicePrincipalType eq 'Application'"` retrieves enterprise applications represented as service principals.
> * `-Property "..."` requests fields needed for the report.
> * `foreach ($app in $apps)` loops through each enterprise application.
> * `Get-MgServicePrincipalAppRoleAssignedTo` retrieves users, groups, or service principals assigned to the app.
> * `[PSCustomObject]` creates one clean report row per application.
> * `AssignedUsers`, `AssignedGroups`, and `AssignedServicePrincipals` count assignment types.
> * `Format-Table` previews the report in the terminal.
> * `Export-Csv -NoTypeInformation` saves the report as a clean CSV file.
>
> **Watch out for:** Some apps may have no explicit assignments if assignment is not required. Also, some Microsoft first-party applications may appear in the enterprise apps list. In a production report, you may separate Microsoft apps from third-party or custom apps.

---

## Step 8: Export App Role Assignments for One Enterprise App

Use this optional script to inspect assignments for a specific app.

```powershell
# ================================
# Export App Role Assignments for One App
# ================================

Import-Module Microsoft.Graph.Applications

$appDisplayName = "AWS IAM Identity Center"

$app = Get-MgServicePrincipal `
    -Filter "displayName eq '$appDisplayName'" `
    -Property "id,displayName,appId" `
    -ErrorAction SilentlyContinue

if (-not $app) {
    Write-Host "Application not found: $appDisplayName" -ForegroundColor Yellow
    return
}

$assignments = Get-MgServicePrincipalAppRoleAssignedTo `
    -ServicePrincipalId $app.Id `
    -All

$assignmentReport = $assignments |
Select-Object `
    PrincipalDisplayName,
    PrincipalType,
    PrincipalId,
    AppRoleId,
    CreatedDateTime

$assignmentReport | Format-Table -AutoSize

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$assignmentReport |
Export-Csv -Path "./reports/app-role-assignments-$($app.DisplayName.Replace(' ','-')).csv" -NoTypeInformation

Write-Host "Assignment report exported for $($app.DisplayName)" -ForegroundColor Green
```

> #### 📘 Script explained
>
> **What it does:** Finds one enterprise application by display name and exports its app role assignments.
>
> **Why it matters:** App-specific reporting helps you prove who has access to one important application. This is useful for audits and access reviews.
>
> **Line by line:**
>
> * `$appDisplayName` stores the application you want to inspect.
> * `Get-MgServicePrincipal -Filter "displayName eq '$appDisplayName'"` finds the enterprise app.
> * `if (-not $app) { ... return }` stops safely if the app is not found.
> * `Get-MgServicePrincipalAppRoleAssignedTo` retrieves assigned users, groups, and service principals.
> * `Select-Object` creates readable output columns.
> * `Export-Csv` saves the assignments to a report file.
>
> **Watch out for:** The app display name must match the enterprise application name in your tenant. If it does not match, search for the app first using `Get-MgServicePrincipal -All | Where-Object DisplayName -like "*AWS*"`.

---

## Step 9: Verify Everything

Use this checklist to confirm completion:

* [ ] Workload identity guide created at `docs/workload-identity-guide.md`
* [ ] Workload identity types compared
* [ ] System-assigned managed identity created or documented
* [ ] User-assigned managed identity created or documented
* [ ] Managed identity granted Azure RBAC access or documented
* [ ] Managed identities listed with PowerShell
* [ ] SAML enterprise application added from the gallery or documented
* [ ] OIDC enterprise application added from the gallery or documented
* [ ] SSO protocol comparison created at `docs/sso-protocol-comparison.md`
* [ ] App assignment requirement reviewed
* [ ] App assignment requirement set to `Yes` on a test app
* [ ] Users or groups assigned to an enterprise app
* [ ] App roles reviewed or created
* [ ] Users or groups assigned to app roles
* [ ] User consent settings configured
* [ ] Permission classifications reviewed
* [ ] Admin consent workflow configured
* [ ] Admin consent workflow tested or documented
* [ ] Application collections created
* [ ] Application collections assigned to groups
* [ ] My Apps portal tested
* [ ] Enterprise application report exported to `./reports/enterprise-apps-report.csv`
* [ ] App role assignment report exported for one app

Recommended screenshots:

* Workload identity guide
* Azure managed identity page
* Azure RBAC role assignment
* Enterprise application overview
* SAML SSO configuration page
* OIDC app configuration page
* App properties showing `Assignment required`
* Users and groups assigned to an app
* App roles page
* Consent and permissions settings
* Admin consent workflow
* Application collections
* My Apps portal
* Enterprise app report CSV
* App role assignment report CSV

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

Your app registration may be missing Microsoft Graph Application permissions, admin consent, or the required administrator role.

Check:

```text
Microsoft Entra admin center
→ App registrations
→ Your app
→ API permissions
```

For enterprise applications, confirm:

```text
Application.ReadWrite.All
Directory.ReadWrite.All
```

For app role assignments, confirm:

```text
AppRoleAssignment.ReadWrite.All
```

For consent management, confirm:

```text
DelegatedPermissionGrant.ReadWrite.All
Policy.ReadWrite.PermissionGrant
```

After changing permissions, click:

```text
Grant admin consent
```

Then disconnect and reconnect to Microsoft Graph.

---

### Managed identity does not appear in Microsoft Entra ID

Check that:

* The Azure resource exists.
* The managed identity was enabled.
* You are looking in the correct tenant.
* Directory replication may need a few minutes.
* You have permission to read service principals.

---

### Storage account name is not accepted

Storage account names must be:

* Globally unique
* Lowercase only
* 3–24 characters
* Letters and numbers only

Try adding random numbers to the name.

---

### Enterprise application cannot be assigned to users

Check that:

* The application exists as an enterprise application.
* You have permission to manage the app.
* Assignment is supported for the application.
* The users or groups exist.
* You are not trying to assign a group where group assignment is not supported by your license or app configuration.

---

### User cannot access the enterprise application

Check:

* Is **Assignment required** set to `Yes`?
* Is the user assigned directly or through a group?
* Has group membership updated?
* Is the correct app role selected?
* Is there a Conditional Access policy blocking access?
* Is the SSO configuration complete?

---

### Admin consent request does not appear

Check:

* Admin consent workflow is enabled.
* The user attempted to access an app requiring permissions they cannot consent to.
* The reviewer is configured.
* The request did not expire.
* The app was not blocked by another policy.

---

### App role assignment report is empty

This may be normal if:

* Assignment is not required.
* No users or groups have been assigned.
* The app does not expose app roles.
* The selected app name does not match.

---

## Key Takeaways for SC-300

1. Use managed identities for Azure resources.
2. Use service principals for custom applications and automation.
3. Use enterprise applications for SaaS integration and SSO.
4. Avoid using user accounts for automation.
5. System-assigned managed identities are tied to one Azure resource.
6. User-assigned managed identities can be reused across multiple resources.
7. SAML uses XML assertions and is common for enterprise SSO.
8. OIDC uses JSON-based tokens and is common for modern apps.
9. App assignment controls who can access an enterprise application.
10. App roles provide application-specific authorization.
11. Directory roles, Azure RBAC roles, and app roles are different.
12. User consent should be restricted.
13. Admin consent workflow gives users a formal approval path.
14. Application collections organize apps in the My Apps portal.
15. Reports and screenshots provide evidence for portfolio and audit purposes.

---

## Portfolio / Interview Summary

In this lab, I planned and implemented workload identities and enterprise application access for the TeachRich Microsoft Entra tenant. I compared managed identities, service principals, enterprise applications, and user accounts. I created or documented system-assigned and user-assigned managed identities, reviewed Azure RBAC access for managed identities, integrated enterprise applications, compared SAML and OIDC, configured app assignment requirements, reviewed app roles, configured user and admin consent controls, created application collections, and exported enterprise application reports.

This project demonstrates practical SC-300 skills in workload identity planning, SaaS integration, managed identities, enterprise application governance, SSO concepts, consent management, app role assignments, and reporting.

---

## What's Next

➡️ **Lab 8:** [App Registrations, Permissions & Defender for Cloud Apps](../lab08-app-registrations/) — Custom app registrations, API permissions, Application Proxy, and cloud app security.

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
