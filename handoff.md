# Handoff: Deployment Stabilization Context

This repository is an Azure SRE Agent demo lab that deploys an AKS-hosted Pet Store app, Azure Chaos Studio experiments, observability resources, and an optional Chaos Engineering Portal. The current work focused on making `scripts\deploy.ps1` match the documented clean deployment experience.

## Current assessment

The upstream repo appears to have been tested incrementally rather than as a clean end-to-end deployment. The architecture is coherent, but several integration points were not fully wired or were too optimistic about Azure control-plane readiness.

In short:

- Chaos Studio Azure resources were provisioned by Bicep, but the required in-cluster Chaos Mesh installation was missing.
- Alert rules could fail because scheduled query rules were created before the Log Analytics workspace was reliably visible to the `Microsoft.Insights` provider.
- The optional portal deployment path had a deterministic PowerShell bug due to an undefined subscription variable.

## Important project structure

- `scripts\deploy.ps1` - main deployment orchestrator.
- `scripts\build-portal.ps1` - builds and deploys the optional Chaos Engineering Portal.
- `infra\bicep\main.bicep` - subscription-scoped Bicep orchestration for the 3-resource-group model.
- `infra\bicep\modules\chaos-studio.bicep` - provisions Azure Chaos Studio targets, capabilities, identities, and experiments.
- `infra\bicep\modules\alerts.bicep` - provisions Azure Monitor scheduled query alert rules.
- `k8s\base\application.yaml` - healthy Pet Store application manifests.
- `k8s\scenarios\` - kubectl-applied breakable scenarios.
- `docs\BREAKABLE-SCENARIOS.md` and `docs\CHAOS-AND-SRE-GUIDE.md` - scenario documentation.

## Failure conditions investigated

### 1. Chaos Mesh not installed in `chaos-testing`

Symptom:

```text
deploy.ps1 completes Azure deployment, but Chaos Mesh is not present in namespace chaos-testing.
```

Root cause:

- Bicep can enable AKS as a Chaos Studio target and create Chaos Studio capabilities/experiments.
- Bicep does not install the in-cluster Chaos Mesh controller, CRDs, daemon, or RBAC.
- The docs implied Bicep handled the complete Chaos Studio setup, but the Kubernetes-side prerequisite was absent.

Conclusion:

- This was not platform drift.
- The Azure-side Chaos Studio resources may have deployed, but the automated experiments requiring Chaos Mesh could not work from a clean deployment unless Chaos Mesh had been manually installed beforehand.

Fix applied:

- Added idempotent Chaos Mesh installation to `scripts\deploy.ps1`.
- Uses Helm:
  - `helm repo add chaos-mesh https://charts.chaos-mesh.org`
  - `helm repo update`
  - `kubectl create namespace chaos-testing --dry-run=client -o yaml | kubectl apply -f -`
  - `helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh --namespace chaos-testing --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock`
- Validates Chaos Mesh pods after install.
- Installs only when Chaos Studio experiments are deployed.
- AKS credentials are now fetched with `az aks get-credentials --admin` because Helm installation creates CRDs/RBAC and requires cluster-admin-level access.

Docs updated:

- `README.md` now lists Helm as a prerequisite and clarifies `deploy.ps1` installs Chaos Mesh.
- `docs\BREAKABLE-SCENARIOS.md` now clarifies that Bicep provisions Azure Chaos Studio resources, while `deploy.ps1` installs Chaos Mesh into AKS.

### 2. `deploy-alerts` failed with "The workspace could not be found"

Observed deployment error:

```json
{
  "code": "DeploymentFailed",
  "target": "/subscriptions/aa7137b7-e6b4-4f40-94ea-89c0a7bc5eea/resourceGroups/rg-srelabmon-sec-01/providers/Microsoft.Resources/deployments/deploy-alerts",
  "message": "At least one resource deployment operation failed."
}
```

Nested operation details showed seven `Microsoft.Insights/scheduledQueryRules` failures:

