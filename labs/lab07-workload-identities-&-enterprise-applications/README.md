# Lab 7: Workload Identities & Enterprise Applications

> **Type:** Big Project | **Time:** 3-4 hours | **SC-300 Domain:** Plan and implement workload identities (20-25%)

---

## Scenario

The development team at Expat Teacher's Lounge needs to integrate several applications with Entra ID. The company uses Salesforce for CRM, a custom internal application for resource management, and several Azure services that need programmatic access to company data. You need to understand the different identity types for applications and workloads, integrate enterprise apps with SSO, and configure proper consent and access controls.

---

## SC-300 Exam Objectives Covered

- Select appropriate identities for applications and Azure workloads
- Create managed identities (system-assigned and user-assigned)
- Assign and use managed identities to access Azure resources
- Plan and implement settings for enterprise applications
- Assign Microsoft Entra roles for managing enterprise apps
- Design and implement integration for SaaS apps
- Assign users, groups, and app roles to enterprise applications
- Configure and manage user and admin consent
- Create and manage application collections

---

## Prerequisites

- Completed Labs 1-6
- Microsoft 365 E5 subscription
- Azure subscription (for managed identity exercises)

---

## Step 1: Understand Workload Identity Types

Create a reference document (`docs/workload-identity-guide.md`):

| Identity Type | What It Is | When To Use | Example |
|---|---|---|---|
| **Managed Identity (System-Assigned)** | Identity tied to a specific Azure resource. Created and deleted with the resource. | An Azure VM that needs to read from Key Vault. An Azure Function that needs to query a database. | VM → Key Vault access |
| **Managed Identity (User-Assigned)** | Standalone identity you create. Can be shared across multiple resources. | Multiple VMs or Functions that need the same access. When the identity should persist if the resource is deleted. | Shared identity for a microservices cluster |
| **Service Principal** | An identity for an application or service. Created when you register an app or grant consent to an enterprise app. | Custom applications that need to authenticate to Entra ID. CI/CD pipelines. Automation scripts. | Your IAM Lifecycle Automation app registration |
| **Enterprise Application** | A service principal that represents a SaaS application in your tenant (from the gallery). | Integrating third-party apps like Salesforce, ServiceNow, Slack. | Salesforce SSO integration |
| **User Account** | A human identity. Never use for automation. | End-user access. | Employee sign-in |

**SC-300 exam tip:** The exam tests whether you can select the RIGHT identity type for a given scenario. A common wrong answer is using a user account for automation — always choose managed identity or service principal.

### How to explain this in an interview:

*"I choose the identity type based on the workload. For Azure resources that need to talk to each other, I use managed identities because they eliminate credential management entirely — no secrets or certificates to rotate. For custom applications and automation, I use service principals with certificate authentication. For SaaS integrations, I use enterprise applications from the Entra gallery. I never use user accounts for automation because they can't be properly governed and they break when someone changes their password."*

---

## Step 2: Create and Use Managed Identities

### 2a: Create a System-Assigned Managed Identity

You need an Azure resource to attach this to. Let's use a simple Storage Account:

1. Go to **portal.azure.com** → search **Storage accounts** → **+ Create**
2. **Resource group:** Create new → `RG-IAMLab`
3. **Storage account name:** `iamlabstorage` (must be globally unique — add random numbers if needed)
4. **Region:** Choose your nearest region
5. Click **Review + Create** → **Create**

Enable the system-assigned managed identity:
1. Go to the storage account → **Identity** (in left sidebar)
2. **System assigned** tab → toggle **Status** to **On**
3. Click **Save**
4. Note the **Object ID** — this is the identity's unique identifier

### 2b: Create a User-Assigned Managed Identity

1. Go to **portal.azure.com** → search **Managed identities** → **+ Create**
2. **Resource group:** `RG-IAMLab`
3. **Name:** `mi-iam-automation`
4. **Region:** Same as your storage account
5. Click **Review + Create** → **Create**

### 2c: Grant Managed Identity Access to a Resource

Give the system-assigned identity access to the storage account:

1. Go to your storage account → **Access control (IAM)**
2. Click **+ Add** → **Add role assignment**
3. **Role:** `Storage Blob Data Reader`
4. **Assign access to:** Managed identity → select your storage account's system-assigned identity
5. Click **Review + assign**

### 2d: Document with PowerShell

