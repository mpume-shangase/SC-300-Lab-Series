# Lab 4: Hybrid Identity with Entra Connect

> **Type:** Big Project | **Time:** 4-5 hours | **SC-300 Domain:** Implement and manage user identities (20-25%)

---

## Scenario

IAM Lab Corp has been running on-premises Active Directory for 15 years. The CTO has decided to move to a hybrid identity model — keeping on-prem AD for legacy applications while extending identities to the cloud for Microsoft 365 and SaaS applications. You need to build the on-prem AD environment, synchronize identities to Entra ID, configure the right authentication method, and ensure the sync is healthy and monitored.

This is the most hands-on lab in Domain 1 and one of the most commonly tested topics on SC-300.

---

## SC-300 Exam Objectives Covered

- Implement and manage Microsoft Entra Connect Sync
- Implement and manage Microsoft Entra Cloud Sync
- Implement and manage password hash synchronization (PHS)
- Implement and manage pass-through authentication (PTA)
- Implement and manage seamless single sign-on (SSO)
- Migrate from AD FS to other authentication mechanisms (conceptual)
- Implement and manage Microsoft Entra Connect Health

---

## Prerequisites

- Completed Labs 1-3
- Microsoft 365 E5 subscription
- A computer capable of running a virtual machine (8GB RAM minimum, 16GB recommended)
- VirtualBox (free) or Hyper-V (Windows Pro/Enterprise) installed
- Windows Server 2022 ISO (evaluation from Microsoft Evaluation Center — free for 180 days)

---

## Step 1: Build the On-Premises Active Directory Lab

### 1a: Download Windows Server

1. Go to **microsoft.com/en-us/evalcenter/evaluate-windows-server-2022**
2. Download the ISO file (Desktop Experience version)
3. This is a free 180-day evaluation — no purchase required

### 1b: Create the Virtual Machine

**Using VirtualBox (Mac or Windows):**

1. Open VirtualBox → click **New**
2. **Name:** `DC01-IAMLabCorp`
3. **Type:** Microsoft Windows
4. **Version:** Windows 2022 (64-bit)
5. **Memory:** 4096 MB (4 GB)
6. **Hard disk:** Create a virtual hard disk, 50 GB, VDI, Dynamically allocated
7. Click **Create**

Before starting the VM:
1. Click **Settings** → **Network**
2. Adapter 1: Attached to **NAT** (for internet access)
3. Click **Settings** → **Storage**
4. Click the empty CD icon → choose the Windows Server ISO
5. Click **OK**

### 1c: Install Windows Server

1. Start the VM
2. Select language, time, keyboard → **Install Now**
3. Select **Windows Server 2022 Standard (Desktop Experience)**
4. Accept license terms
5. Choose **Custom: Install Windows only**
6. Select the drive → **Next** → wait for installation
7. Set an Administrator password (e.g., `IAMLabAdmin2026!`) — write this down
8. Log in to the server

### 1d: Configure Basic Networking

Open PowerShell as Administrator on the server:

```powershell
# Set a static IP address (adjust for your network)
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.2.10 -PrefixLength 24 -DefaultGateway 10.0.2.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.2.10,8.8.8.8

# Rename the server
Rename-Computer -NewName "DC01" -Restart
```

After restart, verify internet connectivity:

```powershell
Test-Connection google.com
```

**Why this matters:** In most enterprises, you'll be connecting to an existing on-prem AD environment. Building one from scratch teaches you the fundamentals and makes you appreciate what Entra Connect is actually synchronizing.

---

## Step 2: Install Active Directory Domain Services

On your Windows Server VM, open PowerShell as Administrator:

```powershell
# Install AD DS role
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Promote to domain controller
Install-ADDSForest `
    -DomainName "iamlabcorp.local" `
    -DomainNetBIOSName "IAMLABCORP" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDNS:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "DSRMPassword2026!" -AsPlainText -Force) `
    -Force:$true
