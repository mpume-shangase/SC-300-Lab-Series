# Lab 8: App Registrations, Permissions & Defender for Cloud Apps

> **Type:** Focused Lab | **Time:** 2–3 hours | **SC-300 Domain:** Plan and implement workload identities (20–25%)

---

## Scenario

The TeachRich development team is building a custom internal application that needs to authenticate users and access Microsoft Graph data. At the same time, the security team has discovered that employees are using unauthorized cloud applications without approval.

You have been asked to register the custom application properly, configure authentication settings, apply the correct Microsoft Graph permissions, create app roles, document Application Proxy for on-premises apps, and review Microsoft Defender for Cloud Apps to discover and control shadow IT.

The TeachRich verified domain used throughout this lab is:

```text
teachrich.com
```

The custom application used in this lab is:

```text
TeachRich Resource Manager
```

---

## SC-300 Exam Objectives Covered

* Plan for app registrations
* Create app registrations
* Configure app authentication
* Configure redirect URIs
* Configure credentials, including client secrets and certificates
* Configure delegated and application API permissions
* Create app roles
* Understand Application Proxy for on-premises applications
* Configure and analyze Cloud Discovery results in Defender for Cloud Apps
* Configure connected apps
* Implement application-enforced restrictions
* Configure Conditional Access App Control
* Create access and session policies in Defender for Cloud Apps
* Implement and manage policies for OAuth apps
* Manage the Cloud App Catalog

---

## Prerequisites

Before starting this lab, you should have:

* Completed Labs 1–7
* A Microsoft 365 E3, Microsoft 365 E5, or similar subscription
* Microsoft Defender for Cloud Apps available if you are completing the MDCA steps
* Microsoft Entra ID P1 or P2 features available
* A verified Microsoft Entra domain, such as `teachrich.com`
* PowerShell 7 installed
* Microsoft Graph PowerShell SDK installed
* Certificate-based Microsoft Graph authentication configured from Lab 1
* Test users and groups created from earlier labs

Example TeachRich groups used in this lab:

```text
DG-Engineering-All
DG-Finance-All
SG-Project-Alpha
SG-FIDO2-Enabled
```

> **Important:** Some Defender for Cloud Apps features depend on licensing, portal availability, and whether Microsoft 365 is connected as an app connector. If a feature is not available in your tenant, document the concept, the portal location, and what would be required in production.

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
DelegatedPermissionGrant.ReadWrite.All
Policy.Read.All
Policy.ReadWrite.PermissionGrant
ServicePrincipalEndpoint.ReadWrite.All
User.Read.All
Group.Read.All
Reports.Read.All
```

Then click:

```text
Grant admin consent
```

Use the permissions based on the tasks you plan to complete:

| Task                                          | Useful permissions                                     |
| --------------------------------------------- | ------------------------------------------------------ |
| Create or manage app registrations            | `Application.ReadWrite.All`                            |
| Manage enterprise apps and service principals | `Application.ReadWrite.All`, `Directory.ReadWrite.All` |
| Assign app roles                              | `AppRoleAssignment.ReadWrite.All`                      |
| Read users and groups                         | `User.Read.All`, `Group.Read.All`                      |
| Review or manage consent grants               | `DelegatedPermissionGrant.ReadWrite.All`               |
| Manage permission grant policies              | `Policy.ReadWrite.PermissionGrant`                     |
| Read usage reports                            | `Reports.Read.All`                                     |
| Read policy settings                          | `Policy.Read.All`                                      |

> **Troubleshooting note:** If you add or change API permissions, always click **Grant admin consent**, then disconnect and reconnect to Microsoft Graph.

> #### 📘 Why these permissions matter
>
> App registrations and enterprise applications are powerful because they define how applications authenticate and what data they can access.
>
> `Application.ReadWrite.All` allows the app to manage application registrations and service principals. `AppRoleAssignment.ReadWrite.All` allows assignment of users, groups, or service principals to app roles. Consent-related permissions are sensitive because they control which apps can access organizational data.
>
> In production, use least privilege. Avoid granting broad application permissions unless they are clearly required and approved.

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
> **Why it matters:** App registrations, permissions, and enterprise app reports are often managed or audited through automation. PowerShell gives you a repeatable way to create evidence for your GitHub portfolio.
>
> **Line by line:**
>
> * `$TenantId` identifies the TeachRich tenant.
> * `$ClientId` identifies the app registration that has Microsoft Graph permissions.
> * `$TenantDomain = "teachrich.com"` stores the verified domain for reuse in lab scripts.
> * `Read-Host ... -AsSecureString` prompts for the PFX password without showing it on screen.
> * `New-Object ... X509Certificate2(...)` loads the certificate and private key from the `.pfx` file.
> * `Connect-MgGraph ... -Certificate $cert` authenticates to Microsoft Graph as the application.
> * `Get-MgContext` confirms that the connection is active.
>
> **Watch out for:** Do not upload your `.pfx` file, certificate password, client secrets, or real app credentials to GitHub.

---

## Step 1: Plan and Create an App Registration

An app registration defines how an application integrates with Microsoft Entra ID.

**Where:** Microsoft Entra admin center → Identity → Applications → App registrations → New registration

---

### 1a: Register the Application

Create a new app registration.

Use these values:

| Field                   | Value                                          |
| ----------------------- | ---------------------------------------------- |
| Name                    | `TeachRich Resource Manager`                   |
| Supported account types | Accounts in this organizational directory only |
| Platform                | Web                                            |
| Redirect URI            | `https://localhost:3000/auth/callback`         |

