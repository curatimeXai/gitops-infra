# Migration Issue Tickets & Scope Backlog

This document tracks all blockers, application-level bugs, hardcoded domain references, data migration requirements, and infrastructure issues discovered during the domain migration from `nightingaleheart.com` to `mlthrive.com`.

All tasks are strictly categorized by domain ownership and operational scope:
1. **Application Fixes** (Code / Docker image rebuilds)
2. **External Auth** (Auth0 tenant configuration)
3. **Storage / Data** (AWS S3/CloudFront data migrations)
4. **Cluster Infrastructure** (Pre-existing Kubernetes / CNI / Cilium networking issues)

---

## Recently Resolved

* **AI-WhatIf Domain Migration (September 14, 2026):** Fully completed. Frontend bundle rebuilt with `*.mlthrive.com` API endpoints, NGINX Ingress and Let's Encrypt TLS secret `aiwhatif-mlthrive-tls` provisioned, legacy Cilium HTTPRoutes removed from GitOps. ArgoCD application is `Synced / Healthy`.
* **Cilium Operator & IPAM Outage (Ticket #11 — September 14, 2026):** Resolved. Fixed Gateway API TLSRoute CRD compatibility (`v1alpha2 served=true`). Both `cilium-operator` pods recovered (`2/2 Ready`), worker node `medium-fbfpz-5brrp` received PodCIDR, Cilium DaemonSet `6/6 Ready`, blocked workloads (Segmentation3D backend, Loki) unblocked and running.
Ticket #2: PollutionMap frontend still uses old API domain and old API ingress host (September 14, 2026) Completed.
* Frontend was calling the wrong API domain. The API base URL was baked into the frontend code at build time, pointing to api.pollutionmap.nightingaleheart.com. I updated it to api.pollutionmap.mlthrive.com and triggered a rebuild/redeploy.
* Backend was rejecting requests from the new domain (CORS). Even after fixing #1, one feature (city search) still failed — the backend has a security setting (CORS) that only allows requests from an approved list of domains, and that list still only had the old nightingaleheart.com domains on it. So the backend was correctly blocking the new domain as "not recognized." I added the new domain to that allowlist and redeployed.
* Both Backend and Frontend shows READY 1/1 ArgoCD is Synced/Healthy.
* Ticket #6: SurgicSense frontend still uses old API domain after mlthrive.com migration (September 14, 2026):
* Root cause: The API URL was hardcoded in the frontend's js/config.js, still pointing at the old dead domain (api.surgicsense.nightingaleheart.com) instead of the new one (api.surgicsense.mlthrive.com).
* Fix: Updated that one line, committed to master. CI automatically built a new image, pushed it, and ArgoCD synced it to the cluster. Confirmed via kubectl that the new image is running, and confirmed live that login now works.
* ArgoCD is Synced/Healthy.
---

# 1. APPLICATION FIXES (Frontend Rebuilds / Hardcoded API URLs)

## Ticket #1: Causal Modeling frontend still uses old nightingaleheart.com API URL after domain migration

**Service:** Causal Modeling (`apps/causal-modeling`)  
**Component:** Frontend container (`causal-modeling-frontend`)  
**Discovered:** September 14, 2026  
**Status:** Open / Ready for Dev  
**Severity:** High (Blocks full deprecation of `nightingaleheart.com`)  

### Description

During the migration from `nightingaleheart.com` to `mlthrive.com`, the Causal Modeling application was tested after the new DNS, Ingress and TLS configuration had been added.

The new frontend and backend domains are working correctly:

```text
https://causal-modeling.mlthrive.com
https://api-causal-modeling.mlthrive.com
```

Infrastructure checks are OK:

```text
ArgoCD: Synced / Healthy
TLS certificates: Ready=True
Frontend: HTTP 200
Backend /health: HTTP 200
Backend /openapi.json: HTTP 200
```

The backend currently exposes the following endpoints:

```text
POST /analyze
POST /analyze.csv
GET  /health
```

The correct new API endpoint is therefore:

```text
https://api-causal-modeling.mlthrive.com/analyze
```

However, inspection of the currently running frontend container shows that the old production API URL is hardcoded inside the built Next.js bundle:

```text
https://api-causal-modeling.nightingaleheart.com/analyze
```

It was found in both the server-side and browser-side bundles:

```text
/app/.next/server/chunks/ssr/src_app_page_tsx_1chiuah._.js
/app/.next/static/chunks/28vg9yrib7uha.js
```

Example from the deployed bundle:

```javascript
let f=`https://api-causal-modeling.nightingaleheart.com/analyze?${e.toString()}`,
    g=await fetch(f,{method:"POST",body:c});
```

The currently deployed frontend image is:

```text
ghcr.io/curatimexai/causal-modeling-frontend:d86b32ad588b1c459eaadde61ad68ef246043988
```

There is currently no API URL configuration passed to the frontend through Kubernetes environment variables or `envFrom`, so the URL appears to be defined directly in the frontend source code or injected during the image build.

### Impact

`nightingaleheart.com` is being retired and is no longer intended to be used. Once the old domain is fully removed, the Causal Modeling frontend will not be able to submit analysis requests because it still sends them to `api-causal-modeling.nightingaleheart.com`.

### Required change

Update the frontend API base URL from `https://api-causal-modeling.nightingaleheart.com` to `https://api-causal-modeling.mlthrive.com`. Rebuild and redeploy the frontend Docker image.

### Acceptance criteria

- No references to `nightingaleheart.com` remain in the Causal Modeling frontend bundle
- Frontend requests are sent to `api-causal-modeling.mlthrive.com`
- `POST /analyze` reaches the new backend successfully
- Application works from `causal-modeling.mlthrive.com`
- ArgoCD remains Synced / Healthy after deployment

---

## Ticket #2: PollutionMap frontend still uses old API domain and old API ingress host

**Service:** PollutionMap (`apps/healthmap-pollutionmap`)  
**Component:** Frontend container (`pollutionmap-service-frontend`) & GitOps Ingress  
**Discovered:** September 14, 2026  
**Status:** Open / Ready for Dev (Ingress manifest fix already committed in GitOps)  
**Severity:** High (Frontend completely broken due to `ERR_NAME_NOT_RESOLVED` to dead domain)  

### Description

During the migration of PollutionMap from `nightingaleheart.com` to `mlthrive.com`, two issues were identified:

The new frontend domain is working:

```text
https://pollutionmap.mlthrive.com -> HTTP 200
```

The new API domain is:

```text
https://api.pollutionmap.mlthrive.com
```

#### 1. Frontend still uses the old API domain

Browser testing shows that the deployed PollutionMap frontend is still sending requests to the old domain:

```text
https://api.pollutionmap.nightingaleheart.com/point
https://api.pollutionmap.nightingaleheart.com/country
```

The browser fails with:

```text
net::ERR_NAME_NOT_RESOLVED
Failed to fetch pollution data: Network Error
Failed to fetch country averages: Network Error
```

The old `nightingaleheart.com` domain is being retired and must no longer be used. The frontend API base URL needs to be changed to `https://api.pollutionmap.mlthrive.com`.

#### 2. Incorrect API hostname in Kubernetes Ingress rules *(Resolved in GitOps)*

The last Ingress rule originally duplicated the old API hostname. This was corrected in GitOps repository commit `a81207e` (`apps/healthmap-pollutionmap/ingress.yaml`).

### Required changes

1. Change PollutionMap frontend API base URL to `https://api.pollutionmap.mlthrive.com`.
2. Rebuild and redeploy the PollutionMap frontend image.
3. Verify `/point` and `/country` through the new API domain.

### Acceptance criteria

- No frontend requests are sent to `api.pollutionmap.nightingaleheart.com`
- PollutionMap frontend uses `api.pollutionmap.mlthrive.com`
- `/point` and `/country` work through the new API domain
- `pollutionmap.mlthrive.com` loads data successfully without network errors
- ArgoCD remains Synced / Healthy after deployment

---

## Ticket #3: EchoGame frontend still uses old API domain after mlthrive.com migration

**Service:** EchoGame (`apps/healthview-echogame`)  
**Component:** Frontend container / image build (`echogame-frontend`)  
**Discovered:** September 14, 2026  
**Status:** Open / Ready for Dev  
**Severity:** High (Frontend unable to fetch questions or predict due to `ERR_NAME_NOT_RESOLVED` to dead domain)  

### Description

During the EchoGame domain migration, the new frontend and API infrastructure were successfully configured and verified:

```text
https://echogame.mlthrive.com     -> HTTP 200
https://api-echogame.mlthrive.com -> HTTP 200
```

The Kubernetes deployment was updated with:

```text
NEXT_PUBLIC_BACKEND_URL=https://api-echogame.mlthrive.com
NEXT_PUBLIC_API_URL=https://api-echogame.mlthrive.com
```

However, the deployed Next.js frontend bundle still contains the old API URL:

```text
GET https://api-echogame.nightingaleheart.com/api/questions
net::ERR_NAME_NOT_RESOLVED
Could not fetch questions: TypeError: Failed to fetch
```

The `NEXT_PUBLIC_*` values were embedded into the frontend bundle during the image build and cannot be overridden by Kubernetes environment variables alone.

### Required work

- Update the frontend build configuration to use `https://api-echogame.mlthrive.com`.
- Rebuild and redeploy the EchoGame frontend image.
- Verify `/api/questions` and `/api/predict` requests are sent to `https://api-echogame.mlthrive.com`.
- Verify that no frontend bundle references `api-echogame.nightingaleheart.com`.

---

## Ticket #4: EchoExplore frontend verification and legacy API references

**Service:** EchoExplore (`apps/healthview-echoexplore`)  
**Component:** Frontend container (`echoexplore-frontend`)  
**Discovered:** September 14, 2026  
**Status:** Open / Pending Frontend Audit  
**Severity:** Medium  

### Description

The infrastructure and backend for EchoExplore are working on `mlthrive.com`:

```text
https://echoexplore.mlthrive.com/        -> HTTP 200
https://api-echoexplore.mlthrive.com/    -> HTTP 200
https://echoexplore.mlthrive.com/predict -> HTTP 200
```

However, given that other Next.js/React frontends in the cluster (Causal Modeling, PollutionMap, EchoGame, HarmoniaHealth) have production API URLs baked into their Docker images at build time, the EchoExplore client bundle must be audited to ensure that no client-side calls are being directed to `api-echoexplore.nightingaleheart.com`.

### Required work

- Audit deployed EchoExplore frontend chunks for occurrences of `nightingaleheart.com`.
- If hardcoded legacy API URLs are found, update build config, rebuild, and redeploy image.
- Verify that ultrasound video predictions and API calls route cleanly to `api-echoexplore.mlthrive.com`.

---

## Ticket #5: HarmoniaHealth frontend still uses old API domain after mlthrive.com migration

**Service:** HarmoniaHealth (`apps/nightingale-harmoniahealth`)  
**Component:** Frontend container / image build (`harmoniahealth-frontend`)  
**Discovered:** September 14, 2026  
**Status:** Open / Ready for Dev  
**Severity:** High (Frontend unable to communicate with backend due to `ERR_NAME_NOT_RESOLVED` to dead domain)  

### Description

The HarmoniaHealth infrastructure migration to `mlthrive.com` is complete:

```text
https://harmoniahealth.mlthrive.com     -> HTTP 200
https://api-harmoniahealth.mlthrive.com -> HTTP 200
```

DNS, NGINX Ingress and Let's Encrypt TLS are working correctly.

However, the deployed frontend still sends requests to the legacy API domain:

```text
https://api-harmoniahealth.nightingaleheart.com/api/auth/register
```

Browser error:

```text
net::ERR_NAME_NOT_RESOLVED
[Auth] Auto-auth failed: AxiosError: Network Error
```

### Required work

- Update the frontend configuration to use `https://api-harmoniahealth.mlthrive.com`.
- Rebuild the HarmoniaHealth frontend image.
- Redeploy the new image.
- Verify that authentication and API requests are sent to `https://api-harmoniahealth.mlthrive.com`.
- Verify that no frontend bundle references `api-harmoniahealth.nightingaleheart.com` anymore.

---

## Ticket #6: SurgicSense frontend still uses old API domain after mlthrive.com migration

**Service:** SurgicSense (`apps/healthview-surgicsense`)  
**Component:** Frontend container / image build (`healthview-surgicsense-frontend`)  
**Discovered:** September 14, 2026  
**Status:** Open / Ready for Dev  
**Severity:** High (Frontend unable to communicate with backend due to `ERR_NAME_NOT_RESOLVED` to dead domain)  

### Description

SurgicSense infrastructure has been migrated to `mlthrive.com`:

```text
Frontend: https://surgicsense.mlthrive.com
API:      https://api.surgicsense.mlthrive.com
```

DNS, TLS and Kubernetes Ingress are working.

However, the frontend still sends API requests to the expired domain:

```text
POST https://api.surgicsense.nightingaleheart.com/auth/register
```

Browser error:

```text
net::ERR_NAME_NOT_RESOLVED
```

### Required work

- Update the frontend API configuration/source to:
  `https://api.surgicsense.mlthrive.com`
- Rebuild and redeploy the SurgicSense frontend image if the value is baked into the image.
- Verify that registration and authentication requests reach `https://api.surgicsense.mlthrive.com`.
- Verify that no frontend bundle references `api.surgicsense.nightingaleheart.com` anymore.

*This is an application configuration/build issue and is separate from the domain/Ingress migration.*

---

# 2. EXTERNAL AUTH (Auth0 Tenant Configuration)

## Ticket #7: Update Auth0 configuration for mlthrive.com (HeartAware, Cardiomegaly, SurgicSense)

**Services:** 
- `nightingale-heartaware` (`heartaware.mlthrive.com`)
- `healthview-cardiomegaly-cnn` (`cardiomegaly-cnn.mlthrive.com`)
- `healthview-surgicsense` (`surgicsense.mlthrive.com`)  
**Component:** Auth0 Applications & Kubernetes Secrets  
**Discovered:** September 14, 2026  
**Status:** Blocked (Waiting for Auth0 Tenant Admin Credentials)  
**Severity:** High (Users cannot complete OAuth login / authentication flow)  

### Description

Several applications in the UpCloud cluster authenticate users against the Auth0 tenant:

```text
https://nightingale-heart-production.eu.auth0.com
```

The TLS, DNS, and Ingress routing for these domains are live on `mlthrive.com`, but the OAuth login flow redirects or fails because the new domains are not registered in the Auth0 Application settings.

### Required Subtasks

#### Subtask 7.1: HeartAware
1. In Auth0 Dashboard (Application: HeartAware), add to:
   - **Allowed Callback URLs:** `https://heartaware.mlthrive.com/auth/callback`
   - **Allowed Logout URLs:** `https://heartaware.mlthrive.com`
   - **Allowed Web Origins:** `https://heartaware.mlthrive.com`
2. Update Kubernetes Secret `heartaware-secrets` in namespace `nightingale-heartaware`:
   - Set `APP_BASE_URL: https://heartaware.mlthrive.com`
3. Restart pod: `kubectl -n nightingale-heartaware rollout restart deployment/nightingale-heartaware-frontend`

#### Subtask 7.2: Cardiomegaly CNN
1. In Auth0 Dashboard (Application: Cardiomegaly), add to:
   - **Allowed Callback URLs:** `https://cardiomegaly-cnn.mlthrive.com/callback`
   - **Allowed Logout URLs:** `https://cardiomegaly-cnn.mlthrive.com`
   - **Allowed Web Origins:** `https://cardiomegaly-cnn.mlthrive.com`
2. Update `cardiomegaly-secret` in namespace `healthview-cardiomegaly-cnn` if base URL or callback URL is stored inside the secret.
3. Restart deployment: `kubectl -n healthview-cardiomegaly-cnn rollout restart deployment/cardiomegaly-frontend`

#### Subtask 7.3: SurgicSense
1. In Auth0 Dashboard (Application: SurgicSense), add to:
   - **Allowed Callback URLs:** `https://surgicsense.mlthrive.com/callback`
   - **Allowed Logout URLs:** `https://surgicsense.mlthrive.com`
   - **Allowed Web Origins:** `https://surgicsense.mlthrive.com`
2. Update `surgicsense-secret` in namespace `healthview-surgicsense` as required.
3. Restart deployment: `kubectl -n healthview-surgicsense rollout restart deployment/healthview-surgicsense-frontend`

---

# 3. STORAGE & DATA (Media & Models Migration from AWS)

## Ticket #8: Migrate EchoGame media storage and CDN from AWS to UpCloud

**Service:** EchoGame (`apps/healthview-echogame`)  
**Component:** Media Storage & CDN (`cdn.echogame.nightingaleheart.com` / AWS CloudFront / S3)  
**Discovered:** September 14, 2026  
**Status:** Open / Backlog (Infra & Data Migration)  
**Severity:** Medium (Media dependency on legacy AWS CloudFront / dead domain)  

### Description

During the `mlthrive.com` domain migration, the EchoGame frontend and API were successfully migrated to `echogame.mlthrive.com` and `api-echogame.mlthrive.com`.

However, EchoGame still depends on the legacy media CDN:

```text
https://cdn.echogame.nightingaleheart.com
```

The legacy CDN is an AWS CloudFront distribution:

```text
Distribution ID:  E138H0T5HME5VT
CloudFront domain: dc3pdcj61u7t1.cloudfront.net
Configured S3 origin: echogame-videos-mp4.s3.eu-central-1.amazonaws.com
```

The application references this CDN through:

```text
NEXT_PUBLIC_S3_BASE_URL
CDN_BASE_URL
```

The old `nightingaleheart.com` DNS no longer resolves, so this media dependency must be migrated before the AWS infrastructure is decommissioned.

### Required work

- Identify the authoritative EchoGame video dataset in AWS.
- Migrate the media files to UpCloud Object Storage.
- Configure a new CDN/public endpoint, e.g. `cdn.echogame.mlthrive.com`.
- Update `NEXT_PUBLIC_S3_BASE_URL` in the frontend.
- Update `CDN_BASE_URL` in the backend.
- Verify video loading through the new domain.
- Decommission the legacy CloudFront/S3 resources after validation.

---

## Ticket #9: Segmentation3D models storage migration AWS -> approved storage

**Service:** 3D Segmentation (`apps/healthview-segmentation3d`)  
**Component:** 3D AI Models Storage (`S3_MODELS_URL`)  
**Discovered:** September 14, 2026  
**Status:** Open / Backlog (Data Migration)  
**Severity:** Medium (External AWS S3 storage dependency)  

### Description

During the domain migration, the Segmentation3D frontend configuration map [apps/healthview-segmentation3d/frontend-configmap.yaml](file:///d:/Projects/Upcloud/gitops-infra/apps/healthview-segmentation3d/frontend-configmap.yaml) was updated to point `API_BASE_URL` to `https://segmentation-api.mlthrive.com`.

However, the 3D model assets are still hosted in an AWS S3 bucket:

```javascript
const S3_MODELS_URL = "https://healthview-segmentation3d-models-prod.s3.eu-central-1.amazonaws.com/models";
```

This S3 dependency remains active on AWS and must be migrated to approved storage (UpCloud Object Storage) prior to shutting down AWS resources.

### Required work

- Replicate the model files from `healthview-segmentation3d-models-prod.s3.eu-central-1.amazonaws.com/models` to UpCloud Object Storage.
- Update `S3_MODELS_URL` in `frontend-configmap.yaml`.
- Decommission the AWS S3 bucket.

---

## Ticket #10: EchoExplore media storage dependency audit and migration

**Service:** EchoExplore (`apps/healthview-echoexplore`)  
**Component:** Ultrasound media storage  
**Discovered:** September 14, 2026  
**Status:** In Audit / Pending Confirmation  
**Severity:** Low / Informational  

### Description

AWS accounts previously contained EchoExplore buckets. Before decommissioning AWS resources, an audit must verify whether the currently deployed EchoExplore backend/frontend actively pulls videos from AWS S3 or if it already relies on local volumes / UpCloud Object Storage. If active AWS storage dependencies exist, a migration plan to UpCloud Object Storage will be executed.

---

## Ticket #13: Audit and verify AWS data migration before AWS decommissioning

**Service:** Cross-cluster Data & Storage (`AWS S3`, `AWS RDS`, UpCloud Object Storage, UpCloud Managed DBs)  
**Component:** AWS Account Decommissioning Gatekeeper  
**Discovered:** September 14, 2026  
**Status:** Open / Audit & Verification in Progress  
**Severity:** High (Data Loss Prevention & Decommissioning Blocker)  

### Description

The application/domain infrastructure has been migrated to UpCloud, and the standard Kubernetes/Ingress/TLS domain migration is now largely complete.

However, AWS still contains a number of S3 buckets (29 total) and RDS databases (8 instances).

Before the AWS account can be decommissioned, we need to verify which AWS resources:
- Are still required by applications;
- Have already been migrated to UpCloud;
- Were only copied during an earlier AWS account migration;
- Are legacy/test/system resources and can be removed;
- Still need to be migrated.

**No AWS S3 bucket or RDS database should be deleted until its status has been confirmed.**

*A detailed standalone audit document is maintained at [aws-data-migration-audit.md](file:///D:/Projects/Upcloud/aws-data-migration-audit.md).*

### Current UpCloud Object Storage

| UpCloud service | Bucket | Approx. size | Purpose |
|---|---|---:|---|
| static_front_end | `main-website` | 97.72 MB | Main static website |
| app-bucket | `aiwhatif` | 65.31 MB | AI-WhatIf application data |
| app-bucket | `surgicsense` | 934.81 MB | SurgicSense application data |
| nightingale-prod-terraform-state | `nightingale-prod-tfstate` | 0.07 MB | Terraform state |
| loki-object-storage | `loki-chunks` | ~14 GB | Loki logs |
| loki-object-storage | `loki-admin` | 0 | Loki |
| loki-object-storage | `loki-ruler` | 0 | Loki |

At the moment there are no dedicated UpCloud application buckets visible for EchoGame, EchoExplore, Segmentation3D, Cardiomegaly, ECG Prediction, LifeSaver, Bogalusa or HeartClusters.

This needs to be compared against the AWS S3 inventory.

### AWS S3 Inventory

| AWS bucket | Application / purpose | Current migration status |
|---|---|---|
| `healthview-cardiomegaly-data-prod` | Cardiomegaly | To verify |
| `healthview-ecgprediction-data-prod` | ECG Prediction | To verify |
| `healthview-echoexplore-videos-prod` | EchoExplore | UpCloud equivalent not found |
| `healthview-echogame-videos-prod` | EchoGame | UpCloud equivalent not found |
| `healthview-segmentation3d-assets-prod` | Segmentation3D | UpCloud equivalent not found |
| `healthview-segmentation3d-models-prod` | Segmentation3D models | **Still referenced by application / migration required or status must be verified** |
| `nightingale-aiwhatif-data-prod` | AI-WhatIf | Compare with UpCloud `aiwhatif` |
| `nightingale-bogalusa-data-prod` | Bogalusa | To verify |
| `nightingale-heartcluster-assets-prod` | HeartClusters | To verify |
| `nightingale-lifesaver-models-prod` | LifeSaver | To verify |
| `aiwhatif-v2-bucket-migrated` | AI-WhatIf / previous migration | Determine whether legacy or source data |
| `echoexplore-videos-migrated` | EchoExplore / previous migration | Determine whether legacy or source data |
| `echogame-videos-mp4-migrated` | EchoGame / previous migration | Determine whether legacy or source data |
| `segmentation-3d-models-migrated` | Segmentation3D / previous migration | Determine whether legacy or source data |
| `cardiomegaly-counterfactuals-migrated` | Cardiomegaly | Determine whether still required |
| `cardiomegaly-frontend-migrated1` | Cardiomegaly frontend | Likely legacy/static artifact – verify |
| `3d-frontend-deployment-migrated` | Segmentation3D frontend | Likely legacy/static artifact – verify |
| `bogalusa-nightingaleheart-migrated` | Bogalusa | Verify |
| `nightingale-heartclusters-frontend-migrated` | HeartClusters | Verify |
| `admin-devsettings-migrated` | Admin/dev settings | Owner/purpose unknown |
| `nightinblaze1-migrated` | Unknown | Identify owner/purpose |
| `ene-testbucket-migrated` | Test bucket | Likely deletion candidate after verification |
| `elasticbeanstalk-eu-central-1-654654611936-migrated` | AWS Elastic Beanstalk | AWS/legacy resource – verify |
| `sagemaker-eu-central-1-654654611936-migrated` | SageMaker | AWS/legacy resource – verify |
| `sagemaker-eu-north-1-654654611936-migrated` | SageMaker | AWS/legacy resource – verify |
| `sagemaker-studio-654654611936-qi3rk2cq8x-migrated` | SageMaker Studio | AWS/legacy resource – verify |
| `spotify-backend-builds-654654611936-eu-west-1-migrated` | Build artifacts | Verify whether still required |
| `aws-cloudtrail-logs-300763413277-5585bf5e` | CloudTrail logs | AWS system/audit data – retention decision required |
| `nightingale-terraform-state-prod-300763413277` | Terraform state | Compare with current UpCloud Terraform state before deletion |

Some `*-migrated` buckets may originate from an earlier AWS-account-to-AWS-account migration rather than the UpCloud migration and should not automatically be considered migrated to UpCloud.

### Database Services

UpCloud currently has two Managed Database services:

| UpCloud service | Engine | State |
|---|---|---|
| `postgres-dbs` | PostgreSQL | Running |
| `mysql-dbs` | MySQL | Running |

AWS still contains the following RDS instances:

| AWS RDS | Engine | Application | Expected UpCloud target |
|---|---|---|---|
| `healthmap-adaptatutor-db-prod` | PostgreSQL | AdaptaTutor | `postgres-dbs` |
| `healthmap-heartsayings-db-prod` | MySQL | Heart Sayings | `mysql-dbs` |
| `healthmap-pollutionmap-db-prod` | MySQL | PollutionMap | `mysql-dbs` |
| `healthmap-worldhealthmap-db-prod` | MySQL | WorldHealthMap | `mysql-dbs` |
| `healthview-cardiomegaly-db-prod` | MySQL | Cardiomegaly | `mysql-dbs` |
| `nightingale-harmoniahealth-db-prod` | PostgreSQL | HarmoniaHealth | `postgres-dbs` |
| `nightingale-lifesaver-db-prod` | PostgreSQL | LifeSaver | `postgres-dbs` |
| `surgicsense-db` | PostgreSQL | SurgicSense | `postgres-dbs` |

The UpCloud database services exist, but the logical databases and their contents have not yet been fully verified against AWS RDS.

### Known Active AWS Dependencies

1. **Segmentation3D** currently has a direct reference to:
   `healthview-segmentation3d-models-prod.s3.eu-central-1.amazonaws.com`
   so this AWS bucket cannot currently be removed. The reference exists in the application configuration.
2. **EchoExplore** also has an AWS/S3 configuration Secret attached to its backend and needs to be verified.
3. **EchoGame** still has an old CDN/S3 dependency and requires separate verification/migration.

### Required Investigation

For each AWS S3 bucket and RDS database:

1. Identify the owning application or system.
2. Determine whether the resource is still used.
3. Find the corresponding UpCloud resource.
4. Compare the AWS and UpCloud data.
5. Verify that the UpCloud copy is complete and sufficiently up to date.
6. Verify that the application uses the UpCloud resource rather than AWS.
7. Mark the AWS resource as one of:
   - `MIGRATED / VERIFIED`
   - `MIGRATION REQUIRED`
   - `LEGACY`
   - `SYSTEM / RETENTION REQUIRED`
   - `UNKNOWN / OWNER REQUIRED`
8. Create separate migration/fix tickets for resources that are still required but not present in UpCloud.

### Acceptance Criteria

AWS can only be considered ready for decommissioning when every S3 bucket and RDS database has an explicit disposition and all required application data has been verified in UpCloud.

---

# 4. CLUSTER INFRASTRUCTURE (Pre-existing Kubernetes / CNI / Cilium Issues)

## Ticket #11: RESOLVED — Cilium operator / Gateway API TLSRoute compatibility blocked IPAM

**Service:** Cluster-wide CNI / IPAM (`kube-system/cilium-operator`, `cilium`)  
**Component:** Cilium Operator & Gateway API TLSRoute CRD compatibility  
**Discovered:** September 14, 2026  
**Resolved:** September 14, 2026  
**Status:** Resolved / Verified  
**Cilium:** 6/6 Ready  
**cilium-operator:** 2/2 Ready  
**medium-fbfpz-5brrp:** Ready  
**PodCIDR:** 192.168.4.0/24  
**Severity:** Critical → Resolved

### Root Cause

The cluster runs Cilium 1.18.6 with Gateway API enabled.

`Cilium 1.18.6` expected `TLSRoute/v1alpha2` (`gateway.networking.k8s.io/v1alpha2`), whereas the installed Gateway API v1.6.1 standard CRD had:

```text
v1        served=true  storage=true
v1alpha2  served=false storage=false
v1alpha3  served=false storage=false
```

Because `v1alpha2` had `served=false`, both `cilium-operator` replicas crashed during startup with:

```text
failed to setup field indexer "backendServiceTLSRouteIndex":
no matches for kind "TLSRoute" in version
"gateway.networking.k8s.io/v1alpha2"
```

Because both Cilium operators were crashed in `CrashLoopBackOff`, cluster-pool IPAM stopped allocating new PodCIDRs.

Worker node `medium-fbfpz-5brrp` was left without a PodCIDR allocation, causing its local Cilium agent (`cilium-jmhx6`) to continuously fail:

```text
required IPv4 PodCIDR not available
```

Workloads scheduled to `medium-fbfpz-5brrp` remained stuck in `ContainerCreating` / `FailedCreatePodSandBox` because the Cilium CNI sandbox could not be created.

### Resolution / Verification

1. **Backup:** A backup of the existing `tlsroutes.gateway.networking.k8s.io` CRD was exported (`tlsroutes-crd-backup-v1.6.1-standard.yaml`).
2. **Compatibility Fix:** A targeted compatibility patch was applied to the CRD:
   ```text
   v1alpha2 served=false  →  v1alpha2 served=true
   ```
   No CRD downgrade, CiliumNode manual edit, or full Gateway API bundle replacement was performed.
3. **Recovery & Verification:**
   * Both `cilium-operator` replicas immediately recovered to `2/2 Ready`.
   * Cilium DaemonSet recovered to `6/6 Ready`.
   * Worker node `medium-fbfpz-5brrp` received PodCIDR `192.168.4.0/24`.
   * The local Cilium agent became `1/1 Running` and initialized `/var/run/cilium/cilium.sock`.
   * Workloads previously blocked on the node (including `healthview-segmentation3d-backend` and Loki) successfully created network sandboxes and became `Running`.
   * Node was uncordoned and is healthy:
     ```text
     Status:            Resolved / Verified
     Cilium:            6/6 Ready
     cilium-operator:   2/2 Ready
     medium-fbfpz-5brrp: Ready
     PodCIDR:           192.168.4.0/24
     ```

### Critical Upgrade Warning / Follow-up

> [!WARNING]
> **CRITICAL INFRASTRUCTURE WARNING: Gateway API CRD Compatibility & cilium-operator**  
> Setting `v1alpha2 served=true` on CRD `tlsroutes.gateway.networking.k8s.io` is a **runtime compatibility fix** for Cilium 1.18.6.  
> If the Gateway API CRDs are reinstalled or upgraded in the future, standard upstream bundles will overwrite this field back to `served=false`.  
> This will immediately crash both `cilium-operator` pods, halting cluster-pool IPAM and preventing pods from launching on newly joined nodes.  
> **Mandatory upgrade procedure:** Before any Kubernetes, Cilium, or Gateway API upgrade, verify `TLSRoute` version requirements and ensure `v1alpha2 served=true` is preserved until Cilium is upgraded to a version that no longer depends on `v1alpha2`.

### Domain Migration Impact

The issue was not caused by the `nightingaleheart.com` → `mlthrive.com` domain migration; it was a pre-existing infrastructure incompatibility revealed during migration validation.

AI-WhatIf no longer relies on Cilium HTTPRoutes. Legacy `nightingaleheart.com` HTTPRoutes were removed from GitOps after migration to NGINX Ingress (`aiwhatif-ingress`, `aiwhatif-mlthrive-tls`), and the ArgoCD application is `Synced / Healthy`.

---

## Ticket #12: AdaptaTutor backend migration to UpCloud appears incomplete

**Service:** AdaptaTutor (`apps/healthmap-adaptatutor`)  
**Component:** Backend Deployment (`healthmap-adaptatutor-backend`), Secret (`adaptatutor-backend-secret`) & Database  
**Discovered:** September 14, 2026 (during domain cutover audit)  
**Status:** Open / Blocked on Legacy Backend Migration  
**Severity:** High (Backend 503, database and secret credentials never configured)  

### Description

Following the resolution of the Cilium/IPAM outage (Ticket #11), the AdaptaTutor backend pod successfully received its CNI network sandbox and was allocated an IPv4 PodIP (`192.168.4.x`). However, the container immediately failed startup and entered **`CrashLoopBackOff`**.

This runtime behavior definitively confirms that the remaining AdaptaTutor failure is an **application, backend configuration, database, and secrets issue**, and **not** a Kubernetes CNI or networking problem.

The production Secret still contains placeholder credentials (`CHANGE_ME`) and legacy origin values, and the API remains unavailable:

```text
DATABASE_URL = CHANGE_ME
AZURE_SPEECH_KEY = CHANGE_ME
AZURE_SPEECH_ENDPOINT = CHANGE_ME
JWT_SECRET_KEY = CHANGE_ME
ADMIN_USERNAME = CHANGE_ME
ADMIN_EMAIL = CHANGE_ME
ADMIN_PASSWORD = CHANGE_ME
FRONTEND_URL = https://adaptatutor.nightingaleheart.com
ALLOWED_ORIGINS = https://adaptatutor.nightingaleheart.com
```

The Secret is managed directly by the ArgoCD application and originates from `apps/healthmap-adaptatutor/secret.example.yaml`. No ExternalSecret, SealedSecret or SOPS-based secret management was identified in the GitOps repository.

### Strategy Note
No further changes should be made to Adaptatutor at this stage of the domain migration. When the engineer or team responsible for backend/data/secrets restores the real production credentials, the origin URLs must be updated in that same change:

```text
FRONTEND_URL=https://adaptatutor.mlthrive.com
ALLOWED_ORIGINS=https://adaptatutor.mlthrive.com
```

### Current state

| Component | Status | Note |
| :--- | :---: | :--- |
| DNS | ✅ | `adaptatutor.mlthrive.com`, `api-adaptatutor.mlthrive.com` pointed to Ingress LB |
| Ingress / TLS | ✅ | Fully cut over to `mlthrive.com` (legacy hosts removed); Let's Encrypt certificates valid |
| Frontend | ✅ | Reachable on new domain (`200 OK`) |
| Backend / Data / Secrets | ❌ | Incomplete previous migration (placeholder credentials, predates domain migration) |

### Required work

When the engineer/team responsible for backend, data and secrets restores the real production credentials:

1. Identify the intended production PostgreSQL database and set `DATABASE_URL`.
2. Recover and configure the required production credentials (`JWT_SECRET_KEY`, `ADMIN_*`).
3. Verify Azure Speech configuration if still required (`AZURE_SPEECH_*`).
4. Update CORS / origin configuration to the new domain in that same change:
   ```yaml
   FRONTEND_URL: "https://adaptatutor.mlthrive.com"
   ALLOWED_ORIGINS: "https://adaptatutor.mlthrive.com"
   ```
5. Restore and scale the backend deployment.
6. Verify `/health` and application functionality.

*Backend/data/secrets configuration is separate from the domain/Ingress migration.*


