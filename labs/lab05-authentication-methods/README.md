# Lab 5: Authentication Methods, MFA & SSPR

> **Type:** Focused Lab | **Time:** 2-3 hours | **SC-300 Domain:** Implement authentication and access management (25-30%)

---

## Scenario

The security team at Expat Teacher's Lounge has flagged that employees are using weak passwords and there's no multi-factor authentication in place. You've been tasked with implementing a comprehensive authentication strategy: deploy multiple authentication methods, enforce MFA for all users, configure self-service password reset so employees stop calling the help desk, and set up password protection to block common and company-specific weak passwords.

---

## SC-300 Exam Objectives Covered

- Plan for authentication
- Implement and manage authentication methods (certificate-based, TAP, OAuth 2.0 tokens, Microsoft Authenticator, passkeys/FIDO2)
- Implement and manage tenant-wide MFA settings
- Configure and deploy self-service password reset (SSPR)
- Implement and manage Windows Hello for Business (conceptual)
- Disable accounts and revoke user sessions
- Implement and manage Microsoft Entra password protection

---

## Prerequisites

- Completed Labs 1-4
- Microsoft 365 E5 subscription
- A mobile phone for testing MFA and Authenticator app
- Microsoft Authenticator app installed on your phone

---

## Step 1: Create an Authentication Strategy Document

Before configuring anything, plan your strategy. Create a document (`docs/authentication-strategy.md`) that defines which authentication methods different user populations should use:

| User Population | Primary Auth | Secondary Auth (MFA) | SSPR Methods | Justification |
|---|---|---|---|---|
| Regular Employees | Password | Microsoft Authenticator push notification | SMS + Email OTP | Balance of security and usability |
| Administrators | Password | FIDO2 security key or Authenticator (number matching) | Authenticator + SMS | Phishing-resistant MFA for privileged accounts |
| External Contractors | Password | Email OTP or SMS | Email OTP | May not have company-managed devices |
| New Employees (Day 1) | Temporary Access Pass | Register Authenticator during onboarding | N/A — register after TAP | Passwordless first-day experience |
| Service Accounts | Certificate-based auth | N/A | N/A | No human interaction |

**Why this matters:** The exam expects you to PLAN authentication, not just configure it. Creating a strategy document shows you think about different user populations and their requirements — this is what senior IAM engineers do.

**SC-300 exam tip:** The exam may present a scenario and ask which authentication method is most appropriate for a given user population. Know the strengths and weaknesses of each method.

### How to explain this in an interview:

*"Before deploying any authentication changes, I created a strategy that maps authentication methods to user populations. Regular employees use Authenticator push notifications for MFA. Administrators are required to use phishing-resistant methods like FIDO2 keys. New employees get a Temporary Access Pass on day one so they can set up their authentication without needing a pre-existing method. This approach balances security with usability across the organization."*

---

## Step 2: Configure Authentication Methods Policy

**Where:** Entra ID → Protection → Authentication methods → Policies

This is where you enable or disable each authentication method for your tenant. Configure each method:

### 2a: Microsoft Authenticator

1. Click **Microsoft Authenticator**
2. **Enable:** Yes
3. **Target:** All users
4. Under **Configure:**
   - **Authentication mode:** Push (recommended), Passwordless, or Both
   - **Require number matching:** Enabled (prevents MFA fatigue attacks)
   - **Show application name:** Enabled (user sees which app is requesting access)
   - **Show geographic location:** Enabled (user sees where the sign-in is from)
5. Click **Save**

**SC-300 exam tip:** Number matching is a critical security feature. Without it, an attacker can spam MFA push notifications hoping the user accidentally approves. With number matching, the user must type the number shown on screen — they can't approve without seeing the sign-in screen.

### 2b: SMS

1. Click **SMS**
2. **Enable:** Yes
3. **Target:** All users
4. Click **Save**

### 2c: Email OTP

1. Click **Email OTP**
2. **Enable:** Yes
3. **Target:** All users
4. Click **Save**

