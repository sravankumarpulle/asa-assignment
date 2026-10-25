# VulnTracker Security Findings

## 1. Overview

I reviewed VulnTracker from both an application and deployment security
perspective.

The review covered the Python/FastAPI API, the Node.js notification service,
application dependencies, container images, and the Helm deployment
configuration.

I used:

- Semgrep for static application security testing
- Trivy for dependency and container vulnerability scanning
- Checkov for Helm/Kubernetes configuration scanning
- Manual code review for authentication, authorization, logging, error
  handling, shared-report links, and network exposure

I did not automatically accept the severity reported by each scanner.
Each result was reviewed in the context of how VulnTracker works and the type
of data it handles.

The findings below are ordered based on the risk they present to the
application.

---

## 2. Prioritised Findings

| ID | Finding | Source | Severity | Business Impact | Introduced In |
|---|---|---|---|---|---|
| VT-001 | SQL injection risk in scan search | Semgrep + Manual review | **Critical** - The search implementation builds SQL using user-controlled input instead of safely parameterizing the query. | An authenticated user could potentially manipulate the search query and access data outside the normal search behavior. Because VulnTracker stores vulnerability details, CVEs, affected components, and remediation information, unauthorized database access would have a significant security impact. | Starter code |
| VT-002 | Passwords are written to application logs | Semgrep + Manual review | **High** - The login implementation logs the password supplied during both normal and failed login attempts. | Anyone with access to application or central logging systems could potentially recover user credentials. Those credentials could then be used to access vulnerability records stored in VulnTracker. | Starter code |
| VT-003 | Missing ownership check when retrieving scans | Manual review | **High** - `GET /scans/{scan_id}` verifies that the caller is authenticated but does not verify that the requested scan belongs to that user. | A logged-in user could potentially request another user's scan by changing the scan ID. This could expose vulnerability descriptions, CVE information, affected components, severity, and remediation notes belonging to another user. | Starter code |
| VT-004 | Vulnerable JWT library | Trivy SCA / Container scan | **Critical** - The installed `python-jose 3.3.0` dependency was reported with security vulnerabilities relevant to JWT handling. | Authentication in VulnTracker depends on JWT tokens. A weakness in token verification could potentially affect the authentication boundary and allow unauthorized access to protected scan data. | Starter dependency |
| VT-005 | Outdated HTTP/form-processing dependencies | Trivy SCA / Container scan | **High** - Trivy reported High-severity issues in dependencies including `python-multipart 0.0.6` and `starlette 0.27.0`. | Malformed HTTP requests could potentially consume excessive CPU or memory and affect API availability. The practical risk depends on whether the vulnerable code paths are reachable through VulnTracker's endpoints. | Starter dependencies |
| VT-006 | Internal exception details are returned to API clients | Manual review | **Medium** - The global exception handler returns the exception message, exception type, traceback, and request path. | Detailed error information can reveal internal implementation details that make it easier for an attacker to understand the application and develop targeted attacks. | Starter code |
| VT-007 | CORS policy is too permissive | Manual review | **Medium** - The API reflects the caller's Origin and allows credentials together with broad HTTP methods and headers. | An untrusted website could receive cross-origin access that was not intended. The final impact depends on how VulnTracker clients store and send their authentication credentials. | Starter code |
| VT-008 | No rate limiting on password-protected shared links | Manual review | **Medium** - The public sharing endpoint verifies the optional password but does not currently limit repeated attempts. | If an attacker obtains a valid sharing token, they could repeatedly try passwords against the public endpoint. This reduces the protection provided by the optional share password. | Task 1 feature |
| VT-009 | Share password is provided in the URL query string | Manual review | **Medium** - The assignment requires the password to be supplied as a query parameter. Query parameters can be captured by browser history, reverse proxies, or access logs if those systems are not configured carefully. | Exposure of the share password would reduce the protection around a vulnerability report sent to a customer or auditor. The password is hashed in the database, but that does not prevent exposure while the request is being sent. | Task 1 feature / assignment requirement |
| VT-010 | FastAPI container contains vulnerable runtime packages | Trivy Container scan | **High** - The image scan reported Critical and High vulnerabilities in runtime components and operating-system packages. | An exploitable runtime vulnerability could affect the confidentiality, integrity, or availability of the FastAPI service. Running the container as a non-root user reduces the impact of some local escalation issues but does not remove the underlying vulnerabilities. | Container image |
| VT-011 | Notification container uses an unsupported/outdated base image | Trivy Container scan | **High** - The Node.js notification image was reported as using an Alpine base that requires lifecycle remediation. | Continuing to run an unsupported or outdated base image increases the chance that operating-system vulnerabilities will remain unpatched. This increases the long-term attack surface of the notification service. | Container image |
| VT-012 | Kubernetes images are referenced using mutable tags | Checkov IaC | **Medium** - Container images are referenced using tags rather than immutable SHA-256 digests. | A tag can later point to different container content. This means the image deployed to Kubernetes might not always be the exact same artifact that was reviewed and security scanned. | Helm deployment |
| VT-013 | Kubernetes workloads are not isolated in a dedicated namespace | Checkov IaC | **Medium** - The deployment does not currently enforce a dedicated VulnTracker namespace. | A dedicated namespace would provide a clearer security boundary for RBAC, NetworkPolicies, resource controls, and operational management. | Helm deployment |
| VT-014 | Container image pull policy uses `IfNotPresent` | Checkov IaC | **Low** - Kubernetes may reuse an image already cached on a node when the configured tag already exists locally. | This can create deployment consistency issues when mutable tags are used. The more important long-term remediation is to deploy images using immutable digests. | Helm deployment |
| VT-015 | Unnecessary direct service exposure | Checkov IaC + Manual review | **High** - Earlier deployment configuration allowed the application service to be directly exposed instead of forcing external traffic through the intended ingress layer. | Direct exposure increases the attack surface and can bypass centralized ingress controls. The Helm design should expose only the FastAPI service through the NGINX Ingress Controller while keeping the notification service internal. | Helm deployment |

