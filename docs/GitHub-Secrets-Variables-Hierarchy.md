# GitHub Secrets and Variables Hierarchy Guide

## 📚 Concept Overview

GitHub has **two main places** to store secrets/variables, with different scopes:

### 1️⃣ **Repository Level** (Settings → Secrets and Variables)
- **Repository Secrets** - Available to ALL workflows in the repo
- **Repository Variables** - Available to ALL workflows in the repo
- Not tied to any specific environment

### 2️⃣ **Environment Level** (Settings → Environments → [ENV_NAME])
- **Environment Secrets** - Only available when deploying to THAT environment
- **Environment Variables** - Only available when deploying to THAT environment
- Tied to specific environment (PROD, UAT, DEV)

---

## 🎯 Key Difference: Scope & Override

**Repository level** = Global, always available  
**Environment level** = Scoped, only when `environment:` is used in job

**Important:** Environment secrets/variables **override** repository ones with the same name!

---

## 📝 Simple Example

### Scenario: Deploy app to different servers

#### Repository Secrets (Settings → Secrets and Variables → Actions → Secrets)
```
APP_NAME = "my-app"
BUILD_VERSION = "1.0.0"
COMMON_API_KEY = "abc123"
```
☑️ These are available in **ALL jobs** (CI and CD)

#### Repository Variables (Settings → Secrets and Variables → Actions → Variables)
```
LOG_LEVEL = "info"
TIMEOUT = "30"
```
☑️ These are available in **ALL jobs**

---

#### Environment: PROD (Settings → Environments → PROD)
**Environment Secrets:**
```
DEPLOY_SERVER = "prod.myapp.com"
DATABASE_PASSWORD = "prod_secret_pass"
AWS_ACCESS_KEY = "prod_aws_key"
```

**Environment Variables:**
```
DB_HOST = "prod-db.myapp.com"
REGION = "us-east-1"
```
☑️ These are **ONLY available in CD job when `environment: PROD`**

---

#### Environment: UAT (Settings → Environments → UAT)
**Environment Secrets:**
```
DEPLOY_SERVER = "uat.myapp.com"
DATABASE_PASSWORD = "uat_secret_pass"
AWS_ACCESS_KEY = "uat_aws_key"
```

**Environment Variables:**
```
DB_HOST = "uat-db.myapp.com"
REGION = "us-west-1"
```
☑️ These are **ONLY available in CD job when `environment: UAT`**

---

#### Environment: DEV (Settings → Environments → DEV)
**Environment Secrets:**
```
DEPLOY_SERVER = "dev.myapp.com"
DATABASE_PASSWORD = "dev_secret_pass"
AWS_ACCESS_KEY = "dev_aws_key"
```

**Environment Variables:**
```
DB_HOST = "dev-db.myapp.com"
REGION = "eu-west-1"
```
☑️ These are **ONLY available in CD job when `environment: DEV`**

---

## 🔄 How They Work Together in Your Workflow

```yaml
jobs:
  CI:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        env:
          # ✅ Can access Repository secrets/variables
          APP_NAME: ${{ vars.APP_NAME }}
          LOG_LEVEL: ${{ vars.LOG_LEVEL }}
          API_KEY: ${{ secrets.COMMON_API_KEY }}
          
          # ❌ CANNOT access Environment secrets/variables
          # DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}  # Not available!

  CD:
    runs-on: ubuntu-latest
    environment: PROD  # 🔑 This unlocks PROD secrets/variables
    steps:
      - name: Deploy
        env:
          # ✅ Can access Repository secrets/variables
          APP_NAME: ${{ vars.APP_NAME }}
          
          # ✅ Can access PROD Environment secrets/variables
          DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}  # "prod.myapp.com"
          DB_HOST: ${{ vars.DB_HOST }}  # "prod-db.myapp.com"
          DB_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}  # "prod_secret_pass"
```

---

## 🎪 Override Example

**Repository Variable:**
```
REGION = "default-region"
```

**PROD Environment Variable:**
```
REGION = "us-east-1"
```

**In your workflow:**
```yaml
CD:
  environment: PROD
  steps:
    - run: echo ${{ vars.REGION }}
      # Outputs: "us-east-1" (Environment overrides Repository!)
```

Without `environment: PROD`:
```yaml
CI:
  steps:
    - run: echo ${{ vars.REGION }}
      # Outputs: "default-region" (Uses Repository variable)
```

---

## 🎯 Your Use Case: Branch → Environment → Secrets/Variables

Your workflow already maps branches to environments:
```yaml
CD:
  environment: ${{ github.ref == 'refs/heads/main' && 'PROD' || ... }}
```

This automatically gives you the right secrets/variables!