### 2d: FIDO2 Security Keys

1. Click **Passkey (FIDO2)**
2. **Enable:** Yes
3. **Target:** Select a group — create a group called `SG-FIDO2-Enabled` and add your admin accounts to it (in production, only admins and high-security users would use FIDO2)
4. Under **Configure:**
   - **Allow self-service set up:** Yes
   - **Enforce attestation:** No (for lab; Yes in production)
5. Click **Save**

### 2e: Temporary Access Pass (TAP)

1. Click **Temporary Access Pass**
2. **Enable:** Yes
3. **Target:** All users
4. Under **Configure:**
   - **Minimum lifetime:** 60 minutes
   - **Maximum lifetime:** 480 minutes (8 hours)
   - **Default lifetime:** 60 minutes
   - **One-time use:** Yes (more secure — each TAP works once)
5. Click **Save**

**Why this matters:** TAP is how you onboard new employees who don't have any authentication method yet. On their first day, you generate a TAP, they use it to sign in and set up Authenticator. It's also used to recover accounts when someone loses their phone.

### 2f: Certificate-based Authentication

1. Click **Certificate-based authentication**
2. Review the configuration — this requires a trusted certificate authority configured
3. For this lab, document the settings page and understand the concept. Full setup requires PKI infrastructure.

**SC-300 exam tip:** Know all six methods, when to use each, and how to target them to specific groups. Know that TAP is for bootstrapping — it's temporary and enables registration of permanent methods.

---

## Step 3: Generate a Temporary Access Pass for a Test User

### 3a: Create the TAP via Portal

1. Go to **Users** → select a test user (e.g., Jessica Thompson)
2. Click **Authentication methods**
3. Click **+ Add authentication method**
4. Select **Temporary Access Pass**
5. Set the lifetime (e.g., 60 minutes)
6. Click **Add**
7. Copy the TAP code — this is the user's temporary "password"

### 3b: Test the TAP Sign-In Experience

1. Open an **incognito browser window**
2. Go to `login.microsoftonline.com`
3. Sign in as `jessica.thompson@expatteacherslounge.com` (or your domain)
4. When prompted for password, enter the TAP code
5. You'll be redirected to set up a permanent authentication method (Authenticator)
6. Follow the prompts to register Microsoft Authenticator on your phone

### 3c: Create a TAP via PowerShell

```powershell
# Generate a TAP for a user
$userId = (Get-MgUser -Filter "displayName eq 'Jessica Thompson'").Id

$tap = New-MgUserAuthenticationTemporaryAccessPassMethod -UserId $userId -BodyParameter @{
    LifetimeInMinutes = 60
    IsUsableOnce = $true
}

Write-Host "TAP for Jessica Thompson: $($tap.TemporaryAccessPass)" -ForegroundColor Green
Write-Host "Valid for: $($tap.LifetimeInMinutes) minutes" -ForegroundColor Cyan
Write-Host "One-time use: $($tap.IsUsableOnce)" -ForegroundColor Cyan
```

**SC-300 exam tip:** TAP is a frequently tested topic. Know how to create one, what the lifetime options are, and the difference between one-time use and multi-use TAPs.

---

## Step 4: Configure Tenant-Wide MFA Settings

There are multiple ways to enforce MFA. Understand all of them:

### 4a: Security Defaults (Simplest — Not Recommended for E5)

**Where:** Entra ID → Overview → Properties → **Manage Security Defaults**

Security Defaults is a free, one-click MFA enforcement. It requires MFA for all admin roles and prompts all users to register for MFA.

1. Review the Security Defaults toggle — for an E5 tenant, you should **disable** Security Defaults because you'll use Conditional Access instead (they conflict)
2. Set to **Disabled** if it's on
3. Click **Save**

**SC-300 exam tip:** Security Defaults and Conditional Access cannot be active at the same time. Know that Security Defaults is for organizations without P1/P2 licenses. With P1/P2, use Conditional Access for granular MFA control.

