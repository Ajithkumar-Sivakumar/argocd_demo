# Lab 5: Argo CD and Kustomize

Deliver one application to dev and prod from Git. Kustomize configures each
environment; Argo CD is the only deployer of application resources.

## What is included

```text
base/
  kustomization.yaml
  configmap.yaml
  migration-job.yaml
  deployment.yaml
  service.yaml
overlays/
  dev/
    kustomization.yaml
    configmap-patch.yaml
  prod/
    kustomization.yaml
    configmap-patch.yaml
argocd/
  kustomization.yaml
  project.yaml
  application-dev.yaml
  application-prod.yaml
```

| Setting | Dev | Prod |
|---|---|---|
| Namespace | `lab5-dev` | `lab5-prod` |
| Deployment replicas | 1 | 2 |
| Web image | `nginx:1.28.1` | `nginx:1.28.0` |
| ConfigMap `APP_MESSAGE` | `Hello from DEV` | `Hello from PROD` |
| Sync | Automated, prune, self-heal | Manual |

The web page displays the ConfigMap value and its namespace. Tags intentionally
differ to demonstrate an environment promotion boundary; these are teaching
pins, not a recommendation to run old images in production.

The migration Job performs a real SQLite schema migration and verifies version
1. Its database lives in an `emptyDir` volume: it is **temporary demo data**, not
a persistent database used by NGINX. For a real application, replace the Job's
image/script with the application's migration tool against a persistent database,
with credentials supplied separately, never committed to Git.

## Prerequisites and conventions

- A disposable Kubernetes cluster and cluster-admin access for installing Argo CD.
  Use a cluster version supported by your chosen Argo CD release.
- `kubectl` with built-in Kustomize, Git, and the Argo CD CLI on `PATH`.
- Cluster egress to GitHub and Docker Hub, and capacity for three web Pods plus
  short-lived migration Jobs.
- PowerShell commands below run from the repository root. Keep port-forward and
  watch terminals open where instructed.
- Use fresh `lab5-dev` and `lab5-prod` namespaces for the **first-sync ordering**
  evidence. Existing Deployments retain their original creation timestamps.
- The default repository is
  `https://github.com/Ajithkumar-Sivakumar/argocd_demo.git`, branch `main`.
  If using a fork, change `spec.source.repoURL` in both Application manifests and
  `spec.sourceRepos` in the AppProject before committing. Change
  `targetRevision` too if your branch is not `main`.

Only Argo CD installation and its AppProject/Application **bootstrap resources**
are applied with `kubectl`. Never run `kubectl apply -k .\overlays\dev`,
`kubectl apply -k .\overlays\prod`, or apply rendered application YAML.
The later `kubectl scale` is a deliberate, one-time drift experiment, not a
deployment method.

## 1. Commit and publish the desired state

This directory is already a Git repository. If starting elsewhere, first create
an empty GitHub repository, run `git init -b main`, and add its URL as `origin`.
For this checkout:

```powershell
git remote -v
git branch --show-current
git add .gitignore README.md base overlays argocd
git commit -m "Add Lab 5 Argo CD and Kustomize manifests"
git push -u origin main
git rev-parse HEAD
```

Do not continue until the files exist on the remote branch: Argo CD reads the
remote repository, not your working directory.

## 2. Render both overlays and capture the diff (no cluster changes)

```powershell
New-Item -ItemType Directory -Force .\evidence | Out-Null

$DevYaml = kubectl kustomize .\overlays\dev
if ($LASTEXITCODE -ne 0) { throw "Dev render failed" }
$DevYaml | Set-Content -Encoding utf8 .\evidence\rendered-dev.yaml
$DevYaml

$ProdYaml = kubectl kustomize .\overlays\prod
if ($LASTEXITCODE -ne 0) { throw "Prod render failed" }
$ProdYaml | Set-Content -Encoding utf8 .\evidence\rendered-prod.yaml
$ProdYaml

git --no-pager diff --no-index --no-color -- .\evidence\rendered-dev.yaml .\evidence\rendered-prod.yaml |
    Tee-Object -FilePath .\evidence\overlays.diff
$DiffExit = $LASTEXITCODE
if ($DiffExit -gt 1) { throw "Diff command failed" }
```

