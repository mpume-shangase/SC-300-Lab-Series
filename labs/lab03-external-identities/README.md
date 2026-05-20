# Lab 3: External Identities & Cross-Tenant Access

> **Type:** Focused Lab | **Time:** 2-3 hours | **SC-300 Domain:** Implement and manage user identities (20-25%)

---

## Scenario

IAM Lab Corp works with external contractors and partner organizations. The legal team needs contractors to access specific SharePoint sites. The partner company needs their employees to collaborate in Teams. You need to configure B2B collaboration, invite external users securely, set up cross-tenant access policies, and ensure external users have appropriate but limited access — with automatic cleanup when the engagement ends.

---

## SC-300 Exam Objectives Covered

- Manage External collaboration settings in Microsoft Entra ID
- Invite external users, individually and in bulk
- Manage external user accounts in Microsoft Entra ID
- Implement Cross-tenant access settings
- Implement and manage cross-tenant synchronization
- Configure external identity providers (SAML, WS-Fed, social providers)

---

## Prerequisites

- Completed Labs 1-2 (tenant configured with users and groups)
- Microsoft 365 E5 subscription
- At least one personal email address to use as a test guest (Gmail, Outlook, etc.)
- Ideally a second Entra ID tenant for cross-tenant testing (optional — can be simulated)

---

## Step 1: Configure External Collaboration Settings

**Where:** Entra ID → External Identities → External collaboration settings

These settings control the overall B2B collaboration policy for your tenant:

| Setting | Recommended Value | Why |
|---------|-------------------|-----|
| Guest user access | **Guest users have limited access to properties and memberships of directory objects** | Prevents guests from enumerating your full directory |
| Guest invite settings | **Member users and users assigned to specific admin roles can invite guest users** | Controls who can bring in external people — not everyone should have this power |
| Enable guest self-service sign-up via user flows | **No** | Keep control over who enters your tenant |
| Collaboration restrictions | **Allow invitations only to the specified domains** OR **Deny invitations to the specified domains** | Control which organizations can be invited |

For the collaboration restrictions, try both options:
1. First, set to **Allow invitations only to specified domains** and add `gmail.com` and `outlook.com`
2. Save and test by inviting a guest from an allowed domain
3. Then switch to **Deny invitations to specified domains** and add a test domain
4. Save and understand the difference

Click **Save** after each configuration.

**Why this matters:** External collaboration settings are your first line of defense for B2B. You need to control who can invite guests, what guests can see, and which organizations are allowed or blocked.

**SC-300 exam tip:** Know all the guest invite restriction levels. Know the difference between allow-list and deny-list approaches to collaboration restrictions. Know what guests can and cannot see at each access level.

### How to explain this in an interview:

*"I configured external collaboration settings to control B2B access. Guest invitations are restricted to specific admin roles — not all employees can invite external users. I implemented domain allow-listing so only approved partner organizations can be invited. Guest users have limited directory visibility to protect internal user information."*

---

## Step 2: Invite External Users Individually

**Where:** Entra ID → Users → + New user → Invite external user

### 2a: Invite via the Portal

1. Click **+ New user** → **Invite external user**
2. **Email:** Use a personal email address (Gmail, Outlook, etc.)
3. **Display name:** `External Contractor - [Name]`
4. **Personal message:** `Welcome to IAM Lab Corp. You've been granted access to collaborate on Project Alpha.`
5. Click **Invite**
6. Check your personal email — you should receive the invitation
7. Click **Accept invitation** in the email and walk through the redemption process

### 2b: Observe the Guest User Object

After the guest accepts:
1. Go to **Users** → **All users**
2. Find the guest user — notice the **User type** column shows **Guest** instead of **Member**
3. Click on the guest user and review their properties:
   - Source: External Microsoft Entra ID (or External email)
   - User type: Guest
   - Creation type: Invitation

### 2c: Invite via PowerShell

```powershell
# Invite a guest user via Graph API
$invitation = New-MgInvitation -InvitedUserEmailAddress "testguest@gmail.com" `
    -InviteRedirectUrl "https://myapplications.microsoft.com" `
    -InvitedUserDisplayName "Test Guest User" `
    -SendInvitationMessage:$true `
    -InvitedUserMessageInfo @{
        CustomizedMessageBody = "Welcome to IAM Lab Corp. Please accept this invitation to access project resources."
    }

Write-Host "Invitation sent. Redemption URL: $($invitation.InviteRedeemUrl)" -ForegroundColor Green
```

**SC-300 exam tip:** Know the difference between invitation and redemption. The invitation creates the guest object; redemption is when the guest actually accepts. Know that guests can redeem using Microsoft accounts, Google federation, email OTP, or direct federation.

