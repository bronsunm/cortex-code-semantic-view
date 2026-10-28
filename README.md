# cortex-code-semantic-view

A small demo of **Snowflake Semantic Studio**, which became generally available on **30 September 2026**. It builds a semantic view over insurance claims and vehicles, puts a Cortex Agent on top of it, and keeps the definition in Git.

## What Semantic Studio gives you

| Feature | What it means |
|---|---|
| Conversational authoring with CoCo | Describe the semantic view in plain language; CoCo creates and refines it |
| YAML editing | Each view is a `.sv.yaml` file you can edit by hand or with CoCo |
| Deployment as Snowflake objects | The YAML is deployed to a real `SEMANTIC VIEW` object that Cortex Analyst and Cortex Agents can query |
| Git-backed version control | Commit, push and pull the YAML to review changes and track history |

Semantic Studio lives in **Snowsight » Projects » Workspaces**. To create a view, select **Add new » Semantic View**. To open an existing one, select **AI & ML » Cortex Analyst**, or ask CoCo to open it by name.

## What is in this repo

| File | Purpose |
|---|---|
| `INSURANCE_UPDATES_SEPT.sv.yaml` | Semantic view definition over `FACT_CLAIMS` and `DIM_VEHICLE` |
| `README.md` | This guide |

### Objects created in Snowflake (all in `DEV_DEMO_INSURANCE_SI.CUSTOMER_360`, owned by `SYSADMIN`)

| Object | Type |
|---|---|
| `INSURANCE_UPDATES_SEPT` | Semantic view |
| `INSURANCE_UPDATES_SEPT_AGENT` | Cortex Agent (uses the semantic view as a Cortex Analyst tool) |
| `CORTEX_CODE_SEMANTIC_VIEW` | Git repository object pointing at this GitHub repo |

### What the semantic view covers

- **Tables:** `FACT_CLAIMS` (as `CLAIMS`) and `DIM_VEHICLE` (as `VEHICLES`), joined on `VEHICLE_ID`.
- **Metrics:** `CLAIM_COUNT`, `TOTAL_CLAIM_COST`, `AVG_CLAIM_COST`.
- **Dimensions:** claim type, severity, status, product, period (YYYYMM), vehicle make, model, year and category.
- **Note:** claims with no vehicle (Home and Contents) show as a blank group when you break results down by vehicle.

## Prerequisites

- A role that can create objects in the target schema. This demo uses `SYSADMIN`, which owns the schema.
- These privileges on the role:
  - `USAGE` on the database and schema
  - `CREATE SEMANTIC VIEW` and `CREATE AGENT` on the schema
  - `SELECT` on `FACT_CLAIMS` and `DIM_VEHICLE`
  - `USAGE` on a warehouse (this demo uses `COCO_HUB_WH`)
- At least one supported LLM available in the account. Check with:
  ```sql
  SHOW MODELS LIKE 'claude-4-sonnet' IN SNOWFLAKE.MODELS;
  ```
- For Git: an API integration (this demo uses `GIT_BRONSUNM`), a secret holding a GitHub username and token with write access (`GITHUB-SECRET-BRONSUN`), and an empty GitHub repository.

## Step by step

### 1. Create the Git repository object

Create the empty repository on GitHub first, then connect it:

```sql
USE ROLE SYSADMIN;

CREATE OR REPLACE GIT REPOSITORY DEV_DEMO_INSURANCE_SI.CUSTOMER_360.CORTEX_CODE_SEMANTIC_VIEW
  API_INTEGRATION = GIT_BRONSUNM
  GIT_CREDENTIALS = DEV_DEMO_INSURANCE_SI.CUSTOMER_360."GITHUB-SECRET-BRONSUN"
  ORIGIN = 'https://github.com/bronsunm/cortex-code-semantic-view';

ALTER GIT REPOSITORY DEV_DEMO_INSURANCE_SI.CUSTOMER_360.CORTEX_CODE_SEMANTIC_VIEW FETCH;
SHOW GIT BRANCHES IN GIT REPOSITORY DEV_DEMO_INSURANCE_SI.CUSTOMER_360.CORTEX_CODE_SEMANTIC_VIEW;
```

### 2. Create a Git-backed workspace

1. In Snowsight, go to **Projects » Workspaces**.
2. Select **From Git repository** and choose this repo, using the same API integration and credentials.
3. This workspace is where you edit the YAML and commit. A shared workspace cannot be connected to Git, so use a personal one.

### 3. Author the semantic view

Pick either route.

**Route A: with CoCo (recommended for the demo).** In the workspace, ask CoCo, for example:

> Create a semantic view called INSURANCE_UPDATES_SEPT in DEV_DEMO_INSURANCE_SI.CUSTOMER_360 from FACT_CLAIMS and DIM_VEHICLE, join them on VEHICLE_ID, and add metrics for claim count, total claim cost and average claim cost.

CoCo writes `INSURANCE_UPDATES_SEPT.sv.yaml` into the workspace.

**Route B: by hand.** Create or edit `INSURANCE_UPDATES_SEPT.sv.yaml` directly. You can mix both: edit the YAML, then ask CoCo to make a further change.

### 4. Edit the YAML

