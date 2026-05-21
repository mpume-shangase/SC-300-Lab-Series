# Lab 9: Identity Governance — Entitlements, Access Reviews & PIM

> **Type:** Big Project | **Time:** 4-5 hours | **SC-300 Domain:** Plan and automate identity governance (20-25%)

---

## Scenario

Audit season is approaching at Expat Teacher's Lounge. The compliance team needs proof that every employee's access has been reviewed by their manager within 90 days, that privileged roles are time-limited and justified, and that access requests follow a controlled approval workflow. A previous audit found that 35% of employees had access they no longer needed, and three former contractors still had active accounts. You need to build the complete identity governance framework to prevent these findings from recurring.

This is the most comprehensive governance lab and covers the entire Domain 4 of SC-300.

---

## SC-300 Exam Objectives Covered

- Plan entitlements
- Create and configure catalogs
- Create and configure access packages
- Manage access requests
- Implement and manage terms of use (ToU)
- Manage the lifecycle of external users
- Configure and manage connected organizations
- Plan for access reviews
- Create and configure access reviews
- Monitor access review activity
- Manually respond to access review activity
- Plan and manage Microsoft Entra roles in PIM (settings and assignments)
- Plan and manage Azure resources in PIM (settings and assignments)
- Plan and configure PIM for Groups
- Manage the PIM request and approval process
- Analyze PIM audit history and reports
- Create and manage break-glass accounts

---

## Prerequisites

- Completed Labs 1-8
- Microsoft 365 E5 subscription (includes Entra ID P2 — required for PIM and access reviews)
- Break-glass accounts already created (from Lab 6)
- Multiple test users across different departments

---

## Step 1: Understand Identity Governance Components

Create a reference document (`docs/identity-governance-overview.md`):

| Component | What It Does | Business Problem It Solves |
|---|---|---|
| **Entitlement Management** | Self-service access request portal with approval workflows | "I need access to X" — controlled request instead of ad-hoc |
| **Access Reviews** | Periodic review of who has access to what | "Does this person still need this access?" — prevents stale access |
| **PIM (Privileged Identity Management)** | Time-limited, approval-based activation of admin roles | "Why does everyone have permanent Global Admin?" — reduces standing privilege |
| **Terms of Use** | Legal agreements users must accept before accessing resources | "Can we prove users agreed to our policies?" — compliance documentation |
| **Lifecycle Workflows** | Automated actions triggered by identity lifecycle events | "What happens automatically when someone joins/leaves?" — reduces manual work |

**SC-300 exam tip:** Know all five components and when to use each. The exam presents scenarios and asks which governance feature addresses the requirement.

### How to explain this in an interview:

*"I built a comprehensive identity governance framework with three pillars. Entitlement management handles access requests with approval workflows — employees request access packages instead of emailing IT. Access reviews ensure every group membership and app assignment is reviewed quarterly by the responsible manager. And PIM eliminates permanent admin roles — admins activate their roles just-in-time with justification and time limits. Together, these prevent access creep, reduce standing privilege, and produce the compliance evidence auditors need."*

---

## Step 2: Create an Access Catalog

**Where:** Entra ID → Identity Governance → Entitlement management → Catalogs

A catalog is a container that groups resources (apps, groups, SharePoint sites) that can be bundled into access packages.

### 2a: Create the Catalog

1. Click **+ New catalog**
2. **Name:** `Expat Teacher's Lounge - General Access`
3. **Description:** `Contains access packages for standard employee access across all departments`
4. **Enabled:** Yes
5. **Enabled for external users:** Yes (contractors may need access too)
6. Click **Create**

### 2b: Add Resources to the Catalog

1. Click into your catalog → **Resources** → **+ Add resources**
2. Add **Groups:**
   - SG-Engineering
   - SG-Sales
   - SG-HR
   - SG-Finance
   - SG-Marketing
   - SG-IT
3. Add **Applications** (enterprise apps from Lab 7 if configured)
4. Click **Add**

### 2c: Create a Second Catalog for Sensitive Resources

1. Create another catalog: `Sensitive Resources`
2. **Enabled for external users:** No (only internal employees)
3. Add resources that require higher approval levels