Click **Register**.

Save these values from the Overview page:

```text
Application (client) ID
Directory (tenant) ID
Object ID
```

### Why this matters

The app registration is the application definition. It tells Microsoft Entra ID how the app signs users in, what redirect URIs are trusted, and what permissions the app may request.

---

### 1b: Add Additional Redirect URIs

Go to the app registration → **Authentication**.

Under **Web**, add a production redirect URI:

```text
https://app.teachrich.com/auth/callback
```

Under **Implicit grant and hybrid flows**:

| Setting       | Recommended Value                                             |
| ------------- | ------------------------------------------------------------- |
| ID tokens     | Enabled if the app uses OIDC sign-in                          |
| Access tokens | Leave unchecked for modern apps using authorization code flow |

Under **Advanced settings**:

| Setting                   | Recommended Value |
| ------------------------- | ----------------- |
| Allow public client flows | No                |

Click **Save**.

### Why this matters

Redirect URIs are security boundaries. Microsoft Entra ID will only send tokens to redirect URIs that are registered on the app.

### SC-300 exam tip

Know the difference between redirect URI platform types:

| Platform                | Use Case                        |
| ----------------------- | ------------------------------- |
| Web                     | Server-side web applications    |
| Single-page application | Browser-based JavaScript apps   |
| Mobile and desktop      | Native apps and desktop clients |

Also know that implicit grant is considered legacy for many modern scenarios. Authorization code flow with PKCE is the recommended modern pattern.

---

### 1c: Optional PowerShell: Create the App Registration

You can create the app registration with PowerShell for repeatable automation.

```powershell
# ================================
# Create TeachRich Resource Manager App Registration
# ================================

Import-Module Microsoft.Graph.Applications

$appName = "TeachRich Resource Manager"
$redirectUri = "https://localhost:3000/auth/callback"

try {
    $existingApp = Get-MgApplication `
        -Filter "displayName eq '$appName'" `
        -ErrorAction SilentlyContinue

    if ($existingApp) {
        Write-Host "[EXISTS] App registration already exists: $appName" -ForegroundColor Yellow
        $existingApp | Select-Object DisplayName, AppId, Id
        return
    }

    $app = New-MgApplication `
        -DisplayName $appName `
        -SignInAudience "AzureADMyOrg" `
        -Web @{
            RedirectUris = @($redirectUri)
        } `
        -ErrorAction Stop

    Write-Host "[CREATED] App registration: $($app.DisplayName)" -ForegroundColor Green
    Write-Host "Application Client ID: $($app.AppId)" -ForegroundColor Cyan
    Write-Host "Application Object ID: $($app.Id)" -ForegroundColor Cyan
}
catch {
    Write-Host "[FAILED] Could not create app registration" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Creates a single-tenant app registration called `TeachRich Resource Manager` with a local development redirect URI.
>
> **Why it matters:** App registration creation can be automated. This is useful when teams need consistent app setup across development, test, and production tenants.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Applications` loads the Graph commands for app registrations.
> * `$appName` stores the application name.
> * `$redirectUri` stores the trusted callback URL.
> * `Get-MgApplication -Filter "displayName eq '$appName'"` checks whether the app already exists.
> * `if ($existingApp) { ... return }` prevents duplicate app registrations.
> * `New-MgApplication` creates the app registration.
> * `-SignInAudience "AzureADMyOrg"` makes the app single-tenant.
> * `-Web @{ RedirectUris = @($redirectUri) }` configures the web platform and redirect URI.
> * `try/catch` gives clear success or failure output.
>
> **Watch out for:** Only register redirect URIs that you control. Do not add random or unverified URLs to an app registration.

---

## Step 2: Configure API Permissions

API permissions define what the application can access.

**Where:** App registration → API permissions

---

### 2a: Add Delegated Permissions