---

## 3. Scanner Results Requiring Manual Validation

Not every scanner result was treated as a confirmed vulnerability. I reviewed
the following findings separately because their severity depends on how the
affected code is actually used.

### JWT `none` algorithm finding

Semgrep reported a possible JWT `none` algorithm issue in `app/auth.py`.

I did not automatically classify this as an authentication bypass. The
important point is whether the JWT decoding logic accepts an attacker-selected
algorithm or explicitly restricts verification to the expected algorithm.

I kept the finding in the raw Semgrep report, but it should only be treated as
a confirmed authentication vulnerability after reviewing the complete JWT
verification path.

### CSRF finding in the notification service

Semgrep reported that CSRF protection was not detected in the Express
notification service.

I did not automatically treat this as High severity because CSRF is mainly a
concern when a browser automatically attaches authentication credentials such
as cookies to requests.

The notification service is intended primarily for internal service-to-service
communication, so the scanner result needs to be considered together with its
actual exposure and authentication model.

### Possible mass assignment in the notification service

Semgrep also identified the use of `Object.assign` as a possible
mass-assignment issue.

This needs to be checked against the actual properties a caller is allowed to
provide. If user-controlled properties can modify security-sensitive
application state, the finding becomes more serious.

For now, I retained the scanner result without treating it as a confirmed
High-severity vulnerability.

---

## 4. Security Controls Added

The deployment work also introduced several controls to reduce the application's
attack surface.

### Container security

The containers are configured to:

- run as a non-root user
- prevent privilege escalation
- drop unnecessary Linux capabilities
- use a restricted security context
- expose only the application ports that are required
- use health checks for application monitoring

### Kubernetes security

The Helm deployment includes:

- CPU and memory requests and limits
- `RuntimeDefault` seccomp configuration
- read-only container filesystems where supported
- Kubernetes service-account token automounting disabled
- liveness and readiness probes
- NetworkPolicies
- ClusterIP services instead of directly exposing application pods
- Secrets Store CSI integration for retrieving secrets from an external
  secrets manager

### Network exposure

The intended deployment flow is:

NGINX Ingress Controller -> FastAPI -> Notification Service

Only the FastAPI service needs external ingress.

The Node.js notification service is used for internal service-to-service
communication and therefore does not need separate public exposure.

These controls reduce the impact of an application vulnerability, but they do
not replace fixing vulnerable application code or dependencies.

---

## 5. Remediation Priority

### Immediate priority

The following issues have the highest potential impact and should be addressed
first:

1. **VT-001 - SQL injection in scan search**
2. **VT-004 - Vulnerable JWT dependency**
3. **VT-002 - Passwords written to logs**
4. **VT-003 - Missing scan ownership validation**

These issues affect core security boundaries such as database access,
authentication, credentials, and authorization.

### High priority

The next group is:

- vulnerable HTTP/form-processing dependencies
- vulnerable packages in container images
- outdated container base images
- unnecessary direct Kubernetes service exposure

These issues mainly affect service availability, container security, and
deployment attack surface.

### Hardening improvements

The remaining findings should also be addressed, but they have lower immediate
risk than the authentication, authorization, and injection issues.

These include:

- limiting repeated shared-link password attempts
- handling share passwords carefully in logs
- tightening CORS configuration
- deploying images by immutable digest
- using a dedicated Kubernetes namespace
- improving image pull behavior

---

## 6. Remediation Approach

Based on the findings above, I would prioritize the remediation work as
follows:

1. Replace the dynamically constructed scan-search SQL with parameterized
   SQLAlchemy queries.

2. Remove passwords and other sensitive authentication values from application
   logs.

3. Enforce ownership checks consistently when reading, searching, updating,
   deleting, or sharing scans.

4. Upgrade vulnerable authentication and HTTP-processing dependencies and run
   regression tests after each upgrade.

5. Rebuild application images using supported and patched minimal base images.

6. Keep only the FastAPI application externally reachable through the NGINX
   Ingress Controller. Keep the notification service internal.

7. Restrict pod-to-pod traffic using Kubernetes NetworkPolicies.

8. Retrieve runtime secrets from the external secrets manager through Secrets
   Store CSI rather than storing credentials in Helm values or manifests.

9. Pin production images using immutable SHA-256 digests once the final images
   are published to the container registry.

10. Add rate limiting and monitoring around the public shared-report endpoint.

11. Make sure share passwords and other sensitive query parameters are redacted
    from ingress, proxy, and application logs.

---

## 7. Conclusion

The initial review found several important weaknesses in the starter
application, particularly around SQL handling, credential logging,
authorization, dependencies, and error handling.

The new shared-report feature also introduces security considerations because
it intentionally allows external access without normal user authentication.
For that reason, the implementation uses unpredictable share tokens,
24-hour expiry, optional password protection, and ownership validation when a
sharing link is created.

The deployment design adds additional defence through non-root containers,
restricted security contexts, NetworkPolicies, external secret management,
resource limits, and controlled ingress through NGINX.

The remaining risks are documented separately in `docs/remediation-plan.md`
together with the proposed remediation effort and compensating controls.