**When you push to `main`:**
- `environment: PROD` is set
- CD job gets `PROD` environment secrets/variables

**When you push to `qa`:**
- `environment: UAT` is set
- CD job gets `UAT` environment secrets/variables

**When you push to `dev`:**
- `environment: DEV` is set
- CD job gets `DEV` environment secrets/variables

---

## 📊 Visual Hierarchy

```
Repository (supreme-chainsaw)
│
├─ Repository Secrets/Variables ──────────┐
│  ├─ APP_NAME                            │
│  ├─ BUILD_VERSION                       │
│  ├─ LOG_LEVEL                           │
│  ├─ TIMEOUT                             │
│  └─ COMMON_API_KEY                      │
│                                          │
└─ Environments                            │
   │                                       │
   ├─ PROD ─────────────────────────┐     │
   │  ├─ DEPLOY_SERVER              │     │
   │  ├─ DATABASE_PASSWORD           │     │
   │  ├─ AWS_ACCESS_KEY              │     │
   │  ├─ DB_HOST                     │     │
   │  └─ REGION                      │     │
   │                                 │     │
   ├─ UAT ──────────────────────────┤     │
   │  ├─ DEPLOY_SERVER              │     │
   │  ├─ DATABASE_PASSWORD           │     │
   │  ├─ AWS_ACCESS_KEY              │     │
   │  ├─ DB_HOST                     │     │
   │  └─ REGION                      │     │
   │                                 │     │
   └─ DEV ──────────────────────────┘     │
      ├─ DEPLOY_SERVER                    │
      ├─ DATABASE_PASSWORD                 │
      ├─ AWS_ACCESS_KEY                    │
      ├─ DB_HOST                           │
      └─ REGION                            │
                                           │
Workflow Jobs:                             │
                                           │
┌────────────────────────┐                │
│  CI Job                │                │
│  (no environment set)  │◄───────────────┘
│                        │  Can access only
│  Available:            │  Repository level
│  - APP_NAME            │
│  - BUILD_VERSION       │
│  - LOG_LEVEL           │
│  - TIMEOUT             │
│  - COMMON_API_KEY      │
└────────────────────────┘

┌────────────────────────┐
│  CD Job                │
│  environment: PROD     │◄─────┬─ Repository level
│                        │      └─ PROD Environment
│  Available:            │
│  - APP_NAME            │ (from Repository)
│  - BUILD_VERSION       │ (from Repository)
│  - LOG_LEVEL           │ (from Repository)
│  - TIMEOUT             │ (from Repository)
│  - COMMON_API_KEY      │ (from Repository)
│  - DEPLOY_SERVER       │ (from PROD)
│  - DATABASE_PASSWORD   │ (from PROD)
│  - AWS_ACCESS_KEY      │ (from PROD)
│  - DB_HOST             │ (from PROD)
│  - REGION              │ (from PROD)
└────────────────────────┘
```

---

## 🔍 Scope Resolution Rules

When a job tries to access `${{ secrets.DEPLOY_SERVER }}`:

1. **Check if job has `environment:` set**
   - ❌ No → Use Repository-level secret (if exists)
   - ✅ Yes → Continue to step 2

2. **Check Environment-level secrets**
   - ✅ Found → Use Environment secret
   - ❌ Not found → Fallback to Repository-level secret

**Example:**
```yaml
# Repository secret
secrets.DEPLOY_SERVER = "default.myapp.com"

# PROD environment secret
secrets.DEPLOY_SERVER = "prod.myapp.com"

jobs:
  CI:
    steps:
      - run: echo ${{ secrets.DEPLOY_SERVER }}
        # Output: "default.myapp.com"
  
  CD:
    environment: PROD
    steps:
      - run: echo ${{ secrets.DEPLOY_SERVER }}
        # Output: "prod.myapp.com" (Environment overrides!)
```

---

## ✅ Best Practices

| Type | Use For | Example |
|------|---------|---------|
| **Repository Secrets** | Common secrets for all envs | `GITHUB_TOKEN`, `DOCKER_USERNAME`, `NPM_TOKEN` |
| **Repository Variables** | Common config for all envs | `APP_NAME`, `BUILD_TIMEOUT`, `NODE_VERSION` |
| **Environment Secrets** | Env-specific sensitive data | `DATABASE_PASSWORD`, `AWS_SECRET_KEY`, `API_TOKEN` |
| **Environment Variables** | Env-specific config | `API_URL`, `DB_HOST`, `REGION`, `CDN_URL` |

### 🎯 Decision Tree

