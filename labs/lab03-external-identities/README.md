# Lab 3: External Identities & Cross-Tenant Access

> **Type:** Focused Lab | **Time:** 2–3 hours | **SC-300 Domain:** Implement and manage user identities (20–25%)

---

## Scenario

TeachRich works with external contractors, training partners, and partner organizations. Contractors need limited access to specific Microsoft 365 resources, such as SharePoint sites and Teams channels. Partner organizations need their employees to collaborate securely with TeachRich staff.

In this lab, you will configure B2B collaboration, invite external users securely, review guest user objects, configure cross-tenant access settings, document external identity provider options, and generate a guest user lifecycle report.

The TeachRich verified domain used throughout this lab is:

```text
teachrich.com
```

---

## SC-300 Exam Objectives Covered

* Manage external collaboration settings in Microsoft Entra ID
* Invite external users individually and in bulk
* Manage external user accounts in Microsoft Entra ID
* Implement cross-tenant access settings
* Understand cross-tenant synchronization concepts
* Configure or document external identity providers, including Google federation and email one-time passcode
* Monitor and report on guest user lifecycle status

---

## Prerequisites

Before starting this lab, you should have:

* Completed Labs 1–2, including tenant configuration and certificate-based authentication
* A Microsoft 365 E3, Microsoft 365 E5, or similar subscription
* A verified Microsoft Entra domain, such as `teachrich.com`
* At least one personal email address to use as a test guest, such as Gmail or Outlook
* Ideally, a second Microsoft Entra tenant for cross-tenant testing. This is optional and can be documented if you do not have one.
* Global Administrator or External Identity Provider Administrator access for the portal steps
* PowerShell 7 installed
* Microsoft Graph PowerShell SDK installed
* Certificate-based authentication configured from Lab 1

