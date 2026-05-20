# 🔐 SC-300 Lab Series: Microsoft Identity & Access Administrator

> Hands-on labs covering every SC-300 exam objective. Built on Microsoft 365 E5 with Microsoft Entra ID. Follow along and build each lab yourself.

[![SC-300](https://img.shields.io/badge/SC--300-Identity_&_Access_Admin-0078D4?style=flat-square&logo=microsoft&logoColor=white)](https://learn.microsoft.com/en-us/credentials/certifications/identity-and-access-administrator/)
[![Entra ID](https://img.shields.io/badge/Microsoft_Entra_ID-Identity-00A4EF?style=flat-square&logo=microsoft-azure&logoColor=white)](https://entra.microsoft.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

---

## About This Series

This is a 10-lab series mapped directly to the official SC-300 exam objectives (updated April 2026). Each lab is a real-world scenario that teaches you the concepts through hands-on configuration, not just theory. By the time you complete all 10 labs, you'll have practiced every skill the exam tests.

All labs use a Microsoft 365 E5 subscription with Microsoft Entra ID P1/P2.

📺 **Each lab is documented as a tutorial on [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d).**

---

## Lab Overview

| Lab | Title | Type | Time | SC-300 Domain |
|-----|-------|------|------|---------------|
| [Lab 1](labs/lab01-tenant-config/) | Tenant Configuration & Role Management | Focused Lab | 2-3 hrs | Domain 1: User Identities |
| [Lab 2](labs/lab02-user-group-lifecycle/) | User & Group Lifecycle with PowerShell | Big Project | 3-4 hrs | Domain 1: User Identities |
| [Lab 3](labs/lab03-external-identities/) | External Identities & Cross-Tenant Access | Focused Lab | 2-3 hrs | Domain 1: User Identities |
| [Lab 4](labs/lab04-hybrid-identity/) | Hybrid Identity with Entra Connect | Big Project | 4-5 hrs | Domain 1: User Identities |
| [Lab 5](labs/lab05-authentication-mfa-sspr/) | Authentication Methods, MFA & SSPR | Focused Lab | 2-3 hrs | Domain 2: Auth & Access |
| [Lab 6](labs/lab06-conditional-access/) | Conditional Access & ID Protection | Big Project | 3-4 hrs | Domain 2: Auth & Access |
| [Lab 7](labs/lab07-workload-identities/) | Workload Identities & Enterprise Apps | Big Project | 3-4 hrs | Domain 3: Workload Identities |
| [Lab 8](labs/lab08-app-registrations/) | App Registrations, Permissions & Defender for Cloud Apps | Focused Lab | 2-3 hrs | Domain 3: Workload Identities |
| [Lab 9](labs/lab09-identity-governance/) | Identity Governance: Entitlements, Access Reviews & PIM | Big Project | 4-5 hrs | Domain 4: Identity Governance |
| [Lab 10](labs/lab10-monitoring-logs/) | Monitoring, Logs & Identity Secure Score | Focused Lab | 2-3 hrs | Domain 4: Identity Governance |

---

## SC-300 Exam Domains

| Domain | Weight | Labs |
|--------|--------|------|
| Implement and manage user identities | 20-25% | Labs 1, 2, 3, 4 |
| Implement authentication and access management | 25-30% | Labs 5, 6 |
| Plan and implement workload identities | 20-25% | Labs 7, 8 |
| Plan and automate identity governance | 20-25% | Labs 9, 10 |

---

## Prerequisites

| Requirement | Details |
|------------|---------|
| **Microsoft 365 E5 Subscription** | Includes Entra ID P2 and all required features |
| **Global Administrator access** | Admin account in your Entra ID tenant |
| **PowerShell 7+** | [Download here](https://github.com/PowerShell/PowerShell) |
| **Microsoft Graph SDK** | `Install-Module Microsoft.Graph -Scope CurrentUser` |
| **A computer** | Windows, Mac, or Linux |

---

## How to Use This Repo

Each lab has its own folder under `labs/` with:

- **README.md** — Full step-by-step instructions
- **screenshots/** — Configuration screenshots for reference
- **scripts/** — Any PowerShell scripts used in the lab
- **docs/** — Supporting documentation (SOPs, diagrams, decision matrices)

Work through the labs in order — each one builds on the previous. Take screenshots as you go for your own documentation.

---

## Study Schedule

| Week | Labs | Focus |
|------|------|-------|
| Week 1 | Labs 1-2 | Tenant config, users, groups, PowerShell |
| Week 2 | Labs 3-4 | External identities, hybrid identity |
| Week 3 | Labs 5-6 | Authentication, MFA, SSPR, Conditional Access |
| Week 4 | Labs 7-8 | Enterprise apps, app registrations, MDCA |
| Week 5 | Labs 9-10 | Governance, PIM, access reviews, monitoring |
| Week 6-7 | Review | Practice exams, revisit weak areas |
| Week 8 | EXAM | Take the SC-300 |

---

## Related Projects

| Project | Description |
|---------|-------------|
| [IAM Lifecycle Automation](https://github.com/mpume-shangase/IAM-Lifecycle-Automation) | Automated JML pipeline with PowerShell & Graph API — demonstrates scripting and automation skills beyond portal configuration |

---

## Connect

📺 [The Cyber Chronicles](https://www.youtube.com/@TheCyberChronicles-l9d) — Video tutorials for each lab

💼 [LinkedIn](https://www.linkedin.com/in/mpume-shangase/)

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

*Built as part of my SC-300 certification journey. If you're studying for the exam, fork this repo and build along with me.*
