# HR Recruitment Analytics

HR teams have recruitment funnel data, interview feedback, and onboarding records, but no shared reporting system identifies which hiring stages contribute most to candidate drop-offs across departments.

This project provides a shared recruitment analytics workflow that transforms raw recruitment CSV data into validated cleaned datasets, an application-grain recruitment fact table, Python/Pandas analytics, SQL analysis definitions, threshold-based alerts, and a Streamlit dashboard.

The current dataset contains applications, interviews, offers, candidates, and jobs. It does not contain a reliable hired outcome, so the project reports accepted offers rather than inventing a hiring result.

## Technology

- Python 3.12
- Pandas
- NumPy
- Pytest
- SQL/PostgreSQL-compatible SQL definitions
- Streamlit
- Docker
- GitHub Actions

FastAPI and React are not part of the current implementation.

## Architecture

```text
Raw CSV files
    |
    v
Normalization and validation
    |
    v
Processed CSV files
    |
    v
Application-grain recruitment fact table
    |
    +--------------------------+
    |                          |
    v                          v
Python/Pandas analytics        SQL schema and queries
    |                          |
    +--------------------------+
                |
                v
KPIs / funnel / source performance / alerts
                |
                v
Streamlit dashboard
```

The normalization stage validates source files, types, identifiers, categories, dates, and foreign keys. The fact-table stage joins candidate and job attributes to applications after aggregating interview and offer events. The analytics layer calculates application-grain recruitment measures. SQL files provide PostgreSQL-compatible schema and analysis definitions. The Streamlit app presents the validated fact data and analytics.

## Data

### Raw data

```text
data/raw/
├── candidates.csv
├── jobs.csv
├── applications.csv
├── interviews.csv
└── offers.csv
```

The main relationships are:

- A candidate can have applications.
- A job can have applications.
- An application belongs to one candidate and one job.
- An application can have multiple interview records.
- An application can have offer records.

### Processed data

```text
data/processed/
├── candidates_clean.csv
├── jobs_clean.csv
├── applications_clean.csv
├── interviews_clean.csv
├── offers_clean.csv
└── recruitment_facts.csv
```

`recruitment_facts.csv` is the main analytical dataset. It has exactly one row per `application_id`. Interview and offer records are aggregated before they are joined to applications, preventing join multiplication.

### Current validated counts

| Dataset | Rows |
|---|---:|
| Candidates | 9,000 |
| Jobs | 2,000 |
| Applications | 15,000 |
| Interviews | 7,196 |
| Offers | 938 |
| Recruitment facts | 15,000 |

`recruitment_facts.application_id` is unique and non-null. Foreign-key validation has passed, raw files are preserved, and the pipeline validates the expected raw input filenames.

## Data Normalization

[scripts/normalize_hr_data.py](scripts/normalize_hr_data.py) loads all five raw files and:

- Validates expected files and columns.
- Converts numeric fields and nullable integer identifiers.
- Parses date fields.
- Normalizes categorical values to lowercase and trims whitespace.
- Fills missing categorical and string values with `unknown`.
- Parses offer `accepted` values as booleans.
- Checks duplicate and null primary IDs.
- Validates application, interview, and offer foreign keys.
- Writes the five cleaned datasets.
- Never modifies the raw input files.

## Recruitment Facts

[scripts/build_recruitment_facts.py](scripts/build_recruitment_facts.py) builds the application-grain fact table by:

- Starting with applications as the base table.
- Joining candidate information by `candidate_id`.
- Joining job information by `job_id`.
- Aggregating interviews by `application_id` before joining.
- Aggregating offers by `application_id` before joining.
- Adding interview counts, offer counts, and accepted-offer information.
- Calculating elapsed days to first interview, offer, and acceptance.
- Preserving applications without interview or offer events.
- Avoiding duplicate application rows caused by event joins.
- Avoiding any invented `hired` outcome.

## Pipeline

[scripts/run_pipeline.py](scripts/run_pipeline.py) validates the five required raw inputs, runs normalization, builds `recruitment_facts.csv`, checks all expected processed outputs, and validates the final 15,000-row application grain.

Run it from the project root:

```powershell
python scripts/run_pipeline.py
```

The current pipeline validates the expected current dataset size of 15,000 recruitment fact rows.