**SC-300 exam tip:** Know that catalogs organize resources for access packaging. Know that catalog owners can manage the resources and access packages within their catalog without being Global Admins — this enables delegated governance.

---

## Step 3: Create Access Packages

**Where:** Entitlement management → Access packages

### 3a: Engineering Standard Access Package

1. Click **+ New access package**
2. **Name:** `Engineering Standard Access`
3. **Description:** `Standard access bundle for Engineering department employees. Includes Engineering security group and development tools.`
4. **Catalog:** `Expat Teacher's Lounge - General Access`
5. Click **Next**

**Resource roles:**
1. Add **SG-Engineering** → Role: Member
2. Add any Engineering-specific enterprise apps if configured
3. Click **Next**

**Requests:**
1. **Who can request:** For users in your directory
2. **Require approval:** Yes
3. **How many stages:** 1
4. **First approver:** Select the Engineering manager (Alex Chen)
5. **Require requestor justification:** Yes
6. **How many days to make a decision:** 7
7. **If no action is taken:** Request is automatically denied
8. Click **Next**

**Requestor information:**
1. Add a question: `What project will you be working on?` (Required, text answer)
2. Click **Next**

**Lifecycle:**
1. **Access package assignments expire:** Yes
2. **Assignments expire after:** 365 days
3. **Allow users to extend access:** Yes
4. **Require approval to extend:** Yes
5. Click **Next**

**Review + Create** → Click **Create**

### 3b: Create Additional Access Packages

Repeat for each department:

| Package Name | Resources | Approver | Expiry |
|---|---|---|---|
| Sales Standard Access | SG-Sales + sales apps | Maria Santos | 365 days |
| Finance Standard Access | SG-Finance + finance apps | Priya Patel | 365 days |
| HR Standard Access | SG-HR + HR apps | James Wilson | 365 days |
| Contractor Limited Access | SG-Contractors (create if needed) | Requesting user's manager | 90 days |

### 3c: Create a Multi-Stage Approval Package

For sensitive access, create a package with two approval stages:

1. **Name:** `Sensitive Data Access`
2. **Catalog:** `Sensitive Resources`
3. **Requests:**
   - Stage 1: Manager approval
   - Stage 2: Security team approval (create a group `SG-Security-Approvers` and add your admin)
4. **Lifecycle:** Expires after 180 days, no extension without re-approval

**SC-300 exam tip:** Know single-stage vs multi-stage approval. Know that access packages can include groups, apps, and SharePoint sites. Know the lifecycle settings (expiry, extension, re-approval).

---

## Step 4: Test the Access Request Flow

### 4a: Request Access as a Regular User

1. Open an **incognito window**
2. Sign in as a test user (e.g., one who is NOT in Engineering)
3. Go to `myaccess.microsoft.com`
4. Browse **Access packages**
5. Find `Engineering Standard Access` → click **Request access**
6. Fill in the justification and project question
7. Click **Submit**

### 4b: Approve as a Manager

1. Sign in as the approver (Alex Chen or your admin account)
2. Go to `myaccess.microsoft.com` → **Approvals**
3. Find the pending request
4. Review the justification and answers
5. Click **Approve** (or **Deny** to test the denial flow)

### 4c: Verify the Assignment

1. Go to **Entitlement management** → **Access packages** → `Engineering Standard Access` → **Assignments**
2. Verify the user now has an active assignment
3. Check that they were added to SG-Engineering

**SC-300 exam tip:** Know the full request-approve-assign flow. Know that `myaccess.microsoft.com` is the self-service portal for end users to request and manage their access.

---

## Step 5: Implement Terms of Use

**Where:** Entra ID → Identity Governance → Terms of use

### 5a: Create a Terms of Use Document

1. Click **+ New terms of use**
2. **Name:** `Acceptable Use Policy`
3. **Display name:** `Expat Teacher's Lounge - Acceptable Use Policy`
4. Upload a PDF document (create a simple one-page AUP document)
5. **Require users to expand the terms:** Yes
6. **Require users to consent on every device:** No
7. **Expire consents:** Yes → **Expire starting on:** 1 year from today → **Frequency:** Yearly
8. **Duration before re-acceptance required:** 30 days
9. Click **Create**