---

## Step 3: Invite External Users in Bulk

**Where:** Entra ID → Users → Bulk operations → Bulk invite

1. Click **Bulk operations** → **Bulk invite**
2. Download the CSV template
3. Fill in 3-4 guest users:

| Email address to invite | Redirection url | Send invite message | Customized message |
|---|---|---|---|
| guest1@gmail.com | https://myapplications.microsoft.com | TRUE | Welcome to IAM Lab Corp |
| guest2@outlook.com | https://myapplications.microsoft.com | TRUE | Welcome to IAM Lab Corp |
| guest3@gmail.com | https://myapplications.microsoft.com | TRUE | Welcome to IAM Lab Corp |

4. Upload and submit
5. Review results

Also do it via PowerShell for comparison:

```powershell
$guestsToInvite = @(
    @{ Email = "bulkguest1@gmail.com"; Name = "Bulk Guest 1" }
    @{ Email = "bulkguest2@gmail.com"; Name = "Bulk Guest 2" }
)

foreach ($guest in $guestsToInvite) {
    try {
        New-MgInvitation -InvitedUserEmailAddress $guest.Email `
            -InviteRedirectUrl "https://myapplications.microsoft.com" `
            -InvitedUserDisplayName $guest.Name `
            -SendInvitationMessage:$true | Out-Null
        Write-Host "[INVITED] $($guest.Email)" -ForegroundColor Green
    } catch {
        Write-Host "[FAILED]  $($guest.Email): $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

---

## Step 4: Manage External User Accounts

### 4a: Review All Guest Users

```powershell
# List all guest users with their status
Get-MgUser -All -Filter "userType eq 'Guest'" | 
    Select-Object DisplayName, Mail, UserPrincipalName, CreatedDateTime,
        @{N='Source';E={$_.ExternalUserState}} |
    Format-Table -AutoSize
```

### 4b: Convert a Guest to a Member (and back)

Sometimes an external contractor becomes a full-time employee:

```powershell
# Convert guest to member
$guest = Get-MgUser -Filter "mail eq 'testguest@gmail.com'"
Update-MgUser -UserId $guest.Id -UserType "Member"
Write-Host "Converted to Member" -ForegroundColor Green

# Convert back to guest
Update-MgUser -UserId $guest.Id -UserType "Guest"
Write-Host "Converted back to Guest" -ForegroundColor Green
```

### 4c: Clean Up Stale Guest Accounts

```powershell
# Find guests who haven't signed in for 90+ days
$staleDate = (Get-Date).AddDays(-90)
Get-MgUser -All -Filter "userType eq 'Guest'" -Property DisplayName,Mail,SignInActivity |
    Where-Object { 
        $_.SignInActivity.LastSignInDateTime -lt $staleDate -or 
        $_.SignInActivity.LastSignInDateTime -eq $null 
    } |
    Select-Object DisplayName, Mail, 
        @{N='LastSignIn';E={$_.SignInActivity.LastSignInDateTime}} |
    Format-Table -AutoSize
```

**SC-300 exam tip:** Know how to manage guest user lifecycle — inviting, monitoring, converting between guest/member, and cleaning up stale accounts. Stale guest accounts are a security risk.

---

## Step 5: Configure Cross-Tenant Access Settings

**Where:** Entra ID → External Identities → Cross-tenant access settings

Cross-tenant access settings control how your tenant interacts with other Entra ID tenants. There are two directions:

- **Inbound:** Controls how users from OTHER tenants access YOUR resources
- **Outbound:** Controls how YOUR users access resources in OTHER tenants

### 5a: Configure Default Settings

1. Click **Default settings**
2. Review the inbound and outbound defaults:
   - **Inbound access:** B2B collaboration — customize to allow/block specific apps or users
   - **Outbound access:** B2B collaboration — customize to allow/block your users from accessing external resources
3. Under **Trust settings:** Enable **Trust multi-factor authentication from Microsoft Entra tenants** — this means if a guest already did MFA in their home tenant, your tenant trusts it instead of making them do MFA again

### 5b: Add an Organization-Specific Policy

1. Click **+ Add organization**
2. Enter a tenant ID or domain name (you can use Microsoft's tenant ID `72f988bf-86f1-41af-91ab-2d7cd011db47` as a test)
3. Configure **Inbound access:**
   - B2B collaboration → Applications → Select **Allow access** to only specific applications (e.g., SharePoint Online only)
4. Configure **Outbound access:**
   - B2B collaboration → Users → Select **Allow access** for all your users
5. Click **Save**

### 5c: Document the Cross-Tenant Architecture

Create a document that explains:
- What your default cross-tenant policy allows
- Which organizations have custom policies and why
- What applications external users can access
- Whether MFA trust is enabled and the implications

**Why this matters:** Cross-tenant access settings are how you control federated identity at the organizational level. In an acquisition scenario (Company A buys Company B), this is how you'd allow Company B's users to access Company A's resources before full identity migration.

**SC-300 exam tip:** Cross-tenant access is a newer topic but heavily tested. Know the difference between inbound and outbound settings, default vs. organization-specific policies, and MFA trust settings.

### How to explain this in an interview:

*"I configured cross-tenant access policies to control how our organization interacts with partner tenants. Inbound policies restrict which external users can access our resources and which applications they can reach. I enabled MFA trust so guests aren't forced to re-authenticate if their home tenant already verified their identity. Each partner organization can have custom policies based on the level of trust we have with them."*

---

## Step 6: Configure an External Identity Provider

**Where:** Entra ID → External Identities → All identity providers

This allows guest users to sign in using their existing accounts instead of creating a new Microsoft account.

### 6a: Configure Google as a Social Identity Provider

1. Go to **External Identities** → **All identity providers**
2. Click **+ Google**
3. You'll need a Google API Client ID and Client Secret:
   - Go to `console.developers.google.com`
   - Create a new project
   - Go to **Credentials** → **Create credentials** → **OAuth client ID**
   - Application type: Web application
   - Authorized redirect URIs: `https://login.microsoftonline.com/te/your-tenant-id/oauth2/authresp`
   - Copy the Client ID and Client Secret
4. Enter them in the Entra ID form
5. Click **Save**

If you don't want to set up Google integration, document the process and take screenshots of the configuration page. The exam tests whether you know WHERE to configure it and what's required.

### 6b: Configure Email One-Time Passcode (OTP)

1. On the identity providers page, verify **Email one-time passcode** is enabled
2. This allows guests without a Microsoft or Google account to authenticate using a code sent to their email

**SC-300 exam tip:** Know the identity provider priority order: Microsoft Entra account → Google federation → SAML/WS-Fed federation → Microsoft account → Email OTP. Know that Email OTP is the fallback for guests who don't have any other account type.

---

## Step 7: Generate an External Users Report

```powershell
# Comprehensive guest user report
$guestReport = Get-MgUser -All -Filter "userType eq 'Guest'" `
    -Property DisplayName,Mail,UserPrincipalName,CreatedDateTime,ExternalUserState,
        SignInActivity,CreationType | 
    Select-Object DisplayName, Mail, CreationType, ExternalUserState,
        @{N='Created';E={$_.CreatedDateTime.ToString('yyyy-MM-dd')}},
        @{N='LastSignIn';E={
            if($_.SignInActivity.LastSignInDateTime){
                $_.SignInActivity.LastSignInDateTime.ToString('yyyy-MM-dd')
            } else { 'Never' }
        }}

$guestReport | Format-Table -AutoSize

# Export
$guestReport | Export-Csv -Path "./reports/guest-users-report.csv" -NoTypeInformation
Write-Host "Guest user report exported to ./reports/guest-users-report.csv" -ForegroundColor Green
Write-Host "Total guests: $($guestReport.Count)" -ForegroundColor Cyan
```

---

## Step 8: Verify Everything

Checklist:

- [ ] External collaboration settings configured (invite restrictions, domain policies)
- [ ] Guest user invited via portal and redemption completed
- [ ] Guest user invited via PowerShell
- [ ] Bulk guest invitations completed
- [ ] Guest user properties reviewed (user type, source, creation type)
- [ ] Guest-to-member conversion tested
- [ ] Stale guest account query created
- [ ] Cross-tenant access default settings configured with MFA trust
- [ ] Organization-specific cross-tenant policy created
- [ ] External identity provider configured (Google or documented)
- [ ] Email OTP enabled
- [ ] Guest user report generated and exported

Take screenshots of external collaboration settings, cross-tenant policies, and identity provider configuration.

---

## Key Takeaways for SC-300

1. **External collaboration settings** control who can invite guests and from which domains
2. **Guest users** have a separate user type with limited default permissions
3. **Cross-tenant access** has inbound (others accessing your resources) and outbound (your users accessing others) policies
4. **MFA trust** prevents double MFA prompts for guests already authenticated in their home tenant
5. **Identity providers** define how guests authenticate — Microsoft, Google, SAML/WS-Fed, or Email OTP
6. **Guest lifecycle management** requires monitoring for stale accounts and automatic cleanup

---

## What's Next

➡️ **Lab 4:** [Hybrid Identity with Entra Connect](../lab04-hybrid-identity/) — On-premises AD synchronization, password hash sync, pass-through authentication, and Seamless SSO

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