### 4b: Per-User MFA (Legacy — Know It for the Exam)

**Where:** Entra ID → Users → Per-user MFA (at the top of the user list)

1. Click **Per-user MFA**
2. This shows all users with their MFA status: Disabled, Enabled, or Enforced
3. **Do not enable anything here** — this is the legacy approach
4. Just review the page and understand it exists

**SC-300 exam tip:** Per-user MFA is legacy but still on the exam. Know the three states (Disabled, Enabled, Enforced) and know that Conditional Access is the modern, recommended approach.

### 4c: Conditional Access MFA (Recommended — Lab 6)

We'll configure MFA through Conditional Access in Lab 6. This is the modern, production approach because it lets you define conditions (location, device, risk level) that trigger MFA instead of requiring it for every single sign-in.

---

## Step 5: Configure Self-Service Password Reset (SSPR)

**Where:** Entra ID → Protection → Password reset

### 5a: Properties

1. **Self service password reset enabled:** Set to **All**
2. Click **Save**

### 5b: Authentication Methods

1. Click **Authentication methods**
2. **Number of methods required to reset:** **2** (production standard — requires two verification methods)
3. **Methods available to users:** Check:
   - **Mobile app notification** (Authenticator push)
   - **Mobile app code** (Authenticator TOTP code)
   - **Email**
   - **Mobile phone** (SMS)
4. Click **Save**

### 5c: Registration

1. Click **Registration**
2. **Require users to register when signing in:** **Yes**
3. **Number of days before users are asked to re-confirm:** **90**
4. Click **Save**

### 5d: Notifications

1. Click **Notifications**
2. **Notify users on password resets:** **Yes**
3. **Notify all admins when other admins reset their password:** **Yes**
4. Click **Save**

### 5e: Test SSPR End-to-End

1. Open an **incognito browser window**
2. Go to `aka.ms/sspr`
3. Enter a test user's UPN
4. Complete the CAPTCHA
5. Choose a verification method (email or phone)
6. Complete the verification
7. Set a new password
8. Verify: go back to the Entra portal → **Audit logs** → filter by activity "Reset password (self-service)" — you should see the event

**SC-300 exam tip:** Know the SSPR configuration options in detail: which methods are available, the registration enforcement settings, and the notification options. Know that admins always require 2 methods regardless of the tenant setting. Know combined registration (MFA + SSPR registration in a single flow).

### How to explain this in an interview:

*"I configured SSPR to require two verification methods — Authenticator app and either SMS or email. Users are forced to register their methods on their next sign-in. Admins get notified when any admin resets their password, which is an important security alert. This reduced our simulated help desk password reset tickets by allowing users to self-serve."*

---

## Step 6: Implement Microsoft Entra Password Protection

**Where:** Entra ID → Protection → Authentication methods → Password protection

### 6a: Configure Custom Banned Password List

1. Click **Password protection**
2. **Custom banned password list:** Set to **Yes**
3. In the **Custom banned password list** field, add company-specific words that shouldn't be in passwords:
   ```
   expatteacher
   teacherslounge
   expat2026
   iamlabcorp
   password123
   ```
4. **Enable password protection on Windows Server Active Directory:** Set to **Yes** if you have hybrid identity (Lab 4)
5. **Mode:** Set to **Enforced** (in production, start with **Audit** to see what would be blocked)
6. Click **Save**

### 6b: Test Password Protection

1. Try to set a test user's password to something containing a banned word (e.g., `ExpatTeacher2026!`)
2. The reset should be rejected with a message about not meeting password requirements
3. Try again with a strong password that doesn't contain banned words — it should succeed

**Why this matters:** Password protection prevents users from choosing passwords that are easy to guess based on company information. Attackers commonly try company name + year + special character as their first brute force attempts.

**SC-300 exam tip:** Know the difference between the global banned password list (Microsoft-maintained, always active) and the custom banned password list (you maintain). Know that password protection can be extended to on-prem AD via the Password Protection proxy agent.

---

## Step 7: Disable an Account and Revoke Sessions

