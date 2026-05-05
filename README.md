# ice-cream-dataops

A [Cognite Data Fusion (CDF)](https://docs.cognite.com/cdf/) DataOps project built during the CDF Bootcamp. It implements a full end-to-end industrial data pipeline — from raw API ingestion through asset modelling to OEE (Overall Equipment Effectiveness) analytics — using [Cognite Toolkit](https://developer.cognite.com/dev/guides/toolkit/).

---

## Overview

The pipeline ingests operational data from a simulated **Ice Cream Factory REST API** across **10 global manufacturing sites**, models it in CDF using the Cognite Core Data Model, and computes OEE metrics per equipment. All resources (groups, datasets, data models, transformations, functions, workflows, hosted extractors) are defined as code and deployed via CI/CD.

**Sites covered:** Houston · Oslo · Kuala Lumpur · Hannover · Nuremberg · Marseille · São Paulo · Chicago · Rotterdam · London

---

## Architecture

```
Ice Cream Factory API (REST)
        │
        ▼
Hosted Extractor  ──────────────────────────────────────────────►  RAW Table
  (assets + timeseries)                                         (ice-cream-factory-db.assets)
                                                                        │
                                          ┌─────────────────────────────┘
                                          ▼
                              Transformation: create_asset_hierarchy
                              (RAW → CogniteAsset nodes in icapi_dm_space)
                                          │
                                          ▼
                              Transformation: contextualize_ts_assets
                              (link CogniteTimeSeries → CogniteAsset in cdf_cdm)
                                          │
                                          ▼
                              CDF Function: icapi_datapoints_extractor
                              (fetch datapoints per site from Ice Cream Factory API)
                                          │
                                          ▼
                              CDF Function: oee_timeseries
                              (calculate OEE metrics → write to oee_ts_space)
```

All four steps are orchestrated by the **`wf_icapi_data_pipeline`** CDF Workflow.

---

## Repository Structure

```
ice-cream-dataops/
├── cdf.toml                          # Cognite Toolkit root config
├── pyproject.toml                    # Python project / dependency manifest
├── .devcontainer/devcontainer.json   # Dev Container (Python 3.11 + Poetry)
├── .github/workflows/deploy.yaml     # CI/CD pipeline (GitHub Actions)
└── ice-cream-dataops/                # Toolkit organization directory
    ├── config.test.yaml              # Environment config for 'test'
    ├── config.prod.yaml              # Environment config for 'prod'
    ├── build_info.test.yaml          # Build metadata
    ├── function_local_venvs/         # Local venvs for function development & testing
    │   ├── icapi_datapoints_extractor/
    │   └── oee_timeseries/
    └── modules/bootcamp/
        ├── data_foundation/          # Base IAM / group setup
        │   └── auth/
        │       └── data_developer.Group.yaml
        ├── ice_cream_api/            # Data ingestion module
        │   ├── auth/                 # icapi_extractors group + capabilities
        │   ├── data_models/          # icapi_dm_space space definition
        │   ├── data_sets/            # ds_icapi dataset
        │   ├── extraction_pipelines/ # ep_icapi_datapoints pipeline
        │   ├── functions/            # icapi_datapoints_extractor CDF Function
        │   │   └── icapi_datapoints_extractor/
        │   │       ├── handler.py
        │   │       ├── ice_cream_factory_api.py
        │   │       └── requirements.txt
        │   ├── hosted_extractors/    # Hosted Extractor (assets + timeseries jobs)
        │   │   └── Ice Cream Factory API/
        │   │       ├── Source.yaml
        │   │       ├── assets/       # Destination, Job, Mapping
        │   │       └── timeseries/   # Destination, Job, Mapping
        │   ├── raw/                  # RAW table definition
        │   ├── transformations/      # SQL transformations + schedules
        │   └── workflows/            # wf_icapi_data_pipeline (Workflow + Trigger + Version)
        └── use_cases/oee/            # OEE use-case module
            ├── auth/                 # data_pipeline_oee group + capabilities
            ├── data_models/          # oee_ts_space space definition
            ├── data_sets/            # ds_uc_oee dataset
            └── functions/            # oee_timeseries CDF Function
                └── oee_timeseries/
                    ├── handler.py
                    └── requirements.txt
```

---

## Modules

### `data_foundation`

Provisions the **`data_developer`** CDF group with broad permissions (datasets, data models, functions, hosted extractors, transformations, workflows, etc.) used by developers to manage the full platform.

### `ice_cream_api`

The core data ingestion module. It manages:

| Resource | External ID | Description |
|---|---|---|
| Dataset | `ds_icapi` | Scopes all Ice Cream Factory source data |
| RAW Table | `ice-cream-factory-db.assets` | Landing zone for asset metadata |
| Data Space | `icapi_dm_space` | Holds CogniteAsset + CogniteTimeSeries nodes |
| Hosted Extractor (Source) | `Ice Cream Factory API` | REST source at `ice-cream-factory.inso-internal.cognite.ai` |
| Hosted Extractor (Jobs) | `ICAPI Assets`, `ICAPI Time series` | Pull assets and OEE timeseries every hour |
| Transformation | `create_asset_hierarchy` | RAW assets → CogniteAsset DM nodes |
| Transformation | `contextualize_ts_assets` | Links CogniteTimeSeries nodes to CogniteAsset nodes |
| CDF Function | `icapi_datapoints_extractor` | Fetches up to 336 hours of datapoints from the API |
| Extraction Pipeline | `ep_icapi_datapoints` | Tracks extractor run status |
| Workflow | `wf_icapi_data_pipeline` | Orchestrates all four pipeline steps in order |
| IAM Group | `icapi_extractors` | Least-privilege service account for the extractor |

#### Workflow task order

```
create_asset_hierarchy  →  contextualize_ts_assets  →  icapi_datapoints_extractor  →  oee_timeseries
```

Each task has 3 retries with a 1-hour timeout. The workflow aborts on any task failure.

### `use_cases/oee`

Computes **Overall Equipment Effectiveness** metrics for every piece of equipment at every site and writes the results back to CDF.

| Resource | External ID | Description |
|---|---|---|
| Dataset | `ds_uc_oee` | Scopes OEE output data |
| Data Space | `oee_ts_space` | Holds calculated OEE CogniteTimeSeries nodes |
| CDF Function | `oee_timeseries` | Calculates and writes OEE metrics (runs per site in parallel) |
| IAM Group | `data_pipeline_oee` | Least-privilege service account for the OEE function |

#### OEE Calculation (per equipment, per minute)

| Metric | Formula |
|---|---|
| **Availability** | `status / planned_status` |
| **Performance** | `(count / status) / (60 / 3)` |
| **Quality** | `good / count` |
| **OEE** | `Availability × Performance × Quality` |
| **Off-Spec** | `count − good` |

Input timeseries (`count`, `good`, `status`, `planned_status`) are read from `icapi_dm_space`. Output timeseries (`oee`, `quality`, `performance`, `availability`, `off_spec`) are written to `oee_ts_space`. Missing output timeseries are created on the fly.

---

## Environments

| Environment | CDF Project | Deployment trigger |
|---|---|---|
| `test` | `cdf-bootcamp-71-test` | Auto on push to `main` |
| `prod` | `cdf-bootcamp-71-prod` | Manual workflow dispatch |

Both environments use the same module set (`modules/`) and the same variable structure; values are injected from GitHub Actions environment secrets/variables.

**CDF Cluster:** `westeurope-1`  
**Identity Provider:** Azure AD (tenant `16e3985b-ebe8-4e24-9da4-933e21a9fc81`)  
**Auth flow:** `client_credentials`

---

## Prerequisites

- Python ≥ 3.11
- [Poetry](https://python-poetry.org/) ≥ 2.0
- [Cognite Toolkit CLI](https://developer.cognite.com/dev/guides/toolkit/) `0.6.53` (`cognite-toolkit`)
- Access to a CDF project (`cdf-bootcamp-71-test` or `cdf-bootcamp-71-prod`)
- An Azure AD service principal with the required client ID and secret

---

## Local Setup

```bash
# Clone the repo
git clone https://github.com/<org>/ice-cream-dataops.git
cd ice-cream-dataops

# Install dependencies
poetry install

# Activate the virtual environment
poetry shell
```

### Configure environment variables

Create a `.env` file (never commit this):

```dotenv
CDF_PROJECT=cdf-bootcamp-71-test
CDF_CLUSTER=westeurope-1
IDP_TENANT_ID=16e3985b-ebe8-4e24-9da4-933e21a9fc81
IDP_TOKEN_URL=https://login.microsoftonline.com/16e3985b-ebe8-4e24-9da4-933e21a9fc81/oauth2/v2.0/token
LOGIN_FLOW=client_credentials

IDP_CLIENT_ID=<your-client-id>
IDP_CLIENT_SECRET=<your-client-secret>

DATA_DEVELOPER_SOURCE_ID=<azure-group-object-id>
ICAPI_EXTRACTORS_CLIENT_ID=<icapi-client-id>
ICAPI_EXTRACTORS_CLIENT_SECRET=<icapi-client-secret>
ICAPI_EXTRACTORS_SOURCE_ID=<icapi-group-object-id>
DATA_PIPELINE_OEE_CLIENT_ID=<oee-client-id>
DATA_PIPELINE_OEE_CLIENT_SECRET=<oee-client-secret>
DATA_PIPELINE_OEE_SOURCE_ID=<oee-group-object-id>
```

### Build and deploy

```bash
# Verify authentication
cdf auth verify

# Build configs for the test environment
cdf build --env=test

# Deploy to the test environment
cdf deploy --env=test

# Deploy to prod (manual)
cdf deploy --env=prod
```

---

## CI/CD

Deployments are handled by `.github/workflows/deploy.yaml` using the official `cognite/toolkit:0.6.53` Docker image.

| Trigger | Environment deployed |
|---|---|
| Push to `main` (changes under `ice-cream-dataops/**`) | `test` |
| Manual `workflow_dispatch` → pick `test` or `prod` | selected environment |

**Required GitHub secrets** (per environment):

| Secret | Description |
|---|---|
| `IDP_CLIENT_SECRET` | Service principal secret for toolkit auth |
| `ICAPI_EXTRACTORS_CLIENT_SECRET` | Secret for the ICAPI extractor service principal |
| `DATA_PIPELINE_OEE_CLIENT_SECRET` | Secret for the OEE pipeline service principal |

**Required GitHub variables** (per environment):

| Variable | Description |
|---|---|
| `IDP_CLIENT_ID` | Client ID for toolkit auth |
| `DATA_DEVELOPER_SOURCE_ID` | Azure AD group object ID for data_developer |
| `ICAPI_EXTRACTORS_CLIENT_ID` | Client ID for ICAPI extractor |
| `ICAPI_EXTRACTORS_SOURCE_ID` | Azure AD group object ID for icapi_extractors |
| `DATA_PIPELINE_OEE_CLIENT_ID` | Client ID for OEE pipeline |
| `DATA_PIPELINE_OEE_SOURCE_ID` | Azure AD group object ID for data_pipeline_oee |

---

## Dev Container

A `.devcontainer/devcontainer.json` is included for VS Code / GitHub Codespaces. It uses the `mcr.microsoft.com/devcontainers/python:3.11` base image with the Poetry feature pre-installed — no local Python setup required.

---

## Function Development

Local virtual environments for each CDF Function live under `function_local_venvs/`. These are used to develop and import-check functions before deploying, without needing a live CDF environment.

```bash
# Example: test imports for the datapoints extractor
cd ice-cream-dataops/function_local_venvs/icapi_datapoints_extractor
pip install -r requirements.txt
python import_check.py
python run_check.py
```

---

## Key Dependencies

| Package | Version | Purpose |
|---|---|---|
| `cognite-toolkit` | `0.6.53` | CDF resource deployment CLI |
| `cognite-sdk` | (transitive) | CDF Python SDK used inside functions |
| `orjson` | (function) | Fast JSON parsing in the extractor function |
| `numpy` | (function) | Vectorised OEE calculations |