> **Important:** Some features in this lab may depend on your tenant license and role assignments. If a blade or option does not appear, document what you see and explain the required feature or role in your lab notes.

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
User.Invite.All
User.ReadWrite.All
Directory.ReadWrite.All
Group.ReadWrite.All
AuditLog.Read.All
Policy.Read.All
Policy.ReadWrite.CrossTenantAccess
```

Then click:

```text
Grant admin consent
```

Use the permissions based on the tasks you plan to complete:

| Task                                           | Useful permissions                                                     |
| ---------------------------------------------- | ---------------------------------------------------------------------- |
| Invite guests                                  | `User.Invite.All`, `User.ReadWrite.All`, `Directory.ReadWrite.All`     |
| Read and update guest users                    | `User.ReadWrite.All`, `Directory.ReadWrite.All`                        |
| Add guests to groups                           | `Group.ReadWrite.All`, `Directory.ReadWrite.All`                       |
| Review sign-in activity                        | `AuditLog.Read.All`, `Directory.Read.All` or `Directory.ReadWrite.All` |
| Read cross-tenant access settings              | `Policy.Read.All`                                                      |
| Manage cross-tenant access settings with Graph | `Policy.ReadWrite.CrossTenantAccess`                                   |

> **Troubleshooting note:** If you change API permissions or role assignments, disconnect and reconnect to Microsoft Graph before testing again.

> #### 📘 Why these permissions matter
>
> This lab uses **app-only authentication**, meaning your script signs in as the application instead of as a person. That is useful for automation, but it also means the app must be granted permission to act across the tenant.
>
> For example, `User.Invite.All` allows the app to create B2B invitations. `AuditLog.Read.All` is needed when you want to report on sign-in activity. Cross-tenant access settings are policy objects, so they require policy permissions. In production, you would grant the smallest set of permissions needed for the automation, not every permission listed above.

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
> **What it does:** Signs in to Microsoft Graph as the app registration created in Lab 1, using the certificate file instead of a username and password.
>
> **Why it matters:** Guest invitations and lifecycle reports are good examples of tasks that may eventually be automated. Certificate-based app-only authentication is a strong pattern because it can run unattended without asking an administrator to sign in interactively.
>
> **Line by line:**
>
> * `$TenantId` identifies the TeachRich Microsoft Entra tenant.
> * `$ClientId` identifies the app registration that has Microsoft Graph permissions.
> * `$TenantDomain = "teachrich.com"` stores the TeachRich domain so you can reuse it in scripts and lab notes.
> * `Read-Host ... -AsSecureString` prompts for the PFX password without displaying it on screen.
> * `New-Object ...X509Certificate2(...)` loads the certificate and private key from the `.pfx` file.
> * `Connect-MgGraph ... -Certificate $cert` authenticates to Microsoft Graph as the application.
> * `Get-MgContext` confirms which tenant and app you are connected as.
>
> **Watch out for:** Do not commit your `.pfx` file or password to GitHub. In a production environment, certificates should be stored securely, such as in a certificate store or Azure Key Vault.

---

## Step 1: Configure External Collaboration Settings

**Where:** Microsoft Entra admin center → External Identities → External collaboration settings

These settings control the overall B2B collaboration policy for the TeachRich tenant.

| Setting                                          | Recommended Value                                                                                    | Why                                                                   |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Guest user access                                | **Guest users have limited access to properties and memberships of directory objects**               | Prevents guests from browsing too much internal directory information |
| Guest invite settings                            | **Member users and users assigned to specific admin roles can invite guest users**                   | Controls who can bring external people into the tenant                |
| Enable guest self-service sign-up via user flows | **No**                                                                                               | Keeps control over who enters the TeachRich tenant                    |
| Collaboration restrictions                       | **Allow invitations only to the specified domains** or **Deny invitations to the specified domains** | Controls which external email domains can be invited                  |

For collaboration restrictions, test both options:

1. First, set the configuration to **Allow invitations only to the specified domains**.
2. Add test domains such as:

```text
gmail.com
outlook.com
```

3. Save and test by inviting a guest from an allowed domain.
4. Then switch to **Deny invitations to the specified domains** and add a test domain that you want to block.
5. Save and understand the difference.

> **Important:** Do not add `teachrich.com` as an external guest domain restriction unless you are testing a very specific scenario. `teachrich.com` is the TeachRich internal tenant domain. Collaboration restrictions are usually used to control **external** domains.

### Why this matters

External collaboration settings are the first layer of defense for B2B collaboration. They determine who can invite guests, what guests can see, and which external domains are allowed or blocked.

### SC-300 exam tip

Know the difference between allow-list and deny-list approaches. An allow-list is stricter because only approved domains can be invited. A deny-list is more open because all domains are allowed except the blocked ones.

### How to explain this in an interview

> I configured external collaboration settings for the TeachRich tenant to control B2B access. Guest invitations are restricted so that not every user can invite external people. Guest users have limited directory visibility, and domain restrictions can be used to allow only approved partner organizations.

---

## Step 2: Invite External Users Individually

**Where:** Microsoft Entra admin center → Identity → Users → All users → New user → Invite external user

---

### 2a: Invite a Guest via the Portal

1. Click **New user**.
2. Select **Invite external user**.
3. Enter a personal test email address, such as Gmail or Outlook.
4. Use a display name like:

```text
External Contractor - Test User
```

5. Add a personal message:

```text
Welcome to TeachRich. You have been invited to collaborate on Project Alpha resources.
```

6. Click **Invite**.
7. Check the personal email inbox.
8. Accept the invitation and complete the redemption process.

---

### 2b: Observe the Guest User Object

After the guest accepts:

1. Go to **Users** → **All users**.
2. Find the guest user.
3. Confirm that **User type** shows:

```text
Guest
```

4. Open the user profile and review:

* Source
* User type
* Creation type
* Identities
* Groups
* Sign-in logs, if available

### Why this matters

The invitation and the redemption are not the same thing. The invitation creates or updates the guest object. Redemption is when the external person accepts and proves control of the email or external identity.

---

### 2c: Invite a Guest via PowerShell

Use a test guest email address that you control.

```powershell
# ================================
# Invite a Single Guest User
# ================================

Import-Module Microsoft.Graph.Identity.SignIns

$guestEmail = "testguest@gmail.com"
$guestDisplayName = "External Contractor - Test Guest"
$redirectUrl = "https://myapplications.microsoft.com"