## Python Analytics

### HR KPIs

[analysis/hr_kpis.py](analysis/hr_kpis.py) calculates candidate, application, interview, offer, accepted-offer, rate, and elapsed-time KPIs. It does not calculate a hired KPI.

### Recruitment funnel

[analysis/recruitment_funnel.py](analysis/recruitment_funnel.py) calculates the observable cumulative funnel:

- Applied
- Screened or later
- Interview or later
- Offer
- Accepted Offer

It also calculates stage drop-offs and role-family funnel results.

### Source performance

[analysis/source_performance.py](analysis/source_performance.py) calculates application volume, interview progression, offers, accepted offers, conversion rates, application scores, and elapsed-time metrics for each application source.

### Hiring alerts

[analysis/hiring_alerts.py](analysis/hiring_alerts.py) identifies transparent threshold-based recruitment risks, including large funnel drop-offs, low source conversion, low offer acceptance, and slow source timing.

## Current KPI Results

| KPI | Value |
|---|---:|
| Total candidates | 7,324 |
| Total applications | 15,000 |
| Total interview records | 7,196 |
| Applications with interviews | 3,570 |
| Total offers | 938 |
| Accepted offers | 699 |
| Interview rate | 23.80% |
| Offer rate | 6.25% |
| Offer acceptance rate | 74.52% |
| Application-to-acceptance rate | 4.66% |
| Average days to first interview | 12.423 |
| Average days to offer | 28.698 |
| Average days to acceptance | 34.984 |

The 7,196 interview figure counts interview records. The 3,570 figure counts unique applications with at least one interview. They are intentionally different measures.

## Recruitment Funnel

| Stage | Count | Conversion |
|---|---:|---:|
| Applied | 15,000 | 100.00% |
| Screened or later | 6,362 | 42.41% |
| Interview or later | 3,570 | 23.80% |
| Offer | 938 | 6.25% |
| Accepted Offer | 699 | 4.66% |

The largest overall drop-off is **Interview or later -> Offer**, with approximately **73.73%** drop-off.

## Role / Function Analysis

There is no explicit department field. The project uses `jobs.role_family` as the function/department proxy.

### Current role-family job counts

| Role family | Jobs |
|---|---:|
| operations | 991 |
| engineering | 425 |
| data | 302 |
| product | 128 |
| customer_success | 77 |
| sales | 77 |

### Current role-family interview-to-offer drop-offs

| Role family | Drop-off |
|---|---:|
| customer_success | 77.17% |
| data | 75.10% |
| engineering | 72.67% |
| operations | 73.05% |
| product | 75.22% |
| sales | 77.18% |

## Source Performance

The current application sources are:

- `job_board`
- `referral`
- `linkedin`
- `outbound`
- `recruiter`

| Source | Applications | Application -> Acceptance |
|---|---:|---:|
| job_board | 7,094 | 4.86% |
| referral | 2,694 | 4.38% |
| linkedin | 2,648 | 4.08% |
| outbound | 1,333 | 4.43% |
| recruiter | 1,231 | 5.61% |

Recruiter has the highest application-to-acceptance rate in the current dataset. LinkedIn has the lowest.

## Hiring Alerts

The current alert output contains five alerts:

1. **High stage drop-off:** Interview or later -> Offer, approximately 73.73%.
2. **Warning low source conversion:** LinkedIn, approximately 4.08% application-to-acceptance.
3. **Warning low offer acceptance:** LinkedIn, approximately 67.92%.
4. **Warning slow recruitment stage:** Outbound offer timing, approximately 30.62 days.
5. **Warning slow recruitment stage:** Referral acceptance timing, approximately 36.04 days.

## SQL

[sql/hr_schema.sql](sql/hr_schema.sql) defines these PostgreSQL-compatible tables:

- `candidates`
- `jobs`
- `applications`
- `interviews`
- `offers`
- `recruitment_facts`

[sql/recruitment_queries.sql](sql/recruitment_queries.sql) contains analysis queries for:

- Overall KPIs
- Recruitment funnel
- Role-family analysis
- Source performance
- Interview analysis
- Offer analysis
- Hiring alerts and bottlenecks

The queries use safe denominator handling with `NULLIF` where appropriate. PostgreSQL has not been configured or executed locally as part of this project; these files provide the schema and query definitions.