Open `INSURANCE_UPDATES_SEPT.sv.yaml` and make a change, for example add synonyms to the `TOTAL_CLAIM_COST` metric:

```yaml
- name: TOTAL_CLAIM_COST
  expr: SUM(CLAIMS.CLAIM_COST)
  description: Total claim cost in NZD.
  synonyms:
    - losses
    - incurred claims
```

### 5. Validate and deploy the semantic view

Deploy from Semantic Studio, or with SQL. The last argument of `SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML` is `TRUE` to validate only and `FALSE` to create:

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COCO_HUB_WH;

-- validate only (creates nothing)
CALL SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML(
  'DEV_DEMO_INSURANCE_SI.CUSTOMER_360',
  $$<paste the contents of INSURANCE_UPDATES_SEPT.sv.yaml>$$,
  TRUE);

-- create or replace the semantic view
CALL SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML(
  'DEV_DEMO_INSURANCE_SI.CUSTOMER_360',
  $$<paste the contents of INSURANCE_UPDATES_SEPT.sv.yaml>$$,
  FALSE);
```

Deploying replaces the existing view, so the role needs the privileges listed above.

### 6. Test the semantic view

```sql
SELECT * FROM SEMANTIC_VIEW(
  DEV_DEMO_INSURANCE_SI.CUSTOMER_360.INSURANCE_UPDATES_SEPT
  DIMENSIONS VEHICLES.VEHICLE_CATEGORY
  METRICS CLAIMS.TOTAL_CLAIM_COST, CLAIMS.CLAIM_COUNT
) ORDER BY 2 DESC;
```

Test run results:

| Vehicle category | Total claim cost (NZD) | Claims |
|---|---|---|
| (none) | 35,200,286 | 6,022 |
| SUV | 18,136,694 | 3,122 |
| Ute | 6,222,696 | 981 |
| Sedan | 4,734,999 | 713 |
| Hatchback | 3,142,211 | 590 |

### 7. Create the Cortex Agent

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE COCO_HUB_WH;

CREATE OR REPLACE AGENT DEV_DEMO_INSURANCE_SI.CUSTOMER_360.INSURANCE_UPDATES_SEPT_AGENT
  COMMENT = 'Insurance claims by vehicle - Semantic Studio demo'
  FROM SPECIFICATION $$
models:
  orchestration: auto
instructions:
  response: "Answer concisely. Currency is NZD. State that claims with no vehicle are Home or Contents claims when vehicle breakdowns show a blank group."
  sample_questions:
    - question: "What is the total claim cost by vehicle category?"
    - question: "Which vehicle makes have the highest average claim cost?"
    - question: "How many claims are open vs closed?"
tools:
  - tool_spec:
      type: cortex_analyst_text_to_sql
      name: claims_analyst
      description: "Claims cost, counts and averages by claim type, severity, status, product and vehicle."
tool_resources:
  claims_analyst:
    semantic_view: DEV_DEMO_INSURANCE_SI.CUSTOMER_360.INSURANCE_UPDATES_SEPT
    execution_environment:
      type: warehouse
      warehouse: COCO_HUB_WH
$$;
```

### 8. Try the agent

In Snowsight, open **AI & ML » Agents**, select `INSURANCE_UPDATES_SEPT_AGENT` and ask the sample questions.

### 9. Share access (optional)

Grant the roles that should use the objects:

```sql
GRANT SELECT ON SEMANTIC VIEW DEV_DEMO_INSURANCE_SI.CUSTOMER_360.INSURANCE_UPDATES_SEPT TO ROLE <role>;
GRANT USAGE ON AGENT DEV_DEMO_INSURANCE_SI.CUSTOMER_360.INSURANCE_UPDATES_SEPT_AGENT TO ROLE <role>;
```

Those roles also need `USAGE` on the database, schema and warehouse.

### 10. Commit to Git

From the Git-backed workspace, use the Git panel:

1. Review the changed files (the diff shows your YAML edits).
2. Enter a commit message.
3. Commit and push to `main`, or to a branch for review.

To bring changes made elsewhere into Snowflake, run `ALTER GIT REPOSITORY ... FETCH;` and pull in the workspace.

## Suggested demo flow

1. Show the empty repo, then the Git-backed workspace.
2. Ask CoCo to build the semantic view (step 3).
3. Make a manual YAML edit, then a second change through CoCo (step 4).
4. Deploy and run the test query (steps 5 and 6).
5. Create the agent and ask it a question (steps 7 and 8).
6. Commit the change and show the Git diff (step 10).

## Notes and limits

- Pushing commits from SQL is not possible. A Snowflake Git repository object can fetch from GitHub; commits and pushes are done from the Git-backed workspace.
- The token in the secret needs write access to the GitHub repo for pushes to work.
- Deploying a semantic view replaces any existing view with the same name.
- Preview-era request-ID debugging was mentioned in the August 2026 preview notes but not in the GA release note, so it is not covered here.

## References

- [Semantic Studio](https://docs.snowflake.com/en/user-guide/views-semantic/semantic-studio)
- [Semantic View Autopilot](https://docs.snowflake.com/en/user-guide/views-semantic/autopilot)
- [Sep 30, 2026: Semantic Studio GA release note](https://docs.snowflake.com/en/release-notes/2026/other/2026-09-30-semantic-studio-ga)