### 5b: Link Terms of Use to a Conditional Access Policy

1. Go to **Conditional Access** → **+ New policy**
2. **Name:** `CA-ToU: Require Acceptable Use Policy`
3. **Users:** All users, exclude break-glass
4. **Target resources:** All cloud apps
5. **Grant:** **Require terms of use** → select `Acceptable Use Policy`
6. **Enable:** Report-only (test first)
7. Click **Create**

### 5c: Test Terms of Use

1. Sign in as a test user (incognito window)
2. The user should be presented with the Terms of Use
3. They must scroll through and accept before accessing any apps
4. Check the acceptance record: **Terms of use** → click your ToU → **Audit logs** / **Consents**

**SC-300 exam tip:** Know that Terms of Use integrates with Conditional Access. Know the expiration and re-acceptance settings. Know that you can view who has and hasn't accepted.

---

## Step 6: Manage External User Lifecycle Through Governance

### 6a: Configure Connected Organizations

**Where:** Entitlement management → Connected organizations

1. Click **+ Add connected organization**
2. **Name:** `Partner Company`
3. **Domain:** Enter a partner domain (use `gmail.com` for testing)
4. **Sponsors:** Add your admin account as the internal sponsor
5. Click **Create**

### 6b: Create an External Access Package

1. Create a new access package: `Partner Collaboration Access`
2. **Catalog:** `Expat Teacher's Lounge - General Access`
3. **Resources:** A specific group or app for external collaboration
4. **Requests:**
   - **Who can request:** For users not in your directory → select your connected organization
   - **Require approval:** Yes (2-stage: internal sponsor → resource owner)
   - **Require requestor information:** Yes — ask for company name, project, and expected duration
5. **Lifecycle:**
   - **Expire after:** 90 days
   - **Allow extension:** Yes, with re-approval
   - **When access expires, remove external user:** Yes (this is key — automatically cleans up guest accounts)
6. Click **Create**

**Why this matters:** This automates the external user lifecycle. When a contractor's access package expires, their guest account is automatically removed. No more stale contractor accounts sitting in your directory.

**SC-300 exam tip:** Know connected organizations, how external users request access packages, and the auto-removal of guest accounts when assignments expire. This is the production answer to "how do you manage contractor lifecycle?"

---

## Step 7: Create Access Reviews

**Where:** Identity Governance → Access reviews

### 7a: Review Group Memberships

1. Click **+ New access review**
2. **Review type:** Teams + Groups
3. **Review scope:** Select specific groups → select `SG-Engineering`, `SG-Sales`, `SG-Finance`
4. Click **Next**

**Reviews settings:**
1. **Review name:** `Quarterly Department Group Review`
2. **Frequency:** Quarterly
3. **Duration (in days):** 14
4. **Start date:** Today
5. **End:** Never (ongoing quarterly reviews)
6. Click **Next**

**Reviewers:**
1. **Select reviewers:** Group owner(s) (or Managers of users if no owner)
2. **Fallback reviewers:** Your admin account
3. Click **Next**

**Settings:**
1. **Auto apply results to resource:** Yes
2. **If reviewers don't respond:** Remove access (this is the key governance control)
3. **Action to apply on denied guest users:** Remove membership
4. **At end of review, send notification to:** Your admin account
5. Click **Next**

**Review + Create** → Click **Create**

### 7b: Review Application Assignments

1. Create another access review:
2. **Review type:** Applications
3. **Review scope:** Select enterprise apps from Lab 7
4. **Reviewers:** Application owner or managers of users
5. **Auto-apply:** Yes
6. **No response action:** Remove access
7. **Frequency:** Quarterly

### 7c: Review Privileged Role Assignments

1. Create another access review:
2. **Review type:** Roles → Microsoft Entra roles
3. **Scope:** Select `Global Administrator`, `User Administrator`, `Security Administrator`
4. **Reviewers:** Self-review (role holders review their own assignments)
5. **Frequency:** Monthly (privileged roles need more frequent review)
6. **No response action:** Remove access

### 7d: Respond to an Access Review

