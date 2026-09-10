# Datavalidation

Wiki: Data Validation during Disaster Recovery (DR)
1. Purpose & Scope
This document establishes the official Standard Operating Procedure (SOP) and data quality assurance framework for validating data integrity, volume parity, and access enablement during a Disaster Recovery (DR) event between Snowflake and Microsoft Fabric.

Primary Objectives
Data Reload Validation (Snowflake $\rightarrow$ Fabric): Verify that data reloaded from Snowflake (CDWP_DB_PROD) into Microsoft Fabric Lakehouse (lh_Bronze) achieves 100% volume parity, zero schema drift, and exact metric checksum matching post-failover.


Data Sharing / Reverse Sync Validation (Fabric $\rightarrow$ Snowflake): Confirm that curated views and key ring entities generated in Fabric (lh_Silver) and exposed back to Snowflake are active, queryable, and synchronized within acceptable latency bounds.


Rapid Incident Remediation: Provide on-call data engineers with automated scripts, acceptance thresholds, and escalation protocols to meet organizational RPO/RTO SLAs.


2. Architecture & Data Flow
+------------------------------------+         (Data Reload: Step 1)        +------------------------------------+
|             SNOWFLAKE              | -----------------------------------> |          FABRIC LAKEHOUSE          |
|          (CDWP_DB_PROD)            |                                      |            (lh_Bronze)             |
|  - MDM_INDIVIDUAL_DIM_WH           |                                      |  - USER_DATAPRODUCTS_..._DIM_WH    |
|  - APPLICATION_POLICY_IND_DIM_WH   |                                      |  - USER_DATAPRODUCTS_..._FACT_WH   |
|  - APPLICATION_POLICY_FACT_WH      |                                      |                                    |
+------------------------------------+                                      +------------------------------------+
                  ▲                                                                            │
                  │                                                                            │ (Transformation)
                  │                                                                            ▼
+------------------------------------+        (Data Sharing: Step 2)        +------------------------------------+
|             SNOWFLAKE              | <----------------------------------- |          FABRIC LAKEHOUSE          |
|      (Shared Consumer Views)       |                                      |            (lh_Silver)             |
|  - shared_key_ring_entity          |                                      |  - key_ring_entity                 |
|  - shared_key_ring_identifier      |                                      |  - key_ring_identifier             |
+------------------------------------+                                      +------------------------------------+

Direction,Source System / Database,Source Object / Table Name,Target System / Database,Target Object / Table Name
Snowflake → Fabric,Snowflake (CDWP_DB_PROD),USER_DATAPRODUCTS.MDM_INDIVIDUAL_DIM_WH,Fabric (lh_Bronze),[cdwp].[USER_DATAPRODUCTS_MDM_INDIVIDUAL_DIM_WH]
Snowflake → Fabric,Snowflake (CDWP_DB_PROD),USER_DATAPRODUCTS.APPLICATION_POLICY_INDIVIDUAL_DIM_WH,Fabric (lh_Bronze),[cdwp].[USER_DATAPRODUCTS_APPLICATION_POLICY_INDIVIDUAL_DIM_WH]
Snowflake → Fabric,Snowflake (CDWP_DB_PROD),USER_DATAPRODUCTS.APPLICATION_POLICY_FACT_WH,Fabric (lh_Bronze),[cdwp].[USER_DATAPRODUCTS_APPLICATION_POLICY_FACT_WH]
Fabric → Snowflake,Fabric (lh_Silver),[dbo].[key_ring_entity],Snowflake (Consumer),Shared Direct Lake / Delta Endpoint
Fabric → Snowflake,Fabric (lh_Silver),[dbo].[key_ring_identifier],Snowflake (Consumer),Shared Direct Lake / Delta Endpoint

3. Enterprise Data Quality (DQ) Rule Matrix
During DR recovery, all validation scripts enforce the seven core dimensions of enterprise data quality across both reload and sharing paths:

DQ Dimension
Target Rule & Assertion
Validation Logic
Acceptance Threshold
Completeness
Zero missing records; zero unexpected NULLs in mandatory primary keys.
COUNT(*), COUNT(col) WHERE col IS NULL
Exact $0\%$ variance
Uniqueness
Primary/Composite keys must remain strictly unique; zero duplicate rows allowed.
COUNT(pk) - COUNT(DISTINCT pk)
Exact $0$ duplicates
Accuracy / Fidelity
Hash totals and distinct entity counts must match source metrics precisely.
COUNT(DISTINCT entity_id), BITXOR_AGG(HASH(*))
Exact $0\%$ variance
Timeliness
Data freshness must satisfy the Recovery Point Objective (RPO $\le 1$ hr).
MAX(ETL_UPDATE_TIMESTAMP) >= Cutoff
Data within $\le 1$ hr of DR event
Validity
Attributes must adhere to domain rules and active record flags.
COUNT(*) WHERE RECORD_ACTIVE_FLAG = 'Y'
$0$ invalid code records
Consistency
Column data types, ordinal positions, and schema precision must match.
System metadata catalog comparison
$0$ schema drift errors
Integrity
Foreign key entity links between dimension and fact tables must remain intact.
LEFT JOIN orphan detection queries
$0$ orphan records