Git diff exit code **1 is expected**: the files differ. Look for namespace
changes on all four resources, replicas `1` versus `2`, image tags, and the
ConfigMap message. The migration image stays the same.

Save a screenshot of the diff and retain the two rendered files and
`evidence\overlays.diff`. Evidence is ignored by Git.

## 3. Install Argo CD and log in

Check the target context carefully. These commands install a non-HA Argo CD
control plane into that cluster; they do not deploy the example application.

```powershell
kubectl config current-context
kubectl cluster-info
kubectl get namespace argocd
# If argocd does not already exist:
kubectl create namespace argocd

# Pinned release. Check its Kubernetes compatibility before installation.
$ArgoVersion = "v3.6.3"
$InstallUrl = "https://raw.githubusercontent.com/argoproj/argo-cd/$ArgoVersion/manifests/install.yaml"
kubectl apply --server-side -n argocd -f $InstallUrl
if ($LASTEXITCODE -ne 0) { throw "Argo CD installation failed" }

kubectl wait --for=condition=Established --timeout=120s crd/applications.argoproj.io crd/appprojects.argoproj.io
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-repo-server -n argocd --timeout=300s
kubectl rollout status statefulset/argocd-application-controller -n argocd --timeout=300s
kubectl get pods -n argocd
```

If Argo CD is already installed, reuse it only with the cluster owner's
permission; do not overwrite a shared control plane. Skip the installation
commands in that case.

Install the matching CLI separately if needed. Official CLI installation:
<https://argo-cd.readthedocs.io/en/stable/cli_installation/>.
For example, with GitHub CLI installed on Windows:

```powershell
New-Item -ItemType Directory -Force .\evidence\tools | Out-Null
gh release download v3.6.3 --repo argoproj/argo-cd --pattern argocd-windows-amd64.exe --dir .\evidence\tools
if ($LASTEXITCODE -ne 0) { throw "CLI download failed" }
Copy-Item .\evidence\tools\argocd-windows-amd64.exe .\evidence\tools\argocd.exe
$env:PATH = "$(Join-Path $PWD 'evidence\tools');$env:PATH"
argocd version --client
```

Use the binary for your machine's architecture. Prefer release checksum/signature
verification according to your organization's software-installation policy.
The PATH adjustment is per terminal; repeat it in other CLI terminals if needed.

In a **separate terminal**, start and leave running:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Back in the main terminal:

```powershell
$EncodedPassword = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}'
if ($LASTEXITCODE -ne 0) { throw "Initial admin secret unavailable; use existing admin credentials" }
$InitialPassword = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($EncodedPassword))
argocd login localhost:8080 --username admin --password $InitialPassword --insecure
if ($LASTEXITCODE -ne 0) { throw "Argo CD login failed" }
Remove-Variable InitialPassword, EncodedPassword

$RepoUrl = "https://github.com/Ajithkumar-Sivakumar/argocd_demo.git"
argocd repo add $RepoUrl --type git
if ($LASTEXITCODE -ne 0) { throw "Repository registration failed" }
argocd repo list | Tee-Object -FilePath .\evidence\repositories.txt
```

`--insecure` accepts the lab server's self-signed certificate through the local
port-forward; do not use that as a production TLS configuration.
Public repositories need no credentials. For a private repository, register
the same URL with a read-only token or SSH credential using `argocd repo add`;
keep credentials out of manifests, screenshots, transcripts, and Git.
Rotate the initial admin password using `argocd account update-password`.
Do not capture login output or the initial admin Secret as evidence.

## 4. Bootstrap the two Applications, then manually approve prod

These are Argo CD control resources, not the application workloads:

