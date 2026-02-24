# ICD-10 dbt Model Templates

Exact SQL templates for each dbt model. Replace `<PLACEHOLDERS>` with user values.

## stg_batched_events.sql

```sql
{{ config(materialized='view', schema='<SCHEMA>') }}

SELECT
    AGILON_MEMBER_ID AS patient_id,
    BATCH_NUMBER AS batch_number,
    BATCHED_EVENTS AS clinical_text,
    MIN_EVENT_DATE AS min_event_date,
    MAX_EVENT_DATE AS max_event_date
FROM {{ source('icd10_demo', '<SOURCE_TABLE>') }}
```

## stg_ground_truth.sql

```sql
{{ config(materialized='view', schema='<SCHEMA>') }}

SELECT
    AGILON_MEMBER_ID AS patient_id,
    EXPECTED_ICD10_CODES AS expected_codes,
    SPLIT(UPPER(TRIM(EXPECTED_ICD10_CODES)), ',') AS expected_codes_array,
    NOTES AS condition_notes
FROM {{ source('icd10_demo', '<GROUND_TRUTH_TABLE>') }}
```

## llm_sp_results.sql (Core LLM Extraction)

This is the main model. It replicates production stored procedure logic:
- Groups all batches per patient using LISTAGG
- Calls Cortex COMPLETE with system + user message structure
- Requests structured JSON output with ICD-10 codes, justifications, evidence
- Parses response with TRY_PARSE_JSON and COALESCE for response field variations

```sql
{{ config(materialized='table', schema='<SCHEMA>') }}

WITH source_data AS (
    SELECT patient_id, batch_number, clinical_text, min_event_date, max_event_date
    FROM {{ ref('stg_batched_events') }}
),

batch_groups AS (
    SELECT
        patient_id,
        MIN(batch_number) AS min_batch,
        MAX(batch_number) AS max_batch,
        MIN(min_event_date) AS min_date,
        MAX(max_event_date) AS max_date,
        LISTAGG(clinical_text, '\n\n---BATCH SEPARATOR---\n\n')
            WITHIN GROUP (ORDER BY batch_number) AS combined_clinical_text
    FROM source_data
    GROUP BY patient_id
),

llm_calls AS (
    SELECT
        patient_id, min_batch, max_batch, min_date, max_date, combined_clinical_text,
        SNOWFLAKE.CORTEX.COMPLETE(
            '{{ var("model_name", "claude-3-5-sonnet") }}',
            [
                {
                    'role': 'system',
                    'content': 'You are an expert medical coding specialist.
Identify all ICD-10 codes that apply to the patient in this batch group.

Include both:
- Explicitly documented codes mentioned directly in clinical notes, conditions, or diagnoses.
- Clinically inferred codes logically supported by evidence found across clinical notes and documents, medications, labs, vitals, or treatment patterns.

For each ICD-10 code provide:
1. The code and concise clinical description.
2. Up to 3 short sentences explaining whether it is explicitly documented or inferred and why the evidence supports it.
3. Supporting evidence items with evidence_id, source_type, and date.

Return your response as valid JSON with this structure:
{
  "patient_id": "string",
  "batch_group_range": "string (e.g. 1-8)",
  "icd10_codes": [
    {
      "code": "ICD-10 code",
      "justification": "explanation",
      "evidence": [{"evidence_id": "string", "source_type": "string", "date": "string"}]
    }
  ],
  "batch_group_summary": "5-8 line summary of key findings"
}

Keep language factual and structured. Stay concise. Return strictly valid JSON.'
                },
                {
                    'role': 'user',
                    'content': 'Analyze this batch group of patient data and identify all clinically appropriate ICD-10 codes.

PATIENT ID: ' || patient_id || '
BATCH GROUP RANGE: ' || min_batch::VARCHAR || '-' || max_batch::VARCHAR || '
DATE RANGE: ' || COALESCE(min_date::VARCHAR, 'N/A') || ' to ' || COALESCE(max_date::VARCHAR, 'N/A') || '

DATA:
' || combined_clinical_text || '

Instructions:
- Include every explicit ICD-10 code found in the data.
- Also include inferred codes when strong clinical reasoning supports them.
- Each code must include: code, justification, and supporting evidence.
- Return valid JSON following the schema provided.'
                }
            ],
            { 'max_tokens': 8000, 'temperature': 0 }
        ) AS llm_raw_response
    FROM batch_groups
),

parsed_results AS (
    SELECT
        patient_id,
        min_batch || '-' || max_batch AS batch_group_range,
        min_date, max_date, combined_clinical_text AS clinical_text,
        llm_raw_response,
        COALESCE(
            llm_raw_response:choices[0]:messages::VARCHAR,
            llm_raw_response:choices[0]:message::VARCHAR,
            llm_raw_response::VARCHAR
        ) AS llm_response_text,
        TRY_PARSE_JSON(
            COALESCE(
                llm_raw_response:choices[0]:messages::VARCHAR,
                llm_raw_response:choices[0]:message::VARCHAR,
                llm_raw_response::VARCHAR
            )
        ) AS parsed_json
    FROM llm_calls
)

SELECT
    '{{ var("experiment_id") }}' AS experiment_id,
    '{{ var("experiment_name") }}' AS experiment_name,
    patient_id, batch_group_range, min_date, max_date, clinical_text,
    llm_response_text AS llm_response,
    parsed_json:icd10_codes AS icd10_codes_json,
    parsed_json:batch_group_summary::VARCHAR AS batch_group_summary,
    ARRAY_SIZE(COALESCE(parsed_json:icd10_codes, ARRAY_CONSTRUCT())) AS num_codes_extracted,
    '{{ var("model_name", "claude-3-5-sonnet") }}' AS model_used,
    CURRENT_TIMESTAMP() AS extracted_at
FROM parsed_results
```