4. Standard Validation Procedures
A. Reload Validation Procedure (Snowflake $\rightarrow$ Fabric)
Pre-Validation Baseline: Execute Snowflake SQL verification queries on CDWP_DB_PROD to extract total record counts, distinct key totals, and max update timestamps.


Reload Execution: Execute the DR reload pipelines into Fabric Lakehouse lh_Bronze.


Automated Reconciliation: Run the PySpark Automated DR Reconciliation Notebook in Microsoft Fabric.


Sign-Off Gate: Confirm that row count discrepancy is $0$ and maximum timestamp lag is within the $1$-hour RPO window.


B. Sharing Validation Procedure (Fabric $\rightarrow$ Snowflake)
Endpoint Health Check: Verify that the Delta Share / Direct Lake connection endpoint from Fabric lh_Silver is active.


Permission Check: Execute test read queries (SELECT TOP 10) from Snowflake using designated consumer roles.


Data Freshness Check: Compare MAX(updated_at) on the shared views against target DR SLAs to confirm near-real-time synchronization.


5. Validation Query & Code Library
5.1 Snowflake Source Baseline Queries (SQL)
Execute in Snowflake (CDWP_DB_PROD) to establish source baselines prior to DR verification:

SQL
-- ====================================================================
-- Snowflake DR Baseline Query
-- ====================================================================
SELECT 
    'MDM_INDIVIDUAL_DIM_WH' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT CURRENT_MDM_INDIVIDUAL_ID) AS distinct_keys,
    MAX(ETL_UPDATE_TIMESTAMP) AS latest_rpo_timestamp
FROM CDWP_DB_PROD.USER_DATAPRODUCTS.MDM_INDIVIDUAL_DIM_WH

UNION ALL

SELECT 
    'APPLICATION_POLICY_INDIVIDUAL_DIM_WH' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT APPLICATION_POLICY_INDIVIDUAL_KEY) AS distinct_keys,
    MAX(ETL_UPDATE_TIMESTAMP) AS latest_rpo_timestamp
FROM CDWP_DB_PROD.USER_DATAPRODUCTS.APPLICATION_POLICY_INDIVIDUAL_DIM_WH

UNION ALL

SELECT 
    'APPLICATION_POLICY_FACT_WH' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT APPLICATION_NUMBER) AS distinct_keys,
    MAX(ETL_UPDATE_TIMESTAMP) AS latest_rpo_timestamp
FROM CDWP_DB_PROD.USER_DATAPRODUCTS.APPLICATION_POLICY_FACT_WH;


5.2 Fabric Lakehouse Target Queries (T-SQL)
Execute against Microsoft Fabric SQL Endpoint (lh_Bronze):

SQL
-- ====================================================================
-- Fabric Bronze Target Verification
-- ====================================================================
SELECT 
    'MDM_INDIVIDUAL_DIM_WH' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT CURRENT_MDM_INDIVIDUAL_ID) AS distinct_keys
FROM [lh_Bronze].[cdwp].[USER_DATAPRODUCTS_MDM_INDIVIDUAL_DIM_WH]

UNION ALL

SELECT 
    'APPLICATION_POLICY_INDIVIDUAL_DIM_WH' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT APPLICATION_POLICY_INDIVIDUAL_KEY) AS distinct_keys
FROM [lh_Bronze].[cdwp].[USER_DATAPRODUCTS_APPLICATION_POLICY_INDIVIDUAL_DIM_WH]

UNION ALL

SELECT 
    'APPLICATION_POLICY_FACT_WH' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT APPLICATION_NUMBER) AS distinct_keys
FROM [lh_Bronze].[cdwp].[USER_DATAPRODUCTS_APPLICATION_POLICY_FACT_WH];


5.3 Automated Cross-System Reconciliation Script (PySpark in Fabric)
Run this notebook in Microsoft Fabric during DR failover to perform automated source-to-target reconciliation:

Python
# ====================================================================
# PySpark Automated DR Reconciliation Engine
# ====================================================================
from pyspark.sql.functions import col, count

# 1. Configure Snowflake Connection Parameters
sf_options = {
    "sfUrl": "account.snowflakecomputing.com",
    "sfUser": dbutils.secrets.get(scope="key-vault-scope", key="snowflake-user"),
    "sfPassword": dbutils.secrets.get(scope="key-vault-scope", key="snowflake-password"),
    "sfDatabase": "CDWP_DB_PROD",
    "sfSchema": "USER_DATAPRODUCTS",
    "sfWarehouse": "COMPUTE_WH"
}