```powershell
kubectl apply -k .\argocd
if ($LASTEXITCODE -ne 0) { throw "Argo CD Application bootstrap failed" }

# Dev starts automatically. Wait for the complete operation and health.
argocd app wait lab5-dev --operation --sync --health --timeout 300
if ($LASTEXITCODE -ne 0) { throw "Dev did not become Synced and Healthy" }

# Prod initially remains OutOfSync/Missing until explicitly approved.
argocd app get lab5-prod --refresh
argocd app diff lab5-prod
# An exit code of 1 from app diff means differences exist, not a failed sync.
argocd app sync lab5-prod
if ($LASTEXITCODE -ne 0) { throw "Prod sync failed" }
argocd app wait lab5-prod --operation --sync --health --timeout 300
if ($LASTEXITCODE -ne 0) { throw "Prod did not become Synced and Healthy" }

argocd app list | Tee-Object -FilePath .\evidence\applications.txt
argocd app get lab5-dev | Tee-Object -FilePath .\evidence\dev-application.txt
argocd app get lab5-prod | Tee-Object -FilePath .\evidence\prod-application.txt
kubectl get deployments,pods,services -n lab5-dev
kubectl get deployments,pods,services -n lab5-prod
```

Expected Application list: **both `lab5-dev` and `lab5-prod` show `Synced` and
`Healthy`**. Dev's policy is automated with prune and self-heal; prod's policy
is manual. Exact CLI column formatting varies by version.

### Why is prod manual?

An unreviewed Git change must not immediately affect production. Manual sync
provides an approval/change-window gate after reviewing `argocd app diff`.
Prod still uses Git as its source of truth; approval invokes Argo CD, never
`kubectl apply` on workloads. Prod intentionally does not auto-correct drift.
Restrict production sync permissions with Argo CD RBAC in a real environment.
Manual sync alone is not an authorization control.

Dev automatically deletes tracked resources removed from Git (`prune`) and
reverts live edits (`selfHeal`). Prod does not automatically prune; review
deletions and use `argocd app sync lab5-prod --prune` only when approved.

## 5. Capture and verify resource creation order

Both overlays inherit the following sequence:

| Wave | Resource | Behavior |
|---|---|---|
| -2 | `lab5-config` ConfigMap | Normal Sync resource, created first |
| -1 | `lab5-db-migration` Job | **Sync** hook; Argo CD waits for success |
| 0 | `lab5-web` Deployment and Service | Created only after the Job succeeds |

The Job must be a **Sync**, not PreSync, hook: a PreSync hook would execute
before the ordinary ConfigMap, regardless of its sync wave.
The Job consumes `APP_MESSAGE` through `configMapKeyRef`, so its successful log
also proves that the ConfigMap was available.

`BeforeHookCreation` retains the finished Job for inspection and replaces it on
the next full sync. Do not add a TTL or `HookSucceeded` deletion policy before
capturing evidence. Do not use selective/resource-only sync for this exercise:
hooks do not execute during selective syncs.

Immediately after the **first** successful dev sync:

```powershell
argocd app get lab5-dev --show-operation |
    Tee-Object -FilePath .\evidence\dev-sync-operation.txt
kubectl get application lab5-dev -n argocd -o yaml |
    Set-Content -Encoding utf8 .\evidence\dev-application-state.yaml
kubectl logs job/lab5-db-migration -n lab5-dev --timestamps=true |
    Tee-Object -FilePath .\evidence\dev-migration.log

$Config = kubectl get configmap lab5-config -n lab5-dev -o json | ConvertFrom-Json
$Job = kubectl get job lab5-db-migration -n lab5-dev -o json | ConvertFrom-Json
$Deployment = kubectl get deployment lab5-web -n lab5-dev -o json | ConvertFrom-Json

@(
    "ConfigMap created : $($Config.metadata.creationTimestamp)"
    "Job created       : $($Job.metadata.creationTimestamp)"
    "Job completed     : $($Job.status.completionTime)"
    "Deployment created: $($Deployment.metadata.creationTimestamp)"
) | Tee-Object -FilePath .\evidence\creation-order.txt

if ($Job.status.succeeded -ne 1 -or -not $Job.status.completionTime) {
    throw "Migration Job has not succeeded"
}
if ([DateTimeOffset]$Config.metadata.creationTimestamp -gt [DateTimeOffset]$Job.metadata.creationTimestamp) {
    throw "ConfigMap was created after the Job"
}
if ([DateTimeOffset]$Job.status.completionTime -gt [DateTimeOffset]$Deployment.metadata.creationTimestamp) {
    throw "Deployment was created before migration completion"
}
"PASS: ConfigMap -> successful migration Job -> Deployment" |
    Tee-Object -FilePath .\evidence\creation-order.txt -Append
```

