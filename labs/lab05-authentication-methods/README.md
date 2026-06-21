# Lab 5: Authentication Methods, MFA & SSPR

> **Type:** Focused Lab | **Time:** 2–3 hours | **SC-300 Domain:** Implement authentication and access management (25–30%)

---

## Scenario

TeachRich has grown quickly, and the security team has identified several authentication risks. Some users are still relying only on passwords, there is no consistent MFA registration process, and employees frequently contact the help desk for password resets.

You have been asked to implement a stronger authentication strategy for the TeachRich tenant. In this lab, you will plan authentication methods for different user populations, configure authentication method policies, enable Microsoft Authenticator, review FIDO2 and Temporary Access Pass, configure self-service password reset, implement password protection, and practice disabling accounts and revoking sessions during a simulated security incident.

The TeachRich verified domain used throughout this lab is:

```text
teachrich.com
```

---

## SC-300 Exam Objectives Covered

* Plan for authentication
* Implement and manage authentication methods
* Configure Microsoft Authenticator
* Configure passkeys / FIDO2 security keys
* Configure Temporary Access Pass
* Understand certificate-based authentication
* Implement and manage tenant-wide MFA settings
* Configure and deploy self-service password reset
* Understand Windows Hello for Business concepts
* Disable accounts and revoke user sessions
* Implement and manage Microsoft Entra password protection
* Understand combined security information registration

---

## Prerequisites

Before starting this lab, you should have:

* Completed Labs 1–4
* A Microsoft 365 E3, Microsoft 365 E5, or similar subscription
* Microsoft Entra ID P1 or P2 features available for Conditional Access and advanced identity controls
* A verified Microsoft Entra domain, such as `teachrich.com`
* PowerShell 7 installed
* Microsoft Graph PowerShell SDK installed
* Certificate-based authentication configured from Lab 1
* A mobile phone for testing MFA
* Microsoft Authenticator installed on your phone
* At least one test user account, such as:

```text
zara.ahmed@teachrich.com
```

> **Important:** Some authentication features depend on tenant licensing, admin roles, and whether your tenant has already been migrated to the latest authentication methods experience. If an option is not available, document what you see and explain what would be required to enable it.

---

## Portal Navigation Reference

The Microsoft Entra admin center sidebar is organised into the following sections. Use this as your reference throughout the lab:

**Entra ID** section (top):
Overview → Users → Groups → Devices → Agents → Enterprise apps → App registrations → Roles & admins → Delegated admin partners → Tenant governance → Domain services → Conditional Access → Multifactor authentication → Identity Secure Score → **Authentication methods** → Account recovery → **Password reset** → Custom security attributes → Certificate authorities → External Identities → Cross-tenant synchronization → Entra Connect → Backup and recovery → Domain names → Custom branding → Mobility → Monitoring & health

**ID Protection** section:
Dashboard → Risk-based Conditional Access → Risky users → Risky workload identities → Risky agents

**ID Governance** | **Verified ID** | **Global Secure Access** | What's new | Billing | Security Store

