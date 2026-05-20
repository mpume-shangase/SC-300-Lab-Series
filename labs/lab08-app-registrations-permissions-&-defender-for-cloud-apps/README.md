# Lab 8: App Registrations, Permissions & Defender for Cloud Apps

> **Type:** Focused Lab | **Time:** 2-3 hours | **SC-300 Domain:** Plan and implement workload identities (20-25%)

---

## Scenario

The development team at Expat Teacher's Lounge is building a custom internal application that needs to authenticate users and access Microsoft Graph data. Meanwhile, the security team has discovered that employees are using over 50 unauthorized cloud applications (shadow IT). You need to register the custom app properly, configure its permissions, and set up Defender for Cloud Apps to discover and control shadow IT.

---

## SC-300 Exam Objectives Covered

- Plan for app registrations
- Create app registrations
- Configure app authentication (redirect URIs, credentials)
- Configure API permissions (delegated vs application)
- Create app roles
- Design and implement integration for on-prem apps (Application Proxy)
- Configure and analyze cloud discovery results (Defender for Cloud Apps)
- Configure connected apps
- Implement application-enforced restrictions
- Configure Conditional Access app control
- Create access and session policies in Defender for Cloud Apps
- Implement and manage policies for OAuth apps
- Manage the Cloud app catalog

---

## Prerequisites

- Completed Labs 1-7
- Microsoft 365 E5 subscription (includes Defender for Cloud Apps)
- Certificate-based auth configured from previous labs

---

## Step 1: Plan and Create an App Registration

**Where:** Entra ID → App registrations → + New registration

### 1a: Register the Application

