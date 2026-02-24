---
name: icd10-deploy
description: "Deploy ICD-10 LLMOps dbt project and Streamlit app to Snowflake"
parent_skill: icd10-llmops
---

# Deploy ICD-10 LLMOps to Snowflake

Deploy the dbt project and Streamlit app to Snowflake, optionally connecting to a Git repository.

## Prerequisites

- Scaffolded project (from `scaffold` intent) or existing project directory
- Snowflake CLI (`snow`) installed
- Snowflake connection configured (run `snow connection test`)

## Workflow

### Step 1: Verify Project Structure

Confirm the project directory contains:
```
dbt_project/
├── dbt_project.yml
├── profiles.yml
├── models/
│   ├── sources.yml
│   ├── staging/ (2 models)
│   ├── llm_extraction/ (3 models)
│   └── evaluation/ (3 models)
streamlit_app/
├── streamlit_app.py
├── environment.yml
└── snowflake.yml
```

### Step 2: Deploy dbt Project

```bash
snow dbt deploy <PROJECT_NAME> \
  --source <PATH_TO_DBT_PROJECT> \
  --database <DATABASE> \
  --schema <SCHEMA> \
  --connection <CONNECTION>
```

Verify deployment:
```sql
SHOW DBT PROJECTS LIKE '<PROJECT_NAME>' IN SCHEMA <DATABASE>.<SCHEMA>;
```

### Step 3: Deploy Streamlit App

```bash
cd <PATH_TO_STREAMLIT_APP>
snow streamlit deploy --connection <CONNECTION>
```

Verify deployment:
```sql
DESCRIBE STREAMLIT <DATABASE>.<SCHEMA>.ICD10_LLMOPS_DEMO;
```

### Step 4 (Optional): Connect to Git Repository

If the user wants Git integration:

**4a. Create GitHub repo and push code:**

```bash
# Initialize and push
cd <PROJECT_ROOT>
git init && git add -A && git commit -m "Initial commit"
git remote add origin https://github.com/<USER>/<REPO>.git
git push -u origin main
```

**4b. Create Snowflake Git Repository:**

Requires an API integration. If one doesn't exist:

```sql
CREATE OR REPLACE API INTEGRATION GITHUB_API_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/<USER>/')
  ALLOWED_AUTHENTICATION_SECRETS = ALL
  ENABLED = TRUE;
```

Create the secret and repository:

```sql
CREATE OR REPLACE SECRET <DATABASE>.<SCHEMA>.GITHUB_SECRET
  TYPE = password
  USERNAME = '<GITHUB_USERNAME>'
  PASSWORD = '<GITHUB_PAT>';

CREATE OR REPLACE GIT REPOSITORY <DATABASE>.<SCHEMA>.ICD10_LLMOPS_REPO
  ORIGIN = 'https://github.com/<USER>/<REPO>.git'
  API_INTEGRATION = GITHUB_API_INTEGRATION
  GIT_CREDENTIALS = <DATABASE>.<SCHEMA>.GITHUB_SECRET;

ALTER GIT REPOSITORY <DATABASE>.<SCHEMA>.ICD10_LLMOPS_REPO FETCH;
```

**4c. Recreate objects from Git:**

```sql
-- Drop locally-deployed objects
DROP DBT PROJECT IF EXISTS <DATABASE>.<SCHEMA>.ICD10_LLMOPS_PROJECT;
DROP STREAMLIT IF EXISTS <DATABASE>.<SCHEMA>.ICD10_LLMOPS_DEMO;

-- Recreate from Git
CREATE DBT PROJECT <DATABASE>.<SCHEMA>.ICD10_LLMOPS_PROJECT
  FROM '@<DATABASE>.<SCHEMA>.ICD10_LLMOPS_REPO/branches/main/dbt_project';

CREATE STREAMLIT <DATABASE>.<SCHEMA>.ICD10_LLMOPS_DEMO
  ROOT_LOCATION = '@<DATABASE>.<SCHEMA>.ICD10_LLMOPS_REPO/branches/main/streamlit_app'
  MAIN_FILE = 'streamlit_app.py'
  QUERY_WAREHOUSE = '<WAREHOUSE>'
  TITLE = 'ICD-10 LLMOps Demo';
```

**4d. Update workflow** (push changes, then fetch):

```sql
-- After pushing to GitHub:
ALTER GIT REPOSITORY <DATABASE>.<SCHEMA>.ICD10_LLMOPS_REPO FETCH;
-- Objects automatically use latest code from the repo stage
```

## Stopping Points

- Before creating any Snowflake objects (confirm names)
- Before creating Git secret (requires PAT -- never log it)

## Output

- Deployed dbt project in Snowflake
- Deployed Streamlit app in Snowflake
- (Optional) Git repository integration for CI/CD workflow
