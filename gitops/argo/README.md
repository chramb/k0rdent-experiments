# installation of Flux + k0rdent

prerequisites:

- proper access to git repo for argo depending git forge (GitHub application in this scenario. [Setup guide by CNCF](https://www.cncf.io/blog/2023/10/27/using-github-apps-with-argocd/)
- argocd cli
- kubernetes cluster


## Setup argo with helm

> [!NOTE]
> The official docs propose [installing argo from manifests](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd) but having them all coupled in helm chart seems cleaner to manage in my opinion.

```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd -n argocd argo/argo-cd --create-namespace
```
```console
NAME: argocd
LAST DEPLOYED: Thu Mar 27 17:29:39 2025
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```

By following the instructions printed to the console you can access the WebUI.

Because argo installation doesn't come with its own ingress for now the easiest way to access WebUI is with following command:
```shell
kubectl port-forward svc/argocd-server 8080:443 -n argocd --address="0.0.0.0"
```


## Add access to repository

Now to add repository to 

You can do it either with argocd cli or webui. 
For GitHub private repo access you'll need to share Application Credentials with Argo, good guide on how to create one is available here: https://www.cncf.io/blog/2023/10/27/using-github-apps-with-argocd/

```shell
kubectl config set-context --current --namespace=argocd
argocd repo add https://github.com/chramb/k0rdent-experiments.git --github-app-id 1180847 --github-app-installation-id 62773882 --github-app-private-key-path /Users/kambrozy/Downloads/argo-test-chramb.2025-03-16.private-key.pem
```
```console
Repository 'https://github.com/chramb/k0rdent-experiments.git' added
```

Now Argo knows about that repository but no application yet is using it, we need to create an application that references this repository for Argo to start tracking and applying resources to our cluster.

```shell
cat > apps/apps.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps
  namespace: argocd
spec:
  project: default
  source:
    path: gitops/argo/apps
    repoURL: https://github.com/chramb/k0rdent-experiments.git
    targetRevision: dev
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
EOF
kubectl apply -f apps/apps.yml 
```
```console
application.argoproj.io/apps created
```

> [!NOTE]
> This is the only time you have to apply changes manually after this everything else should be tracked and reconciled with the use of argo. 

This application resource tracks content of `gitops/argo/apps` directory in which we'll add all the other apps that should be deployed (including argo itself)

## Making argo track itself
<!-- add apps one by one and sync (can be automatic) the only thing it adds is extra label that it is managed by argo. -->

Because we installed argo as helm chart next application to add is for the

```shell
cat > apps/argo.yaml <<EOF
# helm repo add argo https://argoproj.github.io/argo-helm	
# helm install argocd argo/argo-cd --namespace argocd --version 5.36.1 -f values.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-cd
    targetRevision: 7.8.14
  destination:
    server: "https://kubernetes.default.svc"
    namespace: argocd
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF
git add apps/argo.yaml
git commit -m 'Make argo manage itself'
git push
```

Argo should now pick up changes and because we didn't enable automatic syncing the app will be marked as out of sync
```shell
kubectl get apps
```
```console
NAME   SYNC STATUS   HEALTH STATUS
apps   OutOfSync     Healthy
```

To remedy it run `argocd app sync apps`

```shell
kubectl get apps
```
```console
NAME     SYNC STATUS   HEALTH STATUS
apps     Synced        Healthy
argocd   OutOfSync     Healthy
```
The same steps can be followed for installation of k0rdent

```
cat > apps/kcm.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kcm
  namespace: argocd
spec:
  project: default
  source:
    chart: kcm
    repoURL: ghcr.io/k0rdent/kcm/charts
    targetRevision: 0.2.0
  destination:
    name: "in-cluster"
    namespace: kcm-system
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF
git add apps/kcm.yaml
git commit -m 'Add kcm'
git push
```
> [!TIP]
> If argo didn't pick up changes to the repo yet you can force it to refresh it with following command
> ```shell
> argocd app get --refresh kcm
> ```

Now last thing to do is to sync both apps first to create kcm Application object and kcm Application itself.
```shell
argocd app sync apps
argocd app sync kcm
```

After a while we should see successfull installation of k0rdent

```shell
 kubectl get pods -n kcm-system
```
```console
NAME                                                          READY   STATUS    RESTARTS   AGE
azureserviceoperator-controller-manager-6b4dd86894-n75vm      1/1     Running   0          10m
capa-controller-manager-64bbcb9f8-9wjll                       1/1     Running   0          9m53s
capi-controller-manager-66f8998ff5-fnbtp                      1/1     Running   0          11m
capo-controller-manager-588f45c7cf-ltr5h                      1/1     Running   0          9m46s
capv-controller-manager-69f7fc65d8-25t8b                      1/1     Running   0          9m35s
capz-controller-manager-544845f786-ppl4t                      1/1     Running   0          10m
helm-controller-7644c4d5c4-r8rvh                              1/1     Running   0          13m
k0smotron-controller-manager-bootstrap-9fc48d76f-nnktj        2/2     Running   0          11m
k0smotron-controller-manager-control-plane-7df9bc7bf-bvczz    2/2     Running   0          10m
k0smotron-controller-manager-infrastructure-f7f94dd76-m7wjh   2/2     Running   0          9m39s
kcm-cert-manager-895954d88-zvttf                              1/1     Running   0          13m
kcm-cert-manager-cainjector-685ffdf549-nd2qp                  1/1     Running   0          13m
kcm-cert-manager-webhook-59ddc6b56-5gkrd                      1/1     Running   0          13m
kcm-cluster-api-operator-8487958779-xqhl5                     1/1     Running   0          11m
kcm-controller-manager-7998cdb69-tl6dz                        1/1     Running   0          11m
kcm-velero-b68fd5957-vd74c                                    1/1     Running   0          13m
source-controller-6cd7676f7f-k2ggj                            1/1     Running   0          13m
```


Last thing to do is to deploy a child cluster with Argo which can be done haing one app per cluster or grouping multiple clusters into one ArgoCD Application. 
