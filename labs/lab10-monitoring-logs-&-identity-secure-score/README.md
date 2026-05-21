# Lab 10: Monitoring, Logs & Identity Secure Score

> **Type:** Focused Lab | **Time:** 2-3 hours | **SC-300 Domain:** Plan and automate identity governance (20-25%)

---

## Scenario

The CISO at Expat Teacher's Lounge wants a monitoring dashboard for the identity infrastructure. You need to configure centralized logging, write queries to investigate identity events, build visual reports for leadership, and improve the organization's security posture using Identity Secure Score. This is the final lab — it ties together everything you've built across the previous 9 labs.

---

## SC-300 Exam Objectives Covered

- Review and analyze sign-in, audit, and provisioning logs
- Configure diagnostic settings (Log Analytics, storage accounts, Event Hubs)
- Monitor Microsoft Entra ID using KQL queries in Log Analytics
- Analyze Microsoft Entra ID using workbooks and reporting
- Monitor and improve security posture using Identity Secure Score

---

## Prerequisites

- Completed Labs 1-9
- Microsoft 365 E5 subscription
- Azure subscription (for Log Analytics workspace)
- Activity in your tenant from previous labs (sign-ins, role activations, group changes, etc.)

---

## Step 1: Explore Sign-In Logs

**Where:** Entra ID → Monitoring → Sign-in logs

### 1a: Review Recent Sign-Ins

1. Go to **Sign-in logs**
2. You'll see all authentication events — successful and failed
3. Click on any sign-in entry and review:
   - **User:** Who signed in
   - **Application:** What app they accessed
   - **Status:** Success or failure (and the error code if failed)
   - **IP address:** Where they signed in from
   - **Location:** City and country
   - **Device info:** OS, browser, compliant/managed status
   - **Conditional Access:** Which policies were evaluated, applied, or not applied
   - **Authentication details:** Which MFA method was used, how many factors

### 1b: Filter for Specific Events

Practice filtering — this is how you investigate in production:

| Filter | What It Shows | When To Use |
|---|---|---|
| Status = Failure | Failed sign-ins only | Investigating account lockouts or attacks |
| User = specific user | All sign-ins for one person | Investigating a compromised account |
| Application = specific app | All sign-ins to one app | Investigating app-specific issues |
| IP address = specific IP | All sign-ins from one location | Investigating suspicious locations |
| Conditional Access = Failure | Sign-ins blocked by CA policies | Verifying CA policies are working |
| Risk level = High | High-risk sign-ins | Investigating Identity Protection alerts |

### 1c: Investigate a Failed Sign-In

1. Filter for **Status = Failure**
2. Click on a failed sign-in
3. Note the **Error code** and **Failure reason** — common ones:
   - `50126` — Invalid username or password
   - `50053` — Account is locked
   - `50057` — Account is disabled
   - `53003` — Blocked by Conditional Access
   - `50074` — MFA required but not completed
4. Document how you'd respond to each error in a troubleshooting runbook

**SC-300 exam tip:** Sign-in logs are the primary investigation tool. Know how to filter by user, app, status, IP, and CA result. Know common error codes. Know that sign-in logs show the full CA policy evaluation — which policies applied, which didn't, and why.

---

## Step 2: Explore Audit Logs

**Where:** Entra ID → Monitoring → Audit logs

### 2a: Review Recent Changes

1. Go to **Audit logs**
2. These show all directory changes — user created, group modified, role assigned, policy changed
3. Click on any entry and review:
   - **Activity:** What happened (e.g., "Add member to group")
   - **Category:** Which service area (User Management, Group Management, Role Management)
   - **Initiated by:** Who made the change (user or app)
   - **Target:** What was changed
   - **Modified properties:** Before and after values

### 2b: Find Specific Events

Practice these searches:

1. **Find when a user was created:**
   - Filter: Activity = "Add user"
   - Find one of your users from Lab 2

2. **Find when a role was assigned:**
   - Filter: Activity = "Add member to role"
   - Find a PIM activation from Lab 9