Practice the security actions you'd take when an account is compromised:

### 7a: Disable via Portal

1. Go to **Users** → select a test user
2. Click **Edit properties** → set **Account enabled** to **No** → **Save**
3. Note: the user can still have active sessions until tokens expire

### 7b: Revoke Sessions via Portal

1. On the same user page, click **Revoke sessions**
2. Confirm the revocation
3. This invalidates all refresh tokens — the user will be signed out of everything

### 7c: Disable and Revoke via PowerShell

```powershell
$user = Get-MgUser -Filter "displayName eq 'Test User'"

# Disable the account
Update-MgUser -UserId $user.Id -AccountEnabled:$false
Write-Host "[DISABLED] Account disabled" -ForegroundColor Green

# Revoke all sessions
Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/users/$($user.Id)/revokeSignInSessions"
Write-Host "[REVOKED] All sessions revoked" -ForegroundColor Green

# Re-enable when done testing
Update-MgUser -UserId $user.Id -AccountEnabled:$true
Write-Host "[ENABLED] Account re-enabled" -ForegroundColor Yellow
```

**SC-300 exam tip:** Know the difference between disabling an account (prevents new sign-ins) and revoking sessions (kills existing tokens). Know that both should be done during incident response, in the correct order (disable first, then revoke).

---

## Step 8: Configure Combined Registration

**Where:** Entra ID → User settings → User feature settings (or User feature previews)

1. Look for **Combined security info registration**
2. Ensure it's set to **All** or **Enabled**
3. This means users register for both MFA and SSPR in a single flow at `aka.ms/mysecurityinfo` instead of two separate registration processes

Test it:
1. Open an **incognito window**
2. Go to `aka.ms/mysecurityinfo`
3. Sign in as a test user
4. Walk through the combined registration — add a phone number, configure Authenticator, add an email
5. Note how one registration covers both MFA and SSPR

**SC-300 exam tip:** Combined registration is the modern experience and is now the default. Know that it uses `aka.ms/mysecurityinfo` and that it replaced the separate MFA and SSPR registration portals.

---

## Step 9: Verify Everything

Checklist:

- [ ] Authentication strategy document created
- [ ] Microsoft Authenticator enabled with number matching
- [ ] SMS enabled for all users
- [ ] Email OTP enabled for all users
- [ ] FIDO2 enabled for admin group
- [ ] Temporary Access Pass enabled and configured
- [ ] TAP generated and tested for a user
- [ ] Security Defaults disabled (using CA instead)
- [ ] Per-user MFA page reviewed (legacy — not enabled)
- [ ] SSPR enabled for all users with 2 methods required
- [ ] SSPR registration enforced on sign-in
- [ ] SSPR notifications enabled for users and admins
- [ ] SSPR tested end-to-end with audit log verification
- [ ] Custom banned password list configured with company-specific words
- [ ] Password protection tested
- [ ] Account disable and session revocation tested
- [ ] Combined registration configured and tested

Take screenshots of authentication methods policy, SSPR settings, password protection, and the combined registration experience.

---

## Key Takeaways for SC-300

1. **Authentication methods policy** is the central control plane for all authentication methods
2. **Number matching** on Authenticator prevents MFA fatigue attacks — always enable it
3. **TAP** is for bootstrapping new users and account recovery — one-time use is more secure
4. **Security Defaults vs. Conditional Access** — they can't coexist; use CA for E5/P1+ tenants
5. **Per-user MFA** is legacy — know it for the exam but use CA in practice
6. **SSPR** requires methods, registration enforcement, and notifications to be fully configured
7. **Custom banned passwords** prevent company-specific weak passwords
8. **Combined registration** unifies MFA and SSPR registration into one experience
9. **Disable then revoke** is the correct incident response order

---

## What's Next

➡️ **Lab 6:** [Conditional Access & Identity Protection](../lab06-conditional-access/) — Build a Zero Trust Conditional Access framework, configure risk-based policies, and set up Identity Protection

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
