---
name: icd10-scaffold
description: "Scaffold a new ICD-10 LLMOps dbt project from scratch"
parent_skill: icd10-llmops
---

# Scaffold ICD-10 LLMOps Project

Create the full dbt project and Streamlit app for ICD-10 code extraction. This uses the **production-tested Agilon Health SP replication** as the default -- proven to work with Precision 47.5%, Recall 63.3%, F1 54.3% out of the box.

## Prerequisites

- Snowflake account with Cortex LLM access (claude-3-5-sonnet)
- Source table: `BATCHED_EVENTS` with columns `AGILON_MEMBER_ID`, `BATCH_NUMBER`, `BATCHED_EVENTS`, `MIN_EVENT_DATE`, `MAX_EVENT_DATE`
- Ground truth table: `GROUND_TRUTH` with columns `AGILON_MEMBER_ID`, `EXPECTED_ICD10_CODES`, `NOTES`

## Workflow

### Step 1: Gather Configuration

**Ask** the user for only these connection details (everything else has tested defaults):

```
1. Snowflake database name (default: ICD10_POC)
2. Schema name (default: ICD10_DEMO)
3. Warehouse name (default: XSMALL_WH)
4. Snowflake account identifier (e.g., sfsenorthamerica-moizahmeddemo)
5. Snowflake username
6. Snowflake role (default: ACCOUNTADMIN)
```

Defaults for everything else:
- Source table: `BATCHED_EVENTS`
- Ground truth table: `GROUND_TRUTH`
- LLM model: `claude-3-5-sonnet`
- Patient ID column: `AGILON_MEMBER_ID`

### Step 2: Create Project Directory Structure

```bash
mkdir -p <project_dir>/dbt_project/models/staging
mkdir -p <project_dir>/dbt_project/models/llm_extraction
mkdir -p <project_dir>/dbt_project/models/evaluation
mkdir -p <project_dir>/streamlit_app
```

### Step 3: Write All Files

**Load** `references/model-templates.md` for exact SQL for each model.
**Load** `references/streamlit-template.md` for the exact Streamlit app code.

Write these files using the exact templates from references, substituting only the user's database/schema/warehouse/account/user/role values. The SQL logic, prompts, and evaluation formulas must be used exactly as-is -- they are production-tested.

**Files to write (13 total):**

1. `dbt_project/dbt_project.yml`:
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
  model_name: 'claude-3-5-sonnet'

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

2. `dbt_project/profiles.yml`:
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

3. `dbt_project/models/sources.yml`:
```yaml
version: 2
sources:
  - name: icd10_demo
    database: <USER_DATABASE>
    schema: <USER_SCHEMA>
    tables:
      - name: BATCHED_EVENTS
        description: "Source patient batched clinical events"
        columns:
          - name: AGILON_MEMBER_ID
            description: "Patient identifier"
          - name: BATCH_NUMBER
            description: "Batch sequence number"
          - name: BATCHED_EVENTS
            description: "Clinical events text content"
          - name: MIN_EVENT_DATE
            description: "Earliest event date in batch"
          - name: MAX_EVENT_DATE
            description: "Latest event date in batch"
      - name: GROUND_TRUTH
        description: "Ground truth ICD-10 codes for evaluation"
        columns:
          - name: AGILON_MEMBER_ID
            description: "Patient identifier"
          - name: EXPECTED_ICD10_CODES
            description: "Comma-separated expected ICD-10 codes"
          - name: NOTES
            description: "Description of conditions"
```

4-11. **Model SQL files** -- copy exactly from `references/model-templates.md`:
   - `models/staging/stg_batched_events.sql`
   - `models/staging/stg_ground_truth.sql`
   - `models/llm_extraction/llm_sp_results.sql` (the core Cortex COMPLETE call with production SP prompt)
   - `models/llm_extraction/llm_extracted_codes.sql` (LATERAL FLATTEN with code normalization)
   - `models/llm_extraction/llm_call_log.sql`
   - `models/evaluation/eval_code_matches.sql` (ARRAY_INTERSECTION matching)
   - `models/evaluation/eval_metrics_per_record.sql` (per-patient P/R/F1)
   - `models/evaluation/eval_experiment_summary.sql` (micro/macro averaged metrics)

   **Replace only `<SCHEMA>` in the `{{ config() }}` blocks** with the user's schema. All other SQL is used verbatim.

12. `streamlit_app/streamlit_app.py` -- copy from `references/streamlit-template.md`, replacing DATABASE, SCHEMA, and DBT_SCHEMA constants.

13. `streamlit_app/environment.yml`:
```yaml
name: streamlit_env
channels:
  - snowflake
dependencies:
  - streamlit
  - pandas
  - snowflake-snowpark-python
```

14. `streamlit_app/snowflake.yml`:
```yaml
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

### Step 4: Verify Files

Confirm all 14 files were created. The project is ready to deploy.

## Critical Implementation Notes

These are lessons learned from production testing -- DO NOT deviate:

1. **Cortex COMPLETE response parsing**: Always use COALESCE on `choices[0]:messages` AND `choices[0]:message` -- the field name varies between model versions
2. **environment.yml**: Never add version specifiers (e.g., `streamlit>=1.48.0`). SiS rejects `>`, `<`, `=` in dependency names
3. **dbt output schema**: dbt creates tables in `<DATABASE>.<SCHEMA>_<SCHEMA>` (e.g., `ICD10_POC.ICD10_DEMO_ICD10_DEMO`). The Streamlit app must query this schema, not the source schema
4. **ICD-10 code normalization**: Use `UPPER(TRIM(REGEXP_REPLACE(code, '[^A-Za-z0-9.]', '')))` -- keeps dots which are part of ICD-10 format (e.g., E11.9)
5. **Temperature 0, max_tokens 8000**: These are required for deterministic, complete medical coding output

## Output

- Complete dbt project directory (8 models + config files)
- Streamlit dashboard app (3 files)
- Ready for deployment via **DEPLOY** intent

## Next

Tell user to proceed with **DEPLOY** intent to push to Snowflake.