```text
[BadRequest] The workspace could not be found Activity ID: ...
```

Investigation:

- Azure CLI was used against subscription `aa7137b7-e6b4-4f40-94ea-89c0a7bc5eea`.
- The target Log Analytics workspace existed:
  - Resource group: `rg-srelabmon-sec-01`
  - Workspace name: `log-srelab`
  - Provisioning state: `Succeeded`
- The same alert module later validated successfully against the same workspace ID.

Root cause:

- Most likely Azure control-plane/provider propagation delay.
- `Microsoft.OperationalInsights/workspaces` had completed, but `Microsoft.Insights/scheduledQueryRules` could not yet resolve the workspace scope.
- This was a timing/race condition, not an incorrect workspace ID.

Conclusion:

- This may have worked intermittently depending on Azure provider timing.
- It is a platform-readiness issue exposed by the original orchestration, which deployed scheduled query rules in the same Bicep deployment as the workspace.

Fix applied:

- `scripts\deploy.ps1` now forces the main subscription Bicep deployment to pass `deployAlerts=false`.
- It reads the user's desired `deployAlerts` value from `infra\bicep\main.bicepparam`.
- If desired alert deployment is enabled, `deploy.ps1` deploys `infra\bicep\modules\alerts.bicep` separately after main infrastructure deployment.
- Added `Wait-LogAnalyticsWorkspace` helper to poll for workspace visibility.
- Added `Deploy-Alerts` helper with retry logic for the specific `"workspace could not be found"` failure.
- Alert outputs are merged back into the deployment output object and written to `deployment-outputs.json`.
- `README.md` documents that alert rules are deployed post-infrastructure to avoid Log Analytics propagation failures.

Important implementation note:

- `infra\bicep\main.bicep` still has the alerts module gated by `deployAlerts`.
- The script override is intentional: `deploy.ps1` disables in-main alert deployment and handles alert creation post-deployment.
- If someone deploys `main.bicep` directly with `deployAlerts=true`, the original race can still occur.

### 3. Chaos Engineering Portal failed with missing `SubscriptionId`

Observed output:

```text
🌐 Deploying Chaos Engineering Portal...
build-portal.ps1: Missing an argument for parameter 'SubscriptionId'. Specify a parameter of type 'System.String' and try again.
```

Root cause:

- `scripts\deploy.ps1` called:

```powershell
& pwsh -NoLogo -NoProfile -File $portalScript -ResourceGroupName $infraResourceGroupName -WorkloadName $WorkloadName -SubscriptionId $SubscriptionId
```

- `deploy.ps1` has no `$SubscriptionId` variable.
- PowerShell passed the named parameter `-SubscriptionId` without a value.
- `build-portal.ps1` actually has an optional `[string]$SubscriptionId = ''` and can resolve the current subscription itself, but the caller prevented that by supplying an empty named argument.

Conclusion:

- This was a deterministic scripting bug.
- `deploy.ps1 -DeployPortal` likely never worked in upstream at commit `78f6f2f` unless the caller had an unrelated session variable named `$SubscriptionId`.

Fix applied:

- Changed the portal invocation to pass the current Azure account ID:

```powershell
& pwsh -NoLogo -NoProfile -File $portalScript -ResourceGroupName $infraResourceGroupName -WorkloadName $WorkloadName -SubscriptionId $account.id
```

## Files intentionally changed by this stabilization work

- `scripts\deploy.ps1`
  - Added `Test-ChaosStudioDeployment`.
  - Added `Install-ChaosMesh`.
  - Added `Get-BicepBoolParameter`.
  - Added `Wait-LogAnalyticsWorkspace`.
  - Added `Deploy-Alerts`.
  - Main deployment now overrides `deployAlerts=false`.
  - Post-deploy alert deployment is run separately if the parameter file requested alerts.
  - AKS credentials call now uses `--admin`.
  - Chaos Mesh installation is run after AKS credentials are configured.
  - Portal build now passes `-SubscriptionId $account.id`.