```

The server will restart. After restart, log in as `IAMLABCORP\Administrator`.

Verify AD is running:

```powershell
Get-ADDomain
Get-ADForest
```

**SC-300 exam tip:** You don't need to know how to install AD DS for the exam, but understanding the on-prem AD structure (domains, forests, OUs, GPOs) helps you understand what Entra Connect synchronizes.

---

## Step 3: Create the On-Premises Organizational Structure

Create OUs and populate with users:

```powershell
# Create OUs
$departments = @("Engineering", "Sales", "Human Resources", "Finance", "Marketing", "IT")
foreach ($dept in $departments) {
    New-ADOrganizationalUnit -Name $dept -Path "DC=iamlabcorp,DC=local"
    Write-Host "[CREATED] OU: $dept" -ForegroundColor Green
}

# Create a Disabled Users OU
New-ADOrganizationalUnit -Name "Disabled Users" -Path "DC=iamlabcorp,DC=local"

# Create users in each department
$password = ConvertTo-SecureString "UserPass2026!" -AsPlainText -Force

$onPremUsers = @(
    @{First="Michael"; Last="Torres";   Dept="Engineering";     Title="Solutions Architect";    OU="Engineering"}
    @{First="Sarah";   Last="Kim";      Dept="Engineering";     Title="Data Engineer";          OU="Engineering"}
    @{First="David";   Last="Osei";     Dept="Engineering";     Title="Platform Engineer";      OU="Engineering"}
    @{First="Rachel";  Last="Green";    Dept="Sales";           Title="Regional Sales Manager"; OU="Sales"}
    @{First="Ahmed";   Last="Nasser";   Dept="Sales";           Title="Sales Engineer";         OU="Sales"}
    @{First="Laura";   Last="Costa";    Dept="Human Resources"; Title="Benefits Administrator"; OU="Human Resources"}
    @{First="Chris";   Last="Anderson"; Dept="Human Resources"; Title="Training Manager";       OU="Human Resources"}
    @{First="Yuki";    Last="Sato";     Dept="Finance";         Title="Treasury Analyst";       OU="Finance"}
    @{First="Mohammed";Last="Al-Rashid";Dept="Finance";         Title="Compliance Officer";     OU="Finance"}
    @{First="Anna";    Last="Novak";    Dept="Marketing";       Title="Digital Marketing Lead"; OU="Marketing"}
    @{First="Brian";   Last="Oconnor";  Dept="Marketing";       Title="Content Producer";       OU="Marketing"}
    @{First="Kenji";   Last="Watanabe"; Dept="IT";              Title="Systems Administrator";  OU="IT"}
    @{First="Diana";   Last="Mendez";   Dept="IT";              Title="Security Analyst";       OU="IT"}
    @{First="Robert";  Last="Singh";    Dept="IT";              Title="Cloud Administrator";    OU="IT"}
    @{First="Nadia";   Last="Volkov";   Dept="Engineering";     Title="ML Engineer";            OU="Engineering"}
)

foreach ($u in $onPremUsers) {
    $upn = "$($u.First.ToLower()).$($u.Last.ToLower())@iamlabcorp.local"
    $sam = "$($u.First.ToLower()).$($u.Last.ToLower())"
    if ($sam.Length -gt 20) { $sam = $sam.Substring(0,20) }
    
    New-ADUser `
        -GivenName $u.First `
        -Surname $u.Last `
        -Name "$($u.First) $($u.Last)" `
        -DisplayName "$($u.First) $($u.Last)" `
        -SamAccountName $sam `
        -UserPrincipalName $upn `
        -Department $u.Dept `
        -Title $u.Title `
        -Path "OU=$($u.OU),DC=iamlabcorp,DC=local" `
        -AccountPassword $password `
        -Enabled $true `
        -ChangePasswordAtLogon $true
    
    Write-Host "[CREATED] $($u.First) $($u.Last) - $($u.Dept)" -ForegroundColor Green
}

