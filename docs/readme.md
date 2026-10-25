# VulnTracker Security Automation Assignment

VulnTracker is a two-service application for recording vulnerability findings,
tracking remediation, and sending notification events.

The solution contains:

- **FastAPI API** in `app/`
- **Node.js notification service** in `notify/`
- Secure shared-report links
- Production container images
- Helm deployment for Kubernetes/AKS
- Azure Key Vault integration
- NGINX Ingress Controller
- NetworkPolicies and hardened security contexts
- SAST, SCA, container, and IaC security reports
- Prioritised findings, remediation plan, and executive summary

---

## Architecture

```text
                         External Client
                               |
                               v
                    NGINX Ingress Controller
                               |
                               | HTTPS / HTTP
                               v
                    VulnTracker FastAPI Service
                         ClusterIP :8000
                               |
                               | Internal notification call
                               v
                     Node.js Notify Service
                         ClusterIP :3001


Azure Key Vault
      |
      v
AKS Workload Identity
      |
      v
Secrets Store CSI Driver
      |
      v
Read-only secret mount in FastAPI pod
```

Only the FastAPI service is exposed through NGINX Ingress. The Node.js notification
service remains internal to the Kubernetes cluster.

---

## Repository Structure

```text
/
├── app/                            # Python FastAPI application
├── notify/                         # Node.js notification service
├── tests/                          # Python API and security tests
├── Dockerfile                      # FastAPI container image
├── notify/
│   └── Dockerfile                  # Node.js container image
├── helm/
│   └── vulntracker/                # Kubernetes Helm chart
├── reports/
│   ├── sast.semgrep.json
│   ├── sca.trivy.json
│   ├── container.python.trivy.json
│   ├── container.node.trivy.json
│   └── iac.checkov.json
├── docs/
│   ├── findings.md
│   ├── remediation-plan.md
│   └── executive-summary.md
├── requirements.txt
└── README.md
```

---

## Technology Summary

| Technology | Purpose |
|---|---|
| Python 3.11 / FastAPI | Vulnerability tracking REST API |
| Node.js 20 / Express | Internal notification service |
| SQLite / SQLAlchemy | Prototype application data store |
| Docker | Containerisation of both services |
| AKS / Kubernetes | Target runtime platform |
| Helm | Kubernetes deployment packaging |
| NGINX Ingress Controller | Controlled external access to FastAPI |
| Azure Key Vault | External storage for runtime secrets |
| AKS Workload Identity | Passwordless access from AKS to Azure Key Vault |
| Federated Identity Credential | Trust relationship between Kubernetes ServiceAccount and Azure managed identity |
| Secrets Store CSI Driver | Read-only Key Vault secret mounting |
| NetworkPolicy | Restricts ingress between NGINX, FastAPI, and Notify |
| Semgrep | Static application security testing |
| Trivy | Dependency and container vulnerability scanning |
| Checkov | Helm/Kubernetes IaC security scanning |
| Pytest | API and security regression testing |

---

# Task 1: Shared Report Links

Task 1 adds two REST endpoints.

## Create a shared link

```http
POST /scans/{scan_id}/share
Authorization: Bearer <access-token>
Content-Type: application/json
```

Optional password:

```json
{
  "password": "Customer123!"
}
```

Without password:

```json
{}
```

Example response:

```json
{
  "share_url": "http://localhost:8000/share/<random-token>"
}
```

## Open a shared report

Without password:

```http
GET /share/{token}
```

With password:

```http
GET /share/{token}?password=Customer123!
```

## Security controls

- Cryptographically secure random token
- Only the token hash is stored in the database
- Optional password is stored as a secure password hash
- Link expires after 24 hours
- Only the scan owner can generate its share link
- Invalid and expired links are rejected
- Password-protected links reject missing or incorrect passwords
- Public URLs do not expose the scan ID

---

# Local Development

## Requirements

- Python **3.11**
- Node.js **20 or later**
- Docker
- Helm
- Trivy
- Semgrep
- Checkov

## Python API

From the repository root:

```bash
python -m venv .venv
```

Windows CMD:

