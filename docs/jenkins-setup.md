# Jenkins Setup Guide

This document describes how to configure Jenkins to run the two pipelines that power the ALM process for this project.

## Prerequisites 

### Jenkins Service Account

#### Create local service account for Jenkins using PowerShell

```powershell
$password = ConvertTo-SecureString "YourStrongPassword123!" -AsPlainText -Force
New-LocalUser -Name "jenkins-svc" -Password $password -PasswordNeverExpires -UserMayNotChangePassword -Description "Jenkins service account"
```

#### Grant log on as service
1. Sign in with administrator privileges to the computer from which you want to provide Log on as Service permission to accounts.

2. Go to Administrative Tools and select Local Security Policy.

3. Expand Local Policy and select User Rights Assignment. In the right pane, right-click Log on as a service and select Properties.

4. Select Add User or Group option to add the new user.

5. In the Select Users or Groups dialog, find the user you wish to add and select OK.

6. Select OK in the Log on as a service Properties to save the changes.

See: [learn.microsoft.com — Enable Log On as a Service](https://learn.microsoft.com/en-us/system-center/scsm/enable-service-log-on-sm?view=sc-sm-2025)

---

### Installations

- **Jenkins**: [Installation Guide](https://www.jenkins.io/doc/book/installing/windows/)
- **Power Platform CLI**: [Installation Guide](https://learn.microsoft.com/en-us/power-platform/developer/howto/install-cli-msi)
- **git**: [Download](https://git-scm.com/download/win)

---

### Configure Windows Agent Label

The pipelines use `agent { label 'windows' }` and will only run on a node tagged with that label.

1. Go to **Manage Jenkins → Nodes**
2. Click your node (typically **Built-In Node** for a single-machine setup)
3. Click **Configure** in the left sidebar
4. Find the **Labels** field and enter `windows`
5. Click **Save**

---

### Jenkins Plugins Required

Install via **Manage Jenkins → Plugins**:

| Plugin | Purpose |
|--------|---------|
| Pipeline | Core pipeline support |
| Credentials Binding | Inject secrets into shell steps |
| Timestamper | Adds timestamps to console output |
| Workspace Cleanup | `cleanWs()` support |
| Role-based Authorization Strategy | Optionally for Jenkins RBAC

---

## Jenkins Credentials

Configure all of the following under **Manage Jenkins → Credentials → Global**.

| Credential ID | Type | Value |
|--------------|------|-------|
| `dataverse-dev-url` | Secret text | `https://<dev-org>.crm.dynamics.com` |
| `dataverse-test-url` | Secret text | `https://<test-org>.crm.dynamics.com` |
| `dataverse-prod-url` | Secret text | `https://<prod-org>.crm.dynamics.com` |
| `dataverse-tenant-id` | Secret text | Azure AD Tenant ID (GUID) |
| `dataverse-client-id` | Secret text | App Registration Client ID (GUID) |
| `dataverse-client-secret` | Secret text | App Registration Client Secret |
| `github-pat` | Secret text | GitHub Personal Access Token (repo scope) |

> **Security note:** The service principal used in `CLIENT_ID` / `CLIENT_SECRET` should be granted only the **System Administrator** role in Dataverse environments via Azure AD app registration. It should not have broader Azure permissions.

---

## Creating the Azure AD App Registration

The pipelines authenticate to Dataverse using a Service Principal rather than a user account. To set this up:

1. Go to **Azure Portal → Azure Active Directory → App registrations → New registration**
2. Name it something neutral (e.g. `dataverse-ci-agent`)
3. Under **API permissions**, add: `Dynamics CRM → user_impersonation`
4. Create a **Client secret** and copy the value immediately
5. In each Dataverse environment, go to **Settings → Users → Application Users** and add the app registration as a System Administrator

---

## Job 1: Export Pipeline

**New Item → Pipeline → name it `solution-export`**

| Setting | Value |
|---------|-------|
| Pipeline definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | GitHub repo URL |
| Branch | `*/main` |
| Script Path | `jenkins/Jenkinsfile.export` |
| Build Triggers | *(none — manual only)* |

The **Build with Parameters** button will prompt for an optional commit message before running.

---

## Job 2: Deploy Pipeline

**New Item → Pipeline → name it `solution-deploy`**

| Setting | Value |
|---------|-------|
| Pipeline definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | GitHub repo URL |
| Branch | `*/main` |
| Script Path | `jenkins/Jenkinsfile.deploy` |
| Build Triggers | *(none — manual only)* |

Run via **Build with Parameters**. You will be prompted to choose:

- **`test`** — deploys Managed solution to Test only
- **`production`** — deploys Managed to Test, then presents an approval prompt before deploying Managed to Production

---

## Production Approval

The deploy pipeline pauses at the **Approve Production** stage and waits up to 24 hours. To approve:

1. Open the `solution-deploy` build in Jenkins
2. Click the **Approve & Deploy to Production** button (only visible to users in the `release-managers` group)
3. Enter release notes and confirm the backup check
4. The Managed solution will then be imported to Production

To configure the `release-managers` group go to **Manage Jenkins → Manage and Assign Roles** (requires Role Strategy Plugin).

---

## Typical Workflow

```
Developer makes changes in Dev environment
          │
          ▼
Run "solution-export" job in Jenkins (manual)
  → Exports + unpacks solution from Dev
  → Commits changed XML to main
          │
          ▼
Run "solution-deploy" job in Jenkins (manual)
  with DEPLOY_TARGET = test
  → Validates XML
  → Packs Unmanaged + Managed zips
  → Deploys Managed to Test
          │
          │  (tester signs off on Test)
          ▼
Run "solution-deploy" job again
  with DEPLOY_TARGET = production
  → Validates + packs again from latest main
  → Deploys Managed to Test
  → Waits for approval (release manager approves in Jenkins UI)
  → Deploys Managed to Production
```