```powershell
# List all managed identities in your subscription
Get-MgServicePrincipal -Filter "servicePrincipalType eq 'ManagedIdentity'" |
    Select-Object DisplayName, AppId, Id,
        @{N='Type';E={if($_.AlternativeNames -match 'isExplicit=True'){'User-Assigned'}else{'System-Assigned'}}} |
    Format-Table -AutoSize
```

**SC-300 exam tip:** Know the difference between system-assigned (lifecycle tied to resource, one-to-one) and user-assigned (standalone, many-to-many). Know that managed identities eliminate the need for credential management — no secrets to rotate.

---

## Step 3: Add Enterprise Applications from the Gallery

**Where:** Entra ID → Enterprise apps → + New application

### 3a: Add a SAML Application

1. Click **+ New application**
2. Search for **AWS Single Sign-On** (or any SAML app available in the gallery)
3. Click **Create**
4. Go to the application → **Single sign-on** → select **SAML**
5. Configure the SAML settings:
   - **Identifier (Entity ID):** `urn:amazon:webservices` (or the app's default)
   - **Reply URL:** The app's ACS URL
   - **Sign-on URL:** The app's login URL
6. Download the **Federation Metadata XML** or **Certificate (Base64)** — in production, you'd upload this to the application's side
7. Click **Save**

### 3b: Add an OIDC Application

1. Click **+ New application**
2. Search for **Slack** (or another OIDC-based app)
3. Click **Create**
4. Go to the application → **Single sign-on**
5. OIDC apps are typically auto-configured — review the settings

### 3c: Document SSO Protocol Differences

Create `docs/sso-protocol-comparison.md`:

| Feature | SAML 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| Token format | XML (SAML assertion) | JSON (JWT token) |
| Primary use | Enterprise SSO, legacy apps | Modern web and mobile apps |
| Authentication flow | Browser redirect with POST binding | Authorization code flow with PKCE |
| Configuration | Manual — exchange metadata XML | Often auto-configured via discovery endpoint |
| Token size | Larger (XML) | Smaller (JSON) |
| Mobile support | Limited | Native support |
| When to choose | Legacy enterprise apps, on-prem apps | Modern SaaS apps, custom web apps, mobile apps |

**SC-300 exam tip:** Know SAML vs OIDC at a conceptual level. Know that SAML uses assertions (XML) and OIDC uses tokens (JWT). Know that most modern apps use OIDC, but many enterprise apps still use SAML.

---

## Step 4: Assign Users and Groups to Enterprise Apps

### 4a: Configure App Assignment Requirement

1. Go to your enterprise app → **Properties**
2. **Assignment required?** Set to **Yes**
3. Click **Save**

This means only users and groups you explicitly assign can access the app. Without this, all users in the tenant can access it.

### 4b: Assign Users and Groups

1. Go to the app → **Users and groups** → **+ Add user/group**
2. Select a security group (e.g., `SG-Engineering`)
3. If the app has roles, select the appropriate role
4. Click **Assign**

### 4c: Create Custom App Roles

1. Go to the app → **App roles** → **+ Create app role**
2. **Display name:** `Project Manager`
3. **Allowed member types:** Users/Groups
4. **Value:** `ProjectManager`
5. **Description:** `Can manage projects within the application`
6. Click **Apply**

Create additional roles:
- `Viewer` — Read-only access
- `Contributor` — Can create and edit
- `Admin` — Full application administration

Assign users to specific roles:
1. Go to **Users and groups** → **+ Add user/group**
2. Select a user → select the **Project Manager** role
3. Click **Assign**

**SC-300 exam tip:** Know app roles, how to create them, and how they differ from directory roles. App roles are application-specific; directory roles are tenant-wide. Know that roles are included in the token sent to the application.

---

## Step 5: Configure User and Admin Consent

**Where:** Entra ID → Enterprise apps → Consent and permissions

### 5a: Configure User Consent Settings

1. Go to **Consent and permissions** → **User consent settings**
2. Select **Allow user consent for apps from verified publishers, for selected permissions** (recommended)
3. Under the permission classification, click **Permission classifications**
4. Add `openid`, `profile`, `email`, `offline_access` as **Low impact** permissions
5. Click **Save**

This means users can consent to low-risk permissions themselves, but anything beyond that requires admin approval.

### 5b: Configure Admin Consent Workflow

1. Go to **Consent and permissions** → **Admin consent settings**
2. **Users can request admin consent to apps they are unable to consent to:** **Yes**
3. **Select users to review admin consent requests:** Add your admin account
4. **Selected users will receive email notifications for requests:** **Yes**
5. **Selected users will receive request expiration reminders:** **Yes**
6. **Consent request expires after (days):** **30**
7. Click **Save**

### 5c: Test the Consent Flow

1. Sign in as a regular user (incognito window)
2. Try to access an app that requires permissions the user can't consent to
3. The user should see an **approval required** screen with option to request admin consent
4. Sign in as admin → go to **Enterprise apps** → **Admin consent requests**
5. Review and approve or deny the request

**SC-300 exam tip:** Consent management is heavily tested. Know the three consent options (do not allow, allow for verified publishers with selected permissions, allow all). Know the admin consent workflow. Know what "verified publisher" means.

### How to explain this in an interview:

*"I configured the consent framework to balance security with usability. Users can consent to low-impact permissions like reading their own profile, but anything beyond that requires admin approval. We set up an admin consent workflow so when users need access to a new app, the request goes to the IAM team for review instead of being silently blocked."*

---

## Step 6: Create Application Collections

**Where:** Entra ID → Enterprise apps → Application collections

Application collections organize apps in the My Apps portal for users:

1. Click **+ New collection**
2. **Name:** `Engineering Tools`
3. Add engineering-related enterprise apps
4. Click **Create**

Create additional collections:
- `Finance & Accounting`
- `HR & People Operations`
- `Company-Wide`

Assign collections to groups:
1. Click on a collection → **Users and groups**
2. Add the appropriate department group
3. Users in that group will see the collection in their My Apps portal

Test it:
1. Go to `myapplications.microsoft.com` as a test user
2. Verify the collections appear with the correct apps

**SC-300 exam tip:** Know that application collections organize the My Apps portal experience. Know that collections can be targeted to specific groups.

---

## Step 7: Generate Enterprise App Reports

```powershell
# List all enterprise applications with assignment info
$apps = Get-MgServicePrincipal -All -Filter "servicePrincipalType eq 'Application'" |
    Where-Object { $_.AppId -ne $null }

foreach ($app in $apps) {
    $assignments = Get-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $app.Id -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        AppName = $app.DisplayName
        AppId = $app.AppId
        AssignedUsers = ($assignments | Where-Object { $_.PrincipalType -eq 'User' }).Count
        AssignedGroups = ($assignments | Where-Object { $_.PrincipalType -eq 'Group' }).Count
        SignInAudience = $app.SignInAudience
    }
} | Format-Table -AutoSize
```

---

## Step 8: Verify Everything

Checklist:

- [ ] Workload identity guide documented (managed identity vs service principal vs enterprise app)
- [ ] System-assigned managed identity created on Azure resource
- [ ] User-assigned managed identity created
- [ ] Managed identity granted access to a resource (RBAC)
- [ ] Enterprise app added from gallery (SAML)
- [ ] Enterprise app added from gallery (OIDC)
- [ ] SSO protocol comparison documented
- [ ] App assignment required: Yes
- [ ] Users and groups assigned to enterprise apps
- [ ] Custom app roles created (Viewer, Contributor, Admin, Project Manager)
- [ ] Users assigned to specific app roles
- [ ] User consent restricted to verified publishers with low-impact permissions
- [ ] Admin consent workflow configured and tested
- [ ] Application collections created for departments
- [ ] Enterprise app report generated

---

## Key Takeaways for SC-300

1. **Choose the right identity type** — managed identities for Azure resources, service principals for apps, enterprise apps for SaaS
2. **System-assigned** managed identities are tied to one resource; **user-assigned** can be shared
3. **SAML** for legacy enterprise apps; **OIDC** for modern apps
4. **App assignment required = Yes** restricts access to explicitly assigned users/groups
5. **App roles** provide application-specific RBAC inside the token
6. **User consent** should be restricted; use **admin consent workflow** for anything beyond basic permissions
7. **Application collections** organize the My Apps portal experience

---

## What's Next

➡️ **Lab 8:** [App Registrations, Permissions & Defender for Cloud Apps](../lab08-app-registrations/) — Custom app registrations, API permissions, Application Proxy, and cloud app security

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
