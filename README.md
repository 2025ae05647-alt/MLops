# Heart Disease MLOps: End-to-End Educational Project

> **Educational demonstration only.** The UCI Cleveland dataset is small and historical. Model outputs are not validated clinical risk estimates and must not guide medical decisions.

## Dataset and approach

The included CSV is a 14-column Cleveland heart-disease dataset with 13 predictors and a binary target. `target > 0` denotes presence of heart disease. `?` and nonnumeric values become missing values. Rows without a target are dropped. Numeric columns use median imputation and standardization; categorical columns use most-frequent imputation and one-hot encoding. All transformers are fit inside cross-validation folds, avoiding train/test leakage. We use an 80/20 stratified split (random_state=42), five-fold stratified CV, and ROC-AUC for selection. Models: Logistic Regression and Random Forest. Original report CSVs/figures are retained as historical artifacts; **reproduced** outputs use separate filenames and can differ because original split/parameters were not supplied.

## Setup (Python 3.11 recommended)

Run all commands from the repository root. On Windows PowerShell:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
python -m pytest tests -v
python -m src.train --mlflow
python -m src.predict
```

For macOS/Linux replace venv creation/activation with `python3 -m venv .venv && source .venv/bin/activate`.

Dataset already included at `data/processed/heart_disease_clean.csv`. To re-download from UCI (requires internet): `python -m src.download_data`. The UCI processed Cleveland data may contain missing values; training handles these within its pipeline. Source: https://archive.ics.uci.edu/dataset/45/heart+disease

## MLflow

Training with `--mlflow` logs hyperparameters, cross-validation/test metrics, model files and ROC/confusion-matrix plots for each candidate. Start UI with `mlflow ui --backend-store-uri sqlite:///mlflow.db --port 5000`, then open http://127.0.0.1:5000. SQLite tracking database and artifact directories are generated locally and are not checked in.

## Run API

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Open http://127.0.0.1:8000/docs to test POST `/predict` using `sample_patient.json`. Other endpoints: GET `/health`, GET `/metrics`. The prediction is a binary classification, and `confidence` is the predicted class's model probability, **not** clinical confidence or a calibrated probability of disease.

PowerShell example:

```powershell
$body = Get-Content sample_patient.json -Raw
Invoke-RestMethod -Uri http://localhost:8000/predict -Method Post -ContentType application/json -Body $body
```

## Docker

Train first to create the model. From the repository root:

```bash
docker build -t heart-disease-api:latest .
docker run --rm -p 8000:8000 heart-disease-api:latest
```

Test `/health` and `/predict` from another terminal. Save your own terminal screenshots.

## CI/CD

GitHub Actions `.github/workflows/ci.yml` performs linting, Pytest, MLflow-tracked training, smoke inference, and uploads generated artifacts. This is **CI and model-build automation**; it does not automatically publish an image or deploy to Kubernetes. Do not call it fully automated CD until you add a deployment job and registry credentials.

## Kubernetes (Minikube)

Install Docker Desktop, kubectl and Minikube. Start a Docker-backed cluster, then build the image inside Minikube's Docker daemon:

```bash
minikube start --driver=docker
minikube image load heart-disease-api:latest
kubectl apply -f deployment/deployment.yaml
kubectl apply -f deployment/service.yaml
kubectl rollout status deployment/heart-disease-api
kubectl get pods,svc
minikube service heart-disease-api --url
```

Use the URL printed by Minikube. If Minikube image load fails, confirm the image was built locally. `NodePort` provides **local cluster access**, not a public cloud URL.

## Monitoring

The API logs HTTP method, route, status, and duration (without logging raw patient payloads). GET `/metrics` exposes Prometheus counters and latency histograms. For a locally running API, start monitoring via `docker compose -f monitoring/docker-compose.yml up -d`. Open Prometheus at http://localhost:9090 and Grafana at http://localhost:3000 (first-login defaults often admin/admin; change the password). Configure a Prometheus data source at `http://prometheus:9090`. Try `sum(rate(heart_api_requests_total[5m]))` and `histogram_quantile(0.95, sum(rate(heart_api_request_seconds_bucket[5m])) by (le))`. The bundled scrape target assumes the API runs on the host, not inside a separate Minikube network; adapt it for Kubernetes monitoring.

## Submission evidence checklist

Capture **real** screenshots: EDA plots, terminal training logs, MLflow runs, green GitHub Actions job, Docker build/run, `/predict` response, Kubernetes pods/service, Prometheus query/Grafana dashboard. Include the actual GitHub URL, deployment access instructions, and your own short screen recording. A separate 10-page report is required by the assignment and is **not included in this code package**.

## Architecture

`UCI CSV -> cleaning -> train/test split -> fold-safe sklearn preprocessing -> GridSearchCV (LR/RF) -> MLflow + joblib -> FastAPI -> Docker -> Minikube -> Prometheus/Grafana`. GitHub Actions independently runs lint/tests/training and stores artifacts.

## Originality and limitations

Review, modify, execute and explain this reference implementation yourself before academic submission. Report your actual run results. The API is an educational prototype: no authentication, transport TLS, clinical validation, fairness study, calibrated risk, or production-grade data-drift detection is provided.