Delegated permissions act **on behalf of a signed-in user**.

Add the following Microsoft Graph delegated permissions:

```text
User.Read
User.ReadBasic.All
Group.Read.All
Calendars.Read
```

Steps:

1. Click **Add a permission**.
2. Select **Microsoft Graph**.
3. Select **Delegated permissions**.
4. Add the permissions listed above.
5. Click **Add permissions**.

### Why this matters

Delegated permissions are used when a user signs in to the app. The app can only do what the permission allows and what the signed-in user is allowed to do.

---

### 2b: Add Application Permissions

Application permissions act **as the app itself**, with no signed-in user.

Add the following Microsoft Graph application permissions:

```text
User.Read.All
Reports.Read.All
```

Steps:

1. Click **Add a permission**.
2. Select **Microsoft Graph**.
3. Select **Application permissions**.
4. Add the permissions listed above.
5. Click **Add permissions**.
6. Click **Grant admin consent for TeachRich**.
7. Confirm that green checkmarks appear.

### Why this matters

Application permissions are powerful because they are not limited by a signed-in user. They usually require admin consent and should be granted carefully.

---

### 2c: Document Delegated vs Application Permissions

Create this file:

```text
docs/api-permissions-guide.md
```

Use this table:

| Feature         | Delegated Permissions                    | Application Permissions                   |
| --------------- | ---------------------------------------- | ----------------------------------------- |
| User context    | Yes, acts as the signed-in user          | No, acts as the app itself                |
| Consent         | User consent or admin consent            | Admin consent required                    |
| Token contains  | User identity and scopes                 | App identity and roles                    |
| Best use case   | Web apps where users sign in             | Background services, daemons, automation  |
| Access boundary | Permission plus the user’s own access    | Full permission scope granted to the app  |
| Example         | Read my calendar                         | Read all user profiles                    |
| Risk level      | Lower, because user access still matters | Higher, because there is no user boundary |

### SC-300 exam tip

This is heavily tested.

Know this:

* Delegated permissions = user context.
* Application permissions = app context.
* Delegated access is limited by both the permission and the user’s own access.
* Application access is limited only by the permission granted to the app.
* Application permissions require admin consent.

### How to explain this in an interview

> I choose API permissions based on the application model. For apps where a user signs in, I use delegated permissions because access is bounded by the user’s own permissions. For background automation or service-to-service access, I use application permissions and require admin consent. I always apply least privilege and avoid granting broad permissions unless they are required.

---

### 2d: Optional PowerShell: Export App Permissions

```powershell
# ================================
# Export App Registration Permissions
# ================================

Import-Module Microsoft.Graph.Applications

$appName = "TeachRich Resource Manager"

$app = Get-MgApplication `
    -Filter "displayName eq '$appName'" `
    -Property "id,appId,displayName,requiredResourceAccess" `
    -ErrorAction SilentlyContinue

if (-not $app) {
    Write-Host "App registration not found: $appName" -ForegroundColor Yellow
    return
}

$permissionReport = foreach ($resource in $app.RequiredResourceAccess) {
    foreach ($access in $resource.ResourceAccess) {
        [PSCustomObject]@{
            AppName       = $app.DisplayName
            AppId         = $app.AppId
            ResourceAppId = $resource.ResourceAppId
            PermissionId  = $access.Id
            PermissionType = $access.Type
        }
    }
}

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$permissionReport | Format-Table -AutoSize

$permissionReport |
Export-Csv -Path "./reports/teachrich-resource-manager-permissions.csv" -NoTypeInformation

Write-Host "Permission report exported to ./reports/teachrich-resource-manager-permissions.csv" -ForegroundColor Green
```

> #### 📘 Script explained
>
> **What it does:** Reads the API permissions configured on the `TeachRich Resource Manager` app registration and exports them to a CSV file.
>
> **Why it matters:** API permission reporting is useful for audit evidence. It helps show what permissions an app is requesting.
>
> **Line by line:**
>
> * `$appName` stores the application to inspect.
> * `Get-MgApplication -Filter "displayName eq '$appName'"` finds the app registration.
> * `RequiredResourceAccess` contains the API permissions requested by the app.
> * The nested `foreach` loops through each API resource and each permission.
> * `PermissionType` shows whether the permission is a delegated scope or an application role.
> * `Export-Csv` saves the permission evidence to a report.
>
> **Watch out for:** This report shows permission IDs rather than friendly permission names. For portfolio notes, combine the CSV with a portal screenshot showing the friendly names.

---

## Step 3: Configure App Credentials

Credentials allow the application to authenticate.

**Where:** App registration → Certificates & secrets

---

### 3a: Create a Client Secret for Development

1. Go to **Certificates & secrets**.
2. Click **Client secrets**.
3. Click **New client secret**.
4. Use this description:

```text
Dev environment secret
```

5. Choose a short expiry such as:

```text
6 months
```

6. Click **Add**.
7. Copy the secret value immediately.

> **Important:** You can only view the client secret value once. Do not paste it into GitHub, screenshots, notes, or messages.

### Why this matters

Client secrets are easy to create, but they are sensitive credentials. They are acceptable for labs and development but should be avoided for production where possible.

---

### 3b: Upload a Certificate for Production-Style Authentication

1. Go to **Certificates**.
2. Click **Upload certificate**.
3. Upload the public certificate file, such as a `.cer` or `.pem` file.
4. Click **Add**.

### Why this matters

Certificates are preferred for production workloads because the private key does not need to be stored as a plain text secret in the application configuration.

---

### 3c: Document Credential Security

Create this file:

```text
docs/app-credential-security.md
```

Use this content:

```markdown
# App Credential Security