1. Sign in as a reviewer (manager account)
2. Go to `myaccess.microsoft.com` → **Access reviews**
3. Open the pending review
4. For each user, select **Approve** or **Deny** with a reason
5. Click **Submit**

### 7e: Monitor Access Review Progress

Back in the admin portal:
1. Go to **Access reviews** → click on your review
2. Check **Results** — see how many users were reviewed, approved, denied
3. Check **Reviewers** — see who has and hasn't completed their review
4. If a reviewer hasn't responded by the deadline, the auto-remediation kicks in

```powershell
# Get access review status via PowerShell
Get-MgIdentityGovernanceAccessReviewDefinition -All | 
    Select-Object DisplayName, Status, 
        @{N='Scope';E={$_.Scope.AdditionalProperties.query}},
        @{N='Created';E={$_.CreatedDateTime.ToString('yyyy-MM-dd')}} |
    Format-Table -AutoSize
```

**SC-300 exam tip:** Access reviews are heavily tested. Know the reviewer types (self, manager, group owner, specific users). Know auto-apply and no-response actions. Know the difference between reviewing groups, apps, and roles. Know the My Access portal URL (`myaccess.microsoft.com`).

### How to explain this in an interview:

*"I implemented quarterly access reviews for all department group memberships and monthly reviews for privileged roles. Managers are the primary reviewers — they see each person in their group and approve or deny their continued access. If a manager doesn't respond within 14 days, access is automatically removed. This ensures we have documented evidence of access review for SOX and SOC2 compliance, and it directly addresses access creep by forcing regular cleanup."*

---

## Step 8: Configure Privileged Identity Management (PIM)

**Where:** Entra ID → Identity Governance → Privileged Identity Management

PIM is where you eliminate permanent admin privileges. Instead of 10 people having Global Admin 24/7, they activate the role when they need it, with justification, for a limited time.

### 8a: Configure PIM Settings for Global Administrator

1. Go to **PIM** → **Microsoft Entra roles** → **Roles**
2. Click **Global Administrator** → **Settings**
3. Configure:

| Setting | Value | Why |
|---|---|---|
| Maximum activation duration | **8 hours** | Limits how long the role stays active |
| On activation, require | **Azure MFA** | Proves identity before granting admin access |
| Require justification on activation | **Yes** | Creates an audit trail of why the role was activated |
| Require ticket information on activation | **Yes** | Links activation to a change ticket |
| Require approval to activate | **Yes** | Another human must approve the activation |
| Select approver(s) | Your secondary admin or security lead | Separation of duties — can't self-approve |
| Assignment duration — Active | **Allow permanent eligible** → No, max 1 year | Forces annual renewal of eligibility |
| Require MFA on active assignment | **Yes** | Extra security for direct assignments |
| Notification settings | Enable all email notifications | Visibility into role activations |

4. Click **Update**

### 8b: Configure PIM for Other Roles

Repeat the settings configuration for:
- **User Administrator** — 4-hour max activation, require justification, no approval required
- **Security Administrator** — 4-hour max activation, require justification, no approval required
- **Exchange Administrator** — 4-hour max activation, require justification, no approval required

### 8c: Make Role Assignments Eligible (Not Permanent)

1. Go to **PIM** → **Microsoft Entra roles** → **Assignments**
2. Click **+ Add assignments**
3. **Role:** User Administrator
4. **Member:** Select a test user
5. **Assignment type:** **Eligible** (not Active)
6. **Duration:** 1 year
7. Click **Assign**

Remove any existing **permanent** assignments:
1. Go to **Assignments** → **Active assignments**
2. For each permanently assigned role (except break-glass accounts), click **Remove**
3. Re-add them as **Eligible** assignments

### 8d: Activate a Role (Walk Through the Experience)

1. Sign in as the user who has an eligible assignment (incognito window)
2. Go to **PIM** → **My roles** → **Microsoft Entra roles**
3. Find the eligible role → click **Activate**
4. **Duration:** Select how long (up to the max configured)
5. **Reason:** `Performing scheduled user account audit for Q2 compliance review`
6. **Ticket number:** `CHG-2026-0412`
7. Click **Activate**
8. If approval is required, wait for the approver to approve
9. Once activated, the role appears under **Active assignments**
10. After the duration expires, the role is automatically deactivated