Kubernetes timestamps have finite precision, so ties are allowed in the checks.
The sync-wave annotations, operation details, migration log, and timestamps
together demonstrate ordering. Capture this **before** the drift experiment:
later syncs may replace the hook Job but do not recreate the Deployment.
A failed migration intentionally prevents wave 0 from deploying.

## 6. Optional: verify each web page

In separate terminals, leave these running:

```powershell
kubectl port-forward svc/lab5-web -n lab5-dev 8081:80
```

```powershell
kubectl port-forward svc/lab5-web -n lab5-prod 8082:80
```

In the main terminal:

```powershell
(Invoke-WebRequest -UseBasicParsing http://localhost:8081).Content |
    Tee-Object -FilePath .\evidence\dev-page.html
(Invoke-WebRequest -UseBasicParsing http://localhost:8082).Content |
    Tee-Object -FilePath .\evidence\prod-page.html
```

Expect the dev/prod messages and their corresponding namespaces.

## 7. Capture drift detection and automatic self-healing

Use **three terminals** at the repository root. Start both watchers and wait for
their initial rows **before scaling**. Self-healing is fast (often seconds);
running `argocd app get` only after scaling can miss the transient OutOfSync state.

**Terminal A: watch Application sync/health transitions**

```powershell
kubectl get applications.argoproj.io lab5-dev -n argocd --watch `
    -o 'custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status' |
    Tee-Object -FilePath .\evidence\drift-application-watch.txt
```

**Terminal B: watch desired and actual Deployment replicas**

```powershell
kubectl get deployment lab5-web -n lab5-dev --watch `
    -o 'custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,READY:.status.readyReplicas,AVAILABLE:.status.availableReplicas' |
    Tee-Object -FilePath .\evidence\drift-deployment-watch.txt
```

**Terminal C: introduce drift once, then let Argo CD fix it**

```powershell
argocd app wait lab5-dev --operation --sync --health --timeout 300
if ($LASTEXITCODE -ne 0) { throw "Dev must be healthy before the drift test" }

kubectl scale deployment/lab5-web -n lab5-dev --replicas=4 |
    Tee-Object -FilePath .\evidence\drift-scale.txt
if ($LASTEXITCODE -ne 0) { throw "Drift injection failed" }

# Record the immediate state, and request a comparison, NOT a sync.
kubectl get deployment lab5-web -n lab5-dev -o yaml |
    Set-Content -Encoding utf8 .\evidence\drift-immediate-deployment.yaml
argocd app get lab5-dev --refresh |
    Tee-Object -FilePath .\evidence\drift-immediate-application.txt

# A bounded polling loop checks the actual replica count as well as Argo status.
# app wait alone could return before the controller has detected the change.
$Deadline = (Get-Date).AddSeconds(180)
$Healed = $false
do {
    $LiveDeployment = kubectl get deployment lab5-web -n lab5-dev -o json | ConvertFrom-Json
    if ($LASTEXITCODE -ne 0) { throw "Cannot read dev Deployment" }
    $Application = kubectl get application lab5-dev -n argocd -o json | ConvertFrom-Json
    if ($LASTEXITCODE -ne 0) { throw "Cannot read dev Application" }
    $Healed = $LiveDeployment.spec.replicas -eq 1 -and `
        $LiveDeployment.status.readyReplicas -eq 1 -and `
        $LiveDeployment.status.replicas -eq 1 -and `
        $Application.status.sync.status -eq "Synced" -and `
        $Application.status.health.status -eq "Healthy"
    if (-not $Healed) { Start-Sleep -Seconds 1 }
} until ($Healed -or (Get-Date) -ge $Deadline)
if (-not $Healed) { throw "Dev did not self-heal within 180 seconds" }

"PASS: live dev replicas reverted from 4 to Git's desired 1 automatically" |
    Tee-Object -FilePath .\evidence\self-heal-result.txt
argocd app list | Tee-Object -FilePath .\evidence\applications-after-heal.txt
kubectl get deployment lab5-web -n lab5-dev -o yaml |
    Set-Content -Encoding utf8 .\evidence\deployment-after-heal.yaml
```

