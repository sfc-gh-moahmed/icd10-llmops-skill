# ICD-10 LLMOps Architecture

## Pipeline Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        SOURCE DATA                                    │
│                                                                       │
│  BATCHED_EVENTS                    GROUND_TRUTH                       │
│  ┌─────────────────────┐          ┌─────────────────────┐           │
│  │ AGILON_MEMBER_ID    │          │ AGILON_MEMBER_ID    │           │
│  │ BATCH_NUMBER        │          │ EXPECTED_ICD10_CODES│           │
│  │ BATCHED_EVENTS      │          │ NOTES               │           │
│  │ MIN/MAX_EVENT_DATE  │          └─────────┬───────────┘           │
│  └─────────┬───────────┘                    │                        │
└────────────┼────────────────────────────────┼────────────────────────┘
             │                                │
             ▼                                ▼
┌────────────────────────┐    ┌──────────────────────────┐
│  stg_batched_events    │    │  stg_ground_truth         │
│  (view)                │    │  (view)                   │
│  Renames columns       │    │  SPLIT() into code array  │
└────────┬───────────────┘    └──────────┬───────────────┘
         │                               │
         ▼                               │
┌──────────────────────────────┐         │
│  llm_sp_results (table)       │         │
│                               │         │
│  - LISTAGG batches per patient│         │
│  - Cortex COMPLETE call       │         │
│    (system + user messages)   │         │
│  - TRY_PARSE_JSON response    │         │
│  - Extracts icd10_codes_json  │         │
└────────┬─────────────────────┘         │
         │                               │
         ▼                               │
┌──────────────────────────────┐         │
│  llm_extracted_codes (table)  │         │
│                               │         │
│  - LATERAL FLATTEN on JSON    │         │
│  - Individual code + justify  │         │
│  - Normalized code format     │         │
└────────┬─────────────────────┘         │
         │                               │
         ▼                               ▼
┌──────────────────────────────────────────┐
│  eval_code_matches (table)                │
│                                           │
│  - ARRAY_AGG extracted per patient        │
│  - JOIN with ground truth                 │
│  - ARRAY_INTERSECTION = true positives    │
│  - FP = extracted - TP                    │
│  - FN = expected - TP                     │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  eval_metrics_per_record (table)          │
│                                           │
│  - Precision = TP / (TP + FP)            │
│  - Recall = TP / (TP + FN)              │
│  - F1 = 2*TP / (extracted + expected)    │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  eval_experiment_summary (table)          │
│                                           │
│  - Micro-averaged P/R/F1                 │
│  - Macro-averaged P/R/F1                 │
│  - Min/Max/StdDev F1                     │
└──────────────────────────────────────────┘
```

## Cortex COMPLETE Call Pattern

The LLM call uses a structured message array (not a single string prompt):

```
SNOWFLAKE.CORTEX.COMPLETE(
    'model-name',
    [
        { 'role': 'system', 'content': '...' },   -- Expert medical coder persona + JSON schema
        { 'role': 'user',   'content': '...' }     -- Patient data + extraction instructions
    ],
    { 'max_tokens': 8000, 'temperature': 0 }
)
```

**Response parsing:** The response object structure varies. Always use:
```sql
COALESCE(
    response:choices[0]:messages::VARCHAR,
    response:choices[0]:message::VARCHAR,
    response::VARCHAR
)
```

## Evaluation Methodology

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **Precision** | TP / (TP + FP) | Of codes extracted, how many were correct |
| **Recall** | TP / (TP + FN) | Of expected codes, how many were found |
| **F1** | 2*TP / (extracted + expected) | Harmonic mean of P and R |
| **Micro** | Aggregate TP/FP/FN then compute | Weighted by patient code count |
| **Macro** | Average per-patient scores | Equal weight per patient |

Code matching uses `ARRAY_INTERSECTION` on normalized codes (uppercase, alphanumeric + dots only).