```cmd
.venv\Scripts\activate
```

Linux/macOS:

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
python -m pip install -r requirements.txt
```

Start the API from the `app` directory:

```bash
cd app
python -m uvicorn main:app --host 0.0.0.0 --port 8000
```

API endpoints:

```text
API:      http://localhost:8000
Swagger:  http://localhost:8000/docs
Health:   http://localhost:8000/health
```

## Node.js notification service

Open another terminal:

```bash
cd notify
npm ci
npm start
```

The service listens on:

```text
http://localhost:3001
```

---

# Tests

## Python tests

Run from the repository root:

```bash
python -m pytest tests/ -v
```

## Node.js tests

Stop any manually running notification service first because the test suite may
start its own service on port 3001.

```bash
cd notify
npm test
```

---

# Task 2: Security Analysis

Four security scan categories were completed.

## Tools used

| Scan | Tool | Scope |
|---|---|---|
| SAST | Semgrep | Python and Node.js source code |
| SCA | Trivy filesystem scan | Python and Node.js dependencies |
| Container scanning | Trivy image scan | FastAPI and Notify container images |
| IaC scanning | Checkov | Helm/Kubernetes configuration |

Raw reports are stored in `reports/`. Scanner output was reviewed manually and
converted into application-specific findings in `docs/findings.md`.

## SAST with Semgrep

```bash
semgrep scan --config auto --json --output reports/sast.semgrep.json app notify
```

## SCA with Trivy

```bash
trivy fs --scanners vuln --format json --output reports/sca.trivy.json .
```

## Container scans

FastAPI:

```bash
trivy image \
  --scanners vuln \
  --format json \
  --output reports/container.python.trivy.json \
  vulntracker-api:1.0.0
```

Node.js Notify:

```bash
trivy image \
  --scanners vuln \
  --format json \
  --output reports/container.node.trivy.json \
  vulntracker-notify:1.0.0
```

## IaC scan with Checkov

Standard command:

```bash
checkov -d helm/vulntracker --framework helm -o json > reports/iac.checkov.json
```

Windows fallback when the Checkov launcher has a `.py` file-association issue:

```cmd
python .venv\Lib\site-packages\checkov\main.py -d helm\vulntracker --framework helm -o json > reports\iac.checkov.json
```

Validate generated JSON:

```cmd
python -m json.tool reports\iac.checkov.json > nul
```

---

# Task 3: Remediation

The following security issues were prioritised and remediated:

1. Replaced dynamically constructed search SQL with parameterized SQLAlchemy queries.
2. Removed plaintext passwords from login logs.
3. Added scan-owner authorization to prevent cross-user access.
4. Added ownership validation when creating shared links.
5. Removed internal traceback details from client-facing error responses.
6. Added tests for shared-link authorization, expiry, and password validation.
7. Hardened container and Kubernetes runtime configuration.

Findings that were not fully remediated are documented in:

```text
docs/remediation-plan.md
```

The remediation plan records:

- residual risk
- reason for deferral
- expected remediation effort
- compensating controls
- recommended production action

---

# Task 4: Containerisation

## Build FastAPI image

Run from the repository root:

```bash
docker build --no-cache -t vulntracker-api:1.0.0 -f Dockerfile .
```

## Run FastAPI image

```bash
docker run --rm -d \
  --name vulntracker-api \
  -p 8000:8000 \
  vulntracker-api:1.0.0
```

Test:

```bash
curl http://localhost:8000/health
```

Check container health:

```bash
docker inspect --format "{{.State.Health.Status}}" vulntracker-api
```

Stop:

```bash
docker stop vulntracker-api
```

## Build Node.js Notify image

```bash
docker build --no-cache \
  -t vulntracker-notify:1.0.0 \
  -f notify/Dockerfile \
  notify
```

## Run Node.js Notify image

```bash
docker run --rm -d \
  --name vulntracker-notify \
  -p 3001:3001 \
  vulntracker-notify:1.0.0