Write-Host "`nTotal users created: $($onPremUsers.Count)" -ForegroundColor Cyan
```

Verify:

```powershell
Get-ADUser -Filter * -Properties Department,Title | 
    Select-Object Name, Department, Title, UserPrincipalName | 
    Sort-Object Department | Format-Table
```

---

## Step 4: Run IdFix to Identify Synchronization Issues

Before connecting to Entra ID, scan your directory for issues that would cause sync errors:

1. Download IdFix from **microsoft.com/en-us/download/details.aspx?id=36832**
2. Install and run it on your domain controller
3. Click **Query** to scan your directory
4. Review the results — IdFix identifies:
   - Duplicate attributes (two users with the same UPN)
   - Invalid characters in names
   - Blank required fields
   - Format errors

5. Fix any issues it identifies before proceeding
6. Take a screenshot of a clean scan

**Why this matters:** IdFix is the first step before any production Entra Connect deployment. Sync errors are painful to troubleshoot after the fact — much better to fix them upfront.

**SC-300 exam tip:** Know what IdFix does and when to run it. The exam expects you to use IdFix BEFORE configuring Entra Connect.

---

## Step 5: Add a UPN Suffix for Cloud Matching

On-prem UPNs end in `.local` but cloud UPNs use your `.onmicrosoft.com` domain. You need to add a routable UPN suffix:

```powershell
# Add the cloud domain as a UPN suffix
Set-ADForest -Identity "iamlabcorp.local" `
    -UPNSuffixes @{Add="thecyberkhroniclesgmail.onmicrosoft.com"}

# Update all users to use the new UPN suffix
Get-ADUser -Filter * -SearchBase "DC=iamlabcorp,DC=local" | ForEach-Object {
    $newUPN = $_.SamAccountName + "@thecyberkhroniclesgmail.onmicrosoft.com"
    Set-ADUser $_ -UserPrincipalName $newUPN
    Write-Host "[UPDATED] $($_.Name) → $newUPN" -ForegroundColor Green
}
```

**SC-300 exam tip:** UPN matching is a key sync concept. Know that on-prem UPN suffixes must match a verified domain in Entra ID for soft-matching to work. The `.local` suffix cannot be synced directly.

---

## Step 6: Install and Configure Microsoft Entra Connect Sync

### 6a: Download and Install

1. On your Windows Server VM, open a browser
2. Go to **microsoft.com/en-us/download/details.aspx?id=47594** (or search for "Microsoft Entra Connect download")
3. Download and run the installer
4. Accept the license terms → **Continue**

### 6b: Choose Express vs. Custom Settings

For this lab, use **Customize** to understand all the options:

1. Click **Customize**
2. On the install components page, leave defaults → **Install**
3. **User sign-in page:**
   - Select **Password Hash Synchronization** (recommended for most scenarios)
   - Check **Enable Single Sign-On**
   - Click **Next**
4. **Connect to Microsoft Entra ID:**
   - Enter your Global Admin credentials (`admin@thecyberkhroniclesgmail.onmicrosoft.com`)
   - Click **Next**
5. **Connect directories:**
   - Forest: `iamlabcorp.local`
   - Click **Add Directory** → enter domain admin credentials
   - Click **Next**
