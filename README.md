# uptime-watch-deploy

Kubernetes manifests for [uptime-watch](https://github.com/MK-Sindhu/uptime-watch)
on Google Kubernetes Engine. This repo holds no application code: it says
what runs in the cluster, and Argo CD makes the cluster match it (GitOps).

## How a change reaches production

```mermaid
flowchart LR
    dev["push to main<br/>uptime-watch"] --> ci["CI<br/>types, lint, tests"]
    ci --> img["build 5 images<br/>tag = commit SHA"]
    img --> gar[("Artifact Registry")]
    img --> bump["commit new tags to<br/>overlays/production"]
    bump --> repo[("uptime-watch-deploy")]
    repo -->|"watched by"| argo["Argo CD<br/>in the cluster"]
    argo --> gke["GKE: migrate, then<br/>roll out the pods"]
    gar -.->|"pull"| gke
```

1. A push to `main` in uptime-watch runs CI (`.github/workflows/cd.yml` there).
2. CI pushes the images, tagged with the commit SHA, and commits the new tags
   to `apps/uptime-watch/overlays/production/kustomization.yaml` here.
3. Argo CD sees the commit and syncs: config and secrets first, then the
   database migration Job, then the new pods.

Nothing outside the cluster can change it: CI only has write access to this
repo, never to Kubernetes. To roll back, `git revert` the deploy commit.

## What's in here

```
bootstrap/
  argocd/              installs Argo CD (pinned version), applied once by hand
  root-app.yaml        the one Application created by hand: "everything in argocd/apps"
argocd/apps/
  external-secrets.yaml  Helm chart: Secret Manager -> Kubernetes Secrets
  keda.yaml              Helm chart: scales workers on Redis stream lag
  uptime-watch.yaml      the app: apps/uptime-watch/overlays/production
apps/uptime-watch/
  base/                everything the app runs, environment-independent
    api.yaml             API Deployment, Service, autoscaler, disruption budget
    fe.yaml              frontend, plus the load balancer's health check and Cloud Armor
    pusher.yaml          exactly one pusher
    workers.yaml         one worker Deployment per region, scaled by KEDA
    migrate-job.yaml     `prisma db update` + region rows, before every rollout
    cloud-sql-proxy.yaml sidecar added to api, pusher and the migration
    secrets.yaml         where each secret comes from in Secret Manager
    ingress.yaml         static IP, managed TLS certificate, http -> https
  overlays/production/ every value specific to your project, and the image tags
```

What runs where:

| Piece | Where | Why |
|---|---|---|
| fe, api, pusher, workers | GKE Autopilot, `asia-south1` | Autopilot manages the nodes; one cluster is covered by GKE's free tier. |
| Postgres | Cloud SQL, through the Cloud SQL Auth Proxy sidecar | Encrypted, IAM-checked connections; backups by Google. |
| Redis | Memorystore for Redis 7, with AUTH | Private IP inside the VPC. |
| ClickHouse | ClickHouse Cloud (on GCP) | Managed; reached over HTTPS from the cluster's fixed outbound address. |
| Secrets | Secret Manager -> External Secrets -> Secret `uptime-watch` | Nothing secret is committed. |
| Images | Artifact Registry | Pushed by CI through Workload Identity Federation (no keys). |

Both worker regions (`region1`, `region2`) run in the same cluster, so they
are labels rather than separate locations. To make a region real, run its
Deployment and ScaledObject (from `workers.yaml`) in a cluster elsewhere that
can reach Memorystore and ClickHouse; nothing else changes.

## One-time setup

You need `gcloud`, `kubectl` and `gh`, signed in. Run these in order; each
step is a few commands. Replace `MK-Sindhu` if the repos live elsewhere.

```sh
export PROJECT_ID=your-project-id
export REGION=asia-south1
gcloud config set project "$PROJECT_ID"
export PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')
```

### 1. APIs

```sh
gcloud services enable container.googleapis.com artifactregistry.googleapis.com \
  sqladmin.googleapis.com redis.googleapis.com secretmanager.googleapis.com \
  compute.googleapis.com iamcredentials.googleapis.com sts.googleapis.com
```

### 2. Cluster, with one fixed outbound address

Workers make requests to the websites they check. Private nodes send that
traffic through Cloud NAT on a reserved address, which ClickHouse Cloud can
then allow.

```sh
gcloud compute addresses create uptime-watch-egress --region "$REGION"
gcloud compute routers create uptime-watch-router --network default --region "$REGION"
gcloud compute routers nats create uptime-watch-nat --router uptime-watch-router \
  --region "$REGION" --nat-all-subnet-ip-ranges --nat-external-ip-pool uptime-watch-egress

gcloud container clusters create-auto uptime-watch --region "$REGION" --enable-private-nodes
gcloud container clusters get-credentials uptime-watch --region "$REGION"
```

### 3. Artifact Registry

```sh
gcloud artifacts repositories create uptime-watch --repository-format docker --location "$REGION"
# The nodes pull images as the default compute service account.
gcloud artifacts repositories add-iam-policy-binding uptime-watch --location "$REGION" \
  --member "serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role roles/artifactregistry.reader
```

### 4. Postgres (Cloud SQL)

```sh
gcloud sql instances create uptime-watch --database-version POSTGRES_16 \
  --edition ENTERPRISE --tier db-f1-micro --region "$REGION"
gcloud sql databases create uptime --instance uptime-watch
DB_PASSWORD=$(openssl rand -hex 24)
gcloud sql users create uptime --instance uptime-watch --password "$DB_PASSWORD"
```

The migration Job creates the tables on the first sync.

### 5. Redis (Memorystore)

```sh
gcloud redis instances create uptime-watch --region "$REGION" --size 1 --tier basic \
  --redis-version redis_7_2 --enable-auth --network default
REDIS_HOST=$(gcloud redis instances describe uptime-watch --region "$REGION" --format 'value(host)')
REDIS_AUTH=$(gcloud redis instances get-auth-string uptime-watch --region "$REGION" --format 'value(authString)')
```

### 6. ClickHouse (ClickHouse Cloud)

In the ClickHouse Cloud console, create a service on Google Cloud in or near
`asia-south1`. Under its network settings, allow only the NAT address:

```sh
gcloud compute addresses describe uptime-watch-egress --region "$REGION" --format 'value(address)'
```

Note the HTTPS endpoint (`https://<id>.<region>.gcp.clickhouse.cloud:8443`) and
the `default` user's password, then run `CREATE DATABASE uptime` in its SQL
console. The API and workers create the tables on start-up.

### 7. Secrets (Secret Manager)

```sh
put() { printf '%s' "$2" | gcloud secrets create "$1" --data-file=-; }
put uptime-watch-jwt-secret "$(openssl rand -hex 32)"
put uptime-watch-database-url "postgresql://uptime:$DB_PASSWORD@127.0.0.1:5432/uptime"
put uptime-watch-redis-host "$REDIS_HOST"
put uptime-watch-redis-auth "$REDIS_AUTH"
put uptime-watch-clickhouse-url "https://<id>.<region>.gcp.clickhouse.cloud:8443"
put uptime-watch-clickhouse-password "<the default user's password>"
```

To change one later: `printf '%s' new-value | gcloud secrets versions add NAME --data-file=-`.
External Secrets picks it up within an hour; restart the pods to use it.

### 8. Permissions for the pods

With Workload Identity Federation for GKE, IAM roles go straight to
Kubernetes service accounts; there are no keys.

```sh
WI="principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$PROJECT_ID.svc.id.goog/subject"
# api, pusher and the migration connect through the Cloud SQL Auth Proxy.
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "$WI/ns/uptime-watch/sa/uptime-watch" --role roles/cloudsql.client
# External Secrets reads the secrets.
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "$WI/ns/external-secrets/sa/external-secrets" --role roles/secretmanager.secretAccessor
```

### 9. Public address, TLS and Cloud Armor

```sh
gcloud compute addresses create uptime-watch-ip --global
gcloud compute addresses describe uptime-watch-ip --global --format 'value(address)'
```

Point your domain's A record at that address. Without a domain, use
`<address>.sslip.io` (for example `34.120.10.20.sslip.io`).

Cloud Armor blocks floods at Google's edge, before they reach the cluster:
more than 300 requests a minute from one address gets 429s for 5 minutes.

```sh
gcloud compute security-policies create uptime-watch-armor
gcloud compute security-policies rules create 1000 --security-policy uptime-watch-armor \
  --src-ip-ranges "*" --action rate-based-ban \
  --rate-limit-threshold-count 300 --rate-limit-threshold-interval-sec 60 \
  --ban-duration-sec 300 --conform-action allow --exceed-action deny-429 --enforce-on-key IP
```

### 10. Let uptime-watch's CI push images and update this repo

GitHub proves which repo a workflow belongs to with an OIDC token, and
Google trusts that instead of a stored key:

```sh
gcloud iam workload-identity-pools create github --location global
gcloud iam workload-identity-pools providers create-oidc uptime-watch \
  --location global --workload-identity-pool github \
  --issuer-uri https://token.actions.githubusercontent.com \
  --attribute-mapping "google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition "assertion.repository == 'MK-Sindhu/uptime-watch'"
gcloud iam service-accounts create github-ci
gcloud artifacts repositories add-iam-policy-binding uptime-watch --location "$REGION" \
  --member "serviceAccount:github-ci@$PROJECT_ID.iam.gserviceaccount.com" \
  --role roles/artifactregistry.writer
gcloud iam service-accounts add-iam-policy-binding "github-ci@$PROJECT_ID.iam.gserviceaccount.com" \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/MK-Sindhu/uptime-watch"
```

Then the settings CI reads, and a deploy key that can push to this repo only:

```sh
APP=MK-Sindhu/uptime-watch
gh variable set GCP_PROJECT_ID --body "$PROJECT_ID" -R $APP
gh variable set GCP_REGION --body "$REGION" -R $APP
gh variable set GAR_REPOSITORY --body uptime-watch -R $APP
gh variable set GCP_WORKLOAD_IDENTITY_PROVIDER -R $APP \
  --body "projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github/providers/uptime-watch"
gh variable set GCP_SERVICE_ACCOUNT --body "github-ci@$PROJECT_ID.iam.gserviceaccount.com" -R $APP
gh variable set DEPLOY_REPO --body MK-Sindhu/uptime-watch-deploy -R $APP

ssh-keygen -t ed25519 -N "" -C uptime-watch-ci -f deploy_key
gh repo deploy-key add deploy_key.pub --allow-write --title "uptime-watch CI" -R MK-Sindhu/uptime-watch-deploy
gh secret set DEPLOY_REPO_SSH_KEY -R $APP < deploy_key
rm deploy_key deploy_key.pub
```

### 11. Fill in the overlay and push this repo

In `apps/uptime-watch/overlays/production/kustomization.yaml`, replace every
`REPLACE_ME_PROJECT_ID`, `REPLACE_ME_STATIC_IP` and `replace-me.example.com`:

```sh
grep -rn -i "replace.me" apps/
```

If your repos aren't under `MK-Sindhu`, change `repoURL` in
`bootstrap/root-app.yaml` and `argocd/apps/uptime-watch.yaml`. Then commit
and push. Push uptime-watch's `main` too, so CI builds the first images and
writes their tags here.

### 12. Install Argo CD and hand the cluster over to it

```sh
kubectl apply --server-side --force-conflicts -k bootstrap/argocd
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -f bootstrap/root-app.yaml
```

This repo is public, so Argo CD needs no credentials. If you make it
private, add a read-only deploy key: `argocd repo add git@github.com:MK-Sindhu/uptime-watch-deploy.git --ssh-private-key-path <key>`.

The first sync takes a few minutes: External Secrets and KEDA install
first, and the app retries until their resources exist. The managed TLS
certificate can take up to an hour after DNS points at the address.

## Day to day

```sh
# Argo CD's UI on https://localhost:8080 (user: admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8080:443

kubectl -n uptime-watch get pods,scaledobjects,hpa
kubectl -n uptime-watch logs job/migrate -c migrate     # the last migration
kubectl -n uptime-watch logs deploy/worker-region1
```

- **Roll back:** `git revert` the deploy commit and push. Argo CD rolls the
  pods back. The database isn't rolled back: migrations only add.
- **Change a setting:** edit the overlay (or the base), open a pull request;
  the Validate workflow renders and checks the manifests.
- **Change a secret:** add a new version in Secret Manager (step 7).
- **Add a region:** copy a Deployment + ScaledObject pair in `workers.yaml`
  and add the name to `REGIONS` in `base/kustomization.yaml`.
- **Don't** `kubectl edit` resources Argo CD manages: it puts them back
  (self-heal). Change git instead.
