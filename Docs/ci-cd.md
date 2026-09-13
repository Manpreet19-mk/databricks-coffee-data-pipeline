%md

# CI/CD + Unit Testing (Databricks Asset Bundles + GitHub Actions)

This project implements CI/CD automation and unit testing using **Databricks Asset Bundles (DABs)** and **GitHub Actions**.

The objective is to ensure that:
- Databricks jobs, pipelines, and notebooks are fully version-controlled
- Deployments are repeatable and automated across environments
- Core transformation logic is validated using unit tests
- Every change is traceable through Git commits and CI/CD run history

---

## 1. Databricks Asset Bundles (DAB)

Databricks Asset Bundles are used as the deployment framework for this project.

Bundle configuration is defined in:
- `databricks.yml`

Using DAB ensures that all Databricks resources (jobs, pipelines, notebooks) can be deployed consistently using configuration-as-code, instead of manual UI creation.

---

## 2. GitHub Actions CI/CD Pipeline

CI/CD automation is implemented using GitHub Actions.

Workflow file location:
- `.github/workflows/databricks-ci-cd.yml`

The workflow performs:
- bundle validation
- environment-based deployment
- execution of unit tests through a Databricks job

This provides an automated CI/CD pipeline aligned with enterprise Databricks deployment practices.

---

## 3. Branch-Based Deployment Strategy

A branch-based strategy is followed:

- **dev branch**
  - deploys to DEV target
  - runs unit tests after deployment

- **main branch**
  - deploys to PROD target
  - runs unit tests after deployment

This ensures DEV acts as a quality gate before PROD deployment, while PROD remains stable and release-controlled.

---

## 4. Secure Authentication (GitHub Secrets)

Databricks authentication is handled securely using GitHub repository secrets:

- `DATABRICKS_HOST`
- `DATABRICKS_TOKEN`

This ensures:
- credentials are not stored in code
- tokens remain secure
- CI/CD is portable across environments

---

## 5. Unit Testing Implementation

Unit tests were implemented to validate the most critical reusable transformation logic in the pipeline.

A dedicated Databricks job is created using DAB resources:

- `unit_test_runner`

This job executes unit tests as notebook tasks and validates:
- SQL UDF column standardization logic
- DataFrame column standardization utility logic
- Deduplication logic for incremental ingestion

This ensures core transformation logic remains correct and prevents regression when pipeline code evolves.

---

## 6. CI/CD Outcome

The CI/CD workflow confirms that:

- Databricks bundles are validated automatically
- deployments occur automatically based on branch
- unit tests execute automatically after deployment
- success/failure is visible in GitHub Actions run history
- test execution is traceable in Databricks Jobs UI

This makes the solution production-aligned and demonstrates automated deployment and testing practices for Databricks pipelines.

---

## 7. Evidence Captured

CI/CD evidence screenshots were captured and stored in the external evidence folder:

`evidence/cicd/`

Evidence includes:
- GitHub secrets configuration
- GitHub Actions workflow execution (validate + deploy + unit tests)
- Databricks unit test runner job execution success