- `README.md`
  - Added Helm prerequisite.
  - Clarified Chaos Mesh and post-deploy alert behavior.
  - Updated script description for `deploy.ps1`.
- `docs\BREAKABLE-SCENARIOS.md`
  - Clarified Bicep vs. `deploy.ps1` responsibilities for Chaos Studio and Chaos Mesh.

## Other current worktree changes to be careful with

At the time this handoff was written, the working tree also had changes that were not part of the main stabilization work or were pre-existing/user-created:

- `infra\bicep\main.bicep`
  - Shows a local change in git status.
  - Previously observed as changing the AKS version default from `1.32` to `1.34`.
- `infra\bicep\main.bicepparam`
  - Shows local changes.
  - Previously observed as setting Kubernetes version `1.34` and changing SMS action receivers to an empty array.
- `scripts\sec-deploy.ps1`
  - Untracked.
- `scripts\sec-destroy.ps1`
  - Untracked.

Do not revert these unless the user explicitly requests it.

## Validation already performed

- Parsed `scripts\deploy.ps1` successfully using the PowerShell parser.
- Built `infra\bicep\modules\alerts.bicep` successfully.
- Built `infra\bicep\main.bicep`; command exited successfully but emitted existing warnings.
- Confirmed the portal `SubscriptionId` fix leaves `deploy.ps1` syntactically valid.

## What to check if failures continue

### If Chaos Mesh still does not install

Check:

```powershell
helm version
kubectl get ns chaos-testing
kubectl get pods -n chaos-testing
kubectl get crd | Select-String chaos-mesh
```

Likely causes:

- Helm not installed or not on PATH.
- Current identity lacks AKS admin access.
- `az aks get-credentials --admin` failed or context points to the wrong cluster.
- Chaos Mesh chart install failed due to network or container image pull issues.

### If Chaos Studio experiments fail after Chaos Mesh installs

Check:

```powershell
az chaos experiment list --resource-group <infra-rg> --output table
kubectl get pods -n chaos-testing
kubectl get pods -n pets
kubectl get events -n pets --sort-by=.lastTimestamp
```

Likely causes:

- Chaos Studio identity lacks the necessary AKS or resource group permissions.
- The experiment target/capability exists but the in-cluster Chaos Mesh CRDs are not ready.
- App labels/selectors changed and no longer match experiment selectors.

### If alerts fail again

Check:

```powershell
az monitor log-analytics workspace show --ids <workspace-id>
az deployment group show --resource-group <monitor-rg> --name deploy-alerts
az deployment group operation list --resource-group <monitor-rg> --name deploy-alerts --output table
```

Likely causes:

- Workspace propagation still delayed longer than current retry budget.
- Direct Bicep deployment was used instead of `deploy.ps1`, causing alerts to be created inside the main deployment again.
- Workspace ID or monitor resource group differs from expected values.

### If portal deployment fails

Check:

```powershell
az account show --query id -o tsv
.\scripts\build-portal.ps1 -ResourceGroupName <infra-rg> -WorkloadName <workload-name>
```

Likely causes:

- Container build dependencies missing.
- ACR login or push failed.
- AKS context points to the wrong cluster.
- Portal Kubernetes manifests or image substitutions failed.

## Overall judgment

This does not look like one cleanly working deployment that later broke solely because Azure drifted. It looks like a demo repo that had useful pieces, but the full `deploy.ps1` path with Chaos Studio, alert rules, Chaos Mesh, and optional portal was not fully clean-room validated.

The fixes made so far aim to turn the repo from "works if preconditions happened to exist" into an actual reproducible deployment:

- Install the Kubernetes-side Chaos Mesh dependency explicitly.
- Avoid Azure provider readiness races for Log Analytics alert rules.
- Fix optional portal parameter passing.
- Update docs so they describe the actual ownership split between Bicep and scripts.