## Client Secrets

Client secrets are convenient for development, but they are risky for production because they can be copied, leaked, or accidentally committed to source control.

## Certificates

Certificates are preferred for production workloads. The app registration stores the public certificate, while the private key remains protected on the system or in a secure store.

## Production Recommendations

- Prefer certificates or workload identity federation over client secrets.
- Use the shortest reasonable credential lifetime.
- Rotate credentials before expiry.
- Store credentials in a secure vault.
- Never commit secrets, certificates with private keys, or `.pfx` files to GitHub.
- Monitor app credential expiry.
```

### SC-300 exam tip

Know the credential options:

| Credential Type      | Use Case                                      |
| -------------------- | --------------------------------------------- |
| Client secret        | Simple development or short-term testing      |
| Certificate          | Production app authentication                 |
| Federated credential | Modern secretless workload identity scenarios |

---

### 3d: Optional PowerShell: Report App Credentials

```powershell
# ================================
# Report App Credentials
# ================================

Import-Module Microsoft.Graph.Applications

$appName = "TeachRich Resource Manager"

$app = Get-MgApplication `
    -Filter "displayName eq '$appName'" `
    -Property "id,displayName,appId,passwordCredentials,keyCredentials" `
    -ErrorAction SilentlyContinue

if (-not $app) {
    Write-Host "App registration not found: $appName" -ForegroundColor Yellow
    return
}

$credentialReport = @()

foreach ($secret in $app.PasswordCredentials) {
    $credentialReport += [PSCustomObject]@{
        AppName      = $app.DisplayName
        CredentialType = "Client Secret"
        DisplayName  = $secret.DisplayName
        StartDate    = $secret.StartDateTime
        EndDate      = $secret.EndDateTime
    }
}

foreach ($cert in $app.KeyCredentials) {
    $credentialReport += [PSCustomObject]@{
        AppName      = $app.DisplayName
        CredentialType = "Certificate"
        DisplayName  = $cert.DisplayName
        StartDate    = $cert.StartDateTime
        EndDate      = $cert.EndDateTime
    }
}

$credentialReport | Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Lists the secrets and certificates configured on the app registration, including their start and end dates.
>
> **Why it matters:** Credential expiry is an operational risk. If an app credential expires unexpectedly, the application may stop working.
>
> **Line by line:**
>
> * `PasswordCredentials` contains client secrets.
> * `KeyCredentials` contains certificates.
> * The first `foreach` loop adds secrets to the report.
> * The second `foreach` loop adds certificates to the report.
> * The report intentionally does not show secret values.
>
> **Watch out for:** Microsoft Graph does not reveal the client secret value after creation. That is expected and secure.

---

## Step 4: Create App Roles

App roles provide application-specific authorization.

**Where:** App registration → App roles

Create the following roles:

| Display Name     | Value                 | Allowed Member Types | Description                            |
| ---------------- | --------------------- | -------------------- | -------------------------------------- |
| Administrator    | `App.Admin`           | Users/Groups         | Full application administration        |
| Resource Manager | `App.ResourceManager` | Users/Groups         | Can create, edit, and delete resources |
| Viewer           | `App.Viewer`          | Users/Groups         | Read-only access to resources          |
| API Reader       | `App.APIReader`       | Applications         | Service-to-service read access         |

### Why this matters

App roles are included in tokens as role claims. The application can read those role claims and decide what the user or service is allowed to do.

---

### Assign Users to App Roles

App roles are defined on the **app registration**, but assigned on the **enterprise application**.

Steps:

1. Go to **Enterprise applications**.
2. Find:

```text
TeachRich Resource Manager
```

3. Go to **Users and groups**.
4. Click **Add user/group**.
5. Select a user or group.
6. Select a role such as:

```text
Resource Manager
```

7. Click **Assign**.

### SC-300 exam tip

Know this distinction:

| Item            | Where It Is Defined                      | Where It Is Assigned                                |
| --------------- | ---------------------------------------- | --------------------------------------------------- |
| App role        | App registration                         | Enterprise application                              |
| Directory role  | Microsoft Entra roles and administrators | User, group, or service principal                   |
| Azure RBAC role | Azure resource IAM                       | User, group, service principal, or managed identity |

---

## Step 5: Configure Application Proxy

Application Proxy publishes on-premises web applications through Microsoft Entra ID. Users can access internal web apps from the internet with Microsoft Entra pre-authentication and Conditional Access protection, without using a VPN.

**Where:** Microsoft Entra admin center → Enterprise applications → Application Proxy

---

### 5a: Document the Architecture

Create this file:

```text
docs/application-proxy-architecture.md
```

Use this diagram:

```text
User on the internet
        ↓