> **Tip:** When portal navigation does not match lab instructions, use the **search bar** at the top of the Entra admin center. Searching by feature name (for example, "Password protection" or "Authentication methods") is the most reliable way to surface a blade directly.

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
UserAuthenticationMethod.ReadWrite.All
AuditLog.Read.All
Policy.Read.All
Policy.ReadWrite.AuthenticationMethod
```

Then click:

```text
Grant admin consent
```

Use the permissions based on the tasks you plan to complete:

| Task                                             | Useful permissions                              |
| ------------------------------------------------ | ----------------------------------------------- |
| Read and update users                            | `User.ReadWrite.All`, `Directory.ReadWrite.All` |
| Create Temporary Access Pass                     | `UserAuthenticationMethod.ReadWrite.All`        |
| Read authentication methods                      | `UserAuthenticationMethod.ReadWrite.All`        |
| Disable accounts                                 | `User.ReadWrite.All`                            |
| Revoke sessions                                  | `User.ReadWrite.All`                            |
| Review audit/sign-in activity                    | `AuditLog.Read.All`                             |
| Read authentication policies                     | `Policy.Read.All`                               |
| Manage authentication method policies with Graph | `Policy.ReadWrite.AuthenticationMethod`         |

> **Troubleshooting note:** If you change API permissions or role assignments, disconnect and reconnect to Microsoft Graph before testing again.

> #### 📘 Why these permissions matter
>
> Authentication settings are sensitive because they control how users prove their identity. In this lab, your PowerShell scripts may create Temporary Access Passes, disable accounts, and revoke sessions. Those actions require strong Microsoft Graph permissions.
>
> In production, you would not give one app every permission listed above unless it truly needed them. You would separate automation tasks and follow least privilege. For the lab, these permissions help you understand which Microsoft Graph rights are connected to each authentication task.

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
> **Why it matters:** Authentication administration is often automated. Certificate-based authentication allows the script to run as the app itself, without a user typing a password or approving a sign-in prompt.
>
> **Line by line:**
>
> * `$TenantId` identifies the TeachRich tenant.
> * `$ClientId` identifies the app registration that has Microsoft Graph permissions.
> * `$TenantDomain = "teachrich.com"` stores the verified domain for reuse in scripts.
> * `Read-Host ... -AsSecureString` asks for the PFX password without displaying it on screen.
> * `New-Object ... X509Certificate2(...)` loads the certificate and private key from the `.pfx` file.
> * `Connect-MgGraph ... -Certificate $cert` authenticates to Microsoft Graph as the app.
> * `Get-MgContext` confirms the tenant, app, and authentication context.
>
> **Watch out for:** Do not upload your `.pfx` file or password to GitHub. In production, certificates should be stored in a secure certificate store or Azure Key Vault.

---

## Step 1: Create an Authentication Strategy Document

Before configuring anything, create a planning document.

Create the file:

```text
docs/authentication-strategy.md
```

Use the following strategy for TeachRich:

| User Population      | Primary Authentication                          | Secondary Authentication / MFA                           | SSPR Methods                 | Justification                                               |
| -------------------- | ----------------------------------------------- | -------------------------------------------------------- | ---------------------------- | ----------------------------------------------------------- |
| Regular Employees    | Password                                        | Microsoft Authenticator with number matching             | Authenticator + SMS or email | Good balance of security and usability                      |
| Administrators       | Password or passwordless sign-in                | FIDO2 security key or Authenticator with number matching | Authenticator + SMS          | Privileged users need stronger phishing-resistant methods   |
| External Contractors | Password or external identity                   | Email OTP, SMS, or Authenticator                         | Email OTP where applicable   | Contractors may not have TeachRich-managed devices          |
| New Employees        | Temporary Access Pass                           | Register Authenticator during onboarding                 | Register after TAP sign-in   | Allows secure first-day onboarding without a known password |
| Service Accounts     | Certificate-based authentication where possible | Not applicable                                           | Not applicable               | Non-human accounts should not rely on interactive MFA       |

### Why this matters

The SC-300 exam expects you to plan authentication, not only configure settings. A good authentication strategy maps the right method to the right user population.

### SC-300 exam tip

Know when each authentication method is appropriate:

* Microsoft Authenticator is suitable for most users.
* FIDO2/passkeys are stronger for admins and high-risk users.
* Temporary Access Pass is useful for onboarding and recovery.
* Email OTP is useful for external users.
* Certificate-based authentication is useful for non-human or high-assurance scenarios.

### How to explain this in an interview

> I created an authentication strategy for TeachRich that maps authentication methods to user populations. Regular employees use Microsoft Authenticator with number matching. Administrators use stronger methods such as FIDO2 security keys. New employees receive a Temporary Access Pass so they can register permanent authentication methods during onboarding. This balances security, usability, and operational support.

---

## Step 2: Configure Authentication Methods Policy

**Where:** Microsoft Entra admin center → Entra ID → Authentication methods → Policies

This is where you enable or disable authentication methods for the TeachRich tenant.

---

### 2a: Microsoft Authenticator

1. Click **Microsoft Authenticator**.
2. Set **Enable** to:

```text
Yes
```

3. Set **Target** to:

```text
All users
```

4. Under **Configure**, use:

| Setting                  | Recommended Value |
| ------------------------ | ----------------- |
| Authentication mode      | Push or Any       |
| Require number matching  | Enabled           |
| Show application name    | Enabled           |
| Show geographic location | Enabled           |

5. Click **Save**.

### Why this matters

Microsoft Authenticator is one of the main MFA methods used in Microsoft Entra ID. Number matching helps protect against MFA fatigue attacks because users must type the number shown on the sign-in screen.

### SC-300 exam tip

Number matching is important. Without it, an attacker may repeatedly send MFA prompts until a user accidentally approves. With number matching, the user must be looking at the sign-in screen to approve the request.

---

### 2b: SMS

1. Click **SMS**.
2. Set **Enable** to:

```text
Yes
```

3. Set **Target** to:

```text
All users
```

4. Click **Save**.

### Why this matters

SMS is widely available, but it is not as strong as app-based or phishing-resistant methods. It is useful as a backup method, especially for SSPR.

### SC-300 exam tip

SMS is better than password-only authentication, but it is weaker than FIDO2, passkeys, or Authenticator with number matching.

---

### 2c: Email OTP

1. Click **Email OTP**.
2. Set **Enable** to:

```text
Yes
```

3. Set **Target** to:

```text
All users
```

4. Click **Save**.

### Why this matters

Email OTP is useful for guest and external user scenarios. It allows users to authenticate with a one-time code sent to their email.

---

### 2d: Passkey / FIDO2 Security Keys

Create a security group for users who should be allowed to register FIDO2 methods.

Suggested group name:

```text
SG-FIDO2-Enabled
```

Then configure the method:

1. Click **Passkey (FIDO2)**.
2. Set **Enable** to:

```text
Yes
```

3. Set **Target** to:

```text
SG-FIDO2-Enabled
```

4. Under **Configure**, use:

| Setting                  | Lab Value | Production Note                                        |
| ------------------------ | --------- | ------------------------------------------------------ |
| Allow self-service setup | Yes       | Allows users to register their key                     |
| Enforce attestation      | No        | In production, consider Yes for approved key models    |
| Key restrictions         | Optional  | Use in production to allow or block specific key types |

5. Click **Save**.

### Why this matters

FIDO2 and passkeys provide phishing-resistant authentication. They are especially valuable for administrators and high-risk users.

### SC-300 exam tip

Know that FIDO2/passkeys are stronger than SMS and standard push MFA because they are phishing-resistant.

---

### 2e: Temporary Access Pass

1. Click **Temporary Access Pass**.
2. Set **Enable** to:

```text
Yes
```

3. Set **Target** to:

```text
All users
```

4. Under **Configure**, use:

| Setting          | Recommended Lab Value |
| ---------------- | --------------------- |
| Minimum lifetime | 60 minutes            |
| Maximum lifetime | 480 minutes           |
| Default lifetime | 60 minutes            |
| One-time use     | Yes                   |

5. Click **Save**.

### Why this matters

Temporary Access Pass is used to bootstrap authentication. It is helpful when a new employee has not registered Microsoft Authenticator yet or when a user loses access to their existing MFA method.

### SC-300 exam tip

TAP is temporary. It is not a long-term password replacement. It is used to help users register permanent authentication methods.

---

### 2f: Certificate-Based Authentication

1. Click **Certificate-based authentication**.
2. Review the configuration page.
3. Document what is required to fully configure it.

For this lab, you can document the page rather than completing a full PKI deployment.

### Why this matters

Certificate-based authentication requires trusted certificate authorities and certificate lifecycle management. It is commonly used in higher-assurance environments.

### SC-300 exam tip

Know that certificate-based authentication depends on certificates issued by a trusted CA and is not simply a toggle that works without certificate infrastructure.

---

## Step 3: Generate a Temporary Access Pass for a Test User

Use a test user such as:

```text
zara.ahmed@teachrich.com
```

---

### 3a: Create the TAP via the Portal

1. Go to **Users**.
2. Select the test user.
3. Click **Authentication methods**.
4. Click **Add authentication method**.
5. Select **Temporary Access Pass**.
6. Set the lifetime to:

```text
60 minutes
```

7. Set one-time use if available.
8. Click **Add**.
9. Copy the TAP code immediately.

> **Important:** The TAP value may only be shown once. Store it carefully for the test and do not commit it to GitHub.

---

### 3b: Test the TAP Sign-In Experience

1. Open an incognito/private browser window.
2. Go to:

```text
https://login.microsoftonline.com
```

3. Sign in as:

```text
zara.ahmed@teachrich.com
```

4. When prompted for a password or authentication method, enter the TAP.
5. Follow the prompts to register Microsoft Authenticator.
6. Confirm that the user can complete registration.

### Why this matters

TAP is one of the cleanest onboarding methods. It allows a new employee to sign in and register secure authentication methods without needing a permanent password first.

---

### 3c: Create a TAP via PowerShell

```powershell
# ================================
# Generate a Temporary Access Pass
# ================================

