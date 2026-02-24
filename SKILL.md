---
name: icd10-llmops
description: "ICD-10 code extraction using Snowflake Cortex LLMs with dbt evaluation framework. Use for: building ICD-10 extraction pipelines, medical coding with LLMs, clinical NLP evaluation, HCC risk adjustment coding, Cortex COMPLETE for healthcare, dbt medical coding project, ICD-10 precision recall metrics. Triggers: ICD-10, ICD10, medical coding, clinical coding, HCC, risk adjustment, healthcare LLM, clinical NLP, extract diagnosis codes."
---

# ICD-10 LLMOps - Clinical Code Extraction with Snowflake Cortex + dbt

Production-tested framework for extracting ICD-10 codes from clinical text using Snowflake Cortex LLMs + dbt. Replicates the **Agilon Health stored procedure logic** in a dbt evaluation pipeline with Precision, Recall, and F1 metrics against ground truth.

**Baseline metrics (out of the box):** Precision 47.5%, Recall 63.3%, F1 54.3%

## What This Skill Contains

This skill packages the **exact production-tested code** from the Agilon Health ICD-10 POC:

- **LLM Prompt**: Medical coding specialist system prompt that extracts both explicitly documented and clinically inferred ICD-10 codes with evidence-backed justifications
- **Cortex COMPLETE Integration**: Structured message array (system + user roles), JSON output schema, temperature 0, max_tokens 8000
- **8 dbt Models**: Staging (2), LLM extraction (3), evaluation (3) -- all SQL tested and working
- **Streamlit Dashboard**: Run pipeline via `EXECUTE DBT PROJECT`, view extraction results with JSON drill-down, P/R/F1 metrics cards
- **Evaluation Pipeline**: ARRAY_INTERSECTION code matching, per-patient and aggregate metrics, experiment comparison via dbt vars
- **Known gotchas**: Cortex response field COALESCE (`messages` vs `message`), SiS environment.yml restrictions, dbt output schema naming (`SCHEMA_SCHEMA`)

## Architecture

```
Source Data (Clinical Text)
       |
       v
[stg_batched_events] ---- staging ---- [stg_ground_truth]
       |                                       |
       v                                       |
[llm_sp_results] -- Cortex COMPLETE            |
       |                                       |
       v                                       |
[llm_extracted_codes] -- LATERAL FLATTEN        |
       |                                       |
       v                                       v
[eval_code_matches] --- ARRAY_INTERSECTION comparison
       |
       v
[eval_metrics_per_record] -- per-patient P/R/F1
       |
       v
[eval_experiment_summary] -- micro/macro averages
       |
       v
[Streamlit Dashboard] -- visual results + run pipeline
```

## Intent Detection

| Intent | Triggers | Action |
|--------|----------|--------|
| **SCAFFOLD** | "create ICD-10 project", "new medical coding pipeline", "scaffold ICD-10", "set up clinical extraction" | Load `scaffold/SKILL.md` |
| **DEPLOY** | "deploy ICD-10 project", "push to snowflake", "connect to git", "deploy streamlit" | Load `deploy/SKILL.md` |
| **RUN** | "run extraction", "execute pipeline", "check metrics", "run ICD-10", "evaluate results" | Load `run/SKILL.md` |

## Quick Reference

**Key Snowflake objects created:**
- dbt Project: `<DB>.<SCHEMA>.ICD10_LLMOPS_PROJECT`
- Streamlit App: `<DB>.<SCHEMA>.ICD10_LLMOPS_DEMO`
- Git Repository (optional): `<DB>.<SCHEMA>.ICD10_LLMOPS_REPO`

**Source tables required:**
- `BATCHED_EVENTS` - columns: `AGILON_MEMBER_ID`, `BATCH_NUMBER`, `BATCHED_EVENTS`, `MIN_EVENT_DATE`, `MAX_EVENT_DATE`
- `GROUND_TRUTH` - columns: `AGILON_MEMBER_ID`, `EXPECTED_ICD10_CODES`, `NOTES`

## Workflow

```
User Request
     |
     v
Intent Detection
     |
     +--> SCAFFOLD --> Create full dbt project + Streamlit app locally
     |
     +--> DEPLOY ----> Deploy to Snowflake (native dbt + Streamlit)
     |
     +--> RUN -------> Execute pipeline + view evaluation metrics
```

## Stopping Points

- Before creating any Snowflake objects (confirm database/schema/warehouse)
- Before running LLM extraction (costs Cortex credits)
- Before connecting to Git (requires PAT)

## Output

- Complete dbt project with 7 models (staging, extraction, evaluation)
- Streamlit dashboard for running pipeline and viewing results
- Evaluation metrics: Precision, Recall, F1 per patient and aggregate