1. Click **+ New registration**
2. **Name:** `ETL Resource Manager` (ETL = Expat Teacher's Lounge)
3. **Supported account types:** Accounts in this organizational directory only (Single tenant)
4. **Redirect URI:**
   - Platform: **Web**
   - URI: `https://localhost:3000/auth/callback` (typical for a dev app)
5. Click **Register**

Save these values from the Overview page:
- **Application (client) ID**
- **Directory (tenant) ID**

### 1b: Add Additional Redirect URIs

1. Go to **Authentication** in the left menu
2. Under **Web**, click **Add URI**
3. Add: `https://etl-app.expatteacherslounge.com/auth/callback` (production URL)
4. Under **Implicit grant and hybrid flows:**
   - Check **ID tokens** (for OIDC sign-in)
   - Leave **Access tokens** unchecked (use authorization code flow instead)
5. Under **Advanced settings:**
   - **Allow public client flows:** No (this is a web app, not a mobile/desktop app)
6. Click **Save**

**SC-300 exam tip:** Know the different redirect URI platforms (Web, SPA, Mobile/Desktop) and when to use each. Know that implicit grant is legacy — authorization code flow with PKCE is recommended for modern apps.

---

## Step 2: Configure API Permissions

**Where:** App registration → API permissions

### 2a: Add Delegated Permissions

Delegated permissions act on behalf of a signed-in user:

1. Click **+ Add a permission** → **Microsoft Graph** → **Delegated permissions**
2. Add:
   - `User.Read` — Sign in and read user profile
   - `User.ReadBasic.All` — Read basic profiles of all users
   - `Group.Read.All` — Read all groups
   - `Calendars.Read` — Read user's calendar
3. Click **Add permissions**

### 2b: Add Application Permissions

Application permissions act as the app itself (no user context):

1. Click **+ Add a permission** → **Microsoft Graph** → **Application permissions**
2. Add:
   - `User.Read.All` — Read all user profiles
   - `Reports.Read.All` — Read usage reports
3. Click **Add permissions**
4. Click **Grant admin consent for Expat Teacher's Lounge**
5. Verify green checkmarks

### 2c: Document the Difference

Create `docs/api-permissions-guide.md`:

| Feature | Delegated Permissions | Application Permissions |
|---|---|---|
| User context | Yes — acts as the signed-in user | No — acts as the app itself |
| Consent | User consent or admin consent | Admin consent only |
| Token type | Access token with user identity | Access token with app identity |
| Use case | Web apps where a user is signed in | Background services, daemons, automation |
| Scope limitation | Limited by both the permission AND the user's own access | Full permission scope — not limited by any user |
| Example | "Read my calendar" | "Read all users' profiles" |
| Risk level | Lower — bounded by user access | Higher — can access everything the permission allows |

**SC-300 exam tip:** This is one of the most heavily tested topics. Know that delegated permissions are intersection of the permission AND the user's access. Application permissions have NO user context and access everything the permission allows. Know that application permissions ALWAYS require admin consent.

### How to explain this in an interview:

*"I configure API permissions based on the application's needs. For web apps where a user is signed in, I use delegated permissions because they're bounded by the user's own access level — even if the permission allows reading all users, the result is limited to what that specific user can see. For background automation and service-to-service communication, I use application permissions with admin consent, and I apply the principle of least privilege to request only the minimum permissions needed."*

---

## Step 3: Configure App Credentials

**Where:** App registration → Certificates & secrets

### 3a: Create a Client Secret

1. Click **Certificates & secrets** → **Client secrets** → **+ New client secret**
2. **Description:** `Dev environment secret`
3. **Expires:** 6 months (choose the shortest reasonable period)
4. Click **Add**
5. **Copy the secret value immediately** — you can't see it again

### 3b: Upload a Certificate

1. Click **Certificates** tab → **Upload certificate**
2. Upload a certificate (reuse your IAM-Lifecycle-Cert.pem or create a new one)
3. Click **Add**

### 3c: Document Why Certificates Are Preferred

In your app registration documentation, note:

```markdown
## Credential Security
- Client secrets are convenient for development but risky for production
- Secrets can be accidentally committed to source control or shared in messages
- Certificates are more secure — the private key stays on the machine
- Production recommendation: ALWAYS use certificates for production workloads
- Rotate credentials before expiry — set calendar reminders
```

**SC-300 exam tip:** Know both credential types. Know that certificates are recommended for production. Know the maximum secret lifetime (24 months). Know that federated credentials (workload identity federation) are the newest option and eliminate secrets entirely for supported scenarios.

---

## Step 4: Create App Roles

**Where:** App registration → App roles

1. Click **+ Create app role**
2. Create these roles:

| Display Name | Value | Allowed Types | Description |
|---|---|---|---|
| Administrator | `App.Admin` | Users/Groups | Full application administration |
| Resource Manager | `App.ResourceManager` | Users/Groups | Can create, edit, and delete resources |
| Viewer | `App.Viewer` | Users/Groups | Read-only access to resources |
| API Reader | `App.APIReader` | Applications | Service-to-service read access |

3. For each role, fill in the fields and click **Apply**

Assign users to roles via the Enterprise Application:
1. Go to **Enterprise apps** → find `ETL Resource Manager`
2. **Users and groups** → **+ Add user/group**
3. Select a user → select the `Resource Manager` role → **Assign**

**SC-300 exam tip:** Know that app roles for Users/Groups appear in the `roles` claim of the ID/access token. App roles for Applications appear in the `roles` claim of the client credentials token. Know that roles are defined in the app registration but ASSIGNED in the enterprise application.

---

## Step 5: Configure Application Proxy (Conceptual)

Application Proxy publishes on-premises web applications through Entra ID — users access them from the internet with SSO and CA protection without VPN.

**Where:** Entra ID → Enterprise apps → Application Proxy

### Architecture Overview

```
User (Internet) → Entra ID → Application Proxy Cloud Service → 
    Application Proxy Connector (on-prem) → Internal Web App
```

### Configuration Steps (Document Even If Not Fully Implementing)

1. **Install the Proxy Connector** on an on-premises server:
   - Download from the Application Proxy page in Entra portal
   - Install on a Windows Server with outbound HTTPS access
   - The connector establishes an outbound connection to the cloud service — no inbound firewall ports needed

2. **Publish an Internal Application:**
   - Click **+ Add application** → **Configure an on-prem app**
   - **Internal URL:** `http://internalapp.iamlabcorp.local:8080`
   - **External URL:** `https://etl-internal.msappproxy.net` (auto-generated)
   - **Pre-authentication:** Microsoft Entra ID (users must authenticate via Entra before reaching the app)
   - **Connector group:** Default

3. **Configure SSO for the Published App:**
   - Under the app's SSO settings, choose the appropriate method:
     - **Integrated Windows Auth** — for Kerberos-based apps
     - **Header-based** — for apps that read identity from HTTP headers
     - **Password-based** — for apps with traditional login forms

Create `docs/application-proxy-architecture.md` documenting this flow.

**SC-300 exam tip:** Know the Application Proxy architecture. Key points: the connector establishes OUTBOUND connections (no inbound firewall rules needed), pre-authentication happens at Entra ID (so CA policies and MFA apply BEFORE the user reaches the internal app), and no VPN is required.

---

## Step 6: Configure Defender for Cloud Apps

**Where:** security.microsoft.com → Cloud apps (or portal.cloudappsecurity.com)

### 6a: Access the MDCA Portal

1. Go to **security.microsoft.com**
2. Navigate to **Cloud apps** in the left sidebar
3. Or go directly to **portal.cloudappsecurity.com**

### 6b: Review Cloud Discovery

1. Go to **Cloud Discovery** → **Dashboard**
2. Review discovered apps — MDCA analyzes traffic logs to identify which cloud apps are being used
3. Look for:
   - **Sanctioned apps** — approved by your organization
   - **Unsanctioned apps** — not approved (shadow IT)
   - **Risk scores** — how risky each app is based on security, compliance, and legal factors

### 6c: Connect Microsoft 365

1. Go to **Settings** → **Connected apps** → **App connectors**
2. Click **+ Connect an app** → select **Microsoft 365**
3. Follow the prompts to authorize the connection
4. Once connected, MDCA can monitor activity within Microsoft 365 apps

### 6d: Create an Activity Policy

1. Go to **Policies** → **Policy management** → **+ Create policy** → **Activity policy**
2. **Name:** `Alert on Mass File Download`
3. **Filter:**
   - Activity type: Download file
   - Repeated activity: More than 10 times in 5 minutes by the same user
4. **Alert:** Create an alert with high severity
5. **Governance action:** Suspend user (optional — start with alert only)
6. Click **Create**

### 6e: Create a Session Policy

1. Go to **Policies** → **+ Create policy** → **Session policy**
2. **Name:** `Block Download from Unmanaged Devices`
3. **Session control type:** Control file download (with inspection)
4. **Filter:**
   - App: SharePoint Online
   - Device tag: Does not equal Compliant (or Corporate)
5. **Action:** Block download
6. Click **Create**

### 6f: Review OAuth Apps

1. Go to **Cloud Discovery** → **OAuth apps** (or **App governance**)
2. Review all OAuth apps that users have consented to
3. Look for apps with excessive permissions (e.g., "Read and write all files" for a simple tool)
4. Flag or revoke suspicious apps

### 6g: Manage the Cloud App Catalog

1. Go to **Cloud app catalog**
2. Search for a common app (e.g., Dropbox, Zoom)
3. Review its risk score and the factors (security, compliance, legal)
4. Click **Sanctioned** or **Unsanctioned** to tag the app
5. Unsanctioned apps can be blocked through integration with your firewall/proxy

**SC-300 exam tip:** Know the MDCA components: Cloud Discovery (shadow IT), Connected Apps (monitoring sanctioned apps), Policies (activity, session, app discovery, OAuth), and the Cloud App Catalog (risk scoring). Know that session policies require Conditional Access App Control integration.

### How to explain this in an interview:

*"I deployed Defender for Cloud Apps to address shadow IT. Cloud Discovery identified over 50 unauthorized cloud apps employees were using. I created activity policies to alert on suspicious behavior like mass file downloads, and session policies to block downloads from unmanaged devices. The OAuth app review identified third-party apps with excessive permissions that we revoked. This gave us visibility into cloud usage that we didn't have before."*

---

## Step 7: Verify Everything

Checklist:

- [ ] App registration created (ETL Resource Manager)
- [ ] Redirect URIs configured (localhost and production)
- [ ] Delegated permissions added and documented
- [ ] Application permissions added with admin consent
- [ ] Delegated vs Application permissions comparison documented
- [ ] Client secret created (for dev)
- [ ] Certificate uploaded (for production)
- [ ] Credential security documentation written
- [ ] App roles created (Admin, Resource Manager, Viewer, API Reader)
- [ ] Users assigned to app roles
- [ ] Application Proxy architecture documented
- [ ] MDCA portal accessed and explored
- [ ] Cloud Discovery dashboard reviewed
- [ ] Microsoft 365 connected as app connector
- [ ] Activity policy created (mass download alert)
- [ ] Session policy created (block download from unmanaged devices)
- [ ] OAuth apps reviewed
- [ ] Cloud app catalog explored with apps tagged

---

## Key Takeaways for SC-300

1. **Delegated permissions** = user context, bounded by user access. **Application permissions** = app context, full scope
2. **Certificates over secrets** for production credentials
3. **App roles** provide application-level RBAC included in tokens
4. **Application Proxy** publishes on-prem apps without VPN — connector makes outbound connections only
5. **MDCA** is your visibility layer for shadow IT and cloud app security
6. **Session policies** require CA integration and control what users can do WITHIN an app
7. **OAuth app review** catches third-party apps with excessive permissions

---

## What's Next

➡️ **Lab 9:** [Identity Governance: Entitlements, Access Reviews & PIM](../lab09-identity-governance/) — The complete governance framework for audit readiness

---

*Part of the [SC-300 Lab Series](../../). Follow along on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).*
