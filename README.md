# GCP Dataform CI/CD Pipeline with Keyless Jenkins

End-to-end automated pipeline deploying Dataform (`.sqlx`) models from GitHub to BigQuery using a keyless Jenkins runner on Google Compute Engine (GCE) and Service Account Impersonation.

## Architecture Overview

1. **Developer** pushes `.sqlx` transformations and `Jenkinsfile` to **GitHub** (`main` branch).
2. **Jenkins (GCE VM)** authenticates keylessly via the VM metadata server (`sa-jenkins-deployer`) and triggers compilation and execution via the **GCP Dataform REST API**.
3. **GCP Dataform Service Agent** pulls the commit from GitHub using a **Secret Manager** PAT and impersonates `sa-dataform-executor` (`actAs`).
4. **BigQuery** executes the compiled SQL and materializes views/tables in `analytics_prod`.

---

## High-Level Setup Steps

### 1. GCP Project & Services Setup
* Create/select GCP project (`sa-testing-515015`) in region `asia-south1`.
* Enable APIs: **BigQuery**, **Dataform**, **Secret Manager**, **Compute Engine**, and **IAM**.
* Provision the Google-managed Dataform Service Agent (`service-<PROJECT_NUMBER>@gcp-sa-dataform.iam.gserviceaccount.com`).
* Create target BigQuery dataset `analytics_prod`.

### 2. Service Accounts & IAM Permissions
* **Create `sa-jenkins-deployer` (CI/CD Orchestrator):**
  * Grant `roles/dataform.editor` at the project level to create compilations and workflow invocations.
* **Create `sa-dataform-executor` (BigQuery Execution Identity):**
  * Grant `roles/bigquery.jobUser`, `roles/bigquery.dataEditor`, and `roles/bigquery.dataViewer` at the project level.
* **Configure Service Account Impersonation (on `sa-dataform-executor`):**
  * Grant `roles/iam.serviceAccountUser` to `sa-jenkins-deployer`.
  * Grant `roles/iam.serviceAccountTokenCreator` and `roles/iam.serviceAccountUser` to the Google-managed Dataform Service Agent.

### 3. GitHub Repository & Dataform Code
* Create GitHub repository `retail-dataform-native` and generate a Fine-Grained Personal Access Token (PAT) with **Contents: Read and write** access.
* Add core project files to `main`:
  * `workflow_settings.yaml` (target project, region, `analytics_prod` dataset, Dataform Core `3.0.0`)
  * `definitions/stg_orders.sqlx` (staging view)
  * `definitions/fct_order_revenue.sqlx` (downstream revenue table)
  * `Jenkinsfile` (CI/CD pipeline definition)

### 4. Secret Manager & Native GCP Dataform Linking
* Store the GitHub PAT in **Secret Manager** (`github-dataform-pat`).
* Grant `roles/secretmanager.secretAccessor` on the secret to the Google-managed Dataform Service Agent.
* Create native Dataform repository `retail-dataform-repo` in `asia-south1` using `sa-dataform-executor` as the default service account, and link it to the GitHub repository over HTTPS.

### 5. Keyless Jenkins VM Setup
* Launch an Ubuntu GCE VM (`jenkins-cicd-vm`) with `sa-jenkins-deployer` attached as the VM service account (eliminating JSON key files).
* Open firewall port `TCP:8080` (`jenkins-server` network tag).
* Install **OpenJDK 21** and **Jenkins**, then unlock the web UI at `http://<VM_EXTERNAL_IP>:8080`.

### 6. Jenkins Pipeline Execution & Verification
* Create a **Pipeline** job (`deploy-dataform-native`) configured with **Pipeline script from SCM** pointing to `Jenkinsfile` on `main`.
* Pipeline stages executed on build:
  1. **Verify Active SA:** Confirms `sa-jenkins-deployer` is active via VM metadata.
  2. **Compile:** Calls `POST .../compilationResults` against `main` and extracts the compilation ID.
  3. **Invoke Workflow:** Calls `POST .../workflowInvocations` passing `compilationResult` and `invocationConfig.serviceAccount = sa-dataform-executor`.
* Verify execution status in **Dataform Workflow Execution Logs** and query `analytics_prod.fct_order_revenue` in BigQuery.

### 7. Cost Management
* Stop `jenkins-cicd-vm` when idle (`gcloud compute instances stop jenkins-cicd-vm --zone=asia-south1-a`) or delete it after testing to avoid ongoing compute charges.