3. **Find when a CA policy was changed:**
   - Filter: Category = "Policy"
   - Find one of your CA policy modifications from Lab 6

4. **Find when a group membership changed:**
   - Filter: Activity = "Add member to group"
   - Find one of your group assignments

### 2c: Export Audit Data

```powershell
# Export recent audit events via PowerShell
Get-MgAuditLogDirectoryAudit -Top 100 -Sort "activityDateTime desc" |
    Select-Object ActivityDateTime, ActivityDisplayName, Category,
        @{N='InitiatedBy';E={$_.InitiatedBy.User.DisplayName}},
        @{N='Target';E={$_.TargetResources[0].DisplayName}},
        Result |
    Export-Csv -Path "./reports/audit-log-export.csv" -NoTypeInformation

Write-Host "Exported 100 most recent audit events" -ForegroundColor Green
```

**SC-300 exam tip:** Know the difference between sign-in logs (authentication events) and audit logs (directory changes). Know that audit logs show WHO changed WHAT and WHEN — this is the compliance trail.

---

## Step 3: Explore Provisioning Logs

**Where:** Entra ID → Monitoring → Provisioning logs

1. Go to **Provisioning logs**
2. These show automated provisioning events — when Entra ID provisions or deprovisions users to/from connected applications
3. If you configured enterprise apps with provisioning in Lab 7, you'll see events here
4. Review:
   - **Action:** Create, Update, Delete, Disable
   - **Source system:** Where the identity came from
   - **Target system:** Where the identity was provisioned to
   - **Status:** Success or failure
   - **Modified properties:** What attributes were synced

**SC-300 exam tip:** Provisioning logs are separate from sign-in and audit logs. They specifically track automated user provisioning to/from connected applications. Know where to find them and what they show.

---

## Step 4: Configure Diagnostic Settings (Log Analytics)

**Where:** Entra ID → Monitoring → Diagnostic settings

This is where you send Entra ID logs to a centralized location for long-term storage and advanced queries.

### 4a: Create a Log Analytics Workspace

1. Go to **portal.azure.com** → search **Log Analytics workspaces** → **+ Create**
2. **Resource group:** `RG-IAMLab`
3. **Name:** `law-iamlabcorp`
4. **Region:** Choose your nearest region
5. Click **Review + Create** → **Create**

### 4b: Configure Diagnostic Settings

1. Go back to **entra.microsoft.com** → **Monitoring** → **Diagnostic settings**
2. Click **+ Add diagnostic setting**
3. **Diagnostic setting name:** `Send-to-LogAnalytics`
4. **Logs:** Check ALL categories:
   - ✅ AuditLogs
   - ✅ SignInLogs
   - ✅ NonInteractiveUserSignInLogs
   - ✅ ServicePrincipalSignInLogs
   - ✅ ManagedIdentitySignInLogs
   - ✅ ProvisioningLogs
   - ✅ ADFSSignInLogs (if applicable)
   - ✅ RiskyUsers
   - ✅ UserRiskEvents
   - ✅ NetworkAccessTrafficLogs (if applicable)
5. **Destination details:** Check **Send to Log Analytics workspace**
6. Select your subscription and workspace (`law-iamlabcorp`)
7. Click **Save**

**Wait 15-30 minutes for data to start flowing to Log Analytics.**

**Why this matters:** By default, Entra ID keeps sign-in logs for 30 days and audit logs for 30 days. Sending them to Log Analytics gives you long-term retention, advanced querying with KQL, and the ability to build custom dashboards and alerts.

**SC-300 exam tip:** Know the three destinations for diagnostic settings: Log Analytics (for KQL queries and workbooks), Storage Account (for long-term archival), Event Hubs (for streaming to SIEM systems). Know that each destination serves a different purpose and you can send to multiple simultaneously.

---

## Step 5: Write KQL Queries in Log Analytics

**Where:** portal.azure.com → Log Analytics workspace → Logs