**Critical notes:**
- Cortex COMPLETE response field varies: use COALESCE on `choices[0]:messages` and `choices[0]:message`
- TRY_PARSE_JSON handles malformed JSON gracefully (returns NULL)
- Temperature 0 for deterministic output
- max_tokens 8000 for complex patients with many codes

## llm_extracted_codes.sql

```sql
{{ config(materialized='table', schema='<SCHEMA>') }}

WITH source_results AS (
    SELECT experiment_id, experiment_name, patient_id, batch_group_range,
           min_date, max_date, icd10_codes_json, batch_group_summary, model_used, extracted_at
    FROM {{ ref('llm_sp_results') }}
    WHERE icd10_codes_json IS NOT NULL
),

flattened_codes AS (
    SELECT
        r.experiment_id, r.experiment_name, r.patient_id, r.batch_group_range,
        r.min_date, r.max_date,
        f.value:code::VARCHAR AS icd10_code,
        f.value:justification::VARCHAR AS justification,
        f.value:evidence AS evidence_json,
        ARRAY_SIZE(COALESCE(f.value:evidence, ARRAY_CONSTRUCT())) AS evidence_count,
        r.batch_group_summary, r.model_used, r.extracted_at,
        f.index AS code_index
    FROM source_results r,
    LATERAL FLATTEN(input => r.icd10_codes_json) f
)

SELECT *, UPPER(TRIM(REGEXP_REPLACE(icd10_code, '[^A-Za-z0-9.]', ''))) AS icd10_code_normalized
FROM flattened_codes
WHERE icd10_code IS NOT NULL AND LENGTH(TRIM(icd10_code)) > 0
```

## llm_call_log.sql

```sql
{{ config(materialized='table', schema='<SCHEMA>') }}

SELECT
    experiment_id, experiment_name, patient_id, batch_group_range,
    model_used, num_codes_extracted,
    LENGTH(llm_response) AS response_length,
    CASE WHEN icd10_codes_json IS NOT NULL THEN 'SUCCESS' ELSE 'PARSE_FAILED' END AS status,
    extracted_at
FROM {{ ref('llm_sp_results') }}
```

## eval_code_matches.sql