Microsoft Entra ID sign-in
        ↓
Conditional Access and MFA
        ↓
Application Proxy cloud service
        ↓
Application Proxy connector
        ↓
Internal on-premises web application
```

### Why this matters

The Application Proxy connector makes outbound connections to Microsoft cloud services. This means you do not need to open inbound firewall ports to publish the internal app.

---

### 5b: Document the Configuration Steps

If you have an on-premises test environment, complete the steps. If not, document them.

1. Install the Application Proxy connector on a Windows Server.
2. Ensure the server has outbound HTTPS access.
3. In Microsoft Entra admin center, click **Add application**.
4. Select **Configure an on-premises application**.
5. Use an internal URL, such as:

```text
http://internalapp.teachrich.local:8080
```

6. Use an external URL, such as:

```text
https://teachrich-internal.msappproxy.net
```

7. Set **Pre-authentication** to:

```text
Microsoft Entra ID
```

8. Select the connector group.
9. Configure the appropriate SSO method.

Common SSO methods:

| SSO Method                                | Use Case                                       |
| ----------------------------------------- | ---------------------------------------------- |
| Microsoft Entra ID disabled / passthrough | App handles authentication itself              |
| Password-based                            | App uses a traditional username/password form  |
| Integrated Windows Authentication         | Kerberos-based internal apps                   |
| Header-based                              | Apps that accept identity through HTTP headers |

### SC-300 exam tip

Know the key Application Proxy concepts:

* No VPN required.
* Connector uses outbound connections.
* Microsoft Entra pre-authentication protects the app before users reach it.
* Conditional Access can be applied to the published app.
* Useful for internal web apps that are not ready to move to the cloud.

---

## Step 6: Configure Microsoft Defender for Cloud Apps

Microsoft Defender for Cloud Apps helps discover shadow IT, monitor cloud app usage, review OAuth apps, and apply session controls.

**Where:** Microsoft Defender portal → Cloud apps

---

### 6a: Access the Defender for Cloud Apps Portal

Go to:

```text
https://security.microsoft.com
```

Then open:

```text
Cloud apps
```

You may also see the standalone portal:

```text
https://portal.cloudappsecurity.com
```

### Why this matters

Defender for Cloud Apps gives security teams visibility into cloud apps, user activity, OAuth apps, and risky behavior.

---

### 6b: Review Cloud Discovery

Go to:

```text
Cloud apps → Cloud Discovery → Dashboard
```

Review:

| Item              | Meaning                                       |
| ----------------- | --------------------------------------------- |
| Discovered apps   | Cloud apps detected from logs or integrations |
| Users             | Users accessing cloud apps                    |
| Traffic           | Amount of data uploaded/downloaded            |
| Risk score        | Microsoft risk rating for the app             |
| Sanctioned apps   | Approved apps                                 |
| Unsanctioned apps | Apps not approved by the organization         |

### Why this matters

Cloud Discovery helps identify shadow IT. For example, users may be using personal file sharing, AI tools, or project management apps that the security team has not approved.

---

### 6c: Connect Microsoft 365

Go to:

```text
Settings → Cloud Apps → App connectors
```

Then:

1. Click **Connect an app**.
2. Select **Microsoft 365**.
3. Follow the prompts.
4. Confirm the connector status.

### Why this matters

Connecting Microsoft 365 allows Defender for Cloud Apps to monitor activity in Microsoft services such as SharePoint, OneDrive, Exchange, and Teams.

---

### 6d: Create an Activity Policy

Create a policy to alert on suspicious behavior.

**Where:** Defender for Cloud Apps → Policies → Policy management → Create policy → Activity policy

Use these values:

| Setting           | Value                                            |
| ----------------- | ------------------------------------------------ |
| Name              | `Alert on Mass File Download`                    |
| Activity type     | Download file                                    |
| Repeated activity | More than 10 times in 5 minutes by the same user |
| Severity          | High                                             |
| Action            | Alert only for the lab                           |

> **Important:** In a lab, start with alert-only. Avoid automatically suspending users until the policy has been tested.

### Why this matters

Mass file download activity may indicate data theft, compromised accounts, or insider risk.

---

### 6e: Create a Session Policy

Create a session policy to control downloads from unmanaged devices.

**Where:** Defender for Cloud Apps → Policies → Policy management → Create policy → Session policy

Use these values:

| Setting              | Value                                   |
| -------------------- | --------------------------------------- |
| Name                 | `Block Download from Unmanaged Devices` |
| Session control type | Control file download with inspection   |
| App                  | SharePoint Online                       |
| Device tag           | Does not equal compliant or corporate   |
| Action               | Block download                          |

### Why this matters

Session policies control what users can do **inside** a cloud app after sign-in. For example, a user may be allowed to view SharePoint in the browser from an unmanaged device but blocked from downloading files.

### SC-300 exam tip

Session policies usually require Conditional Access App Control integration. Know that Conditional Access controls access to the app, while Defender for Cloud Apps session controls actions inside the app.

---

### 6f: Review OAuth Apps

Go to:

```text
Cloud apps → OAuth apps
```

or:

```text
App governance
```

Review apps that users have consented to.

Look for risky signs:

* High-privilege permissions
* Unverified publisher
* Unused apps
* Apps with many users
* Apps requesting read/write access to mail, files, or users
* Suspicious names or publishers

Possible actions:

| Action           | Meaning                                      |
| ---------------- | -------------------------------------------- |
| Investigate      | Review app publisher, permissions, and users |
| Revoke           | Remove the app’s access                      |
| Ban / Unsanction | Mark the app as not approved                 |
| Notify users     | Warn users about risky apps                  |

### Why this matters

OAuth app abuse is a common way attackers maintain access. A user may consent to a malicious app that continues accessing data even after the user closes the browser.

---

### 6g: Manage the Cloud App Catalog

Go to:

```text
Cloud app catalog
```

Search for common apps such as:

```text
Dropbox
Zoom
Canva
Google Drive
ChatGPT
```

Review:

| Area         | Meaning                               |
| ------------ | ------------------------------------- |
| Risk score   | Overall risk rating                   |
| Security     | Security controls available           |
| Compliance   | Certifications and compliance posture |
| Legal        | Terms, privacy, and data handling     |
| Sanctioned   | Approved for use                      |
| Unsanctioned | Not approved for use                  |

Tag at least one app as:

```text
Sanctioned
```

and one as:

```text
Unsanctioned
```

### Why this matters

The Cloud App Catalog helps security teams evaluate cloud services and decide which apps are approved or blocked.

### How to explain this in an interview

> I used Defender for Cloud Apps to identify shadow IT and review cloud app risk. I connected Microsoft 365, reviewed discovered apps, created an activity policy for mass downloads, created a session policy to block downloads from unmanaged devices, reviewed OAuth apps with excessive permissions, and used the Cloud App Catalog to sanction or unsanction apps based on risk.

---

## Step 7: Generate App Registration and Enterprise App Reports

Create a `reports` folder and export a report of app registrations and enterprise apps.

---

### 7a: Export App Registrations

```powershell
# ================================
# Export App Registrations Report
# ================================

