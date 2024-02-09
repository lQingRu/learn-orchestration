# Orchestration

## Main steps
1. Setup database
- Populate `mapping_collections` + indexes
2. Start `workflow-ab`
3. When `workflow-ab` ended, start `workflow-cde`
4. [Stretch] Run post-validation scripts
5. [Stretch] Run validation e2e scripts
6. If validation okay (or minimally the sanity checks for exception case), proceed to do indexing(?) with manual-indexer
- TODO: Setup ease of indexing to different ES indexes


## Housekeeping steps
1. Remove old mongo collections
2. Periodic clean-up for `xx_BATCH_xx` tables

## Tools

- Argo
  - Mainly used for K8s workflow
  - Each task will run in a K8s container
- Prefect
  - Not as mature as Airflow
  - Does not have as wide spread integration with third party tools unlike Airflow
- Airflow
- Spring cloud data flow
- Celery

## Airflow
- Workers
- DAG
  - Tasks
  - Dependencies
  - Schedule

