# Lab 6: Conditional Access & Identity Protection

> **Type:** Big Project | **Time:** 3–4 hours | **SC-300 Domain:** Implement authentication and access management (25–30%)

---

## Scenario

TeachRich has adopted a Zero Trust security model. The leadership team wants proof that access is controlled based on who the user is, where they are signing in from, what application they are accessing, what device they are using, and how risky the sign-in appears.

You have been asked to design and implement a Conditional Access framework for the TeachRich Microsoft Entra tenant. You will also configure Identity Protection so risky users and risky sign-ins can be detected and handled automatically.

This lab focuses on one of the most important SC-300 areas: Conditional Access, Identity Protection, session controls, named locations, authentication strength, break-glass accounts, and risk-based access decisions.

The TeachRich verified domain used throughout this lab is:

```text
teachrich.com
```

---

## SC-300 Exam Objectives Covered

* Plan Conditional Access policies
* Implement Conditional Access policy assignments
* Configure users, groups, apps, and conditions
* Implement grant controls and session controls
* Test and troubleshoot Conditional Access policies
* Implement session management
* Implement device-based access controls
* Understand Continuous Access Evaluation
* Configure authentication context
* Understand protected actions
* Create Conditional Access policies from templates
* Implement user risk policies
* Implement sign-in risk policies
* Implement MFA registration policy
* Monitor, investigate, and remediate risky users
* Monitor, investigate, and remediate risky sign-ins
* Understand risky workload identities

---

## Prerequisites

Before starting this lab, you should have:

* Completed Labs 1–5
* Authentication methods configured from Lab 5
* Microsoft Authenticator enabled
* Temporary Access Pass reviewed or enabled
* Security Defaults disabled
* Microsoft 365 E5 or Microsoft Entra ID P2 features available for Identity Protection
* PowerShell 7 installed
* Microsoft Graph PowerShell SDK installed
* Certificate-based authentication configured from Lab 1
* A verified Microsoft Entra domain, such as `teachrich.com`
* At least one test user account, such as:

```text
zara.ahmed@teachrich.com
```