6. **Domain and OU filtering:**
   - Select **Sync selected domains and OUs**
   - Check: Engineering, Sales, Human Resources, Finance, Marketing, IT
   - Uncheck: Built-in, Computers, Domain Controllers (you typically don't sync these)
   - Click **Next**
7. **Uniquely identifying users:**
   - Leave default: Users are represented only once across all directories
   - Click **Next**
8. **Filter users and devices:**
   - Leave default: Synchronize all users and devices
   - Click **Next**
9. **Optional features:**
   - Check **Password hash synchronization** (should already be checked)
   - Check **Password writeback** if available
   - Click **Next**
10. **Enable single sign-on:**
    - Enter domain admin credentials to configure Seamless SSO
    - Click **Next**
11. **Ready to configure:**
    - Check **Start the synchronization process when configuration completes**
    - Click **Install**

### 6c: Verify Initial Sync

Wait for the installation and first sync to complete (5-10 minutes), then verify:

On the server:
```powershell
# Check sync status
Get-ADSyncScheduler
Get-ADSyncConnectorRunStatus
```

In the Entra portal:
1. Go to **entra.microsoft.com** → **Entra ID** → **Entra Connect** (or **Microsoft Entra Connect**)
2. Check the **Sync status** — it should show "Synced successfully"
3. Go to **Users** → **All users** — you should see your on-prem users appearing

**SC-300 exam tip:** Know the Express vs. Custom installation paths. Express uses PHS and syncs all users from all domains. Custom lets you select OUs, choose authentication methods, and enable additional features.

### How to explain this in an interview:

*"I deployed Entra Connect with custom settings to have granular control over what gets synchronized. I selected specific OUs to sync, excluding system containers. I chose password hash synchronization for authentication because it provides the best balance of simplicity and reliability — it doesn't require additional infrastructure like AD FS or PTA agents, and it supports leaked credential detection through ID Protection."*

---

## Step 7: Understand Authentication Methods

Create a decision matrix documenting when to use each method:

| Method | How It Works | Pros | Cons | When To Use |
|--------|-------------|------|------|-------------|
| **Password Hash Sync (PHS)** | Hash of on-prem password is synced to Entra ID. Authentication happens in the cloud. | Simplest to deploy. No on-prem infrastructure for auth. Enables leaked credential detection. | Passwords are (hashed) in the cloud — some regulated industries object. | Default recommendation for most organizations. |
| **Pass-Through Auth (PTA)** | Auth request passes through an on-prem agent to AD. Password never leaves on-prem. | Password stays on-prem. Real-time enforcement of on-prem policies (account disabled, logon hours). | Requires on-prem agent (single point of failure without redundancy). No leaked credential detection. | When regulations prohibit any form of password in the cloud. |
| **AD FS** | Federated auth through on-prem AD FS servers. Full control over authentication. | Most control. Supports complex claim rules. Smart card auth. | Most complex. Requires AD FS farm (2+ servers). Expensive to maintain. | Legacy — Microsoft recommends migrating away from AD FS. |
| **Seamless SSO** | Works WITH PHS or PTA. Auto-signs users in from domain-joined devices using Kerberos. | Transparent user experience. No password prompt on corporate devices. | Only works on domain-joined devices. Requires Kerberos. | Add-on to PHS or PTA for better user experience. |

Save this as `docs/authentication-decision-matrix.md` in your GitHub repo.

**SC-300 exam tip:** This comparison is one of the most heavily tested topics. Know the trade-offs of each method, especially PHS vs. PTA. Know that Microsoft recommends PHS as the default. Know that Seamless SSO is an add-on, not a standalone method.

---

## Step 8: Compare Entra Connect Sync vs. Cloud Sync

| Feature | Entra Connect Sync | Entra Cloud Sync |
|---------|-------------------|-----------------|
| Install location | On-prem server (heavyweight) | Lightweight agent on any domain-joined machine |
| High availability | Active-passive (staging mode) | Multiple agents auto-balance |
| Multi-forest | Supported | Supported (simpler setup) |
| OU filtering | Yes | Yes |
| Group writeback | Yes | Limited |
| Device writeback | Yes | No |
| Exchange hybrid | Yes | Limited |
| Custom sync rules | Yes (complex) | No (simple attribute mapping) |
| Password writeback | Yes | Yes |
| Best for | Complex environments with custom sync needs | Simple sync scenarios, multi-forest, and when HA is needed without staging mode |

Document this comparison and include it in your lab notes.

**SC-300 exam tip:** Know when to recommend Cloud Sync vs. Connect Sync. Cloud Sync is newer, simpler, and better for multi-forest scenarios but lacks the advanced customization of Connect Sync.

---

## Step 9: Configure Entra Connect Health

**Where:** Entra ID → Entra Connect → Health (or portal.azure.com → Microsoft Entra Connect Health)

1. Entra Connect Health agent should have been installed automatically with Entra Connect
2. Navigate to the Connect Health portal
3. Review:
   - **Sync errors** — any objects that failed to synchronize
   - **Sync performance** — how long sync cycles take
   - **Password hash sync heartbeat** — confirms PHS is working
   - **Alerts** — any active health alerts

4. If Health wasn't installed automatically, download and install the **Entra Connect Health Agent for Sync** from the portal

**SC-300 exam tip:** Know that Connect Health requires Entra ID P1 or P2. Know what it monitors (sync status, PHS heartbeat, AD FS performance if applicable). Know that it sends health data to the Azure portal, not to an on-prem dashboard.

---

## Step 10: Troubleshoot a Sync Error (Intentional)

Create a sync conflict to practice troubleshooting:

On your domain controller:
```powershell
# Create a user with a UPN that already exists in the cloud
New-ADUser `
    -Name "Jessica Thompson Duplicate" `
    -SamAccountName "jessica.thompson2" `
    -UserPrincipalName "jessica.thompson@thecyberkhroniclesgmail.onmicrosoft.com" `
    -Path "OU=Engineering,DC=iamlabcorp,DC=local" `
    -AccountPassword (ConvertTo-SecureString "UserPass2026!" -AsPlainText -Force) `
    -Enabled $true
```

Wait for the next sync cycle (30 minutes) or force a sync:
```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Then check:
1. Go to **Entra ID** → **Entra Connect** → view sync errors
2. You should see a conflict error for the duplicate UPN
3. Resolve it by changing the on-prem user's UPN or deleting the duplicate
4. Force another sync and verify the error is resolved

**Why this matters:** Sync troubleshooting is a daily activity for IAM admins in hybrid environments. Being able to create, identify, and resolve sync errors demonstrates practical experience.

**SC-300 exam tip:** Know common sync errors: UPN conflicts, invalid characters, attribute format mismatches. Know how to use the Synchronization Service Manager and the Entra Connect Health portal to diagnose issues.

---

## Step 11: Verify Everything

Checklist:

- [ ] Windows Server VM running with AD DS installed
- [ ] Domain `iamlabcorp.local` created
- [ ] OUs created for each department
- [ ] 15 on-prem users created across departments
- [ ] IdFix scanned and issues resolved
- [ ] UPN suffixes updated to match cloud domain
- [ ] Entra Connect installed with custom settings
- [ ] Password Hash Synchronization configured
- [ ] Seamless SSO enabled
- [ ] Initial sync completed — on-prem users visible in Entra portal
- [ ] Authentication decision matrix documented
- [ ] Entra Connect Sync vs Cloud Sync comparison documented
- [ ] Entra Connect Health reviewed
- [ ] Sync error created, diagnosed, and resolved
- [ ] All documentation and screenshots saved

---

## Key Takeaways for SC-300

1. **Entra Connect Sync** is the bridge between on-prem AD and Entra ID — install it on a dedicated server
2. **Password Hash Sync** is the recommended default authentication method — simplest, most reliable, enables ID Protection
3. **Pass-Through Auth** keeps passwords on-prem but requires agent infrastructure
4. **Seamless SSO** is an add-on to PHS or PTA for transparent sign-in from domain-joined devices
5. **IdFix** must be run before any sync deployment to prevent errors
6. **UPN suffixes** must match a verified cloud domain for proper identity matching
7. **Cloud Sync** is the newer, simpler alternative — better for basic scenarios and multi-forest
8. **Connect Health** monitors sync status and requires P1/P2

---

## What's Next

➡️ **Lab 5:** [Authentication Methods, MFA & SSPR](../lab05-authentication-mfa-sspr/) — Configure authentication methods, deploy MFA, set up self-service password reset, and implement password protection

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