try {
    $invitation = New-MgInvitation `
        -InvitedUserEmailAddress $guestEmail `
        -InviteRedirectUrl $redirectUrl `
        -InvitedUserDisplayName $guestDisplayName `
        -SendInvitationMessage:$true `
        -InvitedUserMessageInfo @{
            CustomizedMessageBody = "Welcome to TeachRich. Please accept this invitation to access the resources assigned to you."
        } `
        -ErrorAction Stop

    Write-Host "[INVITED] $guestEmail" -ForegroundColor Green
    Write-Host "Redemption URL: $($invitation.InviteRedeemUrl)" -ForegroundColor Cyan
}
catch {
    Write-Host "[FAILED] $guestEmail" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Sends a B2B invitation to one external user and prints the redemption URL.
>
> **Why it matters:** This is the PowerShell version of the portal invitation process. It is useful when you need a repeatable process for inviting contractors, partners, or temporary project users.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Identity.SignIns` loads the Microsoft Graph command that contains `New-MgInvitation`.
> * `$guestEmail` stores the external email address you are inviting.
> * `$guestDisplayName` controls how the guest appears in the TeachRich directory.
> * `$redirectUrl` controls where the guest lands after accepting the invitation.
> * `New-MgInvitation` creates the B2B invitation and sends the email because `-SendInvitationMessage:$true` is included.
> * `-InvitedUserMessageInfo` adds a custom message to make the invitation clear and professional.
> * `try/catch` gives you clean success or failure output instead of stopping the whole script with an unclear error.
>
> **Watch out for:** If the invitation fails with an insufficient privileges error, check that your app has `User.Invite.All` and that admin consent has been granted. Also check the external collaboration settings from Step 1, because domain restrictions can block invitations.

### SC-300 exam tip

Know the difference between **invitation** and **redemption**. Also know that guests may redeem invitations using Microsoft Entra accounts, Microsoft accounts, Google federation, SAML/WS-Fed federation, or email one-time passcode depending on the tenant configuration.

---

## Step 3: Invite External Users in Bulk

Bulk invitation can be done through the portal or through PowerShell.

---

### 3a: Bulk Invite via the Portal

**Where:** Microsoft Entra admin center → Identity → Users → All users → Bulk operations → Bulk invite

1. Click **Bulk operations**.
2. Select **Bulk invite**.
3. Download the CSV template.
4. Fill in 3–4 guest users.

Example:

| Email address to invite                         | Redirection url                      | Send invite message | Customized message                                   |
| ----------------------------------------------- | ------------------------------------ | ------------------- | ---------------------------------------------------- |
| [guest1@gmail.com](mailto:guest1@gmail.com)     | https://myapplications.microsoft.com | TRUE                | Welcome to TeachRich. Please accept this invitation. |
| [guest2@outlook.com](mailto:guest2@outlook.com) | https://myapplications.microsoft.com | TRUE                | Welcome to TeachRich. Please accept this invitation. |
| [guest3@gmail.com](mailto:guest3@gmail.com)     | https://myapplications.microsoft.com | TRUE                | Welcome to TeachRich. Please accept this invitation. |

5. Upload the CSV.
6. Submit the bulk invitation job.
7. Review the results.

### Why this matters

The portal bulk invite option is useful for one-off invitation batches, especially when an administrator receives a list of contractors from a project manager.

---

### 3b: Create a Bulk Guest CSV for PowerShell

Create a `data` folder and a guest invitation CSV.

```powershell
New-Item -ItemType Directory -Path ./data -Force | Out-Null

@"
Email,DisplayName,Project,EndDate
bulkguest1@gmail.com,External Contractor - Bulk Guest 1,Project Alpha,2026-12-31
bulkguest2@gmail.com,External Contractor - Bulk Guest 2,Project Alpha,2026-12-31
bulkguest3@outlook.com,External Partner - Bulk Guest 3,Partner Collaboration,2026-12-31
"@ | Out-File -FilePath ./data/GuestInvites.csv -Encoding UTF8

Test-Path ./data/GuestInvites.csv

Import-Csv ./data/GuestInvites.csv | Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Creates a small CSV file that contains the external users you want to invite.
>
> **Why it matters:** Separating the guest list from the invitation logic makes the process repeatable. In a real organization, the CSV might come from HR, procurement, a project manager, or a partner onboarding workflow.
>
> **Line by line:**
>
> * `New-Item -ItemType Directory -Path ./data -Force` creates the folder if it does not already exist.
> * The `@" ... "@` block is a here-string that stores multi-line CSV content.
> * `Out-File ... -Encoding UTF8` writes the CSV to disk.
> * `Test-Path` confirms the file exists.
> * `Import-Csv ... | Format-Table` previews the rows before you use them.
>
> **Watch out for:** Keep the CSV clean. Extra spaces, misspelled email addresses, or blocked domains can cause invitation failures.

---

### 3c: Bulk Invite via PowerShell

This script reads the CSV and invites each guest. It also checks whether a guest with the same email already exists, so the script is safer to run more than once.

```powershell
# ================================
# Bulk Invite Guest Users from CSV
# ================================

Import-Module Microsoft.Graph.Users
Import-Module Microsoft.Graph.Identity.SignIns

$csvPath = "./data/GuestInvites.csv"
$redirectUrl = "https://myapplications.microsoft.com"

if (-not (Test-Path $csvPath)) {
    Write-Host "CSV file not found at $csvPath" -ForegroundColor Red
    return
}

$guestsToInvite = Import-Csv $csvPath

foreach ($guest in $guestsToInvite) {

    $email = $guest.Email.Trim()
    $displayName = $guest.DisplayName.Trim()

    Write-Host "`nChecking guest: $email" -ForegroundColor Cyan

    try {
        $existingGuest = Get-MgUser `
            -Filter "mail eq '$email'" `
            -Property "id,displayName,mail,userPrincipalName,userType" `
            -ErrorAction SilentlyContinue

        if ($existingGuest) {
            Write-Host "[EXISTS]  $email already exists as $($existingGuest.UserPrincipalName)" -ForegroundColor Yellow
            continue
        }

        $message = "Welcome to TeachRich. You have been invited for $($guest.Project). Access is expected to end on $($guest.EndDate)."

        New-MgInvitation `
            -InvitedUserEmailAddress $email `
            -InviteRedirectUrl $redirectUrl `
            -InvitedUserDisplayName $displayName `
            -SendInvitationMessage:$true `
            -InvitedUserMessageInfo @{
                CustomizedMessageBody = $message
            } `
            -ErrorAction Stop | Out-Null

        Write-Host "[INVITED] $email - $($guest.Project)" -ForegroundColor Green
    }
    catch {
        Write-Host "[FAILED]  $email" -ForegroundColor Red
        Write-Host $_.Exception.Message -ForegroundColor Yellow
    }
}

Write-Host "`nBulk guest invitation process completed." -ForegroundColor Green
```

> #### 📘 Script explained
>
> **What it does:** Reads the guest invitation CSV, checks whether each guest already exists, sends invitations for new guests, and logs the result for every row.
>
> **Why it matters:** This is closer to a production onboarding pattern. It is repeatable, has error handling, avoids duplicate work where possible, and produces a clear run log.
>
> **Line by line:**
>
> * `Import-Module Microsoft.Graph.Users`    }
>   catch {
>   Write-Host "[FAILED]  $email" -ForegroundColor Red
>   Write- loads user lookup commands.
> * `Import-Module Microsoft.Graph.Identity.SignIns` loads the invitation command.
> * `$csvPath` points to the CSV file created in Step 3b.
> * `if (-not (Test-Path $csvPath)) { ... return }` stops early with a clear error if the file is missing.
> * `Import-Csv` loads the rows as PowerShell objects.
> * `.Trim()` removes accidental spaces around email addresses and display names.
> * `Get-MgUser -Filter "mail eq '$email'"` checks whether the guest already exists.
> * `continue` skips existing guests so the script does not invite the same person repeatedly.
> * `$message` builds a custom message using the project and end date from the CSV.
> * `New-MgInvitation` sends the B2B invitation.
> * `try/catch` makes sure one bad row does not stop the whole batch.
>
> **Watch out for:** The `mail` property is a practical check for this lab, but guest objects can sometimes have unusual identity values after redemption. In production, you may also search by `userPrincipalName`, external identities, or maintain your own invitation tracking report.

---

## Step 4: Manage External User Accounts

After inviting guests, you need to monitor and manage their lifecycle.

---

### 4a: Review All Guest Users

```powershell
# ================================
# List All Guest Users
# ================================

Import-Module Microsoft.Graph.Users

Get-MgUser `
    -All `
    -Filter "userType eq 'Guest'" `
    -Property "displayName,mail,userPrincipalName,createdDateTime,externalUserState,userType" |
Select-Object `
    DisplayName,
    Mail,
    UserPrincipalName,
    UserType,
    CreatedDateTime,
    @{Name="ExternalUserState";Expression={$_.ExternalUserState}} |
Format-Table -AutoSize
```

> #### 📘 Script explained
>
> **What it does:** Lists all guest accounts in the TeachRich tenant and shows their basic lifecycle status.
>
> **Why it matters:** External users are often temporary. You need a quick way to see who is in the tenant, when they were created, and whether they are guests rather than internal members.
>
> **Line by line:**
>
> * `Get-MgUser -All` retrieves all matching users instead of only the first page.
> * `-Filter "userType eq 'Guest'"` limits the result to external guest accounts.
> * `-Property "..."` explicitly requests properties that are not always returned by default.
> * `Select-Object` chooses the columns that make the report readable.
> * The calculated property for `ExternalUserState` surfaces the guest redemption state in a clear column.
> * `Format-Table -AutoSize` prints the results in a clean table.
>
> **Watch out for:** If a column appears blank, it may be because the guest has not redeemed the invitation yet, the property was not populated, or your app does not have enough permission to read that field.

---

### 4b: Convert a Guest to a Member and Back

Sometimes an external contractor becomes a full-time employee. In that case, their user type may need to change from `Guest` to `Member`.

Use a test guest account only.

```powershell
# ================================
# Convert Guest to Member and Back
# ================================

Import-Module Microsoft.Graph.Users

$guestEmail = "testguest@gmail.com"

try {
    $guest = Get-MgUser `
        -Filter "mail eq '$guestEmail'" `
        -Property "id,displayName,mail,userPrincipalName,userType" `
        -ErrorAction Stop

    if (-not $guest) {
        Write-Host "Guest not found: $guestEmail" -ForegroundColor Yellow
        return
    }

    Update-MgUser -UserId $guest.Id -UserType "Member" -ErrorAction Stop
    Write-Host "[UPDATED] $($guest.DisplayName) converted to Member" -ForegroundColor Green

    Start-Sleep -Seconds 5

    Update-MgUser -UserId $guest.Id -UserType "Guest" -ErrorAction Stop
    Write-Host "[UPDATED] $($guest.DisplayName) converted back to Guest" -ForegroundColor Green
}
catch {
    Write-Host "[FAILED] User type update failed" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Yellow
}
```

> #### 📘 Script explained
>
> **What it does:** Finds a guest user by email, changes the user type to `Member`, then changes it back to `Guest`.
>
> **Why it matters:** The SC-300 exam expects you to understand the difference between guest and member accounts. This also demonstrates lifecycle management when a contractor becomes a full employee or when an account was created with the wrong user type.
>
> **Line by line:**
>
> * `$guestEmail` stores the external email address you want to test.
> * `Get-MgUser -Filter "mail eq '$guestEmail'"` finds the matching guest object.
> * `if (-not $guest) { ... return }` stops safely if the user is not found.
> * `Update-MgUser -UserType "Member"` changes the account type to member.
> * `Start-Sleep -Seconds 5` gives the directory a few seconds to process the first change.
> * `Update-MgUser -UserType "Guest"` changes the account back to guest.
> * `try/catch` logs the exact error if the update fails.
>
> **Watch out for:** Changing user type does not automatically give the account the same licensing, group membership, or permissions as a normal internal employee. Review access carefully before using this in a real tenant.

---

### 4c: Find Stale Guest Accounts

Stale guest accounts are a common security risk. This script finds guests that have never signed in or have not signed in for 90 days.

```powershell
# ================================
# Find Stale Guest Accounts
# ================================

Import-Module Microsoft.Graph.Users

$staleDate = (Get-Date).AddDays(-90)

$staleGuests = Get-MgUser `
    -All `
    -Filter "userType eq 'Guest'" `
    -Property "id,displayName,mail,userPrincipalName,createdDateTime,signInActivity" |
Where-Object {
    -not $_.SignInActivity.LastSignInDateTime -or
    $_.SignInActivity.LastSignInDateTime -lt $staleDate
} |
Select-Object `
    DisplayName,
    Mail,
    UserPrincipalName,
    CreatedDateTime,
    @{Name="LastSignIn";Expression={
        if ($_.SignInActivity.LastSignInDateTime) {
            $_.SignInActivity.LastSignInDateTime
        }
        else {
            "Never"
        }
    }}

$staleGuests | Format-Table -AutoSize

Write-Host "Stale guests found: $($staleGuests.Count)" -ForegroundColor Cyan
```

> #### 📘 Script explained
>
> **What it does:** Finds guest accounts where the last sign-in is older than 90 days or where no sign-in is recorded.
>
> **Why it matters:** Guest accounts are often created for short-term projects and then forgotten. A stale guest account may still have access to groups, Teams, SharePoint sites, or applications long after the business need has ended.
>
> **Line by line:**
>
> * `$staleDate = (Get-Date).AddDays(-90)` calculates the cutoff date.
> * `Get-MgUser -Filter "userType eq 'Guest'"` retrieves only guest users.
> * `-Property "...,signInActivity"` explicitly requests sign-in activity.
> * `Where-Object` filters for guests with no sign-in date or a last sign-in older than the cutoff.
> * The calculated `LastSignIn` column displays `Never` when there is no recorded sign-in.
> * `Write-Host` prints the total number of stale guests.
>
> **Watch out for:** Reading `SignInActivity` requires audit-related permissions such as `AuditLog.Read.All`. If your output is blank or you get an authorization error, review the app permissions and admin consent.

### SC-300 exam tip

Know how to manage the guest lifecycle: invite, redeem, review, convert, monitor, and remove stale access.

---

## Step 5: Configure Cross-Tenant Access Settings

**Where:** Microsoft Entra admin center → External Identities → Cross-tenant access settings

Cross-tenant access settings control how the TeachRich tenant interacts with other Microsoft Entra tenants.

There are two directions:

| Direction           | Meaning                                                          |
| ------------------- | ---------------------------------------------------------------- |
| **Inbound access**  | Controls how users from other tenants access TeachRich resources |
| **Outbound access** | Controls how TeachRich users access resources in other tenants   |

---

### 5a: Configure Default Settings

1. Go to **External Identities** → **Cross-tenant access settings**.
2. Click **Default settings**.
3. Review the inbound and outbound defaults.
4. Under **Trust settings**, review the option to trust MFA from Microsoft Entra tenants.

For a lab environment, document whether you enable:

```text
Trust multi-factor authentication from Microsoft Entra tenants
```

### Why this matters

If MFA trust is enabled, TeachRich can trust that the guest completed MFA in their home tenant instead of forcing a second MFA prompt in the TeachRich tenant. This can improve user experience, but it should only be enabled when the partner tenant is trusted.

---

### 5b: Add an Organization-Specific Policy

1. Click **Add organization**.
2. Enter a partner tenant ID or domain name.
3. If you do not have a second tenant, document this step with screenshots and notes.
4. Configure **Inbound access**.
5. Configure **Outbound access**.
6. Save the configuration.

Example lab policy:

| Policy Area                | Example Configuration                                 | Purpose                                           |
| -------------------------- | ----------------------------------------------------- | ------------------------------------------------- |
| Inbound B2B collaboration  | Allow selected external users or groups               | Limits which partner users can access TeachRich   |
| Inbound applications       | Allow only selected apps, such as SharePoint Online   | Limits what partner users can access              |
| Outbound B2B collaboration | Allow TeachRich users to collaborate with the partner | Supports collaboration with the partner tenant    |
| Trust settings             | Trust MFA only for approved partner tenants           | Avoids duplicate MFA prompts for trusted partners |

---

### 5c: Optional PowerShell: Read Cross-Tenant Access Default Settings

This optional command lets you document the current default cross-tenant access policy.

```powershell
# ================================
# Read Cross-Tenant Access Defaults
# ================================

$defaultPolicy = Invoke-MgGraphRequest `
    -Method GET `
    -Uri "https://graph.microsoft.com/v1.0/policies/crossTenantAccessPolicy/default"

$defaultPolicy | ConvertTo-Json -Depth 10
```

> #### 📘 Script explained
>
> **What it does:** Reads the default cross-tenant access policy directly from Microsoft Graph and prints it as JSON.
>
> **Why it matters:** The portal is easier for configuration, but Graph output gives you evidence for documentation, auditing, and GitHub portfolio screenshots.
>
> **Line by line:**
>
> * `Invoke-MgGraphRequest` sends a direct Microsoft Graph request.
> * `-Method GET` means you are reading data, not changing it.
> * The URI points to the default cross-tenant access policy object.
> * `ConvertTo-Json -Depth 10` expands nested policy settings so you can inspect and save them.
>
> **Watch out for:** Reading policy settings requires policy read permissions, such as `Policy.Read.All`. Updating cross-tenant access settings requires stronger policy write permissions and should be done carefully.

---

### 5d: Document the Cross-Tenant Architecture

Create a short note or markdown file that explains:

* What the default cross-tenant policy allows
* Which partner organizations have custom policies
* Which applications external users can access
* Whether MFA trust is enabled
* Why the configuration is appropriate for TeachRich

### How to explain this in an interview

> I configured cross-tenant access settings for the TeachRich tenant. Inbound policies control how partner users access TeachRich resources, while outbound policies control how TeachRich users access partner tenants. I also reviewed MFA trust settings to reduce duplicate MFA prompts only where the partner tenant is trusted.

### SC-300 exam tip

Know the difference between inbound and outbound access, default settings and organization-specific settings, and how MFA trust affects guest authentication.

---

## Step 6: Configure or Document an External Identity Provider

**Where:** Microsoft Entra admin center → External Identities → All identity providers

External identity providers let guests sign in using accounts they already have.

---

### 6a: Configure Google as a Social Identity Provider

1. Go to **External Identities** → **All identity providers**.
2. Click **Google**.
3. Create or use a Google API Client ID and Client Secret.
4. In Google Cloud Console, configure the OAuth client as a web application.
5. Use the redirect URI format shown in the Entra configuration page. It commonly follows this pattern:

```text
https://login.microsoftonline.com/te/your-tenant-id/oauth2/authresp
```

6. Copy the Client ID and Client Secret into Microsoft Entra.
7. Save the configuration.

If you do not want to complete the Google setup, document the process and take screenshots of the configuration page.

### Why this matters

Google federation allows invited Gmail users to redeem invitations with their Google identity instead of creating a separate Microsoft account.

---

### 6b: Configure or Verify Email One-Time Passcode

1. Go to **External Identities** → **All identity providers**.
2. Verify that **Email one-time passcode** is available and enabled.
3. Document what the setting does.

Email OTP allows guests who do not have a Microsoft account, Microsoft Entra account, Google federation, or direct federation to authenticate using a code sent to their email.

### SC-300 exam tip

Know the common guest authentication options:

* Microsoft Entra account
* Microsoft account
* Google federation
* SAML/WS-Fed direct federation
* Email one-time passcode

---

## Step 7: Generate an External Users Report

Create a `reports` folder and export a guest user report.

```powershell
# ================================
# Export Guest User Report
# ================================

Import-Module Microsoft.Graph.Users

New-Item -ItemType Directory -Path ./reports -Force | Out-Null

$guestReport = Get-MgUser `
    -All `
    -Filter "userType eq 'Guest'" `
    -Property "displayName,mail,userPrincipalName,createdDateTime,externalUserState,signInActivity,creationType,userType" |
Select-Object `
    DisplayName,
    Mail,
    UserPrincipalName,
    UserType,
    CreationType,
    ExternalUserState,
    @{Name="Created";Expression={
        if ($_.CreatedDateTime) {
            $_.CreatedDateTime.ToString("yyyy-MM-dd")
        }
        else {
            "Unknown"
        }
    }},
    @{Name="LastSignIn";Expression={
        if ($_.SignInActivity.LastSignInDateTime) {
            $_.SignInActivity.LastSignInDateTime.ToString("yyyy-MM-dd")
        }
        else {
            "Never"
        }
    }}

$guestReport | Format-Table -AutoSize

$guestReport | Export-Csv -Path "./reports/guest-users-report.csv" -NoTypeInformation

Write-Host "Guest user report exported to ./reports/guest-users-report.csv" -ForegroundColor Green
Write-Host "Total guests: $($guestReport.Count)" -ForegroundColor Cyan
```

> #### 📘 Script explained
>
> **What it does:** Creates a CSV report of all TeachRich guest users, including their email, UPN, guest state, creation date, and last sign-in date.
>
> **Why it matters:** Reporting is evidence. A guest user report helps prove that you can monitor external identities, identify stale access, and support access reviews or cleanup activities.
>
> **Line by line:**
>
> * `New-Item -ItemType Directory -Path ./reports -Force` creates the reports folder if needed.
> * `Get-MgUser -Filter "userType eq 'Guest'"` retrieves only guest accounts.
> * `-Property "..."` requests the guest and sign-in properties needed for the report.
> * `Select-Object` shapes the output into clean report columns.
> * The `Created` calculated property formats the creation date as `yyyy-MM-dd`.
> * The `LastSignIn` calculated property shows a date or `Never` when there is no sign-in record.
> * `Format-Table` lets you preview the report in the terminal.
> * `Export-Csv -NoTypeInformation` saves the report as a clean CSV file.
> * The final `Write-Host` lines confirm the export path and total number of guests.
>
> **Watch out for:** If `LastSignIn` is blank or unavailable, check `AuditLog.Read.All` and admin consent. Also remember that reporting does not remove access by itself. Use the report to support an access review or cleanup process.

---

## Step 8: Verify Everything

Use this checklist to confirm completion:

* [ ] External collaboration settings configured
* [ ] Guest invite settings reviewed
* [ ] Domain allow-list or deny-list tested
* [ ] Guest user invited via portal
* [ ] Guest invitation redemption completed
* [ ] Guest user properties reviewed
* [ ] Guest user invited via PowerShell
* [ ] Bulk guest invitation CSV created
* [ ] Bulk guest invitations completed via portal or PowerShell
* [ ] All guest users reviewed with PowerShell
* [ ] Guest-to-member conversion tested with a test account
* [ ] Stale guest account query created
* [ ] Cross-tenant access default settings reviewed
* [ ] Organization-specific cross-tenant access policy created or documented
* [ ] MFA trust setting reviewed and documented
* [ ] External identity provider configuration reviewed or documented
* [ ] Email one-time passcode reviewed
* [ ] Guest user report exported to `./reports/guest-users-report.csv`

Recommended screenshots:

* External collaboration settings
* Guest invite settings
* Domain restriction settings
* Portal guest invitation screen
* Guest user profile showing `User type: Guest`
* PowerShell invitation output
* Bulk invite result
* Cross-tenant access default settings
* Organization-specific cross-tenant policy
* Identity providers page
* Email one-time passcode setting
* Guest user report CSV

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

Your app registration is missing Microsoft Graph Application permissions, admin consent, or an Entra role assignment.

Check:

```text
Microsoft Entra admin center
→ App registrations
→ Your app
→ API permissions
```

For guest invitations, confirm:

```text
User.Invite.All
User.ReadWrite.All
Directory.ReadWrite.All
```

For sign-in reporting, confirm:

```text
AuditLog.Read.All
Directory.Read.All
```

For cross-tenant access policy reading or management, confirm:

```text
Policy.Read.All
Policy.ReadWrite.CrossTenantAccess
```

After changing permissions, click:

```text
Grant admin consent
```

Then disconnect and reconnect to Microsoft Graph.

---

### Error: Invitation is blocked by policy

Check the external collaboration settings.

Common causes:

* The guest email domain is not on the allow-list.
* The guest email domain is on the deny-list.
* Guest invitations are restricted to admins only.
* The app or user does not have permission to invite guests.

---

### Error: Guest user already exists

The guest may already exist from a previous invitation.

Search for the guest:

```powershell
Get-MgUser -All -Filter "userType eq 'Guest'" |
Where-Object {
    $_.Mail -eq "testguest@gmail.com" -or
    $_.UserPrincipalName -like "*testguest*"
} |
Select-Object DisplayName, Mail, UserPrincipalName, UserType |
Format-Table -AutoSize
```

You can then decide whether to reuse the existing guest object, reset redemption, or remove and reinvite the account.

---

### SignInActivity is empty or unavailable

Check that your app has audit log permissions:

```text
AuditLog.Read.All
```

Also remember that a newly invited guest may not have signed in yet, so `LastSignIn` may correctly show as `Never`.

---

### Cross-tenant access settings are not available

Check that:

* You have the correct administrator role.
* Your tenant supports the feature.
* You are in the correct portal blade.
* Your app has policy permissions if using Microsoft Graph.

---

## Key Takeaways for SC-300

1. External collaboration settings control who can invite guests and which external domains are allowed or blocked.
2. Guest users have a separate user type and usually have limited directory visibility.
3. Invitation and redemption are different stages of the B2B lifecycle.
4. Bulk invitation can be done through the portal or automated with PowerShell.
5. Cross-tenant access has inbound and outbound policy directions.
6. Default cross-tenant settings apply broadly, while organization-specific settings apply to selected partner tenants.
7. MFA trust can reduce duplicate MFA prompts for trusted partner tenants.
8. External identity providers determine how guests authenticate.
9. Email one-time passcode is an important fallback authentication method for guests.
10. Guest reports help identify stale external access and support access reviews.

---

## Portfolio / Interview Summary

In this lab, I configured external identity and B2B collaboration settings for the TeachRich Microsoft Entra tenant. I invited guest users through the Microsoft Entra admin center and Microsoft Graph PowerShell, created a repeatable bulk invitation workflow, reviewed and updated guest user accounts, identified stale guests, reviewed cross-tenant access settings, documented external identity provider options, and exported a guest user lifecycle report.

This project demonstrates practical SC-300 skills in external identity management, B2B collaboration, cross-tenant access governance, automation, and reporting.

---

## What's Next

➡️ **Lab 4:** [Hybrid Identity with Entra Connect](../lab04-hybrid-identity/) — On-premises AD synchronization, password hash sync, pass-through authentication, and Seamless SSO.

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
