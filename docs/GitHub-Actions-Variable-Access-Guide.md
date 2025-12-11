# GitHub Actions Variable Access Guide

## Overview

This guide explains the different methods for accessing variables, secrets, and other data in GitHub Actions workflows, including when to use each method and the syntax behind them.

---

## Table of Contents

1. [Six Variable Access Methods](#six-variable-access-methods)
2. [The Dot Operator Syntax](#the-dot-operator-syntax)
3. [Detailed Comparison](#detailed-comparison)
4. [Converting vars to env](#converting-vars-to-env)
5. [When to Use Each Method](#when-to-use-each-method)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)

---

## Six Variable Access Methods

GitHub Actions provides six primary contexts for accessing data:

| Context | Access Pattern | Data Type | Scope |
|---------|---------------|-----------|-------|
| **vars** | `${{ vars.VARIABLE_NAME }}` | Plain text variables | Repository & Environment |
| **secrets** | `${{ secrets.SECRET_NAME }}` | Encrypted secrets | Repository & Environment |
| **env** | `${{ env.VARIABLE_NAME }}` | Environment variables | Workflow-defined |
| **needs** | `${{ needs.job_id.outputs.key }}` | Job outputs | Cross-job data |
| **steps** | `${{ steps.step_id.outputs.key }}` | Step outputs | Within same job |
| **github** | `${{ github.ref }}` | Workflow metadata | Automatic context |

---

## The Dot Operator Syntax

### Basic Structure

```yaml
${{ context.property }}
```

**Breaking it down:**

- `${{` - Opens the expression evaluation block
- `context` - The object/namespace containing the data (vars, secrets, env, github, etc.)
- `.` - The dot operator for property access
- `property` - The specific key/field you want to access
- `}}` - Closes the expression evaluation block

### Nested Access

```yaml
${{ needs.job_name.outputs.artifact_name }}
```

**Breaking it down:**

- `needs` - Context containing data from dependent jobs
- `.job_name` - Specific job (e.g., CI, Build, Test)
- `.outputs` - The outputs object of that job
- `.artifact_name` - The specific output key you defined

### Function Calls

```yaml
${{ fromJson(env.TIMEOUT_MINUTES) }}
```

**Breaking it down:**

- `fromJson()` - Function to parse JSON string
- `env.TIMEOUT_MINUTES` - Argument passed to the function
- Functions can be nested: `${{ contains(github.ref, 'main') }}`

---

## Detailed Comparison

### 1. vars Context

**What it is:** Repository-level and environment-level configuration variables (plain text)

**Setup location:** 
- Repository: Settings → Secrets and variables → Actions → Variables tab
- Environment: Settings → Environments → [Environment name] → Variables

**Access syntax:**
```yaml
${{ vars.APP_NAME }}
${{ vars.BUILD_VERSION }}
${{ vars.LOG_LEVEL }}
```

**Characteristics:**
- ✅ Visible in workflow logs (not masked)
- ✅ Can be used at job level (runs-on, timeout-minutes, etc.)
- ✅ Can be overridden by environment-specific vars
- ✅ Suitable for non-sensitive configuration
- ❌ Not encrypted (don't use for passwords, tokens, keys)

**Example:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: ${{ fromJson(vars.TIMEOUT) }}  # Direct access at job level
    steps:
      - name: Display config
        run: echo "App: ${{ vars.APP_NAME }}"  # Direct access in step
```

---

### 2. secrets Context

**What it is:** Encrypted secrets for sensitive data (passwords, tokens, API keys)

**Setup location:**
- Repository: Settings → Secrets and variables → Actions → Secrets tab
- Environment: Settings → Environments → [Environment name] → Secrets

**Access syntax:**
```yaml
${{ secrets.DEPLOY_TOKEN }}
${{ secrets.DATABASE_PASSWORD }}
${{ secrets.AWS_ACCESS_KEY }}
```

**Characteristics:**
- ✅ Encrypted at rest
- ✅ Automatically masked in logs (displays as `***`)
- ✅ Can be used at job level
- ✅ Can be overridden by environment-specific secrets
- ✅ Required for sensitive data
- ⚠️ Cannot directly view values in logs (security feature)

**Example:**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          DB_PASS: ${{ secrets.DATABASE_PASSWORD }}
        run: |
          echo "Token: $TOKEN"  # Will display as: Token: ***
          deploy.sh --token=$TOKEN --db-pass=$DB_PASS
```

---

### 3. env Context

**What it is:** Environment variables defined within the workflow or inherited from the runner

**Setup location:** Defined in the workflow YAML file

**Access syntax:**
```yaml
${{ env.VARIABLE_NAME }}
```

**Characteristics:**
- ✅ Defined in workflow (top-level, job-level, or step-level)
- ✅ Can store computed values
- ✅ Accessible within steps (not at job level properties)
- ✅ Can be modified during workflow execution
- ❌ NOT available at job-level properties (timeout-minutes, runs-on, if, etc.)
- ✅ Can contain values from vars/secrets

**Definition levels:**
```yaml
# Top-level (available to all jobs)
env:
  GLOBAL_VAR: "value1"
  RETENTION_DAYS: 30

jobs:
  job1:
    # Job-level (available to all steps in this job)
    env:
      JOB_VAR: "value2"
    
    steps:
      - name: Step with env
        # Step-level (available only in this step)
        env:
          STEP_VAR: "value3"
        run: |
          echo "Global: $GLOBAL_VAR"
          echo "Job: $JOB_VAR"
          echo "Step: $STEP_VAR"
```

**Scope limitations:**
```yaml
jobs:
  CI:
    timeout-minutes: ${{ fromJson(env.TIMEOUT) }}  # ❌ DOES NOT WORK - env not available here
    
    steps:
      - name: Use timeout
        run: echo "Timeout: ${{ env.TIMEOUT }}"  # ✅ WORKS - env available in steps
```

---

### 4. needs Context

**What it is:** Data passed between jobs via outputs

**Setup location:** Defined in job outputs, accessed in dependent jobs

**Access syntax:**
```yaml
${{ needs.job_id.outputs.key_name }}
${{ needs.job_id.result }}  # Job status: success, failure, cancelled, skipped
```

**Characteristics:**
- ✅ Enables job-to-job communication
- ✅ Can be used at job-level properties (environment, if conditions)
- ✅ Survives job completion
- ✅ Can pass artifact names, computed values, conditional flags
- ⚠️ Requires explicit outputs definition
- ⚠️ Only available in jobs that list the source job in `needs:`

**Example:**
```yaml
jobs:
  CI:
    outputs:
      artifact-name: build-artifact-${{ github.run_number }}
      target-env: ${{ github.ref == 'refs/heads/main' && 'PROD' || 'DEV' }}
    steps:
      - name: Build
        run: echo "Building..."

  CD:
    needs: CI
    environment: ${{ needs.CI.outputs.target-env }}  # Use output to set environment
    steps:
      - name: Download
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.CI.outputs.artifact-name }}  # Use output for artifact name
      
      - name: Check CI status
        if: needs.CI.result == 'success'
        run: echo "CI succeeded"
```

---

### 5. steps Context

**What it is:** Data passed between steps within the same job

**Setup location:** Defined in step outputs, accessed in subsequent steps

**Access syntax:**
```yaml
${{ steps.step_id.outputs.key_name }}
${{ steps.step_id.conclusion }}  # Step status
```

**Characteristics:**
- ✅ Enables step-to-step communication within a job
- ✅ Can capture command outputs
- ✅ Useful for conditional logic between steps
- ⚠️ Only available within the same job (not cross-job)
- ⚠️ Requires `id:` on the source step

**Example:**
```yaml
jobs:
  build:
    steps:
      - name: Get version
        id: version  # ID required for accessing outputs
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT
      
      - name: Get hash
        id: hash
        run: echo "commit_hash=$(git rev-parse HEAD)" >> $GITHUB_OUTPUT
      
      - name: Use outputs
        run: |
          echo "Version: ${{ steps.version.outputs.version }}"
          echo "Hash: ${{ steps.hash.outputs.commit_hash }}"
      
      - name: Conditional step
        if: steps.version.outputs.version == '1.2.3'
        run: echo "Version matches!"
```

---

### 6. github Context

**What it is:** Automatic context with workflow metadata and GitHub event data

**Setup location:** Automatically provided by GitHub Actions

**Access syntax:**
```yaml
${{ github.ref }}
${{ github.actor }}
${{ github.event_name }}
```

**Characteristics:**
- ✅ Automatically available in all workflows
- ✅ Contains rich metadata about the workflow run
- ✅ Can be used at job and step levels
- ✅ Includes event payload data
- ✅ No setup required

**Common properties:**
```yaml
jobs:
  example:
    steps:
      - name: Display GitHub context
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Branch: ${{ github.ref }}"
          echo "Branch name: ${{ github.ref_name }}"
          echo "Actor: ${{ github.actor }}"
          echo "Event: ${{ github.event_name }}"
          echo "Run number: ${{ github.run_number }}"
          echo "Run ID: ${{ github.run_id }}"
          echo "SHA: ${{ github.sha }}"
          echo "Workflow: ${{ github.workflow }}"
```

**Conditional logic:**
```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - name: Deploy to prod
        run: echo "Deploying..."
```

---

## Converting vars to env

### Why You Might Want This

While you **cannot** directly access `vars` using `env.VARIABLE_NAME`, you **can** copy vars into env variables for easier access within shell scripts.

### Method 1: Copy at Step Level

```yaml
jobs:
  example:
    steps:
      - name: Use vars as env
        env:
          APP_NAME: ${{ vars.APP_NAME }}
          BUILD_VERSION: ${{ vars.BUILD_VERSION }}
          LOG_LEVEL: ${{ vars.LOG_LEVEL }}
        run: |
          # Now you can use shell variables without ${{ }}
          echo "App: $APP_NAME"
          echo "Version: $BUILD_VERSION"
          echo "Log level: $LOG_LEVEL"
```

### Method 2: Copy at Job Level

```yaml
jobs:
  example:
    env:
      APP_NAME: ${{ vars.APP_NAME }}
      BUILD_VERSION: ${{ vars.BUILD_VERSION }}
    steps:
      - name: Step 1
        run: echo "App: $APP_NAME"
      
      - name: Step 2
        run: echo "Version: $BUILD_VERSION"
```

### Method 3: Mix and Override

```yaml
env:
  # Global env from vars
  APP_NAME: ${{ vars.APP_NAME }}

jobs:
  dev:
    env:
      # Job-level override
      ENVIRONMENT: "DEV"
      DB_HOST: ${{ vars.DEV_DB_HOST }}
    steps:
      - name: Deploy
        env:
          # Step-level additions
          DEPLOY_TIME: ${{ github.run_id }}
        run: |
          echo "App: $APP_NAME"  # From global env
          echo "Env: $ENVIRONMENT"  # From job env
          echo "DB: $DB_HOST"  # From job env
          echo "Time: $DEPLOY_TIME"  # From step env
```

---

## When to Use Each Method

### Use `vars` when:
- ✅ You need configuration values (timeouts, versions, app names)
- ✅ Values are non-sensitive and can be visible in logs
- ✅ You want to access values at job-level properties
- ✅ You need easy override by environment-specific values
- ✅ You want centralized configuration management

### Use `secrets` when:
- ✅ Storing passwords, tokens, API keys, certificates
- ✅ Values must be encrypted and masked in logs
- ✅ Compliance requires secret management
- ✅ You need environment-specific secrets (prod vs dev credentials)

### Use `env` when:
- ✅ You want to define workflow-specific variables
- ✅ You need to compute values during workflow execution
- ✅ You want to combine multiple vars/secrets into one variable
- ✅ You prefer shell-style variable access (`$VAR` instead of `${{ vars.VAR }}`)
- ✅ You need variables scoped to specific jobs or steps

### Use `needs` when:
- ✅ Passing data between jobs (artifact names, computed values)
- ✅ Checking if a previous job succeeded/failed
- ✅ Dynamic environment selection based on previous job outputs
- ✅ Conditional job execution based on another job's results

### Use `steps` when:
- ✅ Passing data between steps within the same job
- ✅ Capturing command outputs for use in later steps
- ✅ Building conditional logic based on previous step results
- ✅ Avoiding redundant computations

### Use `github` when:
- ✅ You need workflow metadata (branch, actor, event type)
- ✅ Building conditional logic based on branch/event
- ✅ Constructing dynamic names (artifacts, releases)
- ✅ Accessing event payload data (PR number, commit message)

---

## Practical Examples

### Example 1: Multi-Environment Deployment

```yaml
env:
  RETENTION_DAYS: 30

jobs:
  CI:
    outputs:
      artifact-name: app-${{ github.run_number }}
      target-env: ${{ github.ref == 'refs/heads/main' && 'PROD' || 'DEV' }}
    steps:
      - name: Build
        env:
          APP_NAME: ${{ vars.APP_NAME }}
          VERSION: ${{ vars.BUILD_VERSION }}
        run: |
          echo "Building $APP_NAME version $VERSION"
          # Build logic here
      
      - uses: actions/upload-artifact@v4
        with:
          name: app-${{ github.run_number }}
          retention-days: ${{ fromJson(env.RETENTION_DAYS) }}

  CD:
    needs: CI
    environment: ${{ needs.CI.outputs.target-env }}
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ${{ needs.CI.outputs.artifact-name }}
      
      - name: Deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          DB_HOST: ${{ vars.DB_HOST }}
          REGION: ${{ vars.REGION }}
        run: |
          echo "Deploying to ${{ needs.CI.outputs.target-env }}"
          echo "Region: $REGION"
          echo "DB: $DB_HOST"
          deploy.sh --token=$DEPLOY_TOKEN
```

### Example 2: Conditional Logic with Multiple Contexts

```yaml
jobs:
  test:
    if: github.event_name == 'pull_request'
    steps:
      - name: Run tests
        id: test
        run: |
          # Run tests
          echo "result=success" >> $GITHUB_OUTPUT
      
      - name: Comment on PR
        if: steps.test.outputs.result == 'success'
        run: |
          echo "Tests passed for PR #${{ github.event.pull_request.number }}"

  deploy:
    needs: test
    if: |
      always() && 
      needs.test.result == 'success' &&
      github.ref == 'refs/heads/main'
    environment: ${{ vars.DEPLOY_ENV || 'PROD' }}
    steps:
      - name: Deploy
        env:
          TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          SERVER: ${{ vars.DEPLOY_SERVER }}
        run: deploy.sh --server=$SERVER
```

### Example 3: Mixing All Contexts

```yaml
env:
  GLOBAL_TIMEOUT: 30

jobs:
  build:
    outputs:
      version: ${{ steps.version.outputs.value }}
    env:
      APP_NAME: ${{ vars.APP_NAME }}
    steps:
      - name: Get version
        id: version
        run: echo "value=1.0.${{ github.run_number }}" >> $GITHUB_OUTPUT
      
      - name: Build
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "Building $APP_NAME version ${{ steps.version.outputs.value }}"
          echo "Timeout: $GLOBAL_TIMEOUT seconds"
          # Build with API_KEY

  deploy:
    needs: build
    environment: ${{ github.ref_name == 'main' && 'PROD' || 'DEV' }}
    steps:
      - name: Deploy
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }}"
          echo "To environment: ${{ github.ref_name == 'main' && 'PROD' || 'DEV' }}"
          echo "By: ${{ github.actor }}"
```

---

## Best Practices

### 1. Choose the Right Context

```yaml
# ✅ GOOD: Use vars for non-sensitive config
timeout-minutes: ${{ fromJson(vars.TIMEOUT) }}

# ❌ BAD: Don't use secrets for non-sensitive data (wastes encryption)
timeout-minutes: ${{ fromJson(secrets.TIMEOUT) }}

# ✅ GOOD: Use secrets for sensitive data
env:
  TOKEN: ${{ secrets.API_TOKEN }}

# ❌ BAD: Don't use vars for sensitive data
env:
  TOKEN: ${{ vars.API_TOKEN }}  # Will be visible in logs!
```

### 2. Scope Variables Appropriately

```yaml
# ✅ GOOD: Global env for widely-used values
env:
  RETENTION_DAYS: 30

jobs:
  job1:
    # ✅ GOOD: Job-level for job-specific values
    env:
      JOB_SPECIFIC: "value"
    steps:
      - name: step1
        # ✅ GOOD: Step-level for one-time use
        env:
          STEP_SPECIFIC: "value"
```

### 3. Use Meaningful Names

```yaml
# ✅ GOOD: Clear, descriptive names
${{ vars.DATABASE_CONNECTION_TIMEOUT }}
${{ secrets.AWS_ACCESS_KEY_ID }}
${{ needs.CI.outputs.artifact-name }}

# ❌ BAD: Vague names
${{ vars.TIMEOUT }}
${{ secrets.KEY }}
${{ needs.CI.outputs.name }}
```

### 4. Leverage Environment Overrides

```yaml
# Repository variables (default)
vars.DB_HOST = "dev-db.example.com"
vars.REGION = "us-west-2"

# PROD environment variables (override for prod)
vars.DB_HOST = "prod-db.example.com"
vars.REGION = "us-east-1"

# Workflow automatically uses correct values based on environment
environment: PROD
steps:
  - run: echo "DB: ${{ vars.DB_HOST }}"  # prod-db.example.com
```

### 5. Document Context Usage

```yaml
jobs:
  deploy:
    # Document what contexts you're using and why
    steps:
      - name: Deploy app
        env:
          # From repository vars (config)
          APP_NAME: ${{ vars.APP_NAME }}
          TIMEOUT: ${{ vars.TIMEOUT }}
          
          # From environment vars (env-specific config)
          DB_HOST: ${{ vars.DB_HOST }}
          
          # From secrets (sensitive)
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          
          # From previous job (dynamic)
          ARTIFACT: ${{ needs.CI.outputs.artifact-name }}
          
          # From GitHub context (metadata)
          BRANCH: ${{ github.ref_name }}
        run: deploy.sh
```

---

## Summary Table

| Context | When to Use | Available At | Encrypted | Override Support |
|---------|-------------|--------------|-----------|------------------|
| `vars` | Non-sensitive config | Job & Step | No | Yes (by environment) |
| `secrets` | Sensitive data | Job & Step | Yes | Yes (by environment) |
| `env` | Workflow variables | Step only* | No | Yes (by level) |
| `needs` | Cross-job data | Job & Step | No | N/A |
| `steps` | Within-job data | Step only | No | N/A |
| `github` | Metadata | Job & Step | No | N/A |

*`env` can be defined at job level but accessed only at step level

---

## Conclusion

Understanding these six contexts and when to use each one is crucial for building maintainable, secure, and efficient GitHub Actions workflows. The dot operator (`.`) provides a clean, intuitive way to access nested properties across all these contexts.

**Key Takeaways:**

1. **vars** → Non-sensitive configuration (visible in logs)
2. **secrets** → Sensitive data (encrypted, masked in logs)
3. **env** → Workflow-defined variables (step-level access)
4. **needs** → Cross-job communication (outputs and status)
5. **steps** → Within-job communication (step outputs)
6. **github** → Automatic metadata (branch, actor, events)

Choose the right context based on:
- **Sensitivity** (secrets vs vars)
- **Scope** (global vs job vs step)
- **Source** (GitHub vs user-defined)
- **Purpose** (config vs data flow vs metadata)

---

**Document Version:** 1.0  
**Last Updated:** December 10, 2025  
**Related Files:**
- `.github/workflows/cicd_workflow_testing.yml`
- `docs/CICD-Workflow-Documentation.md`
