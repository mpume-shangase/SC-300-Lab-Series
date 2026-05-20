# Lab 6: Conditional Access & Identity Protection

> **Type:** Big Project | **Time:** 3-4 hours | **SC-300 Domain:** Implement authentication and access management (25-30%)

---

## Scenario

The CISO at Expat Teacher's Lounge has mandated Zero Trust. The board wants proof that access is controlled based on who the user is, where they're signing in from, what device they're using, and how risky the sign-in looks. You need to design and implement a Conditional Access framework and configure Identity Protection to automatically detect and respond to compromised accounts.

This lab covers the largest and most heavily tested SC-300 domain (25-30%).

---

## SC-300 Exam Objectives Covered

- Plan Conditional Access policies
- Implement CA policy assignments (users, groups, apps, conditions)
- Implement CA policy controls (grant, session)
- Test and troubleshoot CA policies
- Implement session management
- Implement device-enforced restrictions
- Implement continuous access evaluation
- Configure authentication context
- Implement protected actions
- Create CA policy from a template
- Implement and manage user risk policies (Identity Protection / CA)
- Implement and manage sign-in risk policies (Identity Protection / CA)
- Implement and manage MFA registration policy
- Monitor, investigate, and remediate risky users and sign-ins
- Monitor, investigate, and remediate risky workload identities

---

## Prerequisites

- Completed Labs 1-5 (especially Lab 5 — authentication methods configured)
- Microsoft 365 E5 subscription (includes Entra ID P2 for Identity Protection)
- Security Defaults **disabled** (done in Lab 5)

---

## Step 1: Design Your Conditional Access Policy Matrix

Before creating any policies, plan them on paper. Create `docs/conditional-access-matrix.md`:

| Policy Name | Users | Apps | Conditions | Grant Controls | Session Controls | Priority |
|---|---|---|---|---|---|---|
| CA001: Require MFA for All Users | All users (exclude break-glass) | All cloud apps | None | Require MFA | None | Baseline |
| CA002: Block Legacy Authentication | All users | All cloud apps | Client apps: Other clients, Exchange ActiveSync | Block access | None | High |
| CA003: Require Compliant Device for Sensitive Apps | All users (exclude break-glass) | SharePoint, Exchange Online | None | Require compliant device OR Hybrid Entra joined | None | Medium |
| CA004: Admin MFA with Phishing-Resistant Methods | Directory roles (all admin roles) | All cloud apps | None | Require authentication strength: Phishing-resistant MFA | Sign-in frequency: 4 hours | Critical |
| CA005: Block Access from Untrusted Countries | All users | All cloud apps | Locations: All locations EXCEPT named trusted locations | Block access | None | High |
| CA006: Require MFA for Risky Sign-Ins | All users | All cloud apps | Sign-in risk: Medium and above | Require MFA | None | High |
| CA007: Force Password Change for High-Risk Users | All users | All cloud apps | User risk: High | Require password change + MFA | None | Critical |

**Why this matters:** Designing policies before building them prevents conflicts, gaps, and lockouts. In production, CA policy changes go through a change advisory board — you wouldn't just click and deploy.

**SC-300 exam tip:** The exam frequently asks you to design a CA policy for a specific scenario. Practice mapping scenarios to assignments + conditions + controls.

---

## Step 2: Create a Break-Glass Emergency Access Account

Before any CA policies, create an account that bypasses everything:

1. Go to **Users** → **+ New user** → **Create new user**
2. **Username:** `breakglass@expatteacherslounge.com`
3. **Display name:** `Break Glass - Emergency Access`
4. Set a **very strong password** (20+ characters, random)
5. **Do NOT assign any MFA methods** to this account
6. Click **Create**

Create a second one:
1. **Username:** `breakglass2@expatteacherslounge.com`
2. Same configuration

Assign Global Administrator to both:
1. Go to **Roles & admins** → **Global Administrator** → **+ Add assignments**
2. Add both break-glass accounts