```
Is the value sensitive (password, token, key)?
│
├─ YES → Use Secret
│   │
│   └─ Is it different per environment?
│       │
│       ├─ YES → Environment Secret
│       └─ NO  → Repository Secret
│
└─ NO → Use Variable
    │
    └─ Is it different per environment?
        │
        ├─ YES → Environment Variable
        └─ NO  → Repository Variable
```

---

## 🚀 Quick Setup Guide

### Step 1: Set Repository-level (for CI and common stuff)

1. Go to: **Settings → Secrets and Variables → Actions**
2. Click **Variables** tab
3. Add variables:
   ```
   APP_NAME = "my-app"
   BUILD_VERSION = "1.0.0"
   LOG_LEVEL = "info"
   TIMEOUT = "30"
   ```
4. Click **Secrets** tab
5. Add secrets:
   ```
   COMMON_API_KEY = "abc123xyz"
   DOCKER_PASSWORD = "secret123"
   ```

### Step 2: Set Environment-level (for CD deployment)

#### For PROD:
1. Go to: **Settings → Environments**
2. Create environment: **PROD**
3. Add environment variables:
   ```
   DB_HOST = "prod-db.myapp.com"
   REGION = "us-east-1"
   ```
4. Add environment secrets:
   ```
   DEPLOY_SERVER = "prod.myapp.com"
   DATABASE_PASSWORD = "prod_secret_pass"
   AWS_ACCESS_KEY = "prod_aws_key"
   ```
5. (Optional) Configure protection rules:
   - ☑️ Required reviewers
   - ☑️ Wait timer
   - ☑️ Allowed branches: `main`

#### For UAT:
1. Create environment: **UAT**
2. Add variables: `DB_HOST`, `REGION` (with UAT values)
3. Add secrets: `DEPLOY_SERVER`, `DATABASE_PASSWORD`, `AWS_ACCESS_KEY` (with UAT values)

#### For DEV:
1. Create environment: **DEV**
2. Add variables: `DB_HOST`, `REGION` (with DEV values)
3. Add secrets: `DEPLOY_SERVER`, `DATABASE_PASSWORD`, `AWS_ACCESS_KEY` (with DEV values)

### Step 3: Use in workflow

```yaml
jobs:
  CI:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        env:
          APP_NAME: ${{ vars.APP_NAME }}
          BUILD_VERSION: ${{ vars.BUILD_VERSION }}
          LOG_LEVEL: ${{ vars.LOG_LEVEL }}
        run: |
          echo "Building $APP_NAME version $BUILD_VERSION"
          # Build logic here

  CD:
    needs: CI
    environment: ${{ needs.CI.outputs.target-env }}  # Dynamic: PROD, UAT, or DEV
    steps:
      - name: Deploy
        env:
          # Repository-level (always available)
          APP_NAME: ${{ vars.APP_NAME }}
          
          # Environment-level (specific to PROD/UAT/DEV)
          DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}
          DB_HOST: ${{ vars.DB_HOST }}
          REGION: ${{ vars.REGION }}
          DB_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
        run: |
          echo "Deploying to $DEPLOY_SERVER"
          echo "Region: $REGION"
          echo "Database: $DB_HOST"
          # Deployment logic here
```

---

## 🛡️ Security Considerations

### Repository Secrets/Variables
- ✅ Accessible to all workflows in the repository
- ⚠️ Anyone with write access can read variables (not secrets)
- ⚠️ Anyone with write access can create workflows that expose secrets

### Environment Secrets/Variables
- ✅ Scoped to specific environments
- ✅ Can require approval before access
- ✅ Can restrict to specific branches
- ✅ Better audit trail (who approved what)
- ✅ More secure for production credentials

### Best Practice: Least Privilege
```yaml
# ❌ BAD: Giving all workflows access to production secrets
Repository Secret: PROD_DATABASE_PASSWORD

# ✅ GOOD: Restricting production secrets to PROD environment
PROD Environment Secret: DATABASE_PASSWORD
  - Required reviewers: 2
  - Allowed branches: main only
  - Wait timer: 5 minutes
```

---

## 🔧 Troubleshooting

### Problem: "Secret not found"

**Symptom:**
```yaml
CD:
  steps:
    - run: echo ${{ secrets.DEPLOY_SERVER }}
      # Output: (empty)
```

**Solution:**
Check if:
1. ✅ Job has `environment:` set
2. ✅ Secret exists in that environment
3. ✅ Secret name matches exactly (case-sensitive)
4. ✅ Environment name matches exactly

---

### Problem: "Getting wrong value"

**Symptom:**
```yaml
CD:
  environment: PROD
  steps:
    - run: echo ${{ vars.REGION }}
      # Expected: "us-east-1"
      # Got: "default-region"
```

**Solution:**
- Environment variable might not exist → falls back to Repository variable
- Check: Settings → Environments → PROD → Variables
- Add missing environment variable