### 8e: Approve a PIM Request (As an Approver)

1. Sign in as the approver
2. Go to **PIM** → **Approve requests**
3. Review the request — see the justification and ticket number
4. Click **Approve** (or **Deny** with a reason)

### 8f: Configure PIM for Azure Resources

1. Go to **PIM** → **Azure resources**
2. **Discover resources** → select your subscription
3. Click on a resource (e.g., your resource group from Lab 7)
4. Go to **Roles** → select **Contributor**
5. Click **Settings** → configure similar to Entra roles (activation duration, MFA, justification)
6. **Add assignments** → make a user eligible for Contributor on the resource group

### 8g: Configure PIM for Groups

1. Create a privileged access group:
   - Go to **Groups** → **+ New group**
   - **Name:** `PAG-SeniorAdmins`
   - **Group type:** Security
   - **Entra roles can be assigned:** **Yes** (this makes it PIM-eligible)
   - Click **Create**

2. Go to **PIM** → **Groups** → find `PAG-SeniorAdmins`
3. Configure settings (activation duration, approval, justification)
4. Add eligible members — these users can activate group membership through PIM

**SC-300 exam tip:** PIM for Groups is a newer topic on the exam. Know that only groups with "Entra roles can be assigned = Yes" can be managed through PIM. Know that PIM for Groups controls group membership activation, not role activation.

---

## Step 9: Review PIM Audit History

**Where:** PIM → Microsoft Entra roles → Audit history (or Resource audit)

1. Go to **Audit history**
2. Review:
   - Who activated which roles
   - When they activated and for how long
   - What justification and ticket number they provided
   - Who approved the activation
3. Filter by date range, role, or user
4. Export the audit data for compliance reporting

```powershell
# Pull PIM audit events via PowerShell
Get-MgAuditLogDirectoryAudit -Filter "activityDisplayName eq 'Add member to role completed (PIM activation)'" -Top 20 |
    Select-Object ActivityDateTime, 
        @{N='User';E={$_.InitiatedBy.User.DisplayName}},
        @{N='Role';E={$_.TargetResources[0].DisplayName}},
        @{N='Result';E={$_.Result}} |
    Format-Table -AutoSize
```

**SC-300 exam tip:** Know where to find PIM audit data and what information it contains. Auditors will ask for proof of just-in-time access and role activation history.

---

## Step 10: Create and Verify Break-Glass Account Monitoring

You created break-glass accounts in Lab 6. Now set up monitoring:

### 10a: Create an Alert for Break-Glass Sign-In