```

View logs:

```bash
docker logs vulntracker-notify
```

Stop:

```bash
docker stop vulntracker-notify
```

## Container security controls

- Pinned minimal base images
- Non-root runtime users
- Docker health checks
- No embedded credentials
- Unnecessary build content excluded through `.dockerignore`
- Kubernetes privilege escalation disabled
- Linux capabilities dropped
- `RuntimeDefault` seccomp profile
- Read-only root filesystem where supported
- CPU and memory requests and limits

---

# Kubernetes and AKS Deployment

## Namespace

Recommended deployment namespace:

```text
vulntracker
```

Create it if it is not managed by the chart or platform process:

```bash
kubectl create namespace vulntracker
```

## Helm validation

```bash
helm lint helm/vulntracker
```

Render manifests:

```bash
helm template vulntracker helm/vulntracker \
  --namespace vulntracker \
  > rendered.yaml
```

Client-side validation:

```bash
kubectl apply --dry-run=client -f rendered.yaml
```

## Install

```bash
helm upgrade --install vulntracker helm/vulntracker \
  --namespace vulntracker \
  --create-namespace
```

## Verify

```bash
kubectl get deployments -n vulntracker
kubectl get pods -n vulntracker
kubectl get services -n vulntracker
kubectl get ingress -n vulntracker
kubectl get networkpolicy -n vulntracker
kubectl get secretproviderclass -n vulntracker
kubectl get poddisruptionbudget -n vulntracker
```

## Remove

```bash
helm uninstall vulntracker --namespace vulntracker
```

---

# NGINX Ingress

NGINX Ingress is the only intended external entry point.

```text
NGINX Ingress
      |
      v
FastAPI ClusterIP :8000
      |
      v
Notify ClusterIP :3001
```

The Node.js notification service is not independently exposed through Ingress.

Example NGINX Ingress host:

```text
vulntracker.example.com
```

The actual host and TLS configuration must be supplied for the target environment.

---

# Network Restrictions

The Helm chart includes NetworkPolicies with the following intent:

- NGINX Ingress namespace can reach FastAPI on TCP 8000.
- FastAPI pods can reach Notify on TCP 3001.
- Notify is not directly reachable from external ingress.
- Both application services use `ClusterIP`.
- No application service requires a public `LoadBalancer`.

Expected ingress namespace:

```text
ingress-nginx
```

If the NGINX Ingress Controller uses another namespace, update:

```yaml
networkPolicy:
  ingressNamespace: ingress-nginx
```

---

# Azure Key Vault Integration

Runtime secrets are designed to come from Azure Key Vault rather than being
stored in source-controlled Helm values or Kubernetes manifests.

The integration uses:

- Azure Key Vault
- AKS Workload Identity
- Kubernetes ServiceAccount
- Federated Identity Credential
- Secrets Store CSI Driver
- Azure Key Vault CSI provider
- `SecretProviderClass`

## Federated identity model

```text
Kubernetes ServiceAccount
          |
          | OIDC token
          v
Federated Identity Credential
          |
          v
Azure Managed Identity
          |
          v
Azure Key Vault
```

The Federated Identity Credential establishes trust between the Kubernetes
ServiceAccount identity and the Azure managed identity without requiring a
client secret.

## Deployment-specific values

The following values must be supplied for the target Azure environment:

```yaml
keyVault:
  enabled: true
  name: "<key-vault-name>"
  tenantId: "<azure-tenant-id>"
  clientId: "<managed-identity-client-id>"
```

These values are identifiers. Actual application secrets must remain in Azure
Key Vault.

Do not commit:

- client secrets
- passwords
- application JWT secrets
- API keys
- database credentials
- Azure service-principal credentials

## Secret delivery

The FastAPI pod mounts Key Vault content using a read-only CSI volume.

```text
Azure Key Vault
      |
Secrets Store CSI Driver
      |
SecretProviderClass
      |
Read-only mount: /mnt/secrets-store
```

---

# Helm Workloads

The chart deploys:

```text
2 Kubernetes Deployments
├── vulntracker-api
└── vulntracker-notify

2 ClusterIP Services
├── vulntracker-api
└── vulntracker-notify

1 NGINX Ingress
└── FastAPI only

