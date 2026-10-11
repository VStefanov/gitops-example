# Local Argo CD and .NET GitOps example

Three standalone Helm application charts deploy ASP.NET Core workloads: `service`, `common-concerns`, and `storybook`. Each creates a Deployment and a ClusterIP Service.

All three initially run Microsoft's [.NET 10 ASP.NET Core sample](https://github.com/dotnet/dotnet-docker/blob/main/README.samples.md). Replace the images with your own published .NET applications when ready. The storybook chart is a .NET web application placeholder for your component showcase; it does not install the JavaScript Storybook tool. No .NET source code is included.

## Layout

```text
charts/
  service/
  common-concerns/
  storybook/
argocd/
  base/
    kustomization.yaml
    applications.yaml
    argo-common-concerns.yaml
    argo-storybook.yaml
env1-non-prod/
  qa/
    kustomization.yaml
    values/
  preprod/
env1-prod/
  prod/
env2-non-prod/
  qa/
  preprod/
env2-prod/
  prod/
```

Each chart includes `Chart.yaml`, `values.yaml`, and templates. The shared `argocd/base` contains:

- `applications.yaml`: an ApplicationSet with a list generator that deploys the service chart.
- `argo-common-concerns.yaml`: an Application for the common-concerns chart.
- `argo-storybook.yaml`: an Application for the storybook chart.

Every environment/stage folder contains a `kustomization.yaml` overlay and `values/service.yaml`, `values/common-concerns.yaml`, and `values/storybook.yaml`. Overlays reuse the base and set only the resource name prefix, destination namespace, generated service Application name, and Helm values paths. Repository URL, revision, chart paths, generators, and sync policy are defined once in the base.

[Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) assembles the Argo CD resources; Argo CD still uses Helm to render each .NET workload. Kustomize is bundled with kubectl; these overlays were verified with Kustomize v5.7.1 in kubectl v1.34.1.

Non-prod environments each contain `qa` and `preprod`; production environments each contain only `prod`. All six combinations target the same local cluster, using separate namespaces such as `env1-non-prod-qa` and `env1-prod-prod`. Non-prod uses one replica; prod uses two. These folders are examples, not a complete production cluster configuration.

## Prerequisites

Install Git, Docker Desktop (running Linux containers), kubectl, [kind](https://kind.sigs.k8s.io/docs/user/quick-start/), and [Helm](https://helm.sh/docs/intro/install/). The Argo CD CLI is optional; install it from the [official instructions](https://argo-cd.readthedocs.io/en/stable/cli_installation/). Ensure each command is on PATH.

Commands below use PowerShell and run from this repository's root. Argo CD and the cluster need network access to GitHub and container registries.

## 1. Create a local Kubernetes cluster

```powershell
kind create cluster --name gitops-example --wait 120s
kubectl config use-context kind-gitops-example
kubectl get nodes
```

Alternatively, enable Kubernetes in Docker Desktop and select `docker-desktop` as your context instead of creating a kind cluster. Keep the chosen context for all following commands.

## 2. Install Argo CD

For a fresh local cluster, use the [official Argo CD installation](https://argo-cd.readthedocs.io/en/stable/getting_started/):

```powershell
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Established crd/applications.argoproj.io crd/applicationsets.argoproj.io --timeout=120s
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-repo-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-applicationset-controller -n argocd --timeout=300s
kubectl get pods -n argocd
```

The full installation includes the ApplicationSet controller. The `stable` branch is convenient for this local example; use a tested release tag for a reproducible installation.

In a separate terminal, keep this command running:

```powershell
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Open https://localhost:8080 and accept the local self-signed certificate. Log in as `admin`. Obtain the initial password in your main terminal:

```powershell
$encodedPassword = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encodedPassword))
```

With the optional CLI, you can log in and change the password:

```powershell
argocd login localhost:8080 --username admin --insecure
argocd account update-password
```

## 2a. Configure a trusted local TLS certificate

Argo CD already enables HTTPS using a generated self-signed certificate. The certificate below secures browser/CLI access to `https://localhost:8080`. Git SSH authentication uses separate keys and host verification; replacing this TLS certificate will not fix an SSH handshake or authentication failure. See [Argo CD TLS configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/tls/).

