# installation of Flux + k0rdent

prerequisites:

- proper access to git repo for flux depending on git forge (this example GitHub and GitHub PAT)
- flux cli
- kubernetes cluster

## Bootstrap according to any of the documented solutions 

> official docs: https://fluxcd.io/flux/installation/bootstrap/

Example for GitHub:

```shell
export GITHUB_TOKEN=<YOUR_TOKEN> # Should be properly secured
flux bootstrap github \
  --token-auth \
  --owner=chramb \
  --repository=k0rdent-experiments \
  --namespace=flux-system \
  --branch=master \
  --path=/gitops/flux \
  --personal=true

```
```console
► connecting to github.com
► cloning branch "master" from Git repository "https://github.com/chramb/k0rdent-experiments.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed component manifests to "master" ("d19995d277e22d0d26f63a1f8013eddb030a9bfe")
► pushing component manifests to "https://github.com/chramb/k0rdent-experiments.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "master" ("677af7ead347c8aae472f811aabe0d53c4a7d888")
► pushing sync manifests to "https://github.com/chramb/k0rdent-experiments.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

We successfuly deployed flux, time for installation of k0rdent to manage it and its clusters the GitOps way

## Clone/Pull git repo after flux bootstrap has been performed

```shell
git clone https://github.com/chramb/k0rdent-experiments.git && cd k0rdent-experiments/gitops/flux
tree
```
```console
.
└── flux-system
    ├── gotk-components.yaml
    ├── gotk-sync.yaml
    └── kustomization.yaml
```

# Test working flux

We'll test if it's working by creating namespace for future use with k0rdent (kcm-system)

Create all needed directories and files:

```shell
mkdir kcm-system
cat > ns.yaml  <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: kcm-system
EOF
cat > kcm-system/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ns.yml
EOF
git add kcm-system
git commit -m 'Add kcm-system namespace'
git push
```

We can watch it progressing with get kustomization --watch

```shell
flux get kustomizations --watch
```
```console
NAME       	REVISION            	SUSPENDED	READY  	MESSAGE                    
flux-system	master@sha1:677af7ea	False    	Unknown	Reconciliation in progress	
flux-system	master@sha1:677af7ea	False	Unknown	Reconciliation in progress	
flux-system	master@sha1:677af7ea	False	True	Applied revision: master@sha1:1c547deb	
flux-system	master@sha1:1c547deb	False	True	Applied revision: master@sha1:1c547deb
```

And to confirm it was created iside the cluster:

```shell
kubectl get ns kcm-system -o yaml
```
```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2025-03-19T13:41:25Z"
  labels:
    kubernetes.io/metadata.name: kcm-system
    kustomize.toolkit.fluxcd.io/name: flux-system
    kustomize.toolkit.fluxcd.io/namespace: flux-system
  name: kcm-system
  resourceVersion: "5714"
  uid: d4ac0ce9-17dd-41c9-8f27-802e36743325
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

We can also see extra labels added by flux to denote it is managed by it.

> [!NOTE]
> This manifest falls under flux-system kustomization inside flux because everything under path `./gitops/flux` is being watched by flux source controller

> [!TIP]
> To not wait for flux to re-check for changes in repo you can run
> ```
> flux reconcile kustomization flux-system --with-source
> ```

# Install kcm helm chart from OCI repository

```
cat > kcm-system/kcm.yaml <<EOF
# Equivalent of `helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version 0.1.0 -n kcm-system`
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: kcm
  namespace: kcm-system
spec:
  interval: 10m
  url: oci://ghcr.io/k0rdent/kcm/charts/kcm
  ref:
    tag: 0.1.0
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kcm
  namespace: kcm-system
spec:
  interval: 10m
  releaseName: kcm
  chartRef:
    kind: OCIRepository
    name: kcm
EOF
echo '- kcm.yaml' >> kcm-system/kustomization.yaml
git add kcm-system
git commit -m 'Install k0rdent'
git push
```

## Check results
```shell
kubectl get pod -n kcm-system
```
```console
NAME                                           READY   STATUS    RESTARTS   AGE
helm-controller-7644c4d5c4-sfwv9               1/1     Running   0          2m56s
kcm-cert-manager-895954d88-68bfb               1/1     Running   0          2m56s
kcm-cert-manager-cainjector-685ffdf549-4xld6   1/1     Running   0          2m56s
kcm-cert-manager-webhook-59ddc6b56-rsrrf       1/1     Running   0          2m56s
kcm-controller-manager-6f4d5bf7bb-g7qt6        1/1     Running   0          2m56s
kcm-velero-b68fd5957-4722g                     1/1     Running   0          2m56s
source-controller-6cd7676f7f-4ntzm             1/1     Running   0          2m56s
```