```sql
{{ config(materialized='table', schema='<SCHEMA>') }}

WITH extracted_by_patient AS (
    SELECT experiment_id, experiment_name, patient_id,
           ARRAY_AGG(DISTINCT extracted_code) AS extracted_codes_array,
           COUNT(DISTINCT extracted_code) AS num_extracted
    FROM (
        SELECT experiment_id, experiment_name, patient_id,
               icd10_code_normalized AS extracted_code
        FROM {{ ref('llm_extracted_codes') }}
    )
    GROUP BY experiment_id, experiment_name, patient_id
),

ground_truth AS (
    SELECT patient_id, expected_codes_array, expected_codes, condition_notes
    FROM {{ ref('stg_ground_truth') }}
),

code_comparison AS (
    SELECT e.*, g.expected_codes_array, g.expected_codes, g.condition_notes,
           ARRAY_SIZE(COALESCE(g.expected_codes_array, ARRAY_CONSTRUCT())) AS num_expected
    FROM extracted_by_patient e
    LEFT JOIN ground_truth g ON e.patient_id = g.patient_id
)

SELECT *,
    ARRAY_SIZE(ARRAY_INTERSECTION(extracted_codes_array, COALESCE(expected_codes_array, ARRAY_CONSTRUCT()))) AS true_positives,
    num_extracted - ARRAY_SIZE(ARRAY_INTERSECTION(extracted_codes_array, COALESCE(expected_codes_array, ARRAY_CONSTRUCT()))) AS false_positives,
    num_expected - ARRAY_SIZE(ARRAY_INTERSECTION(extracted_codes_array, COALESCE(expected_codes_array, ARRAY_CONSTRUCT()))) AS false_negatives
FROM code_comparison
```

## eval_metrics_per_record.sql

```sql
{{ config(materialized='table', schema='<SCHEMA>') }}

SELECT *,
    CASE WHEN num_extracted > 0 THEN true_positives::FLOAT / num_extracted ELSE 0 END AS precision_score,
    CASE WHEN num_expected > 0 THEN true_positives::FLOAT / num_expected ELSE 0 END AS recall_score,
    CASE
        WHEN (num_extracted + num_expected) > 0
        THEN 2 * true_positives::FLOAT / (num_extracted + num_expected)
        ELSE 0
    END AS f1_score
FROM {{ ref('eval_code_matches') }}
```

## eval_experiment_summary.sql

```sql
{{ config(materialized='table', schema='<SCHEMA>') }}

SELECT
    experiment_id, experiment_name,
    COUNT(*) AS total_records,
    SUM(num_extracted) AS total_extracted_codes,
    SUM(num_expected) AS total_expected_codes,
    SUM(true_positives) AS total_true_positives,
    SUM(false_positives) AS total_false_positives,
    SUM(false_negatives) AS total_false_negatives,
    CASE WHEN SUM(true_positives + false_positives) > 0
         THEN ROUND(SUM(true_positives)::FLOAT / SUM(true_positives + false_positives), 4) ELSE 0 END AS micro_precision,
    CASE WHEN SUM(true_positives + false_negatives) > 0
         THEN ROUND(SUM(true_positives)::FLOAT / SUM(true_positives + false_negatives), 4) ELSE 0 END AS micro_recall,
    CASE WHEN SUM(true_positives + false_positives) > 0 AND SUM(true_positives + false_negatives) > 0
         THEN ROUND(
            2 * (SUM(true_positives)::FLOAT / SUM(true_positives + false_positives)) *
                (SUM(true_positives)::FLOAT / SUM(true_positives + false_negatives)) /
            ((SUM(true_positives)::FLOAT / SUM(true_positives + false_positives)) +
             (SUM(true_positives)::FLOAT / SUM(true_positives + false_negatives))), 4)
         ELSE 0 END AS micro_f1,
    ROUND(AVG(precision_score), 4) AS macro_precision,
    ROUND(AVG(recall_score), 4) AS macro_recall,
    ROUND(AVG(f1_score), 4) AS macro_f1,
    ROUND(MIN(f1_score), 4) AS min_f1,
    ROUND(MAX(f1_score), 4) AS max_f1,
    ROUND(STDDEV(f1_score), 4) AS stddev_f1,
    CURRENT_TIMESTAMP() AS evaluated_at
FROM {{ ref('eval_metrics_per_record') }}
GROUP BY experiment_id, experiment_name
```
