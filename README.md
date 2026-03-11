# ICD-10 LLMOps Skill for Cortex Code

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that scaffolds, deploys, and runs an ICD-10 code extraction pipeline using **Snowflake Cortex LLMs** and **dbt**. Built from production-tested code for clinical NLP evaluation with Precision, Recall, and F1 scoring against ground truth.

**Baseline metrics (out of the box):** Precision 47.5% | Recall 63.3% | F1 54.3%

## What It Does

This skill lets you build and operate a complete medical coding evaluation pipeline through natural language in Cortex Code. It extracts ICD-10 diagnosis codes from clinical text (doctor's notes, vitals, patient history) using Snowflake Cortex LLMs, then evaluates extraction quality against expert-labeled ground truth.

### Capabilities

| Intent | Example Triggers | What Happens |
|--------|-----------------|--------------|
| **Scaffold** | "create ICD-10 project", "new medical coding pipeline" | Creates a full dbt project + Streamlit app locally |
| **Deploy** | "deploy ICD-10 project", "push to Snowflake" | Deploys to Snowflake as native dbt project + Streamlit app |
| **Run** | "run extraction", "check metrics", "evaluate results" | Executes the pipeline and returns evaluation metrics |

## Architecture

```
Source Data (Clinical Text + Ground Truth)
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
[Streamlit Dashboard] -- visual results + pipeline control
```

The pipeline uses 8 dbt models across three layers:

- **Staging (2 models):** Normalize source tables and split ground truth codes into arrays
- **LLM Extraction (3 models):** Call Cortex COMPLETE with a medical coding specialist prompt, flatten JSON responses into individual codes with justifications, and log execution metadata
- **Evaluation (3 models):** Match extracted codes against ground truth using `ARRAY_INTERSECTION`, compute per-patient and aggregate Precision/Recall/F1

## Prerequisites

- Snowflake account with [Cortex LLM access](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions) (claude-3-5-sonnet or other supported model)
- [Cortex Code CLI](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) installed
- Source data tables:
  - `BATCHED_EVENTS` -- columns: `AGILON_MEMBER_ID`, `BATCH_NUMBER`, `BATCHED_EVENTS`, `MIN_EVENT_DATE`, `MAX_EVENT_DATE`
  - `GROUND_TRUTH` -- columns: `AGILON_MEMBER_ID`, `EXPECTED_ICD10_CODES`, `NOTES`

## Installation

### As a Remote Skill

Add this repository as a remote skill in your Cortex Code configuration (`~/.cortex/skills.json` or project-level):

```json
{
  "skills": [
    {
      "name": "icd10-llmops",
      "type": "remote",
      "url": "https://github.com/sfc-gh-moahmed/icd10-llmops-skill"
    }
  ]
}
```

Once installed, the skill is invoked automatically when you mention ICD-10 extraction, medical coding, or clinical NLP in Cortex Code.

### Manual Clone

```bash
git clone https://github.com/sfc-gh-moahmed/icd10-llmops-skill.git
```

## Usage

After installing the skill, interact with it through natural language in Cortex Code:

### 1. Scaffold a New Project

```
> Create an ICD-10 extraction project for my clinical data
```

The skill will ask for your Snowflake database, schema, warehouse, and account details, then generate:
- A complete dbt project (8 models + config files)
- A Streamlit dashboard for running the pipeline and viewing results

### 2. Deploy to Snowflake

```
> Deploy my ICD-10 project to Snowflake
```

Deploys via `snow dbt deploy` and `snow streamlit deploy`. Optionally connects to a Git repository for CI/CD.

### 3. Run and Evaluate

```
> Run the ICD-10 extraction pipeline and show me the metrics
```

Executes the dbt pipeline, then queries and displays:
- Aggregate Precision, Recall, and F1 scores
- Per-patient metric breakdowns
- Code-level comparison (true positives, false positives, false negatives)

### 4. Iterate on Prompts

```
> Run the pipeline with a stricter prompt to improve precision
```

Supports experiment comparison via dbt vars (`experiment_id`, `model_name`) so you can compare different LLM models and prompt strategies side by side.

## Skill Structure

```
icd10-llmops-skill/
├── SKILL.md                    # Root skill: intent detection and routing
├── scaffold/
│   └── SKILL.md                # Scaffold sub-skill: project creation
├── deploy/
│   └── SKILL.md                # Deploy sub-skill: Snowflake deployment
├── run/
│   └── SKILL.md                # Run sub-skill: pipeline execution
└── references/
    ├── architecture.md         # Pipeline architecture diagrams
    ├── model-templates.md      # Production-tested SQL for all 8 dbt models
    └── streamlit-template.md   # Streamlit dashboard template
```

## Evaluation Methodology

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **Precision** | TP / (TP + FP) | Of codes extracted, how many were correct |
| **Recall** | TP / (TP + FN) | Of expected codes, how many were found |
| **F1** | 2 * TP / (extracted + expected) | Harmonic mean of Precision and Recall |
| **Micro-averaged** | Aggregate TP/FP/FN then compute | Weighted by patient code count |
| **Macro-averaged** | Average per-patient scores | Equal weight per patient |

Code matching normalizes ICD-10 codes to uppercase alphanumeric + dots (preserving format like `E11.9`) and uses Snowflake's `ARRAY_INTERSECTION` for set comparison.

## Key Technical Details

- **Cortex COMPLETE** is called with a structured message array (system + user roles), temperature 0, and max_tokens 8000
- **Response parsing** uses `COALESCE` on `choices[0]:messages` and `choices[0]:message` to handle field name variations across Cortex model versions
- **dbt output schema** follows the pattern `<SCHEMA>_<SCHEMA>` (e.g., `ICD10_DEMO_ICD10_DEMO`) -- the Streamlit app accounts for this
- **Streamlit in Snowflake** `environment.yml` must not contain version specifiers (`>=`, `<`, `=`)

## Snowflake Objects Created

| Object | Type | Description |
|--------|------|-------------|
| `ICD10_LLMOPS_PROJECT` | dbt Project | Native Snowflake dbt project |
| `ICD10_LLMOPS_DEMO` | Streamlit App | Dashboard for pipeline control and results |
| `ICD10_LLMOPS_REPO` | Git Repository | (Optional) GitHub integration for CI/CD |

## Resources

- [Snowflake Cortex LLM Functions](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions)
- [Cortex Code Documentation](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code)
- [dbt Projects on Snowflake](https://docs.snowflake.com/en/user-guide/dbt-snowflake)
- [Streamlit in Snowflake](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit)
