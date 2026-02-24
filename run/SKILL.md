---
name: icd10-run
description: "Execute ICD-10 LLMOps pipeline and view evaluation metrics"
parent_skill: icd10-llmops
---

# Run ICD-10 LLMOps Pipeline

Execute the dbt pipeline to extract ICD-10 codes and evaluate against ground truth.

## Prerequisites

- Deployed dbt project (from `deploy` intent)
- Source tables populated (`BATCHED_EVENTS`, `GROUND_TRUTH`)
- Warehouse available for Cortex LLM calls

## Workflow

### Step 1: Execute Full Pipeline

**Via SQL:**
```sql
EXECUTE DBT PROJECT <DATABASE>.<SCHEMA>.ICD10_LLMOPS_PROJECT
  ARGS = 'run'
  USING (WAREHOUSE => '<WAREHOUSE>');
```

**Via CLI:**
```bash
snow dbt execute -c <CONNECTION> \
  --database <DATABASE> --schema <SCHEMA> \
  <PROJECT_NAME> run
```

**Selective execution** (useful for re-running just evaluation after prompt changes):
```sql
-- Run only evaluation models (skip LLM calls)
EXECUTE DBT PROJECT <DATABASE>.<SCHEMA>.ICD10_LLMOPS_PROJECT
  ARGS = 'run --select eval_code_matches eval_metrics_per_record eval_experiment_summary'
  USING (WAREHOUSE => '<WAREHOUSE>');
```

### Step 2: Check Evaluation Metrics

**Summary metrics (Precision / Recall / F1):**

```sql
SELECT
    experiment_name,
    total_records,
    total_extracted_codes,
    total_expected_codes,
    ROUND(micro_precision, 4) AS precision,
    ROUND(micro_recall, 4) AS recall,
    ROUND(micro_f1, 4) AS f1,
    evaluated_at
FROM <DATABASE>.<DBT_OUTPUT_SCHEMA>.EVAL_EXPERIMENT_SUMMARY
ORDER BY evaluated_at DESC
LIMIT 5;
```

**Per-patient breakdown:**

```sql
SELECT
    patient_id,
    num_extracted,
    num_expected,
    true_positives,
    false_positives,
    false_negatives,
    ROUND(precision_score, 2) AS precision,
    ROUND(recall_score, 2) AS recall,
    ROUND(f1_score, 2) AS f1
FROM <DATABASE>.<DBT_OUTPUT_SCHEMA>.EVAL_METRICS_PER_RECORD
ORDER BY f1_score DESC;
```

**Code-level comparison:**

```sql
SELECT
    patient_id,
    extracted_codes_array,
    expected_codes_array,
    true_positives,
    false_positives,
    false_negatives
FROM <DATABASE>.<DBT_OUTPUT_SCHEMA>.EVAL_CODE_MATCHES
ORDER BY patient_id;
```

### Step 3: Inspect LLM Outputs

**View extracted codes with justifications:**

```sql
SELECT
    patient_id,
    icd10_code,
    LEFT(justification, 200) AS justification,
    evidence_count,
    model_used
FROM <DATABASE>.<DBT_OUTPUT_SCHEMA>.LLM_EXTRACTED_CODES
ORDER BY patient_id, code_index;
```

**View raw JSON responses:**

```sql
SELECT
    patient_id,
    batch_group_range,
    icd10_codes_json,
    batch_group_summary,
    num_codes_extracted
FROM <DATABASE>.<DBT_OUTPUT_SCHEMA>.LLM_SP_RESULTS
ORDER BY patient_id;
```

### Step 4: Iterate on Prompts

To improve metrics, edit the system/user prompts in `llm_sp_results.sql`:

1. Modify the prompt in the model file
2. If using Git: push changes, then `ALTER GIT REPOSITORY ... FETCH`
3. If local: redeploy with `snow dbt deploy`
4. Re-run: `EXECUTE DBT PROJECT ... ARGS = 'run'`
5. Check new metrics

**Common prompt tuning strategies:**
- Add "Only include codes with strong clinical evidence" to reduce false positives (improve precision)
- Add "Include all codes that could reasonably apply" to reduce false negatives (improve recall)
- Specify exact ICD-10 code format (e.g., "Use format X00.0, not X00")
- Add few-shot examples of correct extractions

### Step 5: Compare Experiments

Change the dbt vars to tag different runs:

```sql
EXECUTE DBT PROJECT <DATABASE>.<SCHEMA>.ICD10_LLMOPS_PROJECT
  ARGS = 'run --vars {"experiment_id": "v2_strict_prompt", "experiment_name": "Strict Prompt v2", "model_name": "claude-3-5-sonnet"}'
  USING (WAREHOUSE => '<WAREHOUSE>');
```

## Key Schema Note

dbt outputs go to `<DATABASE>.<SCHEMA>_<SCHEMA>` by default (e.g., `ICD10_POC.ICD10_DEMO_ICD10_DEMO`). This is because dbt appends the custom schema to the target schema. Adjust queries accordingly.

## Stopping Points

- Before running full pipeline (LLM calls cost Cortex credits)

## Output

- Materialized tables with extraction results and evaluation metrics
- Precision, Recall, F1 scores per patient and aggregate