> **Important:** Conditional Access can lock users out if configured incorrectly. Always create and exclude break-glass accounts before enabling policies. Start policies in **Report-only** mode before turning them **On**.

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
Policy.Read.All
Policy.ReadWrite.ConditionalAccess
Directory.ReadWrite.All
User.ReadWrite.All
Group.ReadWrite.All
AuditLog.Read.All
IdentityRiskyUser.Read.All
IdentityRiskyUser.ReadWrite.All
IdentityRiskEvent.Read.All
RoleManagement.Read.Directory
```

Then click:

```text
Grant admin consent
```

Use the permissions based on the tasks you plan to complete:

| Task                                         | Useful permissions                                                     |
| -------------------------------------------- | ---------------------------------------------------------------------- |
| Read Conditional Access policies             | `Policy.Read.All`                                                      |
| Create or update Conditional Access policies | `Policy.ReadWrite.ConditionalAccess`                                   |
| Create users and groups                      | `User.ReadWrite.All`, `Group.ReadWrite.All`, `Directory.ReadWrite.All` |
| Read sign-in and audit logs                  | `AuditLog.Read.All`                                                    |
| Review risky users                           | `IdentityRiskyUser.Read.All`                                           |
| Remediate risky users                        | `IdentityRiskyUser.ReadWrite.All`                                      |
| Review risk detections                       | `IdentityRiskEvent.Read.All`                                           |
| Read admin role assignments                  | `RoleManagement.Read.Directory`                                        |

> **Troubleshooting note:** If you add or change API permissions, always click **Grant admin consent**, then disconnect and reconnect to Microsoft Graph.

> #### 📘 Why these permissions matter
>
> Conditional Access and Identity Protection are high-impact security controls. A misconfigured policy can block access to the tenant, and a risk policy can force password changes or MFA challenges.
>
> `Policy.ReadWrite.ConditionalAccess` allows the app to manage Conditional Access policies. `AuditLog.Read.All` allows reporting on sign-ins and policy results. Identity Protection permissions allow you to view or remediate risky users and risky sign-ins.
>
> In production, separate duties carefully. Do not give one automation app every permission unless there is a clear business need.

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
> **Why it matters:** Conditional Access and Identity Protection can be managed through the portal, but PowerShell helps with reporting, repeatable configuration, and portfolio evidence.
>
> **Line by line:**
>
> * `$TenantId` identifies the TeachRich tenant.
> * `$ClientId` identifies the app registration that has Microsoft Graph permissions.
> * `$TenantDomain = "teachrich.com"` stores the verified domain for use in scripts.
> * `Read-Host ... -AsSecureString` prompts for the PFX password without showing it on screen.
> * `New-Object ... X509Certificate2(...)` loads the certificate and private key from the `.pfx` file.
> * `Connect-MgGraph ... -Certificate $cert` authenticates to Microsoft Graph as the app.
> * `Get-MgContext` confirms that the connection is active.
>
> **Watch out for:** Do not upload your `.pfx` file, certificate password, tenant secrets, or real Conditional Access export files containing sensitive details to GitHub.

---

## Step 1: Design Your Conditional Access Policy Matrix

Before creating policies, design the framework first.

Create this file:

```text
docs/conditional-access-matrix.md
```

Use the following policy matrix for TeachRich:

| Policy Name                                          | Users                          | Apps                           | Conditions                      | Grant Controls                                                   | Session Controls           | Priority |
| ---------------------------------------------------- | ------------------------------ | ------------------------------ | ------------------------------- | ---------------------------------------------------------------- | -------------------------- | -------- |
| `CA001: Require MFA for All Users`                   | All users except break-glass   | All cloud apps                 | None                            | Require multifactor authentication                               | None                       | Baseline |
| `CA002: Block Legacy Authentication`                 | All users except break-glass   | All cloud apps                 | Client apps: legacy clients     | Block access                                                     | None                       | High     |
| `CA003: Require Compliant Device for Sensitive Apps` | All users except break-glass   | SharePoint and Exchange Online | None                            | Require compliant device or hybrid Microsoft Entra joined device | None                       | Medium   |
| `CA004: Admin Phishing-Resistant MFA`                | Admin roles except break-glass | All cloud apps                 | None                            | Require phishing-resistant MFA                                   | Sign-in frequency: 4 hours | Critical |
| `CA005: Block Access from Untrusted Countries`       | All users except break-glass   | All cloud apps                 | Locations                       | Block access                                                     | None                       | High     |
| `CA006: Require MFA for Risky Sign-Ins`              | All users except break-glass   | All cloud apps                 | Sign-in risk: Medium and above  | Require MFA                                                      | None                       | High     |
| `CA007: Force Password Change for High-Risk Users`   | All users except break-glass   | All cloud apps                 | User risk: High                 | Require password change and MFA                                  | None                       | Critical |
| `CA008: Require Strong MFA for Sensitive Data`       | Selected users                 | Authentication context         | Sensitive action or data access | Require phishing-resistant MFA                                   | Optional                   | High     |

### Why this matters

Conditional Access policies can overlap. Designing the policy matrix first helps prevent gaps, conflicts, and accidental lockouts.

### SC-300 exam tip

The exam often gives a scenario and asks which Conditional Access assignments, conditions, or grant controls should be used. Practice breaking every policy into:

* Users
* Target resources
* Conditions
* Grant controls
* Session controls
* Exclusions

### How to explain this in an interview

> I designed a Conditional Access policy matrix before creating policies in the tenant. This allowed me to map business requirements to access controls, identify policy overlap, plan exclusions for emergency accounts, and test each policy in report-only mode before enforcement.

---

## Step 2: Create Break-Glass Emergency Access Accounts

Break-glass accounts are emergency administrator accounts used if normal administrators are locked out.

Create two cloud-only accounts.

**Where:** Microsoft Entra admin center → Identity → Users → All users → New user → Create new user

Create the first account:

```text
Username: breakglass@teachrich.com
Display name: Break Glass - Emergency Access 1
```

Create the second account:

```text
Username: breakglass2@teachrich.com
Display name: Break Glass - Emergency Access 2
```

Use very strong random passwords.

Recommended password requirements:

* 20+ characters
* Randomly generated
* Stored offline or in a secure enterprise password vault
* Not reused anywhere else
* Known only through the emergency access process

Assign both accounts the **Global Administrator** role.

**Where:** Microsoft Entra admin center → Identity → Roles & admins → Global Administrator → Add assignments

---

### Create a Break-Glass Exclusion Group

Create a group:

```text
Group name: SG-BreakGlass-Accounts
Group type: Security
Membership type: Assigned
```

Add both break-glass accounts as members.

All Conditional Access policies in this lab should exclude:

```text
SG-BreakGlass-Accounts
```

---

### Create a Break-Glass Procedure Document

Create this file:

```text
docs/break-glass-procedure.md
```

Use this template:

```markdown
# Break-Glass Account Procedure

## Accounts

- breakglass@teachrich.com
- breakglass2@teachrich.com

## Purpose

These accounts are used only when normal administrator access is unavailable because of Conditional Access misconfiguration, MFA outage, identity provider issues, or emergency recovery.

## Storage

- Passwords are stored offline or in an approved secure vault.
- Access to the passwords is restricted.
- Any access to these passwords must be logged.

## Conditional Access

- These accounts are excluded from all Conditional Access policies.
- These accounts should not be used for daily administration.

## Monitoring

- Any sign-in to a break-glass account must trigger investigation.
- Sign-ins should be reviewed monthly.
- Accounts should be tested on a scheduled basis.

## Testing

- Test one break-glass account monthly.
- Rotate testing between the two accounts.
- Confirm the account can sign in.
- Confirm the account is still excluded from Conditional Access policies.
```

### Why this matters

If a Conditional Access policy blocks every administrator, break-glass accounts may be the only way back into the tenant.

### SC-300 exam tip

Know that break-glass accounts should be:

* Cloud-only
* Highly privileged
* Excluded from all Conditional Access policies
* Protected with strong passwords
* Monitored closely
* Not used for daily administration

---

## Step 3: Create Named Locations

Named locations allow Conditional Access policies to make decisions based on IP ranges or countries.

**Where:** Microsoft Entra admin center → Protection → Conditional Access → Named locations

---

### 3a: Create a Trusted Office Location

1. Click **Named locations**.
2. Click **IP ranges location**.
3. Use this name:

```text
Trusted Office - Doha
```

4. Select:

```text
Mark as trusted location
```

5. Add your office or lab public IP range.
6. Click **Create**.

> **Important:** If you are using your home IP for testing, document that it is a lab example and not a real corporate trusted location.

---

### 3b: Create a Blocked Countries Location

1. Click **Countries location**.
2. Use this name:

```text
Blocked Countries
```

3. Select several countries you want to use for testing.
4. Click **Create**.

### Why this matters

Named locations make policies easier to read and manage. Instead of typing IP ranges into multiple policies, you define them once and reuse them.

---

### 3c: Create an IP Named Location with PowerShell

```powershell
# ================================
# Create a Named Location
# ================================