Do **not** run `argocd app sync`, reapply an overlay, or scale back to 1 yourself:
that would invalidate the automatic recovery evidence.

After recovery, stop both watchers with **Ctrl+C**. Retain their files and take
screenshots showing:

- Terminal A: `Synced -> OutOfSync -> Synced`.
- Terminal B: desired replicas `1 -> 4 -> 1`. Actual Pods may not reach 4 before
  the controller restores the desired count; this is normal.
- Terminal C: the PASS result and both Applications `Synced` / `Healthy`.

If the Application watcher did not capture `OutOfSync`, do not claim it did.
Restart the watchers, wait for their initial rows, and repeat the one-time scale
experiment. The immediate snapshot can already show a healed value; use the
watch streams as the evidence of the transient state.

## 8. Normal GitOps changes after the lab

Edit the relevant overlay, render/review it, commit, and push. Dev reconciles
automatically. For prod, inspect the diff and explicitly approve a full sync.
The named ConfigMap is read into environment variables at Pod startup; changing
only `APP_MESSAGE` does **not** restart existing web Pods. Include a Pod-template
change (for example a new web image tag or a version annotation) in the same Git
commit when promoting a new message. Do not use a manual rollout to bypass Git.

## Evidence checklist / pass criteria

| Requirement | Evidence to retain |
|---|---|
| Rendered dev and prod + differences | `rendered-dev.yaml`, `rendered-prod.yaml`, `overlays.diff`, diff screenshot |
| Repository connected | `repositories.txt` (successful connection, no credentials) |
| Both Applications Synced and Healthy | `applications.txt`, `applications-after-heal.txt`, Application list screenshot |
| ConfigMap before migration before Deployment | `creation-order.txt`, `dev-migration.log`, `dev-sync-operation.txt`, `dev-application-state.yaml` |
| Dev drift detected | `drift-application-watch.txt` showing OutOfSync; `drift-deployment-watch.txt` showing 4 |
| Dev automatically self-healed | The same watchers returning to Synced / 1; `self-heal-result.txt`, `deployment-after-heal.yaml` |

All paths in the table are under the local `evidence` directory. These files are
created by **running** the lab; no cluster results are fabricated or pre-filled.

## Troubleshooting

- **Repository error / manifest generation error:** confirm the push completed,
  the branch and repository URL match all three bootstrap manifests, and private
  repository credentials are registered. Use `argocd repo list` and
  `argocd app get lab5-dev --hard-refresh`.
- **Migration stalls / wave 0 missing:** inspect
  `kubectl describe job lab5-db-migration -n lab5-dev`,
  `kubectl get pods -n lab5-dev`, and
  `kubectl logs job/lab5-db-migration -n lab5-dev`. Fix the cause in Git.
  After fixing a failure at the same revision, a full
  `argocd app sync lab5-dev` retries the hook. Do this outside the drift test.
- **ImagePullBackOff / Pending:** inspect Pod events, registry connectivity,
  image availability, cluster capacity, and namespace admission policies.
  The root-based NGINX example assumes a normal disposable lab namespace, not
  a namespace enforcing the Kubernetes restricted Pod Security Standard.
- **Dev stays at 4:** inspect Application conditions, controller logs, repo
  connectivity, and `spec.syncPolicy.automated.selfHeal`. Ensure no HPA or
  second controller is managing this Deployment.
- **Port-forward exits:** restart it and log in again if required.
- **Order check fails after a later sync:** original Deployment timestamps cannot
  prove a new hook ran before its initial creation. Use the first-sync evidence.
  Do not delete shared namespaces to repeat the lab.

## Optional cleanup (destructive; disposable lab only)

Stop watchers and port-forwards with Ctrl+C. To remove **only these lab
Applications and their workloads**, explicitly request cascading deletion:

```powershell
argocd app delete lab5-dev --cascade --yes
argocd app delete lab5-prod --cascade --yes
```

After confirming deletion completed, the now-empty lab namespaces may be removed
with `kubectl delete namespace lab5-dev lab5-prod`, and the project with
`kubectl delete appproject lab5 -n argocd`. Do not delete Argo CD itself on a
shared cluster.