Document the break-glass procedure:
```markdown
## Break-Glass Account Procedure
- Accounts: breakglass@expatteacherslounge.com, breakglass2@expatteacherslounge.com
- Passwords: Stored in a physical safe (Location: [office location])
- Password envelopes are sealed and signed — any opening is logged
- These accounts are EXCLUDED from all Conditional Access policies
- Any sign-in to these accounts triggers an immediate alert to the security team
- Monthly test: verify accounts can sign in (use breakglass2, rotate monthly)
```

Create a group for excluding break-glass accounts from CA:
1. Go to **Groups** → **+ New group**
2. **Name:** `SG-BreakGlass-Accounts`
3. **Type:** Security, Assigned
4. Add both break-glass accounts as members
5. Click **Create**

**Why this matters:** If your CA policies lock out all admins (misconfiguration, service outage, MFA failure), break-glass accounts are your only way back in. Every enterprise has them.

**SC-300 exam tip:** Break-glass accounts are a specific exam topic. Know that they should be cloud-only (not synced from on-prem), excluded from ALL CA policies, have no MFA, and be monitored with alerts.

---

## Step 3: Create Named Locations

**Where:** Entra ID → Protection → Conditional Access → Named locations

### 3a: Trusted Office Location (IP-based)

1. Click **+ IP ranges location**
2. **Name:** `Trusted Office - Doha`
3. Check **Mark as trusted location**
4. Add your office IP range (or your home IP — check whatismyip.com)
5. Click **Create**

### 3b: Blocked Countries

1. Click **+ Countries location**
2. **Name:** `Blocked Countries`
3. Select countries you want to block access from (pick 5-10 for testing)
4. Click **Create**

### 3c: Create Locations via PowerShell

```powershell
# Create a named location via Graph API
$params = @{
    "@odata.type" = "#microsoft.graph.ipNamedLocation"
    DisplayName = "Corporate VPN"
    IsTrusted = $true
    IpRanges = @(
        @{
            "@odata.type" = "#microsoft.graph.iPv4CidrRange"
            CidrAddress = "203.0.113.0/24"
        }
    )
}
New-MgIdentityConditionalAccessNamedLocation -BodyParameter $params
Write-Host "[CREATED] Named location: Corporate VPN" -ForegroundColor Green
```

**SC-300 exam tip:** Know the difference between IP-based and country-based named locations. Know that "trusted" locations can be used as a condition in CA policies (e.g., "require MFA only when NOT on a trusted location").

---

## Step 4: Build Conditional Access Policies

**Where:** Entra ID → Protection → Conditional Access → Policies

### CA001: Require MFA for All Users

1. Click **+ New policy**
2. **Name:** `CA001: Require MFA for All Users`
3. **Assignments:**
   - Users: **All users**
   - Exclude: Group `SG-BreakGlass-Accounts`
4. **Target resources:** All cloud apps
5. **Conditions:** None (this is the baseline)
6. **Grant:** **Require authentication strength** → select **Multifactor authentication**
7. **Enable policy:** Set to **Report-only** first (never deploy to "On" without testing)
8. Click **Create**

### CA002: Block Legacy Authentication

1. Click **+ New policy**
2. **Name:** `CA002: Block Legacy Authentication`
3. **Assignments:**
   - Users: **All users**
   - Exclude: `SG-BreakGlass-Accounts`
4. **Target resources:** All cloud apps
5. **Conditions:**
   - **Client apps:** Check **Exchange ActiveSync clients** and **Other clients** — uncheck everything else
6. **Grant:** **Block access**
7. **Enable policy:** **Report-only**
8. Click **Create**

**Why block legacy auth?** Legacy protocols (POP, IMAP, SMTP, older ActiveSync) don't support MFA. If you require MFA everywhere but allow legacy auth, attackers just use a legacy protocol to bypass your MFA policy.

### CA003: Require Compliant Device for Sensitive Apps

1. Click **+ New policy**
2. **Name:** `CA003: Require Compliant Device for Sensitive Apps`
3. **Assignments:**
   - Users: **All users**
   - Exclude: `SG-BreakGlass-Accounts`