2 NetworkPolicies
├── NGINX → FastAPI
└── FastAPI → Notify

2 PodDisruptionBudgets
├── FastAPI
└── Notify

1 ServiceAccount
1 SecretProviderClass
```

---

# Task 5: Executive Summary

The executive summary is available at:

```text
docs/executive-summary.md
```

It covers:

- security posture before and after the assessment
- highest-impact remediations
- top residual risks
- business impact
- recommended production actions

It is written for a CISO-level audience and avoids scanner-specific detail in
the opening section.

---

# Security Reports

The final submission should use clear, assignment-aligned filenames:

```text
reports/
├── sast.semgrep.json
├── sca.trivy.json
├── container.python.trivy.json
├── container.node.trivy.json
└── iac.checkov.json
```

The mandatory FastAPI container report may alternatively be named:

```text
container.trivy.json
```

If only one container report is required, use the FastAPI report because the
assignment explicitly requires a production Dockerfile for that service.

Before submission, avoid ambiguous names such as:

```text
reportscontainer.python.trivy.json
reportssca.trivy.json
task4iac.checkov.json
```

Rename them into the `reports/` directory:

```cmd
move reportscontainer.python.trivy.json reports\container.python.trivy.json
move reportscontainer.node.trivy.json reports\container.node.trivy.json
move reportssca.trivy.json reports\sca.trivy.json
move sast.semgrep.json reports\sast.semgrep.json
move task4iac.checkov.json reports\iac.checkov.json
```

If `reports\iac.checkov.json` already contains the final Helm scan, keep the
final file and remove the older duplicate.

---

# Final Validation

## Python

```bash
python --version
python -m pytest tests/ -v
```

Python must be version 3.11 and all tests must pass.

## Node.js

```bash
cd notify
npm ci
npm test
cd ..
```

## Docker

```bash
docker build -t vulntracker-api:1.0.0 .
docker build -t vulntracker-notify:1.0.0 -f notify/Dockerfile notify
docker images
```

## Helm

```bash
helm lint helm/vulntracker
helm template vulntracker helm/vulntracker --namespace vulntracker > rendered.yaml
```

## Security reports

Windows CMD:

```cmd
python -m json.tool reports\sast.semgrep.json > nul
python -m json.tool reports\sca.trivy.json > nul
python -m json.tool reports\container.python.trivy.json > nul
python -m json.tool reports\container.node.trivy.json > nul
python -m json.tool reports\iac.checkov.json > nul
```

---

# CI and Submission

Before submission:

```bash
git status
git diff
```

Add all completed files:

```bash
git add app notify tests Dockerfile .dockerignore helm reports docs README.md
```

Commit:

```bash
git commit -m "complete security automation assignment"
```

Push:

```bash
git push origin master
```

If the repository uses `main` instead:

```bash
git push origin main
```

Confirm that the GitHub Actions workflow is green before submitting the
repository URL.

---

# Task Summary

## Task 1

- Added expiring shared-report links
- Added optional password protection
- Stored token and password hashes
- Enforced scan ownership
- Added public report retrieval endpoint
- Added tests for valid, invalid, expired, and password-protected links

## Task 2

- Ran Semgrep SAST
- Ran Trivy dependency/SCA scan
- Scanned FastAPI and Notify container images
- Ran Checkov against the Helm chart
- Interpreted scanner output in the context of VulnTracker
- Documented prioritised findings in `docs/findings.md`

## Task 3

- Removed SQL injection-prone query construction
- Removed password values from logs
- Added missing ownership validation
- Hardened Task 1 shared-link authorization
- Reduced exception information disclosure
- Documented deferred risks and compensating controls

## Task 4

- Built hardened Python and Node.js images
- Used non-root container execution
- Added health checks
- Created Kubernetes Deployments and ClusterIP Services
- Added NGINX Ingress
- Added NetworkPolicies and PodDisruptionBudgets
- Added resource limits and restrictive security contexts
- Integrated Azure Key Vault through Workload Identity and Secrets Store CSI

## Task 5

- Created a concise executive security summary
- Described security posture before and after remediation
- Documented residual risks
- Recommended production next steps