Wait for data to populate (15-30 minutes after configuring diagnostic settings), then run these queries:

### 5a: Failed Sign-Ins in the Last 24 Hours

```kql
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| project TimeGenerated, UserDisplayName, AppDisplayName, IPAddress, 
    Location, ResultType, ResultDescription
| order by TimeGenerated desc
```

### 5b: Sign-Ins from Outside Your Country

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| where Location.countryOrRegion != "QA"  // Change to your country code
| project TimeGenerated, UserDisplayName, AppDisplayName, 
    IPAddress, Location.city, Location.countryOrRegion
| order by TimeGenerated desc
```

### 5c: Conditional Access Policy Failures

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| mv-expand ConditionalAccessPolicies
| where ConditionalAccessPolicies.result == "failure"
| project TimeGenerated, UserDisplayName, AppDisplayName,
    PolicyName = ConditionalAccessPolicies.displayName,
    PolicyResult = ConditionalAccessPolicies.result
| order by TimeGenerated desc
```

### 5d: PIM Role Activations

```kql
AuditLogs
| where TimeGenerated > ago(30d)
| where OperationName == "Add member to role completed (PIM activation)"
| project TimeGenerated, 
    User = InitiatedBy.user.displayName,
    Role = TargetResources[0].displayName,
    Result
| order by TimeGenerated desc
```

### 5e: Password Resets (SSPR Activity)

```kql
AuditLogs
| where TimeGenerated > ago(30d)
| where OperationName has "password"
| project TimeGenerated, OperationName,
    User = TargetResources[0].displayName,
    InitiatedBy = InitiatedBy.user.displayName,
    Result
| order by TimeGenerated desc
```

### 5f: User Creation Events

```kql
AuditLogs
| where TimeGenerated > ago(30d)
| where OperationName == "Add user"
| project TimeGenerated,
    NewUser = TargetResources[0].displayName,
    CreatedBy = InitiatedBy.user.displayName,
    Result
| order by TimeGenerated desc
```

### 5g: Break-Glass Account Monitoring

```kql
SigninLogs
| where TimeGenerated > ago(30d)
| where UserPrincipalName startswith "breakglass"
| project TimeGenerated, UserPrincipalName, IPAddress, 
    Location.city, Location.countryOrRegion, 
    AppDisplayName, Status.errorCode
| order by TimeGenerated desc
```

### 5h: MFA Method Usage Analysis

```kql
SigninLogs
| where TimeGenerated > ago(30d)
| where AuthenticationDetails has "MFA"
| mv-expand AuthenticationDetails
| where AuthenticationDetails.authenticationMethod != ""
| summarize Count = count() by 
    Method = tostring(AuthenticationDetails.authenticationMethod)
| order by Count desc
| render barchart
```

Save each query as a **function** or **saved query** in Log Analytics for reuse:
1. Run the query
2. Click **Save** → **Save as query**
3. Name it descriptively (e.g., `IAM-FailedSignIns-24h`)

**SC-300 exam tip:** KQL is tested on the exam. You don't need to be a KQL expert, but know basic query structure: table name, where filters, project (select columns), summarize (aggregate), and order by. Know which table contains which data (SigninLogs for auth, AuditLogs for changes).

### How to explain this in an interview:

*"I configured diagnostic settings to stream all Entra ID logs to a Log Analytics workspace, giving us long-term retention and advanced query capabilities. I wrote KQL queries for key security scenarios: failed sign-ins, sign-ins from unexpected locations, CA policy blocks, PIM activations, and break-glass account usage. These queries run on a schedule and alert the security team when something needs attention."*

---

## Step 6: Use Entra ID Workbooks

**Where:** Entra ID → Monitoring → Workbooks

Workbooks are pre-built visual dashboards. Review the following:

### 6a: Sign-In Analysis Workbook

1. Click **Workbooks** → find **Sign-ins** or **Sign-in analysis**
2. Open it and review:
   - Sign-in trends over time
   - Top applications by sign-in count
   - Geographic distribution of sign-ins
   - Failure analysis