4. **Target resources:** **Select apps** → select **Office 365 SharePoint Online** and **Office 365 Exchange Online**
5. **Conditions:** None
6. **Grant:** **Require device to be marked as compliant** OR **Require Hybrid Microsoft Entra joined device** (select "Require one of the selected controls")
7. **Enable policy:** **Report-only**
8. Click **Create**

### CA004: Admin Phishing-Resistant MFA

1. Click **+ New policy**
2. **Name:** `CA004: Admin Phishing-Resistant MFA`
3. **Assignments:**
   - Users: **Select users and groups** → **Directory roles** → select all admin roles (Global Admin, User Admin, Security Admin, etc.)
   - Exclude: `SG-BreakGlass-Accounts`
4. **Target resources:** All cloud apps
5. **Conditions:** None
6. **Grant:** **Require authentication strength** → select **Phishing-resistant MFA**
7. **Session:** Set **Sign-in frequency** to **4 hours**
8. **Enable policy:** **Report-only**
9. Click **Create**

### CA005: Block Untrusted Countries

1. Click **+ New policy**
2. **Name:** `CA005: Block Access from Untrusted Countries`
3. **Assignments:**
   - Users: **All users**
   - Exclude: `SG-BreakGlass-Accounts`
4. **Target resources:** All cloud apps
5. **Conditions:**
   - **Locations:** Include **Any location**, Exclude **Trusted Office - Doha** and any other trusted locations
6. **Grant:** **Block access**
7. **Enable policy:** **Report-only** (be very careful with this one — test thoroughly before enabling)
8. Click **Create**

### CA-Template: Create from a Template

1. Click **+ New policy from template**
2. Browse the available templates — select **Require multifactor authentication for admins**
3. Review the pre-configured settings
4. Customize the name: `CA-Template: MFA for Admins (from template)`
5. Add the `SG-BreakGlass-Accounts` exclusion
6. Set to **Report-only**
7. Click **Create**

**SC-300 exam tip:** Know that CA policy templates exist and what scenarios they cover. The exam may ask about using templates as a starting point.

---

## Step 5: Test Policies with What-If

**Where:** Conditional Access → Policies → **What if**

The What-If tool simulates a sign-in and shows which CA policies would apply:

### Test 1: Regular User from Trusted Location

1. Click **What if**
2. **User:** Select a regular user (e.g., Jessica Thompson)
3. **Cloud apps:** All cloud apps
4. **IP address:** Enter your trusted office IP
5. Click **What if**
6. Review: Which policies applied? Which were not applicable? Was access granted or blocked?

### Test 2: Admin from Unknown Location

1. **User:** Select your admin account
2. **Cloud apps:** All cloud apps
3. **IP address:** Enter a random foreign IP (e.g., `185.220.101.1`)
4. Click **What if**
5. Review: The admin should be blocked by CA005 (untrusted country)

### Test 3: Regular User Accessing SharePoint from Personal Device

1. **User:** Select a regular user
2. **Cloud apps:** Office 365 SharePoint Online
3. **Device platform:** Windows
4. **Device state:** Non-compliant
5. Click **What if**
6. Review: CA003 should require a compliant device

Document all What-If test results in `docs/conditional-access-test-results.md`.

**SC-300 exam tip:** What-If is a critical troubleshooting tool. The exam frequently asks how to determine which CA policies apply to a specific sign-in scenario. Know how to use What-If and interpret the results.

---

## Step 6: Configure Authentication Context

**Where:** Entra ID → Protection → Conditional Access → Authentication context

Authentication context lets you trigger additional security requirements for specific sensitive actions within an application:

1. Click **+ New authentication context**
2. **Name:** `Sensitive Data Access`
3. **Description:** `Required when accessing classified or restricted information`
4. **ID:** `c1` (auto-assigned)
5. Click **Save**

