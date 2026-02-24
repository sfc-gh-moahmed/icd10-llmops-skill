---
name: icd10-scaffold
description: "Scaffold a new ICD-10 LLMOps dbt project from scratch"
parent_skill: icd10-llmops
---

# Scaffold ICD-10 LLMOps Project

Create the full dbt project and Streamlit app for ICD-10 code extraction.

## Prerequisites

- Snowflake account with Cortex LLM access (claude-3-5-sonnet or similar)
- Source table with batched clinical text per patient
- Ground truth table with expected ICD-10 codes per patient

## Workflow

### Step 1: Gather Configuration

**Ask** the user for:

```
1. Snowflake database name (e.g., ASPEN_AI_POC)
2. Schema name (e.g., ICD10_DEMO)
3. Warehouse name (e.g., XSMALL_WH)
4. Source table name for clinical events (default: BATCHED_EVENTS)
5. Ground truth table name (default: GROUND_TRUTH)
6. LLM model to use (default: claude-3-5-sonnet)
```

Store these as variables for template generation.

### Step 2: Create dbt Project Structure

Create the following directory structure:

```
<project_dir>/
├── dbt_project/
│   ├── dbt_project.yml
│   ├── profiles.yml
│   ├── models/
│   │   ├── sources.yml
│   │   ├── staging/
│   │   │   ├── stg_batched_events.sql
│   │   │   └── stg_ground_truth.sql
│   │   ├── llm_extraction/
│   │   │   ├── llm_sp_results.sql
│   │   │   ├── llm_extracted_codes.sql
│   │   │   └── llm_call_log.sql
│   │   └── evaluation/
│   │       ├── eval_code_matches.sql
│   │       ├── eval_metrics_per_record.sql
│   │       └── eval_experiment_summary.sql
│   └── profiles.yml.template
└── streamlit_app/
    ├── streamlit_app.py
    ├── environment.yml
    └── snowflake.yml
```

### Step 3: Generate dbt_project.yml

**Load** `references/model-templates.md` for the exact SQL templates.

```yaml
name: 'icd10_llmops'
version: '1.0.0'
config-version: 2
profile: 'icd10_llmops'
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
clean-targets: ["target", "dbt_packages"]

vars:
  experiment_id: 'default'
  experiment_name: 'Default Experiment'
  model_name: '<USER_MODEL_NAME>'  # e.g., claude-3-5-sonnet

models:
  icd10_llmops:
    staging:
      +materialized: view
      +schema: <USER_SCHEMA>
    llm_extraction:
      +materialized: table
      +schema: <USER_SCHEMA>
    evaluation:
      +materialized: table
      +schema: <USER_SCHEMA>
```

### Step 4: Generate profiles.yml

```yaml
icd10_llmops:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <USER_ACCOUNT>
      user: <USER_NAME>
      authenticator: externalbrowser
      role: <USER_ROLE>
      warehouse: <USER_WAREHOUSE>
      database: <USER_DATABASE>
      schema: <USER_SCHEMA>
      threads: 4
```

### Step 5: Generate sources.yml

```yaml
version: 2
sources:
  - name: icd10_demo
    database: <USER_DATABASE>
    schema: <USER_SCHEMA>
    tables:
      - name: <SOURCE_TABLE>
        description: "Source patient batched clinical events"
        columns:
          - name: AGILON_MEMBER_ID
            description: "Patient identifier"
          - name: BATCH_NUMBER
          - name: BATCHED_EVENTS
            description: "Clinical events text content"
          - name: MIN_EVENT_DATE
          - name: MAX_EVENT_DATE
      - name: <GROUND_TRUTH_TABLE>
        description: "Ground truth ICD-10 codes for evaluation"
        columns:
          - name: AGILON_MEMBER_ID
          - name: EXPECTED_ICD10_CODES
          - name: NOTES
```

### Step 6: Generate All Model SQL Files

**Load** `references/model-templates.md` and generate each model file using the templates. The key models are:

1. **stg_batched_events.sql** - Renames source columns to standard names
2. **stg_ground_truth.sql** - Parses expected codes into arrays
3. **llm_sp_results.sql** - Core LLM extraction with Cortex COMPLETE (system + user messages, structured JSON output)
4. **llm_extracted_codes.sql** - LATERAL FLATTEN to extract individual codes from JSON
5. **llm_call_log.sql** - Audit log of LLM calls
6. **eval_code_matches.sql** - ARRAY_INTERSECTION comparison with ground truth
7. **eval_metrics_per_record.sql** - Per-patient Precision/Recall/F1
8. **eval_experiment_summary.sql** - Micro/macro averaged metrics

### Step 7: Generate Streamlit App

**Load** `references/streamlit-template.md` and generate the Streamlit app with:
- Source data overview
- dbt pipeline trigger button (EXECUTE DBT PROJECT)
- LLM extraction results viewer with JSON drill-down
- Evaluation metrics dashboard (P/R/F1 cards, per-record table, code comparison)

### Step 8: Generate environment.yml and snowflake.yml

```yaml
# environment.yml
name: streamlit_env
channels:
  - snowflake
dependencies:
  - streamlit
  - pandas
  - snowflake-snowpark-python
```

```yaml
# snowflake.yml
definition_version: 2
entities:
  icd10_llmops_demo:
    type: streamlit
    identifier:
      name: ICD10_LLMOPS_DEMO
    title: "ICD-10 LLMOps Demo"
    query_warehouse: <USER_WAREHOUSE>
    main_file: streamlit_app.py
    artifacts:
      - environment.yml
```

## Output

- Complete dbt project directory ready for deployment
- Streamlit app ready for deployment
- User instructed to proceed with `deploy` intent

## Next

After scaffolding, user should proceed with **DEPLOY** intent to push to Snowflake.