## Streamlit Dashboard

[app.py](app.py) is the Streamlit dashboard entry point. It reads:

```text
data/processed/recruitment_facts.csv
```

The dashboard displays:

- KPI cards
- Recruitment funnel
- Source performance
- Role-family analysis
- Hiring alerts
- Sidebar filters for role family, source, level, location type, employment type, and application stage

Run it with:

```powershell
streamlit run app.py
```

Streamlit uses its default port, `8501`.

The project does not include browser automation or a backend API.

## Project Structure

```text
.
├── .github/
│   └── workflows/
│       └── ci.yml
├── analysis/
│   ├── hr_kpis.py
│   ├── recruitment_funnel.py
│   ├── source_performance.py
│   └── hiring_alerts.py
├── data/
│   ├── raw/
│   │   ├── candidates.csv
│   │   ├── jobs.csv
│   │   ├── applications.csv
│   │   ├── interviews.csv
│   │   └── offers.csv
│   └── processed/
│       ├── candidates_clean.csv
│       ├── jobs_clean.csv
│       ├── applications_clean.csv
│       ├── interviews_clean.csv
│       ├── offers_clean.csv
│       └── recruitment_facts.csv
├── scripts/
│   ├── normalize_hr_data.py
│   ├── build_recruitment_facts.py
│   └── run_pipeline.py
├── sql/
│   ├── hr_schema.sql
│   └── recruitment_queries.sql
├── tests/
│   ├── test_normalize_hr_data.py
│   ├── test_build_recruitment_facts.py
│   ├── test_recruitment_pipeline.py
│   └── test_hr_analytics.py
├── app.py
├── Dockerfile
├── .dockerignore
├── .gitignore
├── requirements.txt
└── README.md
```

## Local Setup

Clone the repository and enter the project directory:

```powershell
git clone <repository-url>
cd <project-directory>
```

Create and activate the virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install dependencies:

```powershell
python -m pip install -r requirements.txt
```

## Testing

The full automated test suite currently passes. The latest local run completed with **45 passed**; execution time can vary by environment.

Run all tests:

```powershell
python -m pytest -q
```

The tests cover:

- Data normalization
- Recruitment fact-table construction
- End-to-end pipeline behavior
- HR KPIs, funnel, source performance, and hiring alerts

No test coverage percentage is claimed.

## Docker

The Docker image is dashboard-only. It contains the Streamlit app, analytics modules, dependencies, and the processed recruitment facts file. It does not run the data pipeline automatically.

Build the image:

```powershell
docker build -t hr-recruitment-analytics .
```

Run the dashboard:

```powershell
docker run --rm -p 8501:8501 hr-recruitment-analytics
```

The dashboard is then available at `http://localhost:8501`.

The Docker image does not include or run the raw-data pipeline, PostgreSQL, FastAPI, React, Redis, or other services.

## CI/CD

[.github/workflows/ci.yml](.github/workflows/ci.yml) provides automated testing rather than deployment. It:

- Runs on pushes.
- Runs when pull requests are opened, synchronized, or reopened.
- Uses `ubuntu-latest`.
- Uses Python 3.12.
- Installs `requirements.txt`.
- Runs `python -m pytest -q`.
- Fails automatically when tests fail.

## Limitations and Data Notes

1. The dataset does not contain a reliable hired outcome, so the project does not invent an `is_hired` field.
2. There is no explicit department field; `jobs.role_family` is used as the function/department proxy.
3. Interview count refers to interview records, while applications with interviews refers to unique applications.
4. `jobs.status` represents requisition/job status and should not be interpreted as a candidate hiring outcome.
5. Recruiter-level performance metrics are not included because the data does not contain a reliable recruiter identifier. `recruiter` is an application source, not a recruiter identity field.
6. SQL files are provided for database schema and query analysis. PostgreSQL execution is not configured locally.
7. The pipeline currently validates the expected 15,000-row recruitment fact dataset, so the orchestration is tied to the current dataset size.

## Conclusion

This project provides an end-to-end recruitment analytics workflow: raw recruitment CSV files are normalized and validated, transformed into an application-grain fact table, analyzed with Python/Pandas and SQL definitions, checked for actionable bottlenecks, and presented through a Streamlit dashboard.