Create a CA policy that uses this authentication context:
1. Go to **Policies** → **+ New policy**
2. **Name:** `CA008: Require Phishing-Resistant MFA for Sensitive Data`
3. **Target resources:** Click **Authentication context** → select `Sensitive Data Access`
4. **Grant:** **Require authentication strength** → **Phishing-resistant MFA**
5. **Enable policy:** **Report-only**
6. Click **Create**

**SC-300 exam tip:** Authentication context is a newer exam topic. Know that applications can declare an authentication context, which then triggers specific CA policies. This allows step-up authentication — regular access needs standard MFA, but sensitive actions need stronger authentication.

---

## Step 7: Configure Session Management

Session controls within CA policies determine how long a session lasts:

Edit CA001 (Require MFA for All Users):
1. Open the policy → **Session**
2. **Sign-in frequency:** Enable → set to **24 hours** (users re-authenticate daily)
3. **Persistent browser session:** **Never persistent** (users must sign in each browser session)
4. Click **Save**

**Continuous Access Evaluation (CAE):**
1. Go to **Conditional Access** → **Session** (in left menu)
2. Review **Continuous Access Evaluation** settings
3. CAE enables near-real-time enforcement of policy changes — when you disable a user or change a CA policy, it takes effect within minutes instead of waiting for token expiry

**SC-300 exam tip:** Know sign-in frequency, persistent browser sessions, and CAE. Know that CAE is enabled by default for supported apps (Exchange, SharePoint, Teams) and that it overrides the default 1-hour token lifetime.

---

## Step 8: Configure Protected Actions

**Where:** Entra ID → Protection → Conditional Access → Authentication context (then linked via CA policy)

Protected actions require additional authentication for specific admin operations — like deleting a CA policy or modifying PIM settings:

1. Go to **Roles & admins** → **Protected actions** (if available in your tenant)
2. Or create a CA policy targeting specific admin actions through authentication context
3. Document the concept even if the feature isn't fully available in your tenant

**SC-300 exam tip:** Protected actions are a newer feature. Know that they add step-up authentication for destructive admin operations (deleting CA policies, modifying PIM settings, removing security alerts).

---

## Step 9: Configure Identity Protection

**Where:** Entra ID → Protection → Identity Protection

### 9a: Configure User Risk Policy

1. Go to **Identity Protection** → **User risk policy** (or create via CA policy)
2. Create a new CA policy:
   - **Name:** `CA006: Force Password Change for High-Risk Users`
   - **Users:** All users, exclude `SG-BreakGlass-Accounts`
   - **Conditions:** **User risk** → select **High**
   - **Grant:** **Require password change** AND **Require multifactor authentication**
   - **Enable:** **Report-only**
   - Click **Create**

### 9b: Configure Sign-In Risk Policy

1. Create a new CA policy:
   - **Name:** `CA007: Require MFA for Risky Sign-Ins`
   - **Users:** All users, exclude `SG-BreakGlass-Accounts`
   - **Conditions:** **Sign-in risk** → select **Medium and above**
   - **Grant:** **Require multifactor authentication**
   - **Enable:** **Report-only**
   - Click **Create**

### 9c: Configure MFA Registration Policy

1. Go to **Identity Protection** → **MFA registration policy**
2. **Users:** All users (exclude `SG-BreakGlass-Accounts`)
3. **Enforce:** **On**
4. Click **Save**

### 9d: Review the Identity Protection Dashboard

1. Go to **Identity Protection** → **Overview**
2. Review:
   - **Risky users** — users whose accounts may be compromised
   - **Risky sign-ins** — sign-ins that look suspicious
   - **Risk detections** — specific events that triggered risk flags
3. Click into **Risky users** and review the details if any exist

### 9e: Simulate a Risky Sign-In (Optional)

If you want to trigger a risk detection:
1. Use a VPN to connect from a different country
2. Sign in as a test user
3. Then immediately sign in from your normal location
4. This should trigger an **impossible travel** detection
5. Go back to Identity Protection → **Risk detections** to see the alert