tables_to_validate = [
    ("MDM_INDIVIDUAL_DIM_WH", "USER_DATAPRODUCTS_MDM_INDIVIDUAL_DIM_WH"),
    ("APPLICATION_POLICY_INDIVIDUAL_DIM_WH", "USER_DATAPRODUCTS_APPLICATION_POLICY_INDIVIDUAL_DIM_WH"),
    ("APPLICATION_POLICY_FACT_WH", "USER_DATAPRODUCTS_APPLICATION_POLICY_FACT_WH")
]

print("--- STARTING AUTOMATED DR RECONCILIATION ---")

for sf_table, fabric_table in tables_to_validate:
    # Read Snowflake Source
    df_sf = spark.read \
        .format("net.snowflake.spark.snowflake") \
        .options(**sf_options) \
        .option("dbtable", sf_table) \
        .load()
    
    # Read Fabric Target
    df_fabric = spark.read.table(f"lh_Bronze.cdwp.{fabric_table}")
    
    sf_cnt = df_sf.count()
    fabric_cnt = df_fabric.count()
    diff = sf_cnt - fabric_cnt
    
    status = "PASSED" if diff == 0 else "FAILED"
    print(f"[{status}] Table: {sf_table} | SF Count: {sf_cnt} | Fabric Count: {fabric_cnt} | Variance: {diff}")
    
    assert diff == 0, f"DR CRITICAL FAILURE: Row count mismatch in {sf_table}! Discrepancy: {diff}"

print("--- ALL DR TABLE PARITY CHECKS COMPLETED SUCCESSFULLY ---")


5.4 Fabric to Snowflake Sharing Validation Queries (Executed from Snowflake)
Verify read accessibility and data latency on shared objects from Snowflake:

SQL
-- ====================================================================
-- Snowflake Consumer Readability & Latency Test
-- ====================================================================
-- 1. Test Shared Key Ring Entity Connectivity
SELECT TOP 10 * 
FROM shared_fabric_db.shared_schema.key_ring_entity;

-- 2. Test Data Freshness & Shared Volume
SELECT 
    CURRENT_TIMESTAMP() AS query_execution_time,
    COUNT(*) AS total_shared_entities,
    MAX(created_at) AS latest_shared_record
FROM shared_fabric_db.shared_schema.key_ring_entity;


6. Disaster Recovery (DR) Sign-Off Checklist
This checklist must be executed and signed off by the On-Call Data Engineer before declaring recovery complete:

Step #
Flow Direction
Target Table / Endpoint
Validation Task
Acceptance Criteria
Status
Sign-Off
1
Snowflake $\rightarrow$ Fabric
USER_DATAPRODUCTS_MDM_INDIVIDUAL_DIM_WH
Row Count Parity
$0$ row variance
[ ]


2
Snowflake $\rightarrow$ Fabric
USER_DATAPRODUCTS_APPLICATION_POLICY_INDIVIDUAL_DIM_WH
Row Count Parity
$0$ row variance
[ ]


3
Snowflake $\rightarrow$ Fabric
USER_DATAPRODUCTS_APPLICATION_POLICY_FACT_WH
Row Count Parity
$0$ row variance
[ ]


4
Snowflake $\rightarrow$ Fabric
All lh_Bronze Tables
RPO Timeliness
Max timestamp $\le 1$ hr lag
[ ]


5
Fabric $\rightarrow$ Snowflake
key_ring_entity
Share Connectivity
SELECT TOP 10 succeeds
[ ]


6
Fabric $\rightarrow$ Snowflake
key_ring_identifier
Shared Data Freshness
Latest timestamp matches Fabric
[ ]



7. DR SLA Benchmarks
Recovery Point Objective (RPO): Maximum acceptable data loss threshold is $\le 1$ hour from the last verified Snowflake snapshot.


Recovery Time Objective (RTO): Full validation suite execution must complete within 15 minutes post-pipeline reload completion.


8. Emergency Escalation & Remediation Protocol
[ Step 1: Detect DQ Failure ] ──► [ Step 2: Rerun PySpark Script ] ──► [ Step 3: Isolate Missing Window ] ──► [ Step 4: Trigger Partial Reload ]
                                                                                                                   │
                                                                                                                   ▼
                                                                                                    [ Step 5: Escalate to On-Call ]


Transient Failure Check: On initial failure, re-run the PySpark reconciliation notebook once to clear potential write-lock conflicts.


Isolate Partition Gap: Execute MIN(ETL_UPDATE_TIMESTAMP) and MAX(ETL_UPDATE_TIMESTAMP) on both sides to identify the exact missing load window.


Trigger Catch-Up Pipeline: Re-execute the specific Azure Data Factory / Fabric Data Factory pipeline for the missing time window.


Escalate: If row count variance persists beyond 30 minutes, notify the Data Platform (DP) On-Call Lead via Slack/Teams alert webhook.

