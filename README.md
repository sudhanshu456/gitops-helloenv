# FluxCD Gitops using image automation and image reflector controller

This is a documentation, designed to guide you through the practical implementation of a GitOps workflow using FluxCD. In this documentation, we'll explore how to leverage the power of image automation and the image reflector controller to streamline your development and deployment processes.

We will showcase this through a deployment pipeline in 2 env (staging and production) wherein using githubactions we will do the CI process of building and pushing the containers to registry , once pushed to registry, the image automation and image reflector controllers will identify the change and make an automatic update to kubernetes manifests and flux will automatically reconcile the changes in stage env. For Prod, to ensure control, changes are promoted to production through a Pull Request (PR) with manual approval.

# Requirements
- Flux CLI that you can download here https://fluxcd.io/docs/cmd/
- k8s cluster. For this demo, we're going to use minikube
- Github account with github actions enabled

To implement this demo, requires some fairly specific setup

- First, you need a running Kubernetes cluster

- Second, you need to fork https://github.com/shashankpai/gitops-helloenv and
  https://github.com/shashankpai/helloenv-app under your own account. You 
  also need to set GITHUB_USER to your GitHub username, and GITHUB_TOKEN to a
  personal access token under your account with `repo` scope.

- Finally, you need to edit `apps/git-repo.yaml` in your clone of
  `gitops-helloenv` to point to your fork of helloenv-app -- change the `url`
  field on line 13 as appropriate.

# FluxCD Gitops using image automation and image reflector controller

Before running make sure if you are using minikube , enable ingress addon

```bash
minikube addons enable ingress
```

Beginning with an empty cluster, our initial task is to bootstrap Flux itself. Flux serves as the foundation upon which we'll bootstrap all other components.

Note the `--owner` and `--repository` switches here: we are explicitly looking
for the `${GITHUB_USER}/gitops-linkerd` repo here.

```
flux bootstrap github \ 
--components-extra=image-reflector-controller,image-automation-controller \
--owner=$GITHUB_USER     --repository=gitops-helloenv     \
--path=./clusters/my-cluster/     --branch=main    \
 --read-write-key     --personal --private=false
```
Now, Flux will proceed by installing its own in-cluster services before moving on to install all the other components we've specified.

To gain insights into its actions, we can inspect its Kustomization resources

```bash
flux get kustomizations
```

The process of bootstrapping everything may take some time. To monitor the progress and ensure everything is proceeding as expected, we can utilize the --watch switch:

```bash
flux get kustomizations --watch
```
```bash
kubectl -n flux-system get pods
NAME                                           READY   STATUS    RESTARTS   AGE
image-automation-controller-6c4fb698d4-zrp78   1/1     Running   0          29s
image-reflector-controller-5dfa39212d-hnnvj    1/1     Running   0          29s
kustomize-controller-424f5ab2a2-u2hwb          1/1     Running   0          29s
source-controller-2wc41z892-axkr1              1/1     Running   0          29s
```
Finally, we need to generate a deploy key so Flux can commit to the helloenv-app repo. The public key should be added to Gihub.

```
$ ssh-keygen -q -N "" -f ./identity
$ ssh-keyscan github.com > ./known_hosts

$ kubectl -n flux-system create secret generic ssh-credentials \
    --from-file=./identity \
    --from-file=./identity.pub \
    --from-file=./known_hosts
```

```bash
kubectl rollout status -n helloenv-prod deployments
watch kubectl get pods -n helloenv-prod

kubectl rollout status -n helloenv-staging deployments
watch kubectl get pods -n helloenv-staging
```
<!-- @clear -->

At this point, our application is running in both stage and production environment with the default available image for prod and stage.

You can also check the ingress that are available for stage and prod 

```
kubectl get ingress --all-namespaces
NAMESPACE          NAME       CLASS    HOSTS                ADDRESS   PORTS   AGE
helloenv-prod      helloenv   <none>   helloenv.prod.com              80      9m22s
helloenv-staging   helloenv   <none>   helloenv.stage.com             80      9m21s
```

add the initial curl calls here 

```
curl helloenv.prod.com

curl helloenv.stage.com
```

# What's Under The Hood