### Create a certificate trusted by Windows

Install [mkcert](https://github.com/FiloSottile/mkcert#installation) using one of its documented Windows installation methods. For example, if Chocolatey is already installed, run this in an administrator PowerShell:

```powershell
choco install mkcert
```

Reopen PowerShell so `mkcert` is on PATH. Install its local development certificate authority, then generate a certificate covering localhost and the loopback addresses. Keep the files outside this repository:

```powershell
mkcert -install
$tlsDirectory = Join-Path $env:USERPROFILE '.argocd-local/tls'
New-Item -ItemType Directory -Path $tlsDirectory -Force | Out-Null
$certificatePath = Join-Path $tlsDirectory 'localhost.pem'
$privateKeyPath = Join-Path $tlsDirectory 'localhost-key.pem'
mkcert -cert-file $certificatePath -key-file $privateKeyPath localhost 127.0.0.1 ::1
```

Approve the Windows trust-store prompt if shown. This certificate is for local development. Keep its private key and mkcert's CA private key out of Git.

### Install the certificate in Argo CD

Confirm you are targeting your local cluster, then create or update the TLS Secret:

```powershell
kubectl config current-context
kubectl create secret tls argocd-server-tls -n argocd --cert="$certificatePath" --key="$privateKeyPath" --dry-run=client -o yaml | kubectl apply -f -
```

The Secret contains `tls.crt` and `tls.key`. Argo CD's API server automatically reloads this Secret; a restart is normally unnecessary. Keep server-side TLS enabled for the existing `8080:443` port-forward.

Keep this running in a separate terminal:

```powershell
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Reopen your browser at https://localhost:8080. With the Argo CD CLI installed, log in without the earlier `--insecure` option:

```powershell
argocd login localhost:8080 --username admin
```

To check certificate trust from Windows without bypassing verification:

```powershell
curl.exe --head https://localhost:8080
```

An HTTP response without a certificate validation error confirms the TLS connection. If trust fails, restart the browser/terminal and confirm `mkcert -install` ran on this machine. If the server still presents its old certificate, allow time for Secret propagation and inspect the certificate issuer in the browser. A different hostname needs a matching certificate name.

## 3. Publish the repository

Argo CD reads Git from inside Kubernetes; it cannot read this local Windows folder. The three shared manifests in `argocd/base` point to `https://github.com/VStefanov/gitops-example.git` and branch `main`, based on this repository's origin. If you use another repository or branch, edit `repoURL` and `targetRevision` in those three base files; all six overlays inherit them.

Commit and push the files to that repository:

```powershell
git add README.md charts argocd env1-non-prod env1-prod env2-non-prod env2-prod
git commit -m "Add local Argo CD example and .NET Helm charts"
git push -u origin main
```

If your current branch is different, switch to/create `main` before pushing, or change the manifests to use that branch.

For a private repository, use **Settings > Repositories > Connect Repo** in the Argo CD UI to configure credentials for the HTTPS URL. Your local Git SSH credentials are not inherited by Argo CD. Keep tokens and private keys out of this repository.

### Connect this repository over SSH instead

The current manifests use HTTPS. If you prefer SSH, configure the SSH repository and change `repoURL` in the three shared manifests under `argocd/base` to `git@github.com:VStefanov/gitops-example.git`. Credentials for an SSH URL do not authenticate the existing HTTPS URL. Alternatively, keep HTTPS and configure a GitHub token with read access in the UI.

For SSH, create a dedicated deploy key outside the repository:

```powershell
$sshDirectory = Join-Path $env:USERPROFILE '.ssh'
New-Item -ItemType Directory -Path $sshDirectory -Force | Out-Null
$deployKeyPath = Join-Path $sshDirectory 'argocd-gitops-example'
ssh-keygen -t ed25519 -C 'argocd-gitops-example' -f $deployKeyPath
```

Use an unused filename. When prompted for a passphrase, press Enter twice: Argo CD needs a key it can use unattended. In GitHub, open this repository's **Settings > Deploy keys > Add deploy key**, paste the contents of `argocd-gitops-example.pub`, and leave write access disabled.

In Argo CD, open **Settings > Repositories > Connect Repo**, select SSH, enter `git@github.com:VStefanov/gitops-example.git`, and paste the **private** key from `argocd-gitops-example`. Preserve its header, footer, and line breaks. Alternatively, after CLI login:

```powershell
argocd repo add git@github.com:VStefanov/gitops-example.git --ssh-private-key-path "$deployKeyPath"
argocd repo list
```

For SSH host verification, check **Settings > Certificates** for the GitHub SSH host entries. If missing or outdated, add trusted entries through the UI or `argocd cert add-ssh --from <verified-known-hosts-file>`. Verify fingerprints against [GitHub's published SSH fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints); do not blindly trust `ssh-keyscan` output or disable host verification. See [Argo CD private repository configuration](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/).

Use the actual connection error to choose the next step:

| Error | What to check |
| --- | --- |
| `x509: certificate signed by unknown authority` connecting to localhost | The local TLS certificate and Windows CA trust from section 2a. |
| `x509` error fetching an HTTPS Git repository | The Git server's certificate/CA trust inside Argo CD; a custom Git CA can be added with `argocd cert add-tls <git-hostname> --from <ca.pem>`. |
| `Permission denied (publickey)` or `unable to authenticate` | The private deploy key configured in Argo CD and its public key registered for this GitHub repository. |
| `knownhosts`, unknown host key, or host key mismatch | Verified Git SSH host entries in Argo CD. |
| SSH timeout, DNS failure, or connection refused | Network access from the Argo CD pods to GitHub on SSH port 22. |

## 4. Deploy an environment

Start with env1 non-prod qa:

```powershell
kubectl kustomize env1-non-prod/qa
kubectl apply -k env1-non-prod/qa
kubectl get deployments,pods,services -n env1-non-prod-qa

kubectl apply -k env2-non-prod/qa
kubectl get deployments,pods,services -n env2-non-prod-qa

kubectl get applicationsets,applications -n argocd
```

The overlay creates the ApplicationSet and both Applications in the `argocd` namespace. The ApplicationSet creates `env1-non-prod-qa-service`; the other resources are `env1-non-prod-qa-common-concerns` and `env1-non-prod-qa-storybook`. Argo CD creates the destination namespace and automatically syncs, prunes removed resources, and corrects drift. Initial reconciliation may take a few minutes.

If you already applied the original files, run `kubectl apply -k` for those same environment folders. The rendered resource names and specifications are preserved, so this updates the existing resources in place. No deletion or recreation is required.

Production uses these overlays:

```powershell
kubectl apply -k env1-prod/prod
kubectl apply -k env2-prod/prod
```

Production resources now use names such as `env1-prod-prod-services` and destination namespace `env1-prod-prod`. If you previously deployed the old production qa/preprod overlays, their resources remain in Kubernetes until you explicitly remove them; applying the new prod overlay creates a separate set of applications. Confirm the new production applications are healthy before retiring the old ones.

To apply all six combinations explicitly:

```powershell
foreach ($environment in @('env1-non-prod', 'env1-prod', 'env2-non-prod', 'env2-prod')) {
    $stages = if ($environment -like '*-non-prod') { @('qa', 'preprod') } else { @('prod') }
    foreach ($stage in $stages) {
        kubectl apply -k "$environment/$stage"
    }
}
```

Use `-k` on an environment/stage folder so kubectl builds its overlay. Do not apply the base directly: its environment fields are placeholders resolved by overlays. Helm values files are read by Argo CD, not applied as Kubernetes resources.

Chart and values changes take effect after committing and pushing. Changes to the shared base or an overlay require `kubectl apply -k` again for each affected environment; this example does not include a parent application to reconcile them.

## 5. Access the .NET workloads

Keep each port-forward running in a separate terminal:

```powershell
kubectl port-forward -n env1-non-prod-qa svc/service 8081:80
kubectl port-forward -n env1-non-prod-qa svc/common-concerns 8082:80
kubectl port-forward -n env1-non-prod-qa svc/storybook 8083:80
```

Open http://localhost:8081, http://localhost:8082, and http://localhost:8083. Initially all three show the ASP.NET Core sample.

## Customize the applications

Chart defaults live in `charts/<chart>/values.yaml`; environment overrides live in `<environment>/<stage>/values/<chart>.yaml`. Argo CD resolves each `valueFiles` path relative to its chart folder.

For example, replace the sample image in `env1-non-prod/qa/values/service.yaml`:

```yaml
image:
  repository: ghcr.io/your-organization/service
  tag: "1.0.0"
```

The image must contain a published application, not just the empty .NET runtime. Configure it to listen on the chart's `containerPort` (8080 by default). The chart sets `ASPNETCORE_HTTP_PORTS` and `ASPNETCORE_ENVIRONMENT`. Use immutable image tags/digests for real releases; the sample's version tag can change upstream.

Set `imagePullSecrets` to reference a registry Secret already created in each destination namespace. Use `envFrom` for existing Secrets/ConfigMaps; `env` is for non-secret values. ASP.NET Core options can use double underscores, e.g. `Logging__LogLevel__Default: Information`.

TCP startup, readiness, and liveness probes are provided so the sample does not require a particular health route. Replace them with application-specific HTTP health checks as needed. Set a probe to `{}` to disable it.

To deploy another service from the same chart in all environments, add a distinct `name`, `releaseName`, and `valuesFile` to the list in `argocd/base/applications.yaml` and create its values file in each environment. Unique release names keep Deployments and Services distinct within the namespace. Apply the affected overlays again. See the [ApplicationSet list generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-List/) and [Argo CD Helm values](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/) documentation.

To add another environment, copy an existing stage folder with its values, then edit `namePrefix` and the small patches in its `kustomization.yaml`: set its destination namespaces, ApplicationSet template name, and values paths. Update `APP_ENVIRONMENT` in the values files. Render with `kubectl kustomize <environment>/<stage>` before applying.

## Validate locally

```powershell
foreach ($chart in @('service', 'common-concerns', 'storybook')) {
    helm lint "charts/$chart" --strict
    helm template $chart "charts/$chart" --namespace env1-non-prod-qa -f "env1-non-prod/qa/values/$chart.yaml"
}
```

To render every overlay locally without contacting Kubernetes:

```powershell
foreach ($environment in @('env1-non-prod', 'env1-prod', 'env2-non-prod', 'env2-prod')) {
    $stages = if ($environment -like '*-non-prod') { @('qa', 'preprod') } else { @('prod') }
    foreach ($stage in $stages) {
        kubectl kustomize "$environment/$stage"
    }
}
```

To check an overlay's Argo resources against an installed cluster's CRDs:

```powershell
kubectl apply --dry-run=server -k env1-non-prod/qa
```

## Troubleshooting and cleanup

If an application reports a comparison error, check that the Git branch is pushed, the repo URL is accessible, and the values file exists. For `ImagePullBackOff`, check the image/tag and registry credentials. For pods that do not become ready, inspect:

```powershell
kubectl describe pods -n env1-non-prod-qa
kubectl logs -n env1-non-prod-qa deployment/service
kubectl get applications -n argocd
kubectl logs -n argocd deployment/argocd-applicationset-controller
```

To remove the entire disposable kind cluster, including Argo CD and all workloads:

```powershell
kind delete cluster --name gitops-example
```

This cleanup applies only to the kind cluster created above.
