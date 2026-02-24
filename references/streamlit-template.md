# Streamlit App Template for ICD-10 LLMOps

Streamlit in Snowflake (SiS) dashboard for running the dbt pipeline and viewing results.

## Key Implementation Notes

- Uses `get_active_session()` (SiS -- no connection params needed)
- Pipeline trigger via `EXECUTE DBT PROJECT` SQL
- dbt outputs go to `<DATABASE>.<SCHEMA>_<SCHEMA>` (dbt appends custom schema)
- environment.yml must use only lowercase dependency names without version specifiers

## streamlit_app.py Template

```python
"""
ICD-10 LLMOps Demo - Streamlit in Snowflake (SiS)
dbt-powered LLM evaluation framework for medical coding extraction
"""
import streamlit as st
from snowflake.snowpark.context import get_active_session
import pandas as pd
from datetime import datetime
import json

session = get_active_session()

# Configuration -- update these for your environment
DATABASE = "<DATABASE>"
SCHEMA = "<SCHEMA>"
DBT_SCHEMA = "<SCHEMA>_<SCHEMA>"  # dbt output schema
DBT_PROJECT = "ICD10_LLMOPS_PROJECT"

st.set_page_config(page_title="ICD-10 LLMOps Demo", page_icon="", layout="wide")

if 'experiment_history' not in st.session_state:
    st.session_state.experiment_history = []

def run_query(sql):
    try:
        return session.sql(sql).to_pandas()
    except Exception as e:
        st.error(f"SQL Error: {e}")
        return pd.DataFrame()

def run_dbt_project():
    try:
        session.sql(f"EXECUTE DBT PROJECT {DATABASE}.{SCHEMA}.{DBT_PROJECT}").collect()
        return True, "Success", ""
    except Exception as e:
        return False, "", str(e)

# Header
st.title("ICD-10 Code Extraction - LLMOps Demo")
st.markdown("**dbt-powered evaluation framework using Snowflake Cortex**")

# Sidebar
st.sidebar.header("Configuration")
st.sidebar.info(f"**Database:** {DATABASE}\n**Source Schema:** {SCHEMA}\n**dbt Output Schema:** {DBT_SCHEMA}")

# Step 1: Source Data
st.header("Step 1: Source Data Overview")
col1, col2 = st.columns(2)

with col1:
    st.subheader("Patient Clinical Events")
    if st.button("Load Source Data", key="load_source"):
        df = run_query(f"""
            SELECT AGILON_MEMBER_ID AS PATIENT_ID, BATCH_NUMBER,
                   LEFT(BATCHED_EVENTS, 200) || '...' AS CLINICAL_PREVIEW,
                   MIN_EVENT_DATE, MAX_EVENT_DATE
            FROM {DATABASE}.{SCHEMA}.BATCHED_EVENTS ORDER BY AGILON_MEMBER_ID
        """)
        if not df.empty:
            st.dataframe(df, use_container_width=True)

with col2:
    st.subheader("Ground Truth Labels")
    if st.button("Load Ground Truth", key="load_gt"):
        df = run_query(f"""
            SELECT AGILON_MEMBER_ID AS PATIENT_ID, EXPECTED_ICD10_CODES, NOTES
            FROM {DATABASE}.{SCHEMA}.GROUND_TRUTH ORDER BY AGILON_MEMBER_ID
        """)
        if not df.empty:
            st.dataframe(df, use_container_width=True)

st.markdown("---")

# Step 2: Run Pipeline
st.header("Step 2: Run LLM Extraction Pipeline")
if st.button("Run dbt Pipeline", type="primary", key="run_dbt"):
    with st.spinner("Running dbt pipeline..."):
        success, stdout, stderr = run_dbt_project()
        if success:
            st.success("Pipeline completed!")
        else:
            st.error(f"Failed: {stderr}")

st.markdown("---")

# Step 3: Results
st.header("Step 3: LLM Extraction Results")
tab1, tab2, tab3 = st.tabs(["Summary", "Extracted Codes", "Raw JSON"])

with tab1:
    if st.button("Load Summary", key="load_summary"):
        df = run_query(f"""
            SELECT PATIENT_ID, BATCH_GROUP_RANGE, NUM_CODES_EXTRACTED,
                   LEFT(BATCH_GROUP_SUMMARY, 150) || '...' AS SUMMARY, MODEL_USED
            FROM {DATABASE}.{DBT_SCHEMA}.LLM_SP_RESULTS ORDER BY PATIENT_ID
        """)
        if not df.empty:
            st.dataframe(df, use_container_width=True)

with tab2:
    if st.button("Load Codes", key="load_codes"):
        df = run_query(f"""
            SELECT PATIENT_ID, ICD10_CODE, LEFT(JUSTIFICATION, 200) AS JUSTIFICATION, EVIDENCE_COUNT
            FROM {DATABASE}.{DBT_SCHEMA}.LLM_EXTRACTED_CODES ORDER BY PATIENT_ID, CODE_INDEX
        """)
        if not df.empty:
            st.dataframe(df, use_container_width=True)
            st.bar_chart(df['ICD10_CODE'].value_counts().head(10))

with tab3:
    if st.button("Load Raw JSON", key="load_json"):
        df = run_query(f"""
            SELECT PATIENT_ID, ICD10_CODES_JSON, BATCH_GROUP_SUMMARY
            FROM {DATABASE}.{DBT_SCHEMA}.LLM_SP_RESULTS ORDER BY PATIENT_ID LIMIT 5
        """)
        if not df.empty:
            for _, row in df.iterrows():
                with st.expander(f"Patient {row['PATIENT_ID']}"):
                    st.write(row['BATCH_GROUP_SUMMARY'])
                    try:
                        st.json(json.loads(row['ICD10_CODES_JSON']))
                    except:
                        st.code(str(row['ICD10_CODES_JSON']))

st.markdown("---")

# Step 4: Evaluation
st.header("Step 4: Evaluation Results")
tab1, tab2 = st.tabs(["Aggregate Metrics", "Per-Patient"])

with tab1:
    if st.button("Load Metrics", key="load_metrics"):
        df = run_query(f"""
            SELECT EXPERIMENT_NAME, TOTAL_RECORDS, TOTAL_EXTRACTED_CODES, TOTAL_EXPECTED_CODES,
                   ROUND(MICRO_PRECISION, 4) AS PRECISION,
                   ROUND(MICRO_RECALL, 4) AS RECALL,
                   ROUND(MICRO_F1, 4) AS F1
            FROM {DATABASE}.{DBT_SCHEMA}.EVAL_EXPERIMENT_SUMMARY ORDER BY EVALUATED_AT DESC LIMIT 5
        """)
        if not df.empty:
            latest = df.iloc[0]
            c1, c2, c3 = st.columns(3)
            c1.metric("Precision", f"{float(latest.get('PRECISION', 0)):.1%}")
            c2.metric("Recall", f"{float(latest.get('RECALL', 0)):.1%}")
            c3.metric("F1", f"{float(latest.get('F1', 0)):.1%}")
            st.dataframe(df, use_container_width=True)

with tab2:
    if st.button("Load Per-Patient", key="load_per_patient"):
        df = run_query(f"""
            SELECT PATIENT_ID, NUM_EXTRACTED, NUM_EXPECTED,
                   TRUE_POSITIVES, FALSE_POSITIVES, FALSE_NEGATIVES,
                   ROUND(PRECISION_SCORE, 2) AS PRECISION,
                   ROUND(RECALL_SCORE, 2) AS RECALL,
                   ROUND(F1_SCORE, 2) AS F1
            FROM {DATABASE}.{DBT_SCHEMA}.EVAL_METRICS_PER_RECORD ORDER BY F1_SCORE DESC
        """)
        if not df.empty:
            st.dataframe(df, use_container_width=True)
```

## environment.yml

```yaml
name: streamlit_env
channels:
  - snowflake
dependencies:
  - streamlit
  - pandas
  - snowflake-snowpark-python
```

**Important:** Do NOT add version specifiers (e.g., `streamlit>=1.48.0`). Snowflake SiS rejects dependency names with `>`, `<`, `=` characters.

## snowflake.yml

```yaml
definition_version: 2
entities:
  icd10_llmops_demo:
    type: streamlit
    identifier:
      name: ICD10_LLMOPS_DEMO
    title: "ICD-10 LLMOps Demo"
    query_warehouse: <WAREHOUSE>
    main_file: streamlit_app.py
    artifacts:
      - environment.yml
```