### 6b: Conditional Access Insights Workbook

1. Find **Conditional Access insights and reporting**
2. Review:
   - Policy impact — how many sign-ins each policy affected
   - Users impacted by each policy
   - Policies in report-only mode vs enforced

### 6c: Authentication Methods Activity Workbook

1. Find **Authentication methods activity**
2. Review:
   - Which MFA methods are being used
   - Registration completion rates
   - Passwordless adoption trends

### 6d: Sensitive Operations Workbook

1. Find **Sensitive operations report**
2. Review:
   - High-privilege actions performed
   - CA policy modifications
   - Role assignment changes

Take screenshots of each workbook for your documentation.

**SC-300 exam tip:** Know the key workbooks and what data they visualize. The exam may ask which workbook to use for a specific monitoring scenario.

---

## Step 7: Monitor and Improve Identity Secure Score

**Where:** Entra ID → Overview → Identity Secure Score (or Protection → Identity Secure Score)

### 7a: Review Your Current Score

1. Go to **Identity Secure Score**
2. Note your current score (as a percentage)
3. Review the breakdown by category:
   - Identity
   - Data
   - Device
   - Apps
   - Infrastructure

### 7b: Review Recommendations

1. Click **Improvement actions**
2. Review each recommendation:
   - **Status:** To address, In progress, Completed, Risk accepted
   - **Score impact:** How much your score improves if implemented
   - **User impact:** How disruptive the change is to users

Common high-impact recommendations:

| Recommendation | Impact | You Completed In |
|---|---|---|
| Enable MFA for all users | High | Lab 6 (CA001) |
| Block legacy authentication | High | Lab 6 (CA002) |
| Designate more than one Global Admin | Medium | Lab 6 (break-glass) |
| Enable SSPR | Medium | Lab 5 |
| Use least privileged admin roles | Medium | Lab 1 (custom roles, AUs) |
| Enable password protection | Medium | Lab 5 |
| Enable PIM | High | Lab 9 |
| Configure access reviews | Medium | Lab 9 |

### 7c: Implement Remaining Recommendations

1. Sort recommendations by **Score impact** (highest first)
2. Implement at least 3 recommendations that you haven't already addressed
3. For each one:
   - Document the current state (before)
   - Implement the recommendation
   - Document the new state (after)
   - Check back in 24 hours to see the score update

### 7d: Track Score Over Time

```powershell
# Note: Identity Secure Score may require beta Graph API
# Document your score manually:
$scoreEntry = [PSCustomObject]@{
    Date = Get-Date -Format 'yyyy-MM-dd'
    Score = "XX%"  # Enter your current score
    ActionsCompleted = "MFA enabled, Legacy auth blocked, PIM configured"
    Notes = "Score improved from XX% to XX% after Lab 9 governance changes"
}
$scoreEntry | Export-Csv -Path "./reports/secure-score-tracking.csv" -Append -NoTypeInformation
```

**SC-300 exam tip:** Know what Identity Secure Score measures and how to improve it. Know that some recommendations can be marked as "Risk accepted" if they don't apply to your organization. Know that the score updates periodically, not in real-time.

### How to explain this in an interview:

*"I used Identity Secure Score as a prioritization framework for security improvements. When I started, the score was around 40%. After implementing MFA for all users, blocking legacy authentication, configuring PIM for admin roles, and setting up access reviews, the score improved to over 75%. I track the score monthly and address new recommendations as they appear."*

---

## Step 8: Create a Monitoring Summary Dashboard

Create `docs/monitoring-dashboard.md` as a reference for what you monitor and how:

```markdown
## Identity Monitoring Dashboard

### Daily Checks
- [ ] Review failed sign-ins (KQL: IAM-FailedSignIns-24h)
- [ ] Check break-glass account sign-ins (KQL: IAM-BreakGlass-Monitor)
- [ ] Review Identity Protection alerts for new risky users

### Weekly Checks
- [ ] Review sign-ins from unusual locations
- [ ] Check CA policy blocks — are legitimate users being blocked?
- [ ] Review PIM activation history — any unusual activations?
- [ ] Check access review completion progress

### Monthly Checks  
- [ ] Review Identity Secure Score and new recommendations
- [ ] Export audit log summary for compliance
- [ ] Verify break-glass account functionality (test sign-in)
- [ ] Review license utilization report
- [ ] Check for stale guest accounts (no sign-in > 90 days)

### Quarterly Checks
- [ ] Comprehensive access review cycle
- [ ] Privileged role assignment audit
- [ ] Application consent review
- [ ] Authentication method usage analysis
```

---

## Step 9: Verify Everything

Checklist:

- [ ] Sign-in logs explored and filtered by multiple criteria
- [ ] Failed sign-in investigated with error code documented
- [ ] Audit logs explored — found user creation, role assignment, group changes
- [ ] Audit log exported via PowerShell
- [ ] Provisioning logs reviewed (if applicable)
- [ ] Log Analytics workspace created
- [ ] Diagnostic settings configured — all log categories sent to Log Analytics
- [ ] KQL: Failed sign-ins query written and saved
- [ ] KQL: Sign-ins from outside country query written
- [ ] KQL: CA policy failures query written
- [ ] KQL: PIM activations query written
- [ ] KQL: Password resets query written
- [ ] KQL: Break-glass monitoring query written
- [ ] KQL: MFA method analysis query written (with bar chart)
- [ ] Sign-in analysis workbook reviewed
- [ ] Conditional Access insights workbook reviewed
- [ ] Authentication methods workbook reviewed
- [ ] Identity Secure Score reviewed
- [ ] At least 3 Secure Score recommendations implemented
- [ ] Score tracked before and after improvements
- [ ] Monitoring dashboard document created

---

## Key Takeaways for SC-300

1. **Sign-in logs** = authentication events. **Audit logs** = directory changes. **Provisioning logs** = automated user sync.
2. **Diagnostic settings** send logs to Log Analytics (queries), Storage (archive), or Event Hubs (SIEM streaming)
3. **KQL** is the query language for Log Analytics — know basic syntax (table, where, project, summarize)
4. **Workbooks** provide pre-built visual dashboards — know the key workbooks and what they show
5. **Identity Secure Score** prioritizes security improvements — track it monthly
6. **Long-term log retention** requires sending logs to Log Analytics or a Storage Account — default retention is only 30 days

---

## Congratulations — You've Completed the SC-300 Lab Series!

You've now hands-on practiced every major SC-300 exam objective across 10 labs:

| Domain | Weight | Labs Completed |
|--------|--------|---|
| Implement and manage user identities | 20-25% | Labs 1, 2, 3, 4 ✅ |
| Implement authentication and access management | 25-30% | Labs 5, 6 ✅ |
| Plan and implement workload identities | 20-25% | Labs 7, 8 ✅ |
| Plan and automate identity governance | 20-25% | Labs 9, 10 ✅ |

### Next Steps

1. **Take the Microsoft official practice assessment** at learn.microsoft.com — it's free
2. **Review weak areas** — go back to the labs where you felt least confident
3. **Schedule your SC-300 exam** — you've done the work, trust your preparation
4. **Film your YouTube episodes** — document each lab for The Cyber Chronicles
5. **Update your GitHub and LinkedIn** — show the world what you've built

### Your Portfolio Now Includes

- 🔐 10 completed SC-300 labs with documentation and screenshots
- 💻 IAM Lifecycle Automation project with PowerShell scripts
- 📺 YouTube tutorial content for 13+ episodes
- 📄 Professional documentation (SOPs, decision matrices, architecture diagrams)
- 🎓 SC-300 certification (coming soon!)

You've gone from zero IAM experience to having a more documented portfolio than most people with 2-3 years in the field. That's something to be proud of.

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