---

### Problem: "Can't access environment secret in CI"

**Symptom:**
```yaml
CI:
  steps:
    - run: echo ${{ secrets.DEPLOY_SERVER }}
      # Output: (empty or repository value)
```

**Explanation:**
- ✅ **This is expected behavior!**
- Environment secrets are **only** available when `environment:` is set
- CI typically doesn't need deployment credentials

**Solution:**
- Move secret access to CD job
- Or add `environment:` to CI job (usually not recommended)

---

## 📚 Real-World Example

### Scenario: Multi-stage deployment pipeline

```yaml
name: Deploy Pipeline

jobs:
  CI:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
      target-env: ${{ github.ref_name == 'main' && 'PROD' || 'DEV' }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Get version
        id: version
        run: echo "value=1.0.${{ github.run_number }}" >> $GITHUB_OUTPUT
      
      - name: Build
        env:
          # Repository variables (common config)
          APP_NAME: ${{ vars.APP_NAME }}
          NODE_VERSION: ${{ vars.NODE_VERSION }}
          
          # Repository secrets (common credentials)
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: |
          echo "Building $APP_NAME with Node $NODE_VERSION"
          npm config set //registry.npmjs.org/:_authToken=$NPM_TOKEN
          npm install
          npm run build
      
      - uses: actions/upload-artifact@v4
        with:
          name: app-${{ steps.version.outputs.value }}
          path: dist/

  Deploy-DEV:
    needs: CI
    if: github.ref_name == 'dev'
    environment: DEV
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-${{ needs.CI.outputs.version }}
      
      - name: Deploy to DEV
        env:
          # Repository variables (still available)
          APP_NAME: ${{ vars.APP_NAME }}
          
          # DEV environment variables
          API_URL: ${{ vars.API_URL }}           # https://dev-api.myapp.com
          CDN_URL: ${{ vars.CDN_URL }}           # https://dev-cdn.myapp.com
          DB_HOST: ${{ vars.DB_HOST }}           # dev-db.myapp.com
          
          # DEV environment secrets
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          AWS_KEY: ${{ secrets.AWS_ACCESS_KEY }}
          DB_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
        run: |
          echo "Deploying $APP_NAME to DEV"
          echo "API: $API_URL"
          echo "CDN: $CDN_URL"
          ./deploy.sh --env=dev --token=$DEPLOY_TOKEN

  Approval-PROD:
    needs: CI
    if: github.ref_name == 'main'
    environment: PROD-Approval
    runs-on: ubuntu-latest
    steps:
      - run: echo "Waiting for approval..."

  Deploy-PROD:
    needs: [CI, Approval-PROD]
    environment: PROD
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-${{ needs.CI.outputs.version }}
      
      - name: Deploy to PROD
        env:
          # Repository variables (still available)
          APP_NAME: ${{ vars.APP_NAME }}
          
          # PROD environment variables (DIFFERENT from DEV!)
          API_URL: ${{ vars.API_URL }}           # https://api.myapp.com
          CDN_URL: ${{ vars.CDN_URL }}           # https://cdn.myapp.com
          DB_HOST: ${{ vars.DB_HOST }}           # prod-db.myapp.com
          
          # PROD environment secrets (DIFFERENT from DEV!)
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          AWS_KEY: ${{ secrets.AWS_ACCESS_KEY }}
          DB_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
        run: |
          echo "Deploying $APP_NAME to PROD"
          echo "API: $API_URL"
          echo "CDN: $CDN_URL"
          ./deploy.sh --env=prod --token=$DEPLOY_TOKEN
```

---

## 📖 Summary

| Aspect | Repository Level | Environment Level |
|--------|------------------|-------------------|
| **Scope** | All jobs | Only jobs with `environment:` set |
| **Override** | Can be overridden by environment | Overrides repository |
| **Setup Location** | Settings → Secrets and Variables → Actions | Settings → Environments → [NAME] |
| **Use Case** | Common config, CI tasks | Deployment, env-specific config |
| **Protection** | Basic | Advanced (approvals, branch restrictions) |
| **Visibility** | All workflows | Specific environment workflows |

**Key Takeaway:**
- Use **Repository** for common stuff (CI, builds)
- Use **Environment** for deployment stuff (CD, env-specific)
- Environment values **override** repository values with same name
- Jobs without `environment:` can only see repository-level

---

**Document Version:** 1.0  
**Last Updated:** December 10, 2025  
**Related Files:**
- `.github/workflows/cicd_workflow_testing.yml`
- `docs/CICD-Workflow-Documentation.md`
- `docs/GitHub-Actions-Variable-Access-Guide.md`