1. Go to **Entra ID** → **Monitoring** → **Diagnostic settings**
2. Ensure sign-in logs are being sent to Log Analytics (we'll configure this fully in Lab 10)

For now, create a simple monitoring script:

```powershell
# Check for break-glass account sign-ins in the last 24 hours
$breakGlassUPNs = @(
    "breakglass@expatteacherslounge.com",
    "breakglass2@expatteacherslounge.com"
)

foreach ($upn in $breakGlassUPNs) {
    $signIns = Get-MgAuditLogSignIn -Filter "userPrincipalName eq '$upn'" -Top 5
    if ($signIns) {
        Write-Host "[ALERT] Break-glass account $upn was used!" -ForegroundColor Red
        $signIns | Select-Object CreatedDateTime, Status, IpAddress, 
            @{N='Location';E={$_.Location.City + ', ' + $_.Location.CountryOrRegion}} |
            Format-Table
    } else {
        Write-Host "[OK] No sign-ins detected for $upn" -ForegroundColor Green
    }
}
```

### 10b: Document Break-Glass Procedures

Update your break-glass documentation from Lab 6 with governance controls:

```markdown
## Break-Glass Governance
- Monthly verification: Sign in with breakglass2 to confirm it works (rotate which account monthly)
- Access review: Break-glass accounts are reviewed in the monthly privileged role review
- Sign-in monitoring: Any sign-in triggers an immediate alert to the security team
- Password storage: Sealed envelopes in physical safe. Opening is logged and requires two-person authorization.
- Audit trail: All break-glass usage is documented with incident number and reason
```

**SC-300 exam tip:** Break-glass accounts must be excluded from CA policies, PIM, and MFA. They should be monitored with alerts. Know that you should have at least 2, they should be cloud-only (not synced from on-prem), and they should not be tied to any individual person.

---

## Step 11: Generate Governance Reports

```powershell
# Access review summary
Write-Host "=== ACCESS REVIEW STATUS ===" -ForegroundColor Cyan
Get-MgIdentityGovernanceAccessReviewDefinition -All |
    Select-Object DisplayName, Status, 
        @{N='Created';E={$_.CreatedDateTime.ToString('yyyy-MM-dd')}},
        @{N='Frequency';E={$_.Settings.RecurrenceSettings.RecurrenceType}} |
    Format-Table -AutoSize

# PIM role activation summary
Write-Host "`n=== PIM ELIGIBLE ASSIGNMENTS ===" -ForegroundColor Cyan
Get-MgRoleManagementDirectoryRoleEligibilitySchedule -All |
    Select-Object @{N='Principal';E={
        (Get-MgDirectoryObject -DirectoryObjectId $_.PrincipalId).AdditionalProperties.displayName
    }},
    @{N='Role';E={
        (Get-MgDirectoryRoleDefinition -UnifiedRoleDefinitionId $_.RoleDefinitionId).DisplayName
    }},
    @{N='EndDate';E={$_.ScheduleInfo.Expiration.EndDateTime.ToString('yyyy-MM-dd')}} |
    Format-Table -AutoSize

# Entitlement management summary
Write-Host "`n=== ACCESS PACKAGES ===" -ForegroundColor Cyan
Get-MgEntitlementManagementAccessPackage -All |
    Select-Object DisplayName, CreatedDateTime,
        @{N='Catalog';E={$_.CatalogId}} |
    Format-Table -AutoSize
```

Export all reports to the `reports/` folder for documentation.

---

## Step 12: Verify Everything

Checklist:

- [ ] Identity governance overview document created
- [ ] Access catalog created with resources added
- [ ] Access packages created (Engineering, Sales, Finance, HR, Contractor, Sensitive)
- [ ] Multi-stage approval package configured
- [ ] Access request tested end-to-end (request → approve → verify assignment)
- [ ] Terms of Use created and linked to CA policy
- [ ] Terms of Use tested with a user
- [ ] Connected organization created for external access
- [ ] External access package with auto-removal configured
- [ ] Quarterly access reviews created for groups
- [ ] Application access review created
- [ ] Privileged role access review created (monthly)
- [ ] Access review responded to as a reviewer
- [ ] Access review progress monitored
- [ ] PIM settings configured for Global Admin (8hr, MFA, approval, justification)
- [ ] PIM settings configured for other admin roles
- [ ] Permanent role assignments converted to eligible
- [ ] PIM activation tested (request → justify → approve → activate)
- [ ] PIM for Azure resources configured
- [ ] PIM for Groups configured (PAG-SeniorAdmins)
- [ ] PIM audit history reviewed
- [ ] Break-glass monitoring script created
- [ ] Break-glass governance procedures documented
- [ ] Governance reports generated

---

## Key Takeaways for SC-300

1. **Entitlement management** = self-service access requests with approval workflows and lifecycle automation
2. **Access packages** bundle resources (groups, apps, sites) with policies (who can request, who approves, when it expires)
3. **Access reviews** force periodic verification that access is still needed — auto-remediation removes access on no-response
4. **PIM** eliminates standing privilege — eligible assignments activate just-in-time with justification and time limits
5. **PIM for Groups** manages group membership activation, not role activation
6. **Terms of Use** integrate with CA to require legal acceptance before access
7. **Connected organizations** enable governed external access with automatic guest cleanup
8. **Break-glass accounts** are excluded from everything but monitored for everything

---

## What's Next

➡️ **Lab 10:** [Monitoring, Logs & Identity Secure Score](../lab10-monitoring-logs/) — Log Analytics, KQL queries, workbooks, and security posture improvement

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*