Import-Module Microsoft.Graph.Users
Import-Module Microsoft.Graph.Identity.SignIns

$UserPrincipalName = "zara.ahmed@teachrich.com"

try {
    $user = Get-MgUser `
        -UserId $UserPrincipalName `
        -Property "id,displayName,userPrincipalName" `
        -ErrorAction Stop

    $tap = New-MgUserAuthenticationTemporaryAccessPassMethod `
        -UserId $user.Id `
        -BodyParameter @{
            LifetimeInMinutes = 60
            IsUsableOnce     = $true
        } `
        -ErrorAction Stop

    Write-Host "[TAP CREATED] $($user.DisplayName) <$($user.UserPrincipalName)>" -ForegroundColor Green
    Write-Host "Temporary Access Pass: $($tap.TemporaryAccessPass)" -ForegroundColor Cyan
    Write-Host "Valid for: $($tap.LifetimeInMinutes) minutes" -ForegroundColor Yellow
    Write-Host "One-time use: $($tap.IsUsableOnce)" -ForegroundColor Yellow
}
catch {
    Write-Host "[FAILED] Could not create TAP for $UserPrincipalName" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Finds the test user and creates a one-time Temporary Access Pass that is valid for 60 minutes.
>
> **Why it matters:** This is the automation version of onboarding a user. A help desk or IAM team could generate a TAP for a new starter or for a user who lost access to their MFA device.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Users` loads user management commands.
> * `Import-Module Microsoft.Graph.Identity.SignIns` loads authentication-related commands, including TAP cmdlets.
> * `$UserPrincipalName` stores the test user you are creating the TAP for.
> * `Get-MgUser -UserId $UserPrincipalName` retrieves the user object.
> * `New-MgUserAuthenticationTemporaryAccessPassMethod` creates the TAP.
> * `LifetimeInMinutes = 60` makes the TAP valid for one hour.
> * `IsUsableOnce = $true` means the TAP can only be used one time.
> * `try/catch` gives clear success or failure output.
>
> **Watch out for:** Treat the TAP like a temporary password. Do not paste real TAP values into GitHub screenshots or documentation. Redact the value before publishing.

### SC-300 exam tip

Know how TAP supports onboarding and recovery. Also know the difference between one-time and multi-use TAPs.

---

## Step 4: Configure Tenant-Wide MFA Settings

There are multiple ways to enforce MFA. The lab focuses on understanding each approach.

---

### 4a: Security Defaults

**Where:** Microsoft Entra admin center → Entra ID → Overview → Properties → Manage security defaults

Security Defaults is a simple tenant-wide protection baseline.

1. Open **Manage security defaults**.
2. Review whether Security Defaults is enabled.
3. If you are using Conditional Access in later labs, Security Defaults should be disabled.
4. Save your setting if you make a change.

### Why this matters

Security Defaults is useful for smaller tenants without Microsoft Entra ID P1 or P2. It is simple, but it is not granular.

### SC-300 exam tip

Security Defaults and Conditional Access are not used together. If you need granular MFA policies, use Conditional Access.

---

### 4b: Per-User MFA

**Where:** Microsoft Entra admin center → Entra ID → Users → All users → Per-user MFA

1. Open the **Per-user MFA** page.
2. Review the MFA states.
3. Do not enable anything for this lab unless you are specifically testing legacy MFA.
4. Document that this is the legacy approach.

The three per-user MFA states are:

| State    | Meaning                                                      |
| -------- | ------------------------------------------------------------ |
| Disabled | MFA is not enabled for the user through legacy per-user MFA  |
| Enabled  | User is required to register MFA at next sign-in             |
| Enforced | User has registered MFA and must use it                      |

### Why this matters

Per-user MFA is legacy, but it can still appear in exam questions and real tenants.

### SC-300 exam tip

The modern approach is Conditional Access. Per-user MFA is important to recognize, especially in older environments.

---

### 4c: Conditional Access MFA

Conditional Access MFA will be configured in Lab 6.

For now, document that Conditional Access is the preferred production approach because it can evaluate:

* User or group
* Application
* Location
* Device compliance
* Sign-in risk
* User risk
* Client app
* Session controls

### Why this matters

Conditional Access allows TeachRich to require MFA in the right situations instead of using the same rule for every sign-in.

---

## Step 5: Configure Self-Service Password Reset

**Where:** Microsoft Entra admin center → Entra ID → Password reset

---

### 5a: Properties

1. Go to **Password reset**.
2. Click **Properties**.
3. Set **Self service password reset enabled** to:

```text
All
```

4. Click **Save**.

### Why this matters

Enabling SSPR for all users reduces help desk workload and gives users a secure recovery path.

---

### 5b: Authentication Methods

1. Click **Authentication methods**.
2. Set **Number of methods required to reset** to:

```text
2
```

3. Select available methods, such as:

```text
Mobile app notification
Mobile app code
Email
Mobile phone
```

4. Click **Save**.

### Why this matters

Requiring two methods improves security. A single weak recovery method can become a target for account takeover.

### SC-300 exam tip

Know which methods can be used for SSPR and know that administrators have stricter reset requirements.

---

### 5c: Registration

1. Click **Registration**.
2. Set **Require users to register when signing in** to:

```text
Yes
```

3. Set **Number of days before users are asked to reconfirm** to:

```text
90
```

4. Click **Save**.

### Why this matters

SSPR only works if users have registered their recovery methods before they need them. Registration enforcement makes sure users prepare in advance.

---

### 5d: Notifications

1. Click **Notifications**.
2. Set **Notify users on password resets** to:

```text
Yes
```

3. Set **Notify all admins when other admins reset their password** to:

```text
Yes
```

4. Click **Save**.

### Why this matters

Password reset notifications are important security signals. Admin password resets are especially sensitive and should generate alerts.

---

### 5e: Test SSPR End-to-End

1. Open an incognito/private browser window.
2. Go to:

```text
https://aka.ms/sspr
```

3. Enter a test user UPN, such as:

```text
zara.ahmed@teachrich.com
```

4. Complete the CAPTCHA.
5. Choose the required verification methods.
6. Complete verification.
7. Set a new password.
8. Go back to Microsoft Entra admin center.
9. Open **Audit logs**.
10. Filter for password reset activity.

### How to explain this in an interview

> I configured SSPR for the TeachRich tenant and required two verification methods. Users are prompted to register their security information, and notifications are enabled for password reset activity. This reduces help desk dependency while maintaining a secure recovery process.

---

## Step 6: Implement Microsoft Entra Password Protection

**Where:** Microsoft Entra admin center → Entra ID → Authentication methods → Password protection

> **Portal note:** Click **Authentication methods** in the Entra ID section of the sidebar, then select **Password protection** from the sub-menu or tabs inside that blade. If the sub-item is not visible, use the search bar at the top of the portal and search for "Password protection".

---

### 6a: Configure Custom Banned Password List

1. Click **Password protection**.
2. Set **Custom banned password list** to:

```text
Yes
```

3. Add TeachRich-specific banned words:

```text
teachrich
teachrich2026
teachrichacademy
iamlabcorp
password123
welcome2026
qatar2026
```

4. Set **Enable password protection on Windows Server Active Directory** based on your lab setup.
5. For cloud-only testing, focus on the cloud password protection settings.
6. Set mode to:

```text
Enforced
```

For production, you may start with:

```text
Audit
```

7. Click **Save**.

### Why this matters

Attackers often try passwords based on company names, seasons, years, and common patterns. A custom banned password list blocks passwords that are predictable for your organization.

### SC-300 exam tip

Know the difference between:

* Microsoft global banned password list
* Custom banned password list
* Cloud password protection
* On-premises password protection using agents

---

### 6b: Test Password Protection

1. Try to reset a test user's password to something that includes a banned word, such as:

```text
TeachRich2026!
```

2. Confirm that it is rejected.
3. Try a stronger password that does not contain banned words.
4. Confirm that it is accepted.
5. Screenshot or document the result.

### Why this matters

Testing proves that the policy is not just configured but working.

---

## Step 7: Disable an Account and Revoke Sessions

This step simulates what you would do during an account compromise.

---

### 7a: Disable via the Portal

1. Go to **Users**.
2. Select a test user.
3. Click **Edit properties**.
4. Set **Account enabled** to:

```text
No
```

5. Click **Save**.

### Why this matters

Disabling the account prevents new sign-ins. However, existing sessions may still remain active until tokens expire or are revoked.

---

### 7b: Revoke Sessions via the Portal

1. Open the same user profile.
2. Click **Revoke sessions**.
3. Confirm the action.

### Why this matters

Revoking sessions invalidates refresh tokens and forces the user to sign in again. During an incident, disabling the account and revoking sessions should be done together.

---

### 7c: Disable and Revoke via PowerShell

```powershell
# ================================
# Disable User and Revoke Sessions
# ================================

Import-Module Microsoft.Graph.Users

$UserPrincipalName = "zara.ahmed@teachrich.com"

try {
    $user = Get-MgUser `
        -UserId $UserPrincipalName `
        -Property "id,displayName,userPrincipalName,accountEnabled" `
        -ErrorAction Stop

    Update-MgUser `
        -UserId $user.Id `
        -AccountEnabled:$false `
        -ErrorAction Stop

    Write-Host "[DISABLED] $($user.DisplayName) <$($user.UserPrincipalName)>" -ForegroundColor Green

    Invoke-MgGraphRequest `
        -Method POST `
        -Uri "https://graph.microsoft.com/v1.0/users/$($user.Id)/revokeSignInSessions" `
        -ErrorAction Stop

    Write-Host "[REVOKED] All sign-in sessions revoked" -ForegroundColor Green

    # Re-enable when done testing
    Update-MgUser `
        -UserId $user.Id `
        -AccountEnabled:$true `
        -ErrorAction Stop

    Write-Host "[ENABLED] Test account re-enabled" -ForegroundColor Yellow
}
catch {
    Write-Host "[FAILED] Could not complete disable/revoke test" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Finds a test user, disables the account, revokes active sign-in sessions, and then re-enables the account for lab use.
>
> **Why it matters:** During a suspected compromise, disabling the account is not always enough. The user may still have active sessions. Revoking sessions helps force sign-out and blocks continued access through existing tokens.
>
> **Line by line:**
>
> * `$UserPrincipalName` stores the test account.
> * `Get-MgUser` retrieves the user object.
> * `Update-MgUser -AccountEnabled:$false` disables the account.
> * `Invoke-MgGraphRequest ... /revokeSignInSessions` revokes the user's refresh tokens.
> * The second `Update-MgUser -AccountEnabled:$true` re-enables the test account after the lab.
> * `try/catch` provides clean error output if any step fails.
>
> **Watch out for:** In a real incident, you would not automatically re-enable the account. You would investigate, reset credentials, review sign-in logs, check MFA methods, and confirm the account is safe first.

### SC-300 exam tip

Know the difference between disabling an account and revoking sessions:

| Action              | Purpose                               |
| ------------------- | ------------------------------------- |
| Disable account     | Blocks new sign-ins                   |
| Revoke sessions     | Invalidates existing refresh tokens   |
| Reset password      | Changes the credential                |
| Review sign-in logs | Helps investigate suspicious activity |

---

## Step 8: Verify Combined Registration

> **Portal note (2026):** The "Combined security information registration" toggle no longer exists in the Microsoft Entra admin center. Combined registration is now enabled by default for all tenants and cannot be disabled. There is nothing to configure. Proceed directly to testing below.

Combined registration allows users to register MFA and SSPR methods in a single experience at `https://aka.ms/mysecurityinfo`. Because it is on by default, your focus in this step is to verify that it is working correctly for TeachRich users.

### Test combined registration

1. Open an incognito/private browser window.
2. Go to:

```text
https://aka.ms/mysecurityinfo
```

3. Sign in as a test user:

```text
zara.ahmed@teachrich.com
```

4. Add Microsoft Authenticator.
5. Add a phone number or email method.
6. Confirm that the same security information page supports both MFA and SSPR method registration.
7. Screenshot the registered methods for your portfolio.

### Why this matters

Combined registration improves user experience because users register security information once instead of completing separate MFA and SSPR registration flows.

### SC-300 exam tip

The exam may still reference the combined registration toggle as a concept. Know that it existed as a configurable setting and that Microsoft enabled it by default for all tenants. In the current portal, there is no toggle to find — combined registration is simply always on.

---

## Step 9: Optional PowerShell: Review Authentication Methods for a User

Use this optional report to verify which authentication methods are registered for a test user.

```powershell
# ================================
# Review User Authentication Methods
# ================================

Import-Module Microsoft.Graph.Users

$UserPrincipalName = "zara.ahmed@teachrich.com"

try {
    $user = Get-MgUser `
        -UserId $UserPrincipalName `
        -Property "id,displayName,userPrincipalName" `
        -ErrorAction Stop

    $methods = Get-MgUserAuthenticationMethod `
        -UserId $user.Id `
        -ErrorAction Stop

    Write-Host "Authentication methods for $($user.DisplayName):" -ForegroundColor Cyan

    $methods |
        Select-Object Id, AdditionalProperties |
        Format-List
}
catch {
    Write-Host "[FAILED] Could not retrieve authentication methods for $UserPrincipalName" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Retrieves the authentication methods registered to a test user.
>
> **Why it matters:** After configuring MFA, SSPR, TAP, or Authenticator, you need a way to verify what methods exist on the account.
>
> **Line by line:**
>
> * `$UserPrincipalName` stores the test account.
> * `Get-MgUser` retrieves the user object.
> * `Get-MgUserAuthenticationMethod` reads the registered authentication methods.
> * `Select-Object Id, AdditionalProperties` displays method details returned by Graph.
> * `try/catch` gives a clear error if permissions are missing.
>
> **Watch out for:** Authentication method output can be nested and may not look as clean as normal user properties. For portfolio evidence, a portal screenshot may be easier to read.

---

## Step 10: Export an Authentication Readiness Report

Create a simple report showing test users and whether their accounts are enabled.

```powershell
# ================================
# Export Authentication Readiness Report
# ================================

Import-Module Microsoft.Graph.Users

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$authReport = Get-MgUser `
    -All `
    -Property "displayName,userPrincipalName,accountEnabled,userType,department,createdDateTime" |
Where-Object {
    $_.UserPrincipalName -like "*@teachrich.com"
} |
Select-Object `
    DisplayName,
    UserPrincipalName,
    UserType,
    Department,
    AccountEnabled,
    @{Name="Created";Expression={
        if ($_.CreatedDateTime) {
            $_.CreatedDateTime.ToString("yyyy-MM-dd")
        }
        else {
            "Unknown"
        }
    }}

$authReport | Format-Table -AutoSize

$authReport | Export-Csv -Path "./reports/authentication-readiness-report.csv" -NoTypeInformation

Write-Host "Authentication readiness report saved to ./reports/authentication-readiness-report.csv" -ForegroundColor Green
Write-Host "Total users reported: $($authReport.Count)" -ForegroundColor Cyan
```

> #### 📘 Script explained
>
> **What it does:** Exports a CSV report of TeachRich users, showing whether their accounts are enabled and basic identity details.
>
> **Why it matters:** Reports provide evidence for your GitHub portfolio. They show that you can configure identity controls and also validate the tenant state afterward.
>
> **Line by line:**
>
> * `New-Item -ItemType Directory -Path ./reports -Force` creates the report folder if it does not exist.
> * `Get-MgUser -All` retrieves users from the tenant.
> * `-Property "..."` requests specific fields that may not be returned by default.
> * `Where-Object { $_.UserPrincipalName -like "*@teachrich.com" }` limits the report to TeachRich domain users.
> * `Select-Object` creates clean report columns.
> * The calculated `Created` property formats the creation date.
> * `Export-Csv -NoTypeInformation` saves the report as a clean CSV.
>
> **Watch out for:** This report does not prove MFA registration by itself. It is a basic readiness report. Combine it with portal screenshots or authentication method reports for stronger evidence.

---

## Step 11: Verify Everything

Use this checklist to confirm completion:

* [ ] Authentication strategy document created at `docs/authentication-strategy.md`
* [ ] Microsoft Authenticator enabled
* [ ] Number matching enabled
* [ ] Application name shown in Authenticator prompts
* [ ] Geographic location shown in Authenticator prompts
* [ ] SMS authentication method reviewed or enabled
* [ ] Email OTP reviewed or enabled
* [ ] `SG-FIDO2-Enabled` group created
* [ ] Passkey/FIDO2 policy configured or documented
* [ ] Temporary Access Pass enabled
* [ ] TAP generated for a test user
* [ ] TAP sign-in tested
* [ ] Security Defaults reviewed
* [ ] Per-user MFA page reviewed as legacy MFA
* [ ] Conditional Access MFA noted for Lab 6
* [ ] SSPR enabled for all users
* [ ] SSPR requires two methods
* [ ] SSPR registration enforced
* [ ] SSPR notifications enabled
* [ ] SSPR tested end-to-end
* [ ] Password protection custom banned list configured
* [ ] Password protection tested
* [ ] Account disable tested
* [ ] Session revocation tested
* [ ] Combined registration tested at aka.ms/mysecurityinfo
* [ ] Authentication methods reviewed for a user
* [ ] Authentication readiness report exported to `./reports/authentication-readiness-report.csv`

Recommended screenshots:

* Authentication methods policy overview
* Microsoft Authenticator configuration
* Temporary Access Pass policy
* TAP created for test user, with the TAP value redacted
* SSPR properties
* SSPR authentication methods
* SSPR notifications
* Password protection custom banned password list
* Per-user MFA page showing legacy state
* Security information page at aka.ms/mysecurityinfo showing registered methods
* PowerShell output for TAP creation
* PowerShell output for disable/revoke test
* Authentication readiness report CSV

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

Your app registration is missing Microsoft Graph Application permissions, admin consent, or the required administrator role.

Check:

```text
Microsoft Entra admin center
→ App registrations
→ Your app
→ API permissions
```

For TAP and authentication methods, confirm:

```text
UserAuthenticationMethod.ReadWrite.All
```

For user updates, confirm:

```text
User.ReadWrite.All
Directory.ReadWrite.All
```

For policy-related tasks, confirm:

```text
Policy.Read.All
Policy.ReadWrite.AuthenticationMethod
```

After changing permissions, click:

```text
Grant admin consent
```

Then disconnect and reconnect to Microsoft Graph.

---

### Temporary Access Pass option is missing

Check that:

* Temporary Access Pass is enabled in the authentication methods policy.
* You have permission to manage authentication methods.
* You are viewing a supported user account.
* The tenant has the required licensing/features.

---

### User cannot register Microsoft Authenticator

Check that:

* Microsoft Authenticator is enabled in the authentication methods policy.
* The user is included in the target group.
* The user is not blocked by Conditional Access.
* The user is using the correct account.
* The Authenticator app is installed and working.

---

### SSPR reset fails

Check that:

* SSPR is enabled for the user.
* The user has registered enough methods.
* The selected methods are allowed.
* The account is not blocked.
* The password meets password protection requirements.

---

### Password rejected during reset

This may be expected if the password contains a banned word.

Check:

* Microsoft global banned password list
* TeachRich custom banned password list
* Password complexity requirements
* Password history settings, if applicable

Try a stronger password that does not include company-related words.

---

### Revoke sessions does not immediately sign the user out everywhere

Session revocation invalidates refresh tokens, but some access tokens may remain valid for a short period.

During an incident, also consider:

* Disabling the account
* Resetting the password
* Reviewing sign-in logs
* Removing suspicious authentication methods
* Removing risky app consents
* Reviewing mailbox rules and forwarding settings

---

## Key Takeaways for SC-300

1. Authentication planning should map methods to user populations.
2. Microsoft Authenticator with number matching reduces MFA fatigue risk.
3. FIDO2/passkeys provide phishing-resistant authentication.
4. Temporary Access Pass is used for onboarding and account recovery.
5. Security Defaults is simple, but Conditional Access is the modern granular approach.
6. Per-user MFA is legacy but still important to recognize.
7. SSPR reduces help desk workload and requires users to register recovery methods.
8. Password protection blocks common and company-specific weak passwords.
9. Disabling an account blocks new sign-ins, while revoking sessions invalidates active refresh tokens.
10. Combined registration is enabled by default for all tenants. Users register MFA and SSPR methods in one place at aka.ms/mysecurityinfo.
11. Reports and screenshots provide evidence for your portfolio.

---

## Portfolio / Interview Summary

In this lab, I planned and configured authentication controls for the TeachRich Microsoft Entra tenant. I created an authentication strategy for different user populations, configured authentication method policies, enabled Microsoft Authenticator with number matching, reviewed FIDO2/passkeys, configured Temporary Access Pass, enabled self-service password reset, implemented Microsoft Entra password protection, tested account disablement and session revocation, and exported an authentication readiness report.

This project demonstrates practical SC-300 skills in authentication planning, MFA, SSPR, password protection, secure onboarding, account recovery, and incident response.

---

## What's Next

➡️ **Lab 6:** [Conditional Access & Identity Protection](../lab06-conditional-access/) — Build a Zero Trust Conditional Access framework, configure risk-based policies, and set up Identity Protection.

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