Import-Module Microsoft.Graph.Identity.SignIns

$params = @{
    "@odata.type" = "#microsoft.graph.ipNamedLocation"
    DisplayName   = "Corporate VPN"
    IsTrusted     = $true
    IpRanges      = @(
        @{
            "@odata.type" = "#microsoft.graph.iPv4CidrRange"
            CidrAddress  = "203.0.113.0/24"
        }
    )
}

try {
    New-MgIdentityConditionalAccessNamedLocation `
        -BodyParameter $params `
        -ErrorAction Stop | Out-Null

    Write-Host "[CREATED] Named location: Corporate VPN" -ForegroundColor Green
}
catch {
    Write-Host "[FAILED] Could not create named location" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Creates an IP-based named location called `Corporate VPN`.
>
> **Why it matters:** Named locations are reusable Conditional Access objects. You can mark trusted corporate IP ranges and then use them in multiple policies.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Identity.SignIns` loads the Conditional Access related cmdlets.
> * `$params` defines the named location object.
> * `"@odata.type" = "#microsoft.graph.ipNamedLocation"` tells Graph this is an IP-based named location.
> * `DisplayName = "Corporate VPN"` sets the name shown in the portal.
> * `IsTrusted = $true` marks the location as trusted.
> * `IpRanges` contains the allowed IP range.
> * `CidrAddress = "203.0.113.0/24"` is an example documentation IP range. Replace it with your real lab IP range.
> * `New-MgIdentityConditionalAccessNamedLocation` creates the named location.
> * `try/catch` gives a clean success or failure message.
>
> **Watch out for:** Do not trust large IP ranges unless they are truly controlled by your organization. A poorly configured trusted location can weaken MFA and access policies.

### SC-300 exam tip

Know the difference between:

| Named Location Type | Use Case                                                 |
| ------------------- | -------------------------------------------------------- |
| IP ranges           | Offices, corporate VPN, known network ranges             |
| Countries           | Allowing or blocking access based on geographic location |
| Trusted location    | A location considered lower risk by Conditional Access   |

---

## Step 4: Build Conditional Access Policies

**Where:** Microsoft Entra admin center → Protection → Conditional Access → Policies

> **Important:** Create all policies in **Report-only** mode first. Do not turn policies **On** until you have tested them with the What-If tool and sign-in logs.

---

### CA001: Require MFA for All Users

1. Click **New policy**.
2. Name the policy:

```text
CA001: Require MFA for All Users
```

3. Under **Users**, select:

```text
All users
```

4. Exclude:

```text
SG-BreakGlass-Accounts
```

5. Under **Target resources**, select:

```text
All cloud apps
```

6. Leave **Conditions** empty.
7. Under **Grant**, select:

```text
Require multifactor authentication
```

Or, if your tenant uses authentication strengths, select:

```text
Require authentication strength: Multifactor authentication
```

8. Set **Enable policy** to:

```text
Report-only
```

9. Click **Create**.

### Why this matters

This is the baseline MFA policy. It ensures users are challenged with MFA when accessing cloud apps, while break-glass accounts remain available for emergency access.

---

### CA002: Block Legacy Authentication

1. Click **New policy**.
2. Name the policy:

```text
CA002: Block Legacy Authentication
```

3. Under **Users**, select:

```text
All users
```

4. Exclude:

```text
SG-BreakGlass-Accounts
```

5. Under **Target resources**, select:

```text
All cloud apps
```

6. Under **Conditions**, configure **Client apps**.
7. Select legacy clients such as:

```text
Exchange ActiveSync clients
Other clients
```

8. Under **Grant**, select:

```text
Block access
```

9. Set **Enable policy** to:

```text
Report-only
```

10. Click **Create**.

### Why this matters

Legacy authentication protocols often do not support MFA. If legacy authentication is allowed, an attacker may bypass MFA by using older protocols.

### SC-300 exam tip

Blocking legacy authentication is one of the most important Conditional Access policies. Know that legacy clients include older protocols such as POP, IMAP, SMTP AUTH, and older ActiveSync clients.

---

### CA003: Require Compliant Device for Sensitive Apps

1. Click **New policy**.
2. Name the policy:

```text
CA003: Require Compliant Device for Sensitive Apps
```

3. Under **Users**, select:

```text
All users
```

4. Exclude:

```text
SG-BreakGlass-Accounts
```

5. Under **Target resources**, select **Select apps**.
6. Choose sensitive apps such as:

```text
Office 365 SharePoint Online
Office 365 Exchange Online
```

7. Under **Grant**, select:

```text
Require device to be marked as compliant
```

You may also select:

```text
Require Microsoft Entra hybrid joined device
```

8. Choose:

```text
Require one of the selected controls
```

9. Set **Enable policy** to:

```text
Report-only
```

10. Click **Create**.

### Why this matters

Sensitive data should not be accessed freely from unmanaged devices. Device-based controls help ensure that access to key apps happens from trusted or compliant devices.

---

### CA004: Admin Phishing-Resistant MFA

1. Click **New policy**.
2. Name the policy:

```text
CA004: Admin Phishing-Resistant MFA
```

3. Under **Users**, choose **Directory roles**.
4. Select admin roles such as:

```text
Global Administrator
Privileged Role Administrator
Security Administrator
Conditional Access Administrator
User Administrator
Authentication Administrator
```

5. Exclude:

```text
SG-BreakGlass-Accounts
```

6. Under **Target resources**, select:

```text
All cloud apps
```

7. Under **Grant**, select:

```text
Require authentication strength
```

8. Choose:

```text
Phishing-resistant MFA
```

9. Under **Session**, set:

```text
Sign-in frequency: 4 hours
```

10. Set **Enable policy** to:

```text
Report-only
```

11. Click **Create**.

### Why this matters

Administrative accounts are high-value targets. Phishing-resistant MFA reduces the chance that an attacker can compromise an admin account through fake login pages or MFA fatigue attacks.

### SC-300 exam tip

Authentication strength lets you specify which MFA methods are acceptable. For admins, phishing-resistant options are preferred.

---

### CA005: Block Access from Untrusted Countries

1. Click **New policy**.
2. Name the policy:

```text
CA005: Block Access from Untrusted Countries
```

3. Under **Users**, select:

```text
All users
```

4. Exclude:

```text
SG-BreakGlass-Accounts
```

5. Under **Target resources**, select:

```text
All cloud apps
```

6. Under **Conditions**, open **Locations**.
7. Include:

```text
Any location
```

8. Exclude trusted locations such as:

```text
Trusted Office - Doha
Corporate VPN
```

9. Under **Grant**, select:

```text
Block access
```

10. Set **Enable policy** to:

```text
Report-only
```

11. Click **Create**.

> **Important:** Be careful with country/location blocking. IP geolocation is not perfect, and users may travel. Use report-only mode and sign-in logs before enforcement.

### Why this matters

Location-based blocking can reduce exposure to regions where the organization does not operate. However, it must be tested carefully to avoid blocking legitimate users.

---

### CA-Template: Create a Policy from a Template

1. Go to **Conditional Access** → **Policies**.
2. Click **New policy from template**.
3. Choose a template such as:

```text
Require multifactor authentication for admins
```

4. Review the preconfigured settings.
5. Rename the policy:

```text
CA-Template: MFA for Admins
```

6. Add the break-glass exclusion:

```text
SG-BreakGlass-Accounts
```

7. Set the policy to:

```text
Report-only
```

8. Click **Create**.

### Why this matters

Templates help administrators start from Microsoft-recommended patterns instead of building every policy from scratch.

### SC-300 exam tip

Know that Conditional Access policy templates exist and can be used as a starting point. They should still be reviewed and customized before enforcement.

---

## Step 5: Test Policies with the What-If Tool

**Where:** Microsoft Entra admin center → Protection → Conditional Access → What If

The What-If tool simulates a sign-in and shows which Conditional Access policies would apply.

---

### Test 1: Regular User from Trusted Location

Use a regular user such as:

```text
zara.ahmed@teachrich.com
```

Test settings:

| Field           | Value             |
| --------------- | ----------------- |
| User            | Zara Ahmed        |
| Cloud apps      | All cloud apps    |
| IP address      | Trusted office IP |
| Device platform | Any               |

Review:

* Which policies apply?
* Which policies do not apply?
* Is MFA required?
* Is access blocked?

---

### Test 2: Admin from Unknown Location

Use an administrator account.

Test settings:

| Field           | Value                      |
| --------------- | -------------------------- |
| User            | Admin account              |
| Cloud apps      | All cloud apps             |
| IP address      | Unknown or foreign test IP |
| Device platform | Any                        |

Review:

* Does the admin phishing-resistant MFA policy apply?
* Does the location-based block policy apply?
* Is access blocked or allowed?

---

### Test 3: Regular User Accessing SharePoint from a Personal Device

Test settings:

| Field           | Value                        |
| --------------- | ---------------------------- |
| User            | Regular TeachRich user       |
| Cloud app       | Office 365 SharePoint Online |
| Device platform | Windows                      |
| Device state    | Non-compliant                |

Review:

* Does the compliant device policy apply?
* Does access require a compliant device?
* Would the user be blocked if the policy was turned on?

---

### Document the Results

Create this file:

```text
docs/conditional-access-test-results.md
```

Use this format:

```markdown
# Conditional Access What-If Test Results

## Test 1: Regular User from Trusted Location

User: zara.ahmed@teachrich.com  
App: All cloud apps  
Location: Trusted Office - Doha  
Result: [Document result here]  
Policies applied: [List policies]  
Notes: [Add observations]

## Test 2: Admin from Unknown Location

User: [Admin account]  
App: All cloud apps  
Location: Unknown test location  
Result: [Document result here]  
Policies applied: [List policies]  
Notes: [Add observations]

## Test 3: SharePoint from Non-Compliant Device

User: zara.ahmed@teachrich.com  
App: SharePoint Online  
Device state: Non-compliant  
Result: [Document result here]  
Policies applied: [List policies]  
Notes: [Add observations]
```

### Why this matters

What-If testing helps prove that policies are working before they affect real users.

### SC-300 exam tip

What-If is one of the main troubleshooting tools for Conditional Access. Know how to use it to determine which policies apply to a sign-in scenario.

---

## Step 6: Configure Authentication Context

Authentication context allows an application to trigger a specific Conditional Access policy for sensitive actions or data.

**Where:** Microsoft Entra admin center → Protection → Conditional Access → Authentication context

---

### 6a: Create an Authentication Context

1. Click **New authentication context**.
2. Use this name:

```text
Sensitive Data Access
```

3. Use this description:

```text
Required when accessing confidential TeachRich resources
```

4. Save the context.

---

### 6b: Create a Policy for the Authentication Context

1. Go to **Conditional Access** → **Policies**.
2. Click **New policy**.
3. Name the policy:

```text
CA008: Require Strong MFA for Sensitive Data
```

4. Under **Target resources**, select:

```text
Authentication context
```

5. Select:

```text
Sensitive Data Access
```

6. Under **Grant**, select:

```text
Require authentication strength
```

7. Choose:

```text
Phishing-resistant MFA
```

8. Set **Enable policy** to:

```text
Report-only
```

9. Click **Create**.

### Why this matters

Authentication context supports step-up authentication. A user may have normal access with standard MFA, but sensitive actions can require stronger MFA.

### SC-300 exam tip

Authentication context is used when applications need to trigger stronger Conditional Access controls for specific actions or sensitive resources.

---

## Step 7: Configure Session Management

Session controls define how long users stay signed in and how often they must reauthenticate.

---

### 7a: Configure Sign-In Frequency

Edit:

```text
CA001: Require MFA for All Users
```

Under **Session**, configure:

```text
Sign-in frequency: 24 hours
```

This means users must satisfy the sign-in requirements again after the configured period.

---

### 7b: Configure Persistent Browser Session

In the same policy, review:

```text
Persistent browser session
```

For a stricter policy, select:

```text
Never persistent
```

### Why this matters

Session controls help balance user experience and security. Sensitive users or admins may need shorter session lifetimes than regular employees.

---

### 7c: Review Continuous Access Evaluation

Continuous Access Evaluation allows supported Microsoft services to respond to important security events faster than normal token expiration.

Review the settings and document the concept.

Examples of events where CAE helps:

* User account disabled
* Password changed
* User risk increased
* Location policy change
* Token should be revoked

### SC-300 exam tip

Know that Continuous Access Evaluation can enforce changes closer to real time for supported apps such as Exchange Online, SharePoint Online, and Teams.

---

## Step 8: Understand Protected Actions

Protected actions require extra authentication for sensitive administrator operations.

Examples of protected actions may include:

* Deleting a Conditional Access policy
* Modifying privileged role settings
* Changing high-impact security settings
* Performing destructive administrative actions

For this lab:

1. Review where protected actions appear in your tenant.
2. Document the concept.
3. If the feature is not available, note that it depends on tenant capabilities and supported admin actions.

### Why this matters

Protected actions reduce the risk of a compromised admin session being used to make dangerous changes. They can require step-up authentication before allowing specific admin operations.

### SC-300 exam tip

Know that protected actions use Conditional Access authentication context to require stronger authentication for selected administrative actions.

---

## Step 9: Configure Identity Protection

**Where:** Microsoft Entra admin center → Protection → Identity Protection

Identity Protection uses Microsoft risk detections to identify suspicious users and sign-ins.

---

### 9a: Configure a User Risk Policy

Create a Conditional Access policy.

1. Go to **Conditional Access** → **Policies**.
2. Click **New policy**.
3. Name the policy:

```text
CA006: Force Password Change for High-Risk Users
```

4. Under **Users**, select:

```text
All users
```

5. Exclude:

```text
SG-BreakGlass-Accounts
```

6. Under **Target resources**, select:

```text
All cloud apps
```

7. Under **Conditions**, select **User risk**.
8. Choose:

```text
High
```

9. Under **Grant**, select:

```text
Require password change
Require multifactor authentication
```

10. Set **Enable policy** to:

```text
Report-only
```

11. Click **Create**.

### Why this matters

User risk means Microsoft has detected signs that the user account may be compromised. A high-risk user should be forced through strong remediation before regaining access.

---

### 9b: Configure a Sign-In Risk Policy

Create another Conditional Access policy.

1. Click **New policy**.
2. Name the policy:

```text
CA007: Require MFA for Risky Sign-Ins
```

3. Under **Users**, select:

```text
All users
```

4. Exclude:

```text
SG-BreakGlass-Accounts
```

5. Under **Target resources**, select:

```text
All cloud apps
```

6. Under **Conditions**, select **Sign-in risk**.
7. Choose:

```text
Medium and above
```

8. Under **Grant**, select:

```text
Require multifactor authentication
```

9. Set **Enable policy** to:

```text
Report-only
```

10. Click **Create**.

### Why this matters

Sign-in risk is about the current sign-in attempt. If the sign-in looks suspicious, Conditional Access can require MFA before access is granted.

---

### 9c: Configure MFA Registration Policy

**Where:** Identity Protection → MFA registration policy

1. Select:

```text
All users
```

2. Exclude:

```text
SG-BreakGlass-Accounts
```

3. Set enforcement to:

```text
On
```

4. Save the policy.

### Why this matters

Risk-based policies depend on users being able to complete MFA. The MFA registration policy helps ensure users have registered their security information.

---

### 9d: Review the Identity Protection Dashboard

Go to **Identity Protection** → **Overview**.

Review:

| Area            | Meaning                                  |
| --------------- | ---------------------------------------- |
| Risky users     | Users whose accounts may be compromised  |
| Risky sign-ins  | Suspicious sign-in attempts              |
| Risk detections | The specific detections that caused risk |
| Risk history    | Past risk events and remediation         |

### Why this matters

Identity Protection is not only about creating policies. You must also investigate detections and understand how users become risky.

---

### 9e: Optional: Simulate a Risky Sign-In

Only do this if it is safe in your lab environment.

Possible test:

1. Connect to a VPN in another country.
2. Sign in as a test user.
3. Disconnect and sign in from your normal location.
4. Review Identity Protection risk detections later.

> **Important:** Risk detections may not appear immediately, and not every test triggers risk. Do not rely on risky sign-in simulation as the only evidence for this lab.

### SC-300 exam tip

Know the difference between:

| Risk Type      | Meaning                                   | Example Response                                                     |
| -------------- | ----------------------------------------- | -------------------------------------------------------------------- |
| User risk      | The user account may be compromised       | Require password change                                              |
| Sign-in risk   | The current sign-in attempt is suspicious | Require MFA                                                          |
| Risk detection | The event that caused risk                | Impossible travel, leaked credentials, unfamiliar sign-in properties |

### How to explain this in an interview

> I configured Identity Protection for TeachRich using risk-based Conditional Access policies. High-risk users are required to change their password and complete MFA. Medium or high-risk sign-ins require MFA. This creates an automated response model where suspicious activity is challenged or remediated without waiting for manual intervention.

---

## Step 10: Move Policies from Report-Only to On

After testing:

1. Review What-If results.
2. Review sign-in logs.
3. Confirm break-glass accounts are excluded.
4. Enable policies one at a time.
5. Start with lower-risk policies.
6. Monitor for unexpected blocks.
7. Keep screenshots and notes.

Recommended order:

| Order | Policy                                               | Reason                                                          |
| ----- | ---------------------------------------------------- | --------------------------------------------------------------- |
| 1     | `CA002: Block Legacy Authentication`                 | High security value and usually low user impact                 |
| 2     | `CA001: Require MFA for All Users`                   | Baseline protection                                             |
| 3     | `CA007: Require MFA for Risky Sign-Ins`              | Risk-based protection                                           |
| 4     | `CA006: Force Password Change for High-Risk Users`   | Strong remediation for compromised users                        |
| 5     | `CA004: Admin Phishing-Resistant MFA`                | Critical but must confirm admins have registered strong methods |
| 6     | `CA003: Require Compliant Device for Sensitive Apps` | Can impact unmanaged devices                                    |
| 7     | `CA005: Block Access from Untrusted Countries`       | Can cause travel or geolocation issues                          |

### Production best practice

Never enable all policies at once. Use report-only mode, test carefully, enable one policy, monitor, then continue.

---

## Step 11: Generate a Conditional Access Policy Report

Create a `reports` folder and export your Conditional Access policies.

```powershell
# ================================
# Export Conditional Access Policy Report
# ================================

Import-Module Microsoft.Graph.Identity.SignIns

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$policies = Get-MgIdentityConditionalAccessPolicy -All

$policyReport = $policies |
Select-Object `
    DisplayName,
    State,
    CreatedDateTime,
    ModifiedDateTime,
    @{Name="IncludeUsers";Expression={
        if ($_.Conditions.Users.IncludeUsers) {
            $_.Conditions.Users.IncludeUsers -join "; "
        }
        else {
            "None"
        }
    }},
    @{Name="ExcludeUsers";Expression={
        if ($_.Conditions.Users.ExcludeUsers) {
            $_.Conditions.Users.ExcludeUsers -join "; "
        }
        else {
            "None"
        }
    }},
    @{Name="IncludeGroups";Expression={
        if ($_.Conditions.Users.IncludeGroups) {
            $_.Conditions.Users.IncludeGroups -join "; "
        }
        else {
            "None"
        }
    }},
    @{Name="ExcludeGroups";Expression={
        if ($_.Conditions.Users.ExcludeGroups) {
            $_.Conditions.Users.ExcludeGroups -join "; "
        }
        else {
            "None"
        }
    }},
    @{Name="IncludeApplications";Expression={
        if ($_.Conditions.Applications.IncludeApplications) {
            $_.Conditions.Applications.IncludeApplications -join "; "
        }
        else {
            "None"
        }
    }},
    @{Name="GrantControls";Expression={
        if ($_.GrantControls.BuiltInControls) {
            $_.GrantControls.BuiltInControls -join "; "
        }
        else {
            "None"
        }
    }},
    @{Name="SessionControls";Expression={
        if ($_.SessionControls.SignInFrequency) {
            "Sign-in frequency: $($_.SessionControls.SignInFrequency.Value) $($_.SessionControls.SignInFrequency.Type)"
        }
        else {
            "None"
        }
    }}

$policyReport | Format-Table DisplayName, State, GrantControls, SessionControls -AutoSize

$policyReport | Export-Csv -Path "./reports/ca-policies-report.csv" -NoTypeInformation

Write-Host "Conditional Access policy report exported to ./reports/ca-policies-report.csv" -ForegroundColor Green
Write-Host "Total policies reported: $($policyReport.Count)" -ForegroundColor Cyan
```

> #### 📘 Script explained
>
> **What it does:** Reads all Conditional Access policies and exports a CSV report showing policy names, states, assignments, exclusions, grant controls, and session controls.
>
> **Why it matters:** Conditional Access reporting gives you evidence for your portfolio and helps with audit readiness. It also makes it easier to review which policies are in report-only mode, enabled, or disabled.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Identity.SignIns` loads the Conditional Access policy cmdlets.
> * `New-Item -ItemType Directory -Path ./reports -Force` creates the reports folder if it does not exist.
> * `Get-MgIdentityConditionalAccessPolicy -All` retrieves all Conditional Access policies.
> * `Select-Object` shapes the output into a readable report.
> * The calculated properties turn nested policy settings into simple CSV columns.
> * `IncludeUsers`, `ExcludeUsers`, `IncludeGroups`, and `ExcludeGroups` show who the policy applies to and who is excluded.
> * `IncludeApplications` shows which apps are targeted.
> * `GrantControls` shows controls such as MFA, block, password change, or compliant device.
> * `SessionControls` shows sign-in frequency if configured.
> * `Format-Table` previews the report in the terminal.
> * `Export-Csv -NoTypeInformation` saves the report as a clean CSV file.
>
> **Watch out for:** Some values may appear as IDs rather than friendly names. This is normal with Graph output. For a deeper production report, you can resolve group IDs, user IDs, and application IDs into display names.

---

## Step 12: Export Risky Users Report

Use this optional report if your tenant has Identity Protection data.

```powershell
# ================================
# Export Risky Users Report
# ================================

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

try {
    $riskyUsers = Invoke-MgGraphRequest `
        -Method GET `
        -Uri "https://graph.microsoft.com/v1.0/identityProtection/riskyUsers"

    $riskReport = $riskyUsers.value |
    Select-Object `
        Id,
        UserPrincipalName,
        RiskLevel,
        RiskState,
        RiskDetail,
        IsDeleted,
        IsProcessing

    $riskReport | Format-Table -AutoSize

    $riskReport | Export-Csv -Path "./reports/risky-users-report.csv" -NoTypeInformation

    Write-Host "Risky users report exported to ./reports/risky-users-report.csv" -ForegroundColor Green
    Write-Host "Total risky users reported: $($riskReport.Count)" -ForegroundColor Cyan
}
catch {
    Write-Host "[FAILED] Could not export risky users report" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Uses Microsoft Graph to retrieve risky users from Identity Protection and exports them to a CSV file.
>
> **Why it matters:** Risky user reporting supports investigation and remediation. It helps show whether any users are currently flagged as low, medium, or high risk.
>
> **Line by line:**
>
> * `Invoke-MgGraphRequest` sends a direct request to the Identity Protection risky users endpoint.
> * `-Method GET` means the script is reading data, not changing it.
> * The URI points to risky users in Microsoft Graph.
> * `$riskyUsers.value` contains the returned risky user records.
> * `Select-Object` creates readable columns for the report.
> * `RiskLevel` shows whether the user is low, medium, high, or hidden.
> * `RiskState` shows whether the risk is active, remediated, dismissed, or confirmed compromised.
> * `Export-Csv` saves the report for evidence.
>
> **Watch out for:** A lab tenant may have no risky users. That is a good result. If the report is empty, document that no risky users were present at the time of testing.

---

## Step 13: Verify Everything

Use this checklist to confirm completion:

* [ ] Conditional Access policy matrix created at `docs/conditional-access-matrix.md`
* [ ] Two break-glass accounts created
* [ ] Break-glass accounts use `teachrich.com`
* [ ] Break-glass accounts assigned Global Administrator
* [ ] `SG-BreakGlass-Accounts` group created
* [ ] Break-glass accounts added to the exclusion group
* [ ] Break-glass procedure documented
* [ ] Trusted office named location created
* [ ] Blocked countries named location created
* [ ] Optional PowerShell named location created
* [ ] `CA001: Require MFA for All Users` created in report-only mode
* [ ] `CA002: Block Legacy Authentication` created in report-only mode
* [ ] `CA003: Require Compliant Device for Sensitive Apps` created in report-only mode
* [ ] `CA004: Admin Phishing-Resistant MFA` created in report-only mode
* [ ] `CA005: Block Access from Untrusted Countries` created in report-only mode
* [ ] Conditional Access template policy reviewed or created
* [ ] What-If testing completed for at least three scenarios
* [ ] What-If results documented at `docs/conditional-access-test-results.md`
* [ ] Authentication context created
* [ ] `CA008: Require Strong MFA for Sensitive Data` created
* [ ] Session controls reviewed or configured
* [ ] Continuous Access Evaluation documented
* [ ] Protected actions concept documented
* [ ] User risk policy configured
* [ ] Sign-in risk policy configured
* [ ] MFA registration policy reviewed or enabled
* [ ] Identity Protection dashboard reviewed
* [ ] Conditional Access policies moved from report-only to on only after testing
* [ ] Conditional Access report exported to `./reports/ca-policies-report.csv`
* [ ] Risky users report exported if data is available

Recommended screenshots:

* Conditional Access policy matrix
* Break-glass accounts
* `SG-BreakGlass-Accounts` group membership
* Named locations
* CA001 policy summary
* CA002 policy summary
* CA003 policy summary
* CA004 policy summary
* CA005 policy summary
* Conditional Access What-If results
* Authentication context
* Session controls
* Identity Protection overview
* Risky users page
* Risky sign-ins page
* Conditional Access report CSV
* Risky users report CSV, if available

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

Your app registration may be missing required Microsoft Graph Application permissions, admin consent, or administrator role assignment.

Check:

```text
Microsoft Entra admin center
→ App registrations
→ Your app
→ API permissions
```

For Conditional Access, confirm:

```text
Policy.Read.All
Policy.ReadWrite.ConditionalAccess
```

For user and group work, confirm:

```text
User.ReadWrite.All
Group.ReadWrite.All
Directory.ReadWrite.All
```

For Identity Protection, confirm:

```text
IdentityRiskyUser.Read.All
IdentityRiskEvent.Read.All
```

After changing permissions, click:

```text
Grant admin consent
```

Then disconnect and reconnect to Microsoft Graph.

---

### Conditional Access policy is blocking too many users

Check:

* Did you start in report-only mode?
* Are break-glass accounts excluded?
* Are the correct users and groups included?
* Are guest users included by mistake?
* Are locations configured correctly?
* Are device compliance requirements realistic?
* Did you use **What-If** before enabling?

If needed, use a break-glass account to sign in and disable the policy.

---

### Break-glass account is affected by Conditional Access

Check every Conditional Access policy and confirm the break-glass exclusion group is excluded.

Search for:

```text
SG-BreakGlass-Accounts
```

under the **Exclude** section of each policy.

---

### Named location is not working as expected

Check:

* The IP range is correct.
* The public IP is not changing.
* The location is marked as trusted only if appropriate.
* VPN traffic exits from the expected IP.
* Country detection may not be perfect because IP geolocation can vary.

---

### What-If results are different from expected

Check:

* User selection
* App selection
* Conditions
* Device state
* Client app
* IP address
* Named location
* Policy state
* Exclusions

Remember that What-If is a simulation. Sign-in logs provide real sign-in evidence.

---

### Risky users report is empty

This may be normal in a clean lab tenant.

Check:

* Identity Protection is available in the tenant.
* The account has the correct permissions.
* No risky users currently exist.
* Risk detections may take time to appear.

An empty report can still be documented as evidence that no risky users were present at the time of testing.

---

### Admin cannot satisfy phishing-resistant MFA

Check that the admin has registered an allowed method such as:

* FIDO2 security key
* Passkey
* Certificate-based authentication, if configured
* Windows Hello for Business, if available and accepted by the authentication strength

Do not enable the admin phishing-resistant policy until admins have registered the required methods.

---

## Key Takeaways for SC-300

1. Conditional Access should be designed before deployment.
2. Break-glass accounts are essential for emergency recovery.
3. Break-glass accounts should be excluded from all Conditional Access policies.
4. Report-only mode helps test policies safely.
5. What-If is a key Conditional Access troubleshooting tool.
6. Blocking legacy authentication protects against MFA bypass.
7. Authentication strength allows specific MFA methods to be required.
8. Phishing-resistant MFA is preferred for administrators.
9. Named locations support IP-based and country-based access decisions.
10. Authentication context enables step-up authentication for sensitive actions.
11. Session controls manage sign-in frequency and browser persistence.
12. Continuous Access Evaluation supports faster enforcement for supported services.
13. Identity Protection enables risk-based automated responses.
14. User risk and sign-in risk are different conditions.
15. Reports and screenshots provide evidence for a portfolio or audit.

---

## Portfolio / Interview Summary

In this lab, I designed and implemented a Conditional Access and Identity Protection framework for the TeachRich Microsoft Entra tenant. I created a Conditional Access policy matrix, configured break-glass emergency access accounts, created named locations, built baseline and risk-based Conditional Access policies, tested policies with What-If, reviewed authentication context and protected actions, configured session controls, reviewed Identity Protection, and exported Conditional Access and risky user reports.

This project demonstrates practical SC-300 skills in Zero Trust access control, Conditional Access planning, policy testing, authentication strength, break-glass design, Identity Protection, risk-based access, reporting, and operational security.

---

## What's Next

➡️ **Lab 7:** [Workload Identities & Enterprise Applications](../lab07-workload-identities/) — Managed identities, service principals, enterprise app integration, SSO, and consent management.

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