Import-Module Microsoft.Graph.Applications

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$appRegistrations = Get-MgApplication `
    -All `
    -Property "id,appId,displayName,signInAudience,createdDateTime,passwordCredentials,keyCredentials"

$appRegReport = $appRegistrations |
Select-Object `
    DisplayName,
    AppId,
    SignInAudience,
    CreatedDateTime,
    @{Name="ClientSecrets";Expression={$_.PasswordCredentials.Count}},
    @{Name="Certificates";Expression={$_.KeyCredentials.Count}}

$appRegReport |
    Sort-Object DisplayName |
    Format-Table -AutoSize

$appRegReport |
    Sort-Object DisplayName |
    Export-Csv -Path "./reports/app-registrations-report.csv" -NoTypeInformation

Write-Host "App registrations report exported to ./reports/app-registrations-report.csv" -ForegroundColor Green
Write-Host "Total app registrations reported: $($appRegReport.Count)" -ForegroundColor Cyan
```

> #### 📘 Script explained
>
> **What it does:** Exports a CSV report of app registrations, including sign-in audience and the number of secrets and certificates.
>
> **Why it matters:** App registration reporting helps identify applications that may have risky credential configurations, such as too many client secrets or missing certificates.
>
> **Line by line:**
>
> * `Get-MgApplication -All` retrieves all app registrations.
> * `-Property "..."` requests fields needed for the report.
> * `PasswordCredentials.Count` counts client secrets.
> * `KeyCredentials.Count` counts certificates.
> * `Export-Csv` saves the report as evidence.
>
> **Watch out for:** This report does not expose secret values. It only reports the presence and count of credentials.

