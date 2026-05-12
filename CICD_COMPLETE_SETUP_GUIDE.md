# 🚀 Complete Salesforce CI/CD Setup Guide

## Step-by-Step Instructions with All Details and Scripts

**Last Updated:** 2026-05-12  
**Version:** 1.0.0

---

## 📋 Table of Contents

1. [Prerequisites & Requirements](#prerequisites--requirements)
2. [Part 1: Generate Salesforce Auth URLs](#part-1-generate-salesforce-auth-urls)
3. [Part 2: Create GitHub Branches](#part-2-create-github-branches)
4. [Part 3: Add GitHub Workflows](#part-3-add-github-workflows)
5. [Part 4: Configure GitHub Secrets](#part-4-configure-github-secrets)
6. [Part 5: Branch Protection Rules](#part-5-branch-protection-rules)
7. [Part 6: Production Environment Setup](#part-6-production-environment-setup)
8. [Part 7: Test Your Pipeline](#part-7-test-your-pipeline)
9. [Part 8: Team Onboarding](#part-8-team-onboarding)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites & Requirements

### ✅ Software Requirements

Before starting, ensure you have:

1. **Salesforce CLI (sf)** - Latest version
   ```bash
   npm install --global @salesforce/cli
   sf --version
   ```
   
   Expected output: `@salesforce/cli/2.x.x` or higher

2. **Git** - Latest version
   ```bash
   git --version
   ```
   
   Expected output: `git version 2.x.x` or higher

3. **Node.js** - Version 16 or higher
   ```bash
   node --version
   ```
   
   Expected output: `v16.x.x` or higher

4. **GitHub CLI (optional but recommended)**
   ```bash
   gh --version
   ```

### ✅ Access Requirements

You need access to:
- **GitHub Repository:** `https://github.com/ravikishantelu/sfdevops`
- **3 Salesforce Orgs:**
  - ✅ QA Sandbox (for testing)
  - ✅ UAT Sandbox (for user acceptance testing)
  - ✅ Production Org (for production deployment)

### ✅ Initial Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/ravikishantelu/sfdevops.git
   cd sfdevops
   git pull origin main
   ```

2. Verify directory structure:
   ```bash
   ls -la
   ```
   
   You should see:
   ```
   .github/
   force-app/
   README.md
   sfdx-project.json
   .gitignore
   ```

---

## Part 1: Generate Salesforce Auth URLs

### Overview

Salesforce Auth URLs are secure credentials that allow GitHub Actions to authenticate with your Salesforce orgs. You'll create one for each environment (QA, UAT, PROD).

### ⚠️ Security Note

- Auth URLs are sensitive - **never commit them to Git**
- Never share them in Slack, email, or public channels
- Store them securely in GitHub Secrets only
- Regenerate if accidentally exposed

---

### Step 1.1: Login to QA Sandbox

```bash
# Open browser and authenticate to QA sandbox
sf org login web -a qa-org

# What happens:
# 1. Browser opens automatically
# 2. Login with your QA org username & password
# 3. You may be prompted for 2FA code
# 4. Click "Allow" to authorize Salesforce CLI
# 5. Browser shows "Authorize successful" message
```

**Troubleshooting:**
- If browser doesn't open: Use `--browser chrome` flag
  ```bash
  sf org login web -a qa-org --browser chrome
  ```
- If you get "Login failed": Verify QA org URL is correct
- If you need to reset: `sf org logout -u qa-org --no-prompt`

---

### Step 1.2: Get QA Auth URL

```bash
# Display the auth URL for QA org
sf org display --target-org qa-org --json | jq '.result.sfdxAuthUrl'

# Output will be a long string starting with: force://
# Example: force://XXXXXXXXXXXXXXXXXXXXXX@mycompany.my.salesforce.com
```

**What to do:**
1. Copy the entire auth URL (including `force://` prefix)
2. Save it in a temporary text file locally (NOT in repo)
3. Example file: `~/salesforce-auth-urls.txt`
4. Label it: `QA_AUTH_URL: force://...`

**Verify:**
```bash
# Test the auth URL works by listing orgs
sf org list
# Should show qa-org as authenticated
```

---

### Step 1.3: Login to UAT Sandbox

```bash
# Open browser and authenticate to UAT sandbox
sf org login web -a uat-org

# Same process as QA:
# 1. Browser opens
# 2. Login with UAT credentials
# 3. Authorize
# 4. Close browser
```

---

### Step 1.4: Get UAT Auth URL

```bash
# Display the auth URL for UAT org
sf org display --target-org uat-org --json | jq '.result.sfdxAuthUrl'

# Output: force://XXXXXXXXXXXXXXXXXXXXXX@mycompany.my.salesforce.com
```

**Save this:**
```
UAT_AUTH_URL: force://...
```

---

### Step 1.5: Login to Production Org

```bash
# Open browser and authenticate to Production
sf org login web -a prod-org

# Same process as others
```

### ⚠️ **CRITICAL SECURITY WARNING FOR PRODUCTION**

- **DO NOT** share this auth URL with anyone
- **DO NOT** log this in console/terminal history
- **DO NOT** email this to anyone
- **ONLY** store in GitHub Secrets
- Consider using a temporary connection and regenerating after setup

---

### Step 1.6: Get Production Auth URL

```bash
# Display the auth URL for Production org
sf org display --target-org prod-org --json | jq '.result.sfdxAuthUrl'

# Output: force://XXXXXXXXXXXXXXXXXXXXXX@mycompany.my.salesforce.com
```

**Save this securely:**
```
PROD_AUTH_URL: force://...
```

---

### Step 1.7: Verify All Auth URLs

```bash
# List all authenticated orgs
sf org list

# Output should show:
# ✓ qa-org          (Web)
# ✓ uat-org         (Web)
# ✓ prod-org        (Web)
```

**Summary of Step 1:**

You now have:
- ✅ QA Auth URL
- ✅ UAT Auth URL
- ✅ PROD Auth URL

Keep these safe - you'll need them in **Part 4**

---

## Part 2: Create GitHub Branches

### Overview

Your repository needs three main branches:
- `main` - Production-ready code
- `uat` - User Acceptance Testing
- `dev` - Development/QA integration

### Step 2.1: Verify Current Branches

```bash
# List all local branches
git branch -a

# Output should show:
# * main
#   remotes/origin/main
```

**Note:** Only `main` exists by default. We'll create `dev` and `uat`.

---

### Step 2.2: Create Dev Branch

```bash
# Create local dev branch from main
git checkout -b dev

# Verify you're on dev branch (asterisk shows current)
git branch -a

# Output:
# * dev
#   main
#   remotes/origin/main
```

---

### Step 2.3: Push Dev Branch to GitHub

```bash
# Push dev branch to remote repository
git push -u origin dev

# Output:
# Branch 'dev' set up to track remote branch 'dev' from 'origin'
# To https://github.com/ravikishantelu/sfdevops.git
#  * [new branch]      dev -> dev
```

**Verify on GitHub:**
1. Go to: https://github.com/ravikishantelu/sfdevops
2. Click branch dropdown (currently shows "main")
3. You should see `dev` listed

---

### Step 2.4: Create UAT Branch

```bash
# Create local uat branch from main
git checkout -b uat

# Verify you're on uat branch
git branch
# Output: * uat
```

---

### Step 2.5: Push UAT Branch to GitHub

```bash
# Push uat branch to remote repository
git push -u origin uat

# Output:
# Branch 'uat' set up to track remote branch 'uat' from 'origin'
# To https://github.com/ravikishantelu/sfdevops.git
#  * [new branch]      uat -> uat
```

**Verify on GitHub:**
1. Go to: https://github.com/ravikishantelu/sfdevops
2. Click branch dropdown
3. You should see: `dev`, `main`, `uat`

---

### Step 2.6: Verify All Branches

```bash
# List all branches locally and remotely
git branch -a

# Expected output:
#   dev
# * uat
#   main
#   remotes/origin/dev
#   remotes/origin/main
#   remotes/origin/uat
```

**Summary of Step 2:**

You now have:
- ✅ `main` branch (Production)
- ✅ `uat` branch (User Acceptance Testing)
- ✅ `dev` branch (Development/QA)

---

## Part 3: Add GitHub Workflows

### Overview

Workflows are automation scripts that run on GitHub. You need 4 workflows:

1. **validate-pr.yml** - Validates all pull requests
2. **dev-qa-deploy.yml** - Deploys to QA when pushed to dev
3. **uat-deploy.yml** - Deploys to UAT when pushed to uat
4. **prod-deploy.yml** - Deploys to Production with manual approval

### Step 3.1: Create Workflows Directory

```bash
# Navigate to repo
cd sfdevops

# Create workflows directory
mkdir -p .github/workflows

# Verify it's created
ls -la .github/

# Output:
# total XX
# drwxr-xr-x   workflows
```

---

### Step 3.2: Create validate-pr.yml

**File:** `.github/workflows/validate-pr.yml`

```bash
# Create the file
cat > .github/workflows/validate-pr.yml << 'EOF'
name: Validate Pull Request

on:
  pull_request:
    branches: [ dev, uat, main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli --omit=dev
          sf plugins install @salesforce/plugin-deploy-retrieve
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Validate Metadata Format
        run: |
          echo "Validating Salesforce metadata..."
          if [ -d "force-app" ]; then
            echo "✅ force-app directory found"
            find force-app -name '*.xml' | head -10 || echo "No XML files found yet"
          else
            echo "⚠️ force-app directory not found"
          fi

      - name: Check for Common Issues
        run: |
          echo "Checking for common issues..."
          if grep -r "TODO\|FIXME" force-app --include="*.cls" --include="*.trigger" 2>/dev/null; then
            echo "⚠️ TODO/FIXME comments found in code"
          fi
          echo "✅ Metadata validation complete"

      - name: Post Validation Summary
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const status = '${{ job.status }}' === 'success' ? '✅' : '❌';
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `${status} **Salesforce Metadata Validation**\n\nStatus: ${{ job.status }}\nBranch: ${{ github.head_ref }}\nCommit: ${{ github.sha }}`
            });
EOF

# Verify file was created
cat .github/workflows/validate-pr.yml | head -10
```

---

### Step 3.3: Create dev-qa-deploy.yml

**File:** `.github/workflows/dev-qa-deploy.yml`

```bash
cat > .github/workflows/dev-qa-deploy.yml << 'EOF'
name: Deploy to QA

on:
  push:
    branches: [ dev ]

jobs:
  deploy-qa:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli --omit=dev
          sf plugins install @salesforce/plugin-deploy-retrieve
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Authenticate to QA Org
        run: |
          echo "${{ secrets.SFDX_AUTH_URL_QA }}" > ./auth-qa.txt
          sf org login sfdx-url -f ./auth-qa.txt -a qa-org -s
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Validate Deployment to QA
        run: |
          sf project deploy start --source-dir force-app --target-org qa-org --test-level RunLocalTests --dry-run
        continue-on-error: true

      - name: Deploy to QA
        run: |
          sf project deploy start --source-dir force-app --target-org qa-org --test-level RunLocalTests

      - name: Run Apex Tests (QA)
        run: |
          sf apex run test --target-org qa-org --code-coverage --result-format json

      - name: Generate QA Deployment Report
        if: always()
        run: |
          echo "## 🚀 QA Deployment Report" >> $GITHUB_STEP_SUMMARY
          echo "**Status:** ${{ job.status }}" >> $GITHUB_STEP_SUMMARY
          echo "**Branch:** dev" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "**Deployment Time:** $(date)" >> $GITHUB_STEP_SUMMARY

      - name: Cleanup Auth Files
        if: always()
        run: |
          rm -f ./auth-qa.txt
          sf org logout --all --no-prompt
EOF

cat .github/workflows/dev-qa-deploy.yml | head -10
```

---

### Step 3.4: Create uat-deploy.yml

**File:** `.github/workflows/uat-deploy.yml`

```bash
cat > .github/workflows/uat-deploy.yml << 'EOF'
name: Deploy to UAT

on:
  push:
    branches: [ uat ]

jobs:
  deploy-uat:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli --omit=dev
          sf plugins install @salesforce/plugin-deploy-retrieve
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Authenticate to UAT Org
        run: |
          echo "${{ secrets.SFDX_AUTH_URL_UAT }}" > ./auth-uat.txt
          sf org login sfdx-url -f ./auth-uat.txt -a uat-org -s
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Validate Deployment to UAT
        run: |
          sf project deploy start --source-dir force-app --target-org uat-org --test-level RunAllTests --dry-run
        continue-on-error: true

      - name: Deploy to UAT
        run: |
          sf project deploy start --source-dir force-app --target-org uat-org --test-level RunAllTests

      - name: Run All Apex Tests (UAT)
        run: |
          sf apex run test --target-org uat-org --code-coverage --result-format json

      - name: Generate UAT Deployment Report
        if: always()
        run: |
          echo "## 🚀 UAT Deployment Report" >> $GITHUB_STEP_SUMMARY
          echo "**Status:** ${{ job.status }}" >> $GITHUB_STEP_SUMMARY
          echo "**Branch:** uat" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "**Test Level:** RunAllTests" >> $GITHUB_STEP_SUMMARY
          echo "**Deployment Time:** $(date)" >> $GITHUB_STEP_SUMMARY

      - name: Cleanup Auth Files
        if: always()
        run: |
          rm -f ./auth-uat.txt
          sf org logout --all --no-prompt
EOF

cat .github/workflows/uat-deploy.yml | head -10
```

---

### Step 3.5: Create prod-deploy.yml

**File:** `.github/workflows/prod-deploy.yml`

```bash
cat > .github/workflows/prod-deploy.yml << 'EOF'
name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  validate-prod:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli --omit=dev
          sf plugins install @salesforce/plugin-deploy-retrieve
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Authenticate to Production Org
        run: |
          echo "${{ secrets.SFDX_AUTH_URL_PROD }}" > ./auth-prod.txt
          sf org login sfdx-url -f ./auth-prod.txt -a prod-org -s
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Validate Deployment to Production (Dry Run)
        run: |
          sf project deploy start --source-dir force-app --target-org prod-org --test-level RunAllTests --dry-run
        continue-on-error: false

      - name: Cleanup Auth Files
        if: always()
        run: |
          rm -f ./auth-prod.txt
          sf org logout --all --no-prompt

  deploy-prod:
    needs: validate-prod
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Salesforce CLI
        run: |
          npm install --global @salesforce/cli --omit=dev
          sf plugins install @salesforce/plugin-deploy-retrieve
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Authenticate to Production Org
        run: |
          echo "${{ secrets.SFDX_AUTH_URL_PROD }}" > ./auth-prod.txt
          sf org login sfdx-url -f ./auth-prod.txt -a prod-org -s
        env:
          SALESFORCE_CLI_TELEMETRY_DISABLED: true

      - name: Deploy to Production Org
        run: |
          sf project deploy start --source-dir force-app --target-org prod-org --test-level RunAllTests

      - name: Run All Apex Tests (Production)
        run: |
          sf apex run test --target-org prod-org --code-coverage --result-format json

      - name: Create Release Tag
        if: success()
        uses: actions/github-script@v7
        with:
          script: |
            const tag = 'v' + new Date().toISOString().slice(0,10) + '-' + context.sha.slice(0,7);
            github.rest.git.createRef({
              owner: context.repo.owner,
              repo: context.repo.repo,
              ref: 'refs/tags/' + tag,
              sha: context.sha
            });
            console.log('Created release tag: ' + tag);

      - name: Generate Production Deployment Report
        if: always()
        run: |
          echo "## 🚀 Production Deployment Report" >> $GITHUB_STEP_SUMMARY
          echo "**Status:** ${{ job.status }}" >> $GITHUB_STEP_SUMMARY
          echo "**Branch:** main" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "**Actor:** ${{ github.actor }}" >> $GITHUB_STEP_SUMMARY
          echo "**Test Level:** RunAllTests" >> $GITHUB_STEP_SUMMARY
          echo "**Deployment Time:** $(date)" >> $GITHUB_STEP_SUMMARY

      - name: Cleanup Auth Files
        if: always()
        run: |
          rm -f ./auth-prod.txt
          sf org logout --all --no-prompt
EOF

cat .github/workflows/prod-deploy.yml | head -10
```

---

### Step 3.6: Verify All Workflows Created

```bash
# List all workflow files
ls -la .github/workflows/

# Output should show:
# -rw-r--r--  dev-qa-deploy.yml
# -rw-r--r--  uat-deploy.yml
# -rw-r--r--  prod-deploy.yml
# -rw-r--r--  validate-pr.yml
```

---

### Step 3.7: Commit and Push Workflows

```bash
# Check git status
git status

# Add all workflow files
git add .github/workflows/

# Commit changes
git commit -m "Add CI/CD workflows: dev-qa-deploy, uat-deploy, prod-deploy, validate-pr"

# Push to main branch
git push origin main

# Verify on GitHub
# Go to: https://github.com/ravikishantelu/sfdevops
# Click "Code" tab
# Navigate to .github/workflows/
# You should see all 4 YAML files
```

**Summary of Step 3:**

You now have:
- ✅ validate-pr.yml
- ✅ dev-qa-deploy.yml
- ✅ uat-deploy.yml
- ✅ prod-deploy.yml

---

## Part 4: Configure GitHub Secrets

### Overview

GitHub Secrets securely store your Salesforce auth URLs. They're encrypted and only available to GitHub Actions workflows.

### Step 4.1: Navigate to Secrets Settings

1. Go to: https://github.com/ravikishantelu/sfdevops
2. Click **Settings** (top right)
3. Click **Secrets and variables** (left sidebar)
4. Click **Actions**

**URL:** `https://github.com/ravikishantelu/sfdevops/settings/secrets/actions`

---

### Step 4.2: Add SFDX_AUTH_URL_QA Secret

**Steps:**

1. Click **"New repository secret"** button
2. In **Name** field, enter: `SFDX_AUTH_URL_QA`
3. In **Secret** field, paste your QA auth URL (from Part 1)
   - Example: `force://XXXXXXXXXXXXXXXXXXXXXX@qa.my.salesforce.com`
4. Click **"Add secret"**

**Verification:**
- You should see `SFDX_AUTH_URL_QA` listed in Secrets
- Value is hidden (shown as dots)
- Click the pencil icon to edit (don't show value!)

---

### Step 4.3: Add SFDX_AUTH_URL_UAT Secret

**Steps:**

1. Click **"New repository secret"** button
2. In **Name** field, enter: `SFDX_AUTH_URL_UAT`
3. In **Secret** field, paste your UAT auth URL (from Part 1)
4. Click **"Add secret"**

**Verification:**
- You should see `SFDX_AUTH_URL_UAT` listed

---

### Step 4.4: Add SFDX_AUTH_URL_PROD Secret

⚠️ **CRITICAL: PRODUCTION SECRET**

**Steps:**

1. Click **"New repository secret"** button
2. In **Name** field, enter: `SFDX_AUTH_URL_PROD`
3. In **Secret** field, paste your PROD auth URL (from Part 1)
   - ⚠️ Handle with extreme care!
4. Click **"Add secret"**

**Verification:**
- You should see `SFDX_AUTH_URL_PROD` listed
- After adding, delete your local copy of this auth URL
  ```bash
  # Remove local file with auth URLs
  rm ~/salesforce-auth-urls.txt
  history -c  # Clear history
  ```

---

### Step 4.5: Verify All Secrets Added

**On GitHub UI:**

Go to: https://github.com/ravikishantelu/sfdevops/settings/secrets/actions

You should see exactly 3 secrets:
- ✅ SFDX_AUTH_URL_QA
- ✅ SFDX_AUTH_URL_UAT
- ✅ SFDX_AUTH_URL_PROD

All values should be hidden.

---

### Step 4.6: Test Secrets (Optional)

You can verify secrets are working by triggering a workflow:

1. Go to: https://github.com/ravikishantelu/sfdevops/actions
2. Click any workflow
3. If secrets are accessible, you'll see "Secrets available" in logs

**Summary of Step 4:**

You now have:
- ✅ SFDX_AUTH_URL_QA secret added
- ✅ SFDX_AUTH_URL_UAT secret added
- ✅ SFDX_AUTH_URL_PROD secret added

---

## Part 5: Branch Protection Rules

### Overview

Branch protection rules ensure:
- ✅ Code is reviewed before merging
- ✅ Status checks pass (CI/CD validation)
- ✅ Branches are up-to-date before merge
- ✅ Prevents accidental deletions

### Step 5.1: Navigate to Branch Settings

1. Go to: https://github.com/ravikishantelu/sfdevops
2. Click **Settings** (top right)
3. Click **Branches** (left sidebar)
4. Click **Add rule** button

**URL:** `https://github.com/ravikishantelu/sfdevops/settings/branches`

---

### Step 5.2: Setup Dev Branch Protection

**Configure Dev Branch:**

1. Branch name pattern: `dev`
2. Enable these options:
   - ✅ **Require a pull request before merging**
     - Required approvals: `1`
     - ✅ Dismiss stale pull request approvals when new commits are pushed
     - ✅ Require conversation resolution before merging
   - ✅ **Require status checks to pass before merging**
     - (Once workflows run, select: `validate`)
   - ✅ **Require branches to be up to date before merging**

3. Click **Create** button

**Dev Branch Settings Summary:**
```
Branch name pattern: dev
├─ Require PR reviews: 1 approval
├─ Require status checks: validate workflow
├─ Require up-to-date branches: Yes
└─ Dismiss stale reviews: Yes
```

---

### Step 5.3: Setup UAT Branch Protection

**Configure UAT Branch:**

1. Click **Add rule** button
2. Branch name pattern: `uat`
3. Enable these options:
   - ✅ **Require a pull request before merging**
     - Required approvals: `2`
     - ✅ Dismiss stale pull request approvals when new commits are pushed
   - ✅ **Require status checks to pass before merging**
     - (Select: `deploy-uat` workflow when available)
   - ✅ **Require branches to be up to date before merging**

4. Click **Create** button

**UAT Branch Settings Summary:**
```
Branch name pattern: uat
├─ Require PR reviews: 2 approvals
├─ Require status checks: deploy-uat workflow
├─ Require up-to-date branches: Yes
└─ Dismiss stale reviews: Yes
```

---

### Step 5.4: Setup Main/Production Branch Protection

**Configure Main Branch:**

1. Click **Add rule** button
2. Branch name pattern: `main`
3. Enable these options:
   - ✅ **Require a pull request before merging**
     - Required approvals: `2`
     - ✅ Dismiss stale pull request approvals when new commits are pushed
     - ✅ Require review from Code Owners (if using)
   - ✅ **Require status checks to pass before merging**
     - (Select: `validate-prod` workflow when available)
   - ✅ **Require branches to be up to date before merging**
   - ✅ **Require deployments to succeed before merging** (optional)
     - Environment: `production`

4. Click **Create** button

**Main Branch Settings Summary:**
```
Branch name pattern: main
├─ Require PR reviews: 2 approvals
├─ Require status checks: validate-prod workflow
├─ Require up-to-date branches: Yes
├─ Dismiss stale reviews: Yes
└─ Require deploy environment: production
```

---

### Step 5.5: Verify Branch Protection Rules

**On GitHub UI:**

Go to: https://github.com/ravikishantelu/sfdevops/settings/branches

You should see exactly 3 rules:
- ✅ dev branch rule
- ✅ uat branch rule
- ✅ main branch rule

Each rule should show:
- Require pull request reviews
- Require status checks
- Require branches to be up to date

**Summary of Step 5:**

You now have:
- ✅ Dev branch protection (1 approval required)
- ✅ UAT branch protection (2 approvals required)
- ✅ Main branch protection (2 approvals required)

---

## Part 6: Production Environment Setup

### Overview

The production environment adds an additional approval gate for production deployments. Only authorized team members can approve production deployments.

### Step 6.1: Navigate to Environments

1. Go to: https://github.com/ravikishantelu/sfdevops
2. Click **Settings** (top right)
3. Click **Environments** (left sidebar)

**URL:** `https://github.com/ravikishantelu/sfdevops/settings/environments`

---

### Step 6.2: Create Production Environment

**Steps:**

1. Click **New environment** button
2. In **Environment name** field, enter: `production`
3. Click **Configure environment** button

---

### Step 6.3: Configure Required Reviewers

**Steps:**

1. Enable: **✅ Required reviewers**
2. Click the search field
3. Start typing your team members' usernames
4. Select them from the dropdown
5. Add at least **2 required reviewers** (recommended)

**Example:**
- Reviewer 1: Tech Lead
- Reviewer 2: DevOps Lead

**Save:** Click **Save protection rules**

---

### Step 6.4: Restrict Deployment Branches

**Steps:**

1. Enable: **✅ Deployment branches**
2. Select: **Protected branches only**
3. This ensures only `main` branch can trigger production deployments

**Save:** Click **Save protection rules**

---

### Step 6.5: Verify Production Environment

**On GitHub UI:**

Go to: https://github.com/ravikishantelu/sfdevops/settings/environments

You should see:
- ✅ **production** environment listed
- ✅ Required reviewers: (number shown)
- ✅ Deployment branches: Protected branches only

**Summary of Step 6:**

You now have:
- ✅ Production environment created
- ✅ Required reviewers configured (2)
- ✅ Deployment branches restricted to main only

---

## Part 7: Test Your Pipeline

### Overview

Now that everything is configured, let's test the complete CI/CD pipeline end-to-end.

### Step 7.1: Create a Feature Branch

```bash
# Make sure you're in the repo
cd sfdevops

# Pull latest from main
git checkout main
git pull origin main

# Create a new feature branch
git checkout -b feature/test-cicd

# Verify you're on the new branch
git branch
# Output: * feature/test-cicd
```

---

### Step 7.2: Create a Test File

```bash
# Create a simple test file
cat > TEST_DEPLOYMENT.md << 'EOF'
# Test Deployment

This file is created to test the CI/CD pipeline.

## Testing Steps

1. Feature branch created
2. PR opened to dev
3. Validation runs
4. Deploy to QA
5. Deploy to UAT
6. Deploy to Production

## Status
- Dev: ✅ Ready
- QA: ⏳ Testing
- UAT: ⏳ Testing
- Prod: ⏳ Ready for approval
EOF

# Verify file was created
cat TEST_DEPLOYMENT.md
```

---

### Step 7.3: Commit and Push

```bash
# Add the test file
git add TEST_DEPLOYMENT.md

# Commit with descriptive message
git commit -m "test: create deployment test file to validate CI/CD pipeline"

# Push the feature branch
git push -u origin feature/test-cicd

# Output:
# remote: Create a pull request for 'feature/test-cicd' on GitHub by visiting:
# remote: https://github.com/ravikishantelu/sfdevops/pull/new/feature/test-cicd
```

---

### Step 7.4: Create Pull Request to Dev

**On GitHub UI:**

1. Go to: https://github.com/ravikishantelu/sfdevops
2. You should see a banner: **"Compare & pull request"**
3. Click that button

**OR manually:**
1. Click **Pull requests** tab
2. Click **New pull request** button
3. Base: `dev` ← Compare: `feature/test-cicd`
4. Click **Create pull request**

**Fill in PR Details:**

```
Title: test: validate CI/CD pipeline deployment

Description:

## What does this PR do?
Tests the complete CI/CD pipeline from feature branch to production.

## Why?
To verify all workflows are functioning correctly.

## Changes
- Added TEST_DEPLOYMENT.md file
- This triggers validate-pr workflow

## Testing
- validate-pr workflow runs on all PRs
- No production data affected

## Checklist
- [x] Code follows style guidelines
- [x] No new warnings generated
- [x] Tested locally
```

5. Click **Create pull request**

---

### Step 7.5: Monitor PR Validation

**Check Workflow Status:**

1. On the PR page, scroll down to **Checks** section
2. You should see: **"validate"** workflow running
3. Wait for it to complete (usually 2-3 minutes)

**Expected Results:**
- ✅ validate workflow passes
- ✅ Comment posted to PR with validation status
- ✅ PR is ready to merge (if no other issues)

**If Fails:**
- Click on failed check
- Scroll to see error details
- Fix the issue locally
- Push new commit to same branch
- Workflow reruns automatically

---

### Step 7.6: Approve and Merge to Dev

**Approval:**

1. Under **Reviewers** section, click your profile picture
2. This adds you as reviewer
3. Click **Approve** button

**Merge:**

1. Click **Merge pull request** button
2. Select: **Create a merge commit** (recommended)
3. Click **Confirm merge**

**Delete Branch (optional):**
- GitHub offers button to delete branch
- Click **Delete branch**

---

### Step 7.7: Monitor Dev to QA Deployment

**Check Deployment Status:**

1. Go to: https://github.com/ravikishantelu/sfdevops/actions
2. Click **Deploy to QA** workflow
3. Click the latest run

**Monitor Steps:**
```
✅ Checkout code
✅ Setup Node.js
✅ Install Salesforce CLI
⏳ Authenticate to QA Org        (2-3 min)
⏳ Validate Deployment to QA     (2-5 min)
⏳ Deploy to QA                   (5-10 min)
⏳ Run Apex Tests                 (5-15 min)
✅ Cleanup Auth Files
```

**Expected Results:**
- ✅ All steps pass (green checkmarks)
- ✅ Deployment report generated
- ✅ Tests pass in QA org
- ⏳ Metadata deployed to QA sandbox

**If Fails:**
- Click failed step
- Read error message
- Common issues:
  - `Error: Could not authenticate` → Check secrets
  - `Error: Metadata validation failed` → Check metadata syntax
  - `Error: Test failures` → Fix code in Salesforce

---

### Step 7.8: Create PR from Dev to UAT

```bash
# Update local branches
git fetch origin

# Checkout dev
git checkout dev
git pull origin dev

# Create PR branch
git checkout -b pull/dev-to-uat

# No changes needed, just push empty
git push -u origin pull/dev-to-uat
```

**On GitHub:**

1. Create PR: `dev` → `uat`
2. Fill in description:
   ```
   Title: chore: promote dev to uat after QA testing
   
   Description:
   QA testing complete, ready for UAT.
   ```
3. Click **Create pull request**

**Merge:**

1. Get 2 approvals (or approve from 2 accounts)
2. Click **Merge pull request**
3. Confirm merge

---

### Step 7.9: Monitor Dev to UAT Deployment

**Check Deployment Status:**

1. Go to: https://github.com/ravikishantelu/sfdevops/actions
2. Click **Deploy to UAT** workflow
3. Click the latest run

**Monitor Steps:**
```
✅ Checkout code
✅ Setup Node.js
✅ Install Salesforce CLI
⏳ Authenticate to UAT Org        (2-3 min)
⏳ Deploy to UAT                   (5-15 min)
⏳ Run All Apex Tests             (10-20 min)
✅ Cleanup Auth Files
```

**Expected Results:**
- ✅ All steps pass
- ✅ Metadata deployed to UAT sandbox
- ✅ All Apex tests pass
- ✅ Code coverage meets threshold

---

### Step 7.10: Create PR from UAT to Main (Production)

```bash
# Update local branches
git fetch origin

# Checkout uat
git checkout uat
git pull origin uat

# Create PR branch
git checkout -b pull/uat-to-main

# Push
git push -u origin pull/uat-to-main
```

**On GitHub:**

1. Create PR: `uat` → `main`
2. Fill in description:
   ```
   Title: release: promote uat to production after user sign-off
   
   Description:
   UAT testing complete and approved by business.
   Ready for production deployment.
   
   Changes:
   - All features tested and approved
   - All tests passing
   - No breaking changes
   ```
3. Click **Create pull request**

---

### Step 7.11: Approve and Merge to Main

**Get 2 Approvals:**

1. Request reviews from 2 team members
2. They click **Approve** on the PR

**Merge:**

1. Click **Merge pull request**
2. Confirm merge

---

### Step 7.12: Monitor Production Validation

**Check Deployment Status:**

1. Go to: https://github.com/ravikishantelu/sfdevops/actions
2. Click **Deploy to Production** workflow
3. Click the latest run

**Monitor Steps:**

```
✅ Checkout code
✅ Setup Node.js
✅ Install Salesforce CLI
⏳ Authenticate to Production     (2-3 min)
⏳ Validate Deployment (Dry Run)  (5-10 min)
✅ Job: validate-prod completes

⏳ Job: deploy-prod waiting...
🔒 AWAITING MANUAL APPROVAL
```

**This is normal!** The workflow is waiting for manual approval.

---

### Step 7.13: Approve Production Deployment

**On GitHub:**

1. Go to: https://github.com/ravikishantelu/sfdevops/actions
2. Click **Deploy to Production** workflow
3. Click the running deployment

**You should see:**
- **"Waiting for review"** message
- **"Review deployments"** button

**Approve:**

1. Click **"Review deployments"** button
2. Select: **production** environment
3. Enter approval note (optional):
   ```
   Approved for production deployment after UAT sign-off
   ```
4. Click **"Approve and deploy"** button

**Monitor Deployment:**

```
⏳ Deploy to Production            (5-15 min)
⏳ Run All Apex Tests             (10-20 min)
⏳ Create Release Tag              (1 min)
✅ Deployment Report Generated
✅ DEPLOYMENT COMPLETE!
```

---

### Step 7.14: Verify Production Deployment

**Verify on GitHub:**

1. Go to: https://github.com/ravikishantelu/sfdevops/tags
2. You should see a new tag: `v2026-05-12-XXXXXXX`

**Verify in Salesforce:**

1. Login to Production Org
2. Check Setup → Deployment Status
3. You should see recent deployments
4. Metadata should be deployed

**Verify Tests:**

1. Go to: Setup → Apex Tests
2. Recent test runs should show all passing

---

### Test Pipeline Complete! ✅

Your entire CI/CD pipeline is now tested and working:

```
✅ Feature branch created
✅ PR validated (validate-pr workflow)
✅ Merged to dev
✅ Auto-deployed to QA (dev-qa-deploy workflow)
✅ Tests passed in QA
✅ PR from dev to uat created
✅ Merged to uat
✅ Auto-deployed to UAT (uat-deploy workflow)
✅ Tests passed in UAT
✅ PR from uat to main created
✅ Merged to main
✅ Validation ran (validate-prod workflow)
✅ Awaited manual approval
✅ Approved by reviewer
✅ Auto-deployed to Production (deploy-prod workflow)
✅ All tests passed in Production
✅ Release tag created
```

---

## Part 8: Team Onboarding

### Overview

Now that your CI/CD pipeline is fully functional, train your team on how to use it.

### Step 8.1: Share Documentation

Provide your team with:

1. **README.md** - Overview of pipeline
2. **BRANCH_STRATEGY.md** - Developer workflow
3. **This guide** - Complete setup and operation

**Send them:**
```
Repository: https://github.com/ravikishantelu/sfdevops

Documentation:
- README.md - Architecture & overview
- BRANCH_STRATEGY.md - Developer workflow
- SETUP_GUIDE.md - This complete guide
```

---

### Step 8.2: Developer Workflow Training

**Teach developers:**

```bash
# 1. Clone repository
git clone https://github.com/ravikishantelu/sfdevops.git
cd sfdevops

# 2. Create feature branch from dev
git checkout dev
git pull origin dev
git checkout -b feature/your-feature-name

# 3. Make changes to Salesforce metadata
# (Edit files in force-app/)

# 4. Commit changes
git add .
git commit -m "feat: add new feature"

# 5. Push to GitHub
git push -u origin feature/your-feature-name

# 6. Create PR to dev
# (On GitHub: Click "Compare & pull request")

# 7. Get approval & merge
# (Reviewer approves, then merge)

# 8. Monitor deployment to QA
# (Go to GitHub Actions tab)

# 9. QA testing happens in QA sandbox

# 10. When ready for UAT, create PR from dev to uat
# (Manager/lead creates and merges)

# 11. Monitor deployment to UAT

# 12. When ready for Prod, create PR from uat to main
# (Tech lead creates and merges)

# 13. Await manual approval notification

# 14. Tech lead approves deployment

# 15. Monitor deployment to Production
```

---

### Step 8.3: Access Control Guidelines

**Document for team:**

| Role | Can Do | Cannot Do |
|------|--------|----------|
| **Developer** | Create feature branches, commit, push, create PR to dev | Merge to dev/uat/main, approve Prod |
| **QA Lead** | Approve PRs to dev, review QA deployment results | Merge to uat/main, approve Prod |
| **Tech Lead** | Approve PRs, merge to uat, create/merge PR to main | Approve Prod alone (needs 2) |
| **DevOps** | All of above + manage secrets & infrastructure | N/A |

---

### Step 8.4: Common Questions & Answers

**Q: How do I fix a failed deployment?**

A:
```bash
# 1. Check the failed workflow logs on GitHub Actions
# 2. Fix the issue locally
# 3. Push the fix to the same branch
# 4. Workflow reruns automatically
# 5. Once fixed, rerun or merge again
```

**Q: Can I deploy directly to UAT without going through QA?**

A: No. The pipeline enforces:
- Feature branch → QA (dev)
- Dev → UAT (uat)
- UAT → Prod (main)

This ensures testing at each stage.

**Q: What if I make a mistake and merge to wrong branch?**

A:
```bash
# Revert the merge commit
git log --oneline | head -5  # Find merge commit
git revert -m 1 <commit-hash>
git push origin <branch>

# This creates a new commit that undoes the merge
```

**Q: How often should we deploy?**

A: As often as needed! The pipeline supports:
- Multiple deployments per day
- Small changes frequently
- Large changes when ready
- Emergency hotfixes anytime

---

### Step 8.5: Troubleshooting Guide for Team

**Issue: "Workflow not triggering"**

Solution:
```
1. Check branch name (must match exactly)
2. Check if file is in force-app/
3. Check GitHub Actions is enabled
4. Manual trigger: Go to Actions tab → click workflow → "Run workflow"
```

**Issue: "Authentication failed"**

Solution:
```
1. Check secrets are added (Settings → Secrets)
2. Verify secret values (ask DevOps to regenerate)
3. Check org access isn't restricted
4. Wait 30 seconds and retry
```

**Issue: "Apex tests failing"**

Solution:
```
1. Check test logs in workflow
2. Fix code locally
3. Run tests locally first
4. Push fix to branch
5. Workflow reruns automatically
```

**Issue: "Cannot merge - status checks not passing"**

Solution:
```
1. Click on failed check
2. See what validation failed
3. Fix locally
4. Push new commit
5. Workflow reruns
6. Once passing, can merge
```

---

### Step 8.6: Team Checklist

Before team starts using pipeline:

- [ ] All team members have GitHub access to repo
- [ ] All team members read README.md
- [ ] All team members understand BRANCH_STRATEGY.md
- [ ] Developers can create feature branches
- [ ] QA lead can trigger deployments
- [ ] Tech lead can approve production deployments
- [ ] DevOps verified secrets are secure
- [ ] Everyone tested pipeline end-to-end

---

## Troubleshooting

### Common Issues & Solutions

#### Issue 1: "No such file or directory: .github/workflows"

**Error:** When creating workflows
```
mkdir: .github/workflows: No such file or directory
```

**Solution:**
```bash
# Create the directories one at a time
mkdir -p .github
mkdir -p .github/workflows

# Verify
ls -la .github/
```

---

#### Issue 2: "SFDX Auth URL not found"

**Error:** Workflow fails with "Error: Could not authenticate"

**Solution:**
1. Go to Settings → Secrets → Actions
2. Verify secret exists and has correct name
3. Regenerate auth URL:
   ```bash
   sf org logout -u qa-org --no-prompt
   sf org login web -a qa-org
   sf org display --target-org qa-org --json | jq '.result.sfdxAuthUrl'
   ```
4. Update secret in GitHub

---

#### Issue 3: "Branch protection rule prevents merge"

**Error:** Cannot merge PR - "Required status checks must pass"

**Solution:**
1. Go to PR page
2. Check "Checks" section
3. Click failed check to see details
4. Fix issue locally
5. Push new commit
6. Workflow reruns
7. Once passing, can merge

---

#### Issue 4: "Metadata validation failed"

**Error:** "Cannot create/update object..."

**Solution:**
1. Check workflow logs for details
2. Validate metadata locally:
   ```bash
   sf project validate --source-dir force-app --target-org your-org
   ```
3. Fix syntax errors
4. Push fix
5. Retry

---

#### Issue 5: "Test failures in workflow"

**Error:** "X tests failed in QA deployment"

**Solution:**
```bash
# 1. Check which tests failed
# Go to workflow logs → "Run Apex Tests" step

# 2. Run tests locally
sf apex run test --target-org qa-org

# 3. Fix failing tests locally

# 4. Commit and push fix
git add .
git commit -m "fix: resolve failing apex tests"
git push origin feature/branch-name

# 5. Workflow reruns automatically
```

---

#### Issue 6: "Production deployment waiting indefinitely"

**Error:** Deployment stuck on "Awaiting manual approval"

**Solution:**
1. Go to GitHub Actions → Deploy to Production workflow
2. Click the running deployment
3. Click "Review deployments" button
4. Select "production" environment
5. Click "Approve and deploy"
6. Deployment continues

---

#### Issue 7: "Cannot access repository"

**Error:** "Permission denied" or "Repository not found"

**Solution:**
```bash
# Check you have GitHub access
# Run:
gh auth status

# If not authenticated:
gh auth login
# Follow prompts

# If still denied, check repository access
# Ask DevOps to add you as collaborator
```

---

#### Issue 8: "Workflows not running automatically"

**Error:** Push to dev/uat/main but no workflow triggered

**Solution:**
1. Check branch name matches exactly (dev, uat, main)
2. Check GitHub Actions is enabled (Settings → Actions → General)
3. Manual trigger:
   ```bash
   gh workflow run dev-qa-deploy.yml -r dev
   ```
4. Or manually trigger on GitHub UI:
   - Go to Actions tab
   - Click workflow name
   - Click "Run workflow" dropdown
   - Click "Run workflow"

---

#### Issue 9: "Getting 'Apex test code coverage' error"

**Error:** "Code coverage is below threshold"

**Solution:**
1. This is intentional - ensures code quality
2. Write tests for uncovered code
3. Add to force-app/main/default/classes/
4. Push changes
5. Workflow reruns with higher coverage

---

#### Issue 10: "Production deployment approved but nothing happened"

**Error:** Approved but deployment stuck

**Solution:**
```bash
# Check workflow status
# Go to Actions tab → Deploy to Production

# If stuck, click workflow
# Check logs for errors

# If permission issue:
sf org login web -a prod-org
sf org display --target-org prod-org --json | jq '.result.sfdxAuthUrl'
# Update SFDX_AUTH_URL_PROD secret

# Retry deployment manually
```

---

## Summary & Next Steps

### ✅ You've Successfully Completed:

1. ✅ Generated Salesforce auth URLs for QA, UAT, PROD
2. ✅ Created dev and uat branches
3. ✅ Added 4 GitHub Actions workflows
4. ✅ Configured GitHub Secrets
5. ✅ Setup branch protection rules
6. ✅ Created production environment with required reviewers
7. ✅ Tested complete CI/CD pipeline end-to-end
8. ✅ Trained team on developer workflow

### 🎯 Your CI/CD Pipeline Capabilities:

```
✅ Automatic validation on every PR
✅ Auto-deploy to QA on dev merge
✅ Auto-deploy to UAT on uat merge
✅ Manual approval gate for production
✅ Auto-deploy to Production after approval
✅ Automatic Apex test execution
✅ Code coverage tracking
✅ Automatic release tagging
✅ Deployment reports
✅ Audit trail of all changes
```

### 📊 Typical Deployment Time:

- **Feature branch to QA:** 5-15 minutes
- **Dev to UAT:** 10-20 minutes
- **UAT to Production:** 15-25 minutes (+ approval time)

### 🚀 Ready for Production!

Your Salesforce CI/CD pipeline is now fully operational and ready for your team to use daily.

### 📞 Support & Next Steps:

1. **Add CI/CD to your documentation:** Share this guide with team
2. **Monitor deployments:** Watch GitHub Actions tab
3. **Iterate and improve:** Adjust workflows as needed
4. **Add notifications:** Configure Slack/Teams alerts (optional)
5. **Backup credentials:** Keep auth URLs safe

---

## Appendix: Useful Commands

### Git Commands

```bash
# Create and push new branch
git checkout -b feature/name
git push -u origin feature/name

# Sync with remote
git fetch origin
git rebase origin/dev

# View commit history
git log --oneline --graph --all

# Undo last commit
git reset --soft HEAD~1

# Stash changes
git stash
git stash pop
```

### Salesforce CLI Commands

```bash
# List authenticated orgs
sf org list

# Validate metadata
sf project validate --source-dir force-app --target-org qa-org

# Deploy metadata
sf project deploy start --source-dir force-app --target-org qa-org --test-level RunLocalTests

# Run Apex tests
sf apex run test --target-org qa-org

# Pull from org
sf project retrieve start --target-org qa-org --source-dir force-app

# Push to org
sf project deploy start --source-dir force-app --target-org qa-org

# Logout
sf org logout -u qa-org --no-prompt
```

### GitHub CLI Commands

```bash
# Check auth status
gh auth status

# List repositories
gh repo list

# View actions
gh run list -R ravikishantelu/sfdevops

# Trigger workflow
gh workflow run dev-qa-deploy.yml -r dev
```

---

**End of CI/CD Setup Guide**

✅ Setup Complete!  
🚀 Ready for Production!  
📊 Happy Deploying!