**SC-300 exam tip:** Identity Protection is heavily tested. Know the three risk-based policies (user risk, sign-in risk, MFA registration). Know the risk levels (low, medium, high). Know how to investigate and remediate risky users (dismiss risk, confirm compromise, reset password).

### How to explain this in an interview:

*"I configured Identity Protection with risk-based Conditional Access policies. High-risk users are automatically forced to change their password and verify with MFA. Medium-risk sign-ins require MFA before access is granted. This provides automated threat response — we don't need a human to manually investigate every suspicious sign-in. The system responds in real time."*

---

## Step 10: Move Policies from Report-Only to On

Once you've tested all policies with What-If and verified they work as expected:

1. Open each policy
2. Change from **Report-only** to **On**
3. Click **Save**
4. Start with the least disruptive policies first (CA002: Block Legacy Auth)
5. Monitor the **Sign-in logs** for any unexpected blocks
6. Enable the remaining policies one at a time, monitoring after each

**Production best practice:** Enable policies during business hours when the security team is available to respond to issues. Never enable all policies at once.

---

## Step 11: Generate a CA Policy Report

```powershell
# List all Conditional Access policies with their state
Get-MgIdentityConditionalAccessPolicy -All | 
    Select-Object DisplayName, State, CreatedDateTime,
        @{N='GrantControls';E={$_.GrantControls.BuiltInControls -join ', '}},
        @{N='SessionControls';E={
            if($_.SessionControls.SignInFrequency){
                "Sign-in: $($_.SessionControls.SignInFrequency.Value) $($_.SessionControls.SignInFrequency.Type)"
            } else { 'None' }
        }} |
    Format-Table -AutoSize

# Export to CSV
Get-MgIdentityConditionalAccessPolicy -All | 
    Select-Object DisplayName, State, CreatedDateTime | 
    Export-Csv -Path "./reports/ca-policies-report.csv" -NoTypeInformation
Write-Host "CA policy report exported" -ForegroundColor Green
```

---

## Step 12: Verify Everything

Checklist:

- [ ] CA policy matrix designed and documented
- [ ] Break-glass accounts created (2) with Global Admin, excluded from all CA
- [ ] Break-glass procedure documented
- [ ] Named locations configured (trusted office, blocked countries)
- [ ] CA001: MFA for all users (report-only, then on)
- [ ] CA002: Block legacy authentication (report-only, then on)
- [ ] CA003: Compliant device for sensitive apps (report-only)
- [ ] CA004: Phishing-resistant MFA for admins (report-only)
- [ ] CA005: Block untrusted countries (report-only)
- [ ] CA from template created
- [ ] What-If testing completed for 3+ scenarios and documented
- [ ] Authentication context created and linked to CA policy
- [ ] Session management configured (sign-in frequency, CAE)
- [ ] Protected actions concept documented
- [ ] Identity Protection: User risk policy configured
- [ ] Identity Protection: Sign-in risk policy configured
- [ ] Identity Protection: MFA registration policy enabled
- [ ] Identity Protection dashboard reviewed
- [ ] CA policies report generated with PowerShell

---

## Key Takeaways for SC-300

1. **Design before you deploy** — create a policy matrix mapping scenarios to controls
2. **Break-glass accounts** are mandatory — cloud-only, no MFA, excluded from all CA, monitored
3. **Report-only mode** is your safety net — always test before enabling
4. **What-If** is the primary troubleshooting tool for CA policies
5. **Block legacy auth** is one of the highest-impact security policies you can deploy
6. **Authentication strength** replaces the old "require MFA" grant — it lets you specify which MFA methods are acceptable
7. **Authentication context** enables step-up authentication for sensitive actions
8. **Identity Protection** automates threat response with risk-based CA policies
9. **CAE** enables near-real-time policy enforcement instead of waiting for token expiry
10. **Never deploy all policies at once** — enable one at a time, monitor, then proceed

---

## What's Next

➡️ **Lab 7:** [Workload Identities & Enterprise Applications](../lab07-workload-identities/) — Managed identities, service principals, enterprise app integration, SSO, and consent management

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