---

### 7b: Export Enterprise Applications

```powershell
# ================================
# Export Enterprise Applications Report
# ================================

Import-Module Microsoft.Graph.Applications

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$enterpriseApps = Get-MgServicePrincipal `
    -All `
    -Filter "servicePrincipalType eq 'Application'" `
    -Property "id,appId,displayName,accountEnabled,servicePrincipalType,signInAudience"

$enterpriseAppReport = foreach ($app in $enterpriseApps) {

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

$enterpriseAppReport |
    Sort-Object AppName |
    Format-Table AppName, AccountEnabled, AssignedUsers, AssignedGroups -AutoSize

$enterpriseAppReport |
    Sort-Object AppName |
    Export-Csv -Path "./reports/enterprise-applications-report.csv" -NoTypeInformation

Write-Host "Enterprise applications report exported to ./reports/enterprise-applications-report.csv" -ForegroundColor Green
Write-Host "Total enterprise applications reported: $($enterpriseAppReport.Count)" -ForegroundColor Cyan
```

> #### 📘 Script explained
>
> **What it does:** Exports enterprise applications and counts their user, group, and service principal assignments.
>
> **Why it matters:** Enterprise app reporting supports application governance. It helps you understand which apps exist and who has access.
>
> **Line by line:**
>
> * `Get-MgServicePrincipal -Filter "servicePrincipalType eq 'Application'"` retrieves enterprise applications.
> * `Get-MgServicePrincipalAppRoleAssignedTo` retrieves app role assignments.
> * `AssignedUsers`, `AssignedGroups`, and `AssignedServicePrincipals` count assignment types.
> * `Export-Csv` saves the report to the reports folder.
>
> **Watch out for:** Some apps may appear with zero assignments if assignment is not required or if they are Microsoft first-party applications.

---

### 7c: Export OAuth Permission Grants

```powershell
# ================================
# Export OAuth Permission Grants
# ================================

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

try {
    $oauthGrants = Invoke-MgGraphRequest `
        -Method GET `
        -Uri "https://graph.microsoft.com/v1.0/oauth2PermissionGrants"

    $oauthGrantReport = $oauthGrants.value |
    Select-Object `
        Id,
        ClientId,
        ConsentType,
        PrincipalId,
        ResourceId,
        Scope

    $oauthGrantReport | Format-Table -AutoSize

    $oauthGrantReport |
    Export-Csv -Path "./reports/oauth-permission-grants-report.csv" -NoTypeInformation

    Write-Host "OAuth permission grants report exported to ./reports/oauth-permission-grants-report.csv" -ForegroundColor Green
    Write-Host "Total OAuth grants reported: $($oauthGrantReport.Count)" -ForegroundColor Cyan
}
catch {
    Write-Host "[FAILED] Could not export OAuth permission grants" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Exports OAuth delegated permission grants from the tenant.
>
> **Why it matters:** OAuth grants show where users or admins have allowed applications to access data. Reviewing these grants helps identify risky or excessive consent.
>
> **Line by line:**
>
> * `Invoke-MgGraphRequest` sends a direct request to Microsoft Graph.
> * The `/oauth2PermissionGrants` endpoint returns delegated permission grants.
> * `ConsentType` shows whether consent was granted for one user or for all users.
> * `Scope` lists the delegated permissions granted.
> * `Export-Csv` saves the report.
>
> **Watch out for:** The report may show IDs instead of friendly names. Use portal screenshots or additional lookup scripts for easier reading.

---

## Step 8: Verify Everything

Use this checklist to confirm completion:

* [ ] App registration created for `TeachRich Resource Manager`
* [ ] App registration uses `teachrich.com` branding
* [ ] Development redirect URI configured
* [ ] Production redirect URI configured
* [ ] Application Client ID recorded
* [ ] Directory Tenant ID recorded
* [ ] Delegated permissions added
* [ ] Application permissions added
* [ ] Admin consent granted where required
* [ ] Delegated vs application permissions guide created at `docs/api-permissions-guide.md`
* [ ] Client secret created for development
* [ ] Certificate uploaded for production-style authentication
* [ ] Credential security document created at `docs/app-credential-security.md`
* [ ] App roles created
* [ ] Users or groups assigned to app roles through the enterprise application
* [ ] Application Proxy architecture documented
* [ ] Defender for Cloud Apps portal accessed
* [ ] Cloud Discovery dashboard reviewed
* [ ] Microsoft 365 connected as an app connector, if available
* [ ] Activity policy created or documented
* [ ] Session policy created or documented
* [ ] OAuth apps reviewed
* [ ] Cloud App Catalog reviewed
* [ ] At least one app sanctioned or unsanctioned
* [ ] App registrations report exported to `./reports/app-registrations-report.csv`
* [ ] Enterprise applications report exported to `./reports/enterprise-applications-report.csv`
* [ ] OAuth permission grants report exported to `./reports/oauth-permission-grants-report.csv`

Recommended screenshots:

* App registration overview
* Authentication redirect URI settings
* API permissions page
* Admin consent confirmation
* Certificates and secrets page, with values redacted
* App roles page
* Enterprise app users and groups assignment
* Application Proxy page
* Defender for Cloud Apps dashboard
* Cloud Discovery dashboard
* Microsoft 365 app connector
* Activity policy
* Session policy
* OAuth apps page
* Cloud App Catalog
* App registrations report CSV
* Enterprise applications report CSV
* OAuth permission grants report CSV

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

For app registration management, confirm:

```text
Application.ReadWrite.All
Directory.ReadWrite.All
```

For app role assignments, confirm:

```text
AppRoleAssignment.ReadWrite.All
```

For consent and OAuth grant reporting, confirm:

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

### Redirect URI error during sign-in

Check that:

* The redirect URI in the app matches the URI used by the application exactly.
* The protocol is correct: `https` vs `http`.
* The path is correct.
* There is no missing trailing slash.
* The correct platform type is configured.

---

### Admin consent button is unavailable

Check that:

* You are signed in as an administrator with the correct role.
* The app has permissions requiring admin consent.
* You are viewing the correct app registration.
* The tenant allows admin consent by authorized administrators.

---

### Client secret value disappeared

This is expected. The value is only shown once.

Create a new secret if needed, and store it securely.

Do not upload the value to GitHub.

---

### App role does not appear for assignment

Check that:

* The app role is enabled.
* The allowed member type is correct.
* You are assigning the role from the enterprise application, not only viewing the app registration.
* The app registration and enterprise application are linked correctly.

---

### Defender for Cloud Apps is not visible

Check that:

* The tenant has the correct Microsoft Defender licensing.
* You have the correct security administrator permissions.
* You are using the Microsoft Defender portal.
* Cloud app security features may take time to appear after licensing.

---

### Session policy is not working

Check that:

* Conditional Access App Control is configured.
* The app is supported.
* The user session is routed through Defender for Cloud Apps.
* The policy filters match the test scenario.
* The user is using a browser session where session control can apply.

---

### OAuth apps list is empty

This may be normal in a clean lab tenant.

Check:

* Users have actually consented to apps.
* Microsoft 365 is connected.
* App governance or OAuth app visibility is available in your tenant.
* You have permission to view OAuth app data.

---

## Key Takeaways for SC-300

1. App registrations define how applications integrate with Microsoft Entra ID.
2. Enterprise applications represent app instances in your tenant.
3. Redirect URIs must match exactly.
4. Delegated permissions act on behalf of a signed-in user.
5. Application permissions act as the app itself.
6. Application permissions usually require admin consent.
7. Certificates are preferred over client secrets for production workloads.
8. App roles provide application-level authorization.
9. App roles are defined on the app registration and assigned on the enterprise application.
10. Application Proxy publishes internal web apps without VPN.
11. Defender for Cloud Apps helps discover shadow IT.
12. Cloud Discovery identifies unsanctioned cloud applications.
13. Session policies control actions inside cloud apps.
14. OAuth app review helps detect risky third-party consent.
15. Reports and screenshots provide evidence for portfolio and audit purposes.

---

## Portfolio / Interview Summary

In this lab, I created and configured an app registration for the TeachRich Resource Manager application. I configured redirect URIs, added delegated and application Microsoft Graph permissions, granted admin consent where required, created development and production-style credentials, documented credential security, created app roles, assigned users and groups through the enterprise application, documented Application Proxy, reviewed Defender for Cloud Apps, created cloud app security policies, reviewed OAuth apps, and exported app registration, enterprise application, and OAuth permission grant reports.

This project demonstrates practical SC-300 skills in app registrations, workload identity security, delegated and application permissions, consent governance, app roles, Application Proxy concepts, Defender for Cloud Apps, shadow IT discovery, OAuth app review, and reporting.

---

## What's Next

➡️ **Lab 9:** [Identity Governance: Entitlements, Access Reviews & PIM](../lab09-identity-governance/) — The complete governance framework for audit readiness.

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
