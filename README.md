# FluxCD Gitops using image automation and image reflector controller

This is a documentation, designed to guide you through the practical implementation of a GitOps workflow using FluxCD. In this documentation, we'll explore how to leverage the power of image automation and the image reflector controller to streamline your development and deployment processes.

We will showcase this through a deployment pipeline in 2 env (staging and production) wherein using githubactions we will do the CI process of building and pushing the containers to registry , once pushed to registry, the image automation and image reflector controllers will identify the change and make an automatic update to kubernetes manifests and flux will automatically reconcile the changes in stage env. For Prod, to ensure control, changes are promoted to production through a Pull Request (PR) with manual approval.
![gitops-image-automation](https://github.com/shashankpai/gitops-helloenv/assets/24639491/f54850ab-1d8a-41d7-b197-4d20530f3046)



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
for the `${GITHUB_USER}/gitops-helloenv` repo here.

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

Now that we have things running, let's back up and look at _exactly_ how Flux
pulls everything together. A good first step here is to look back at the `flux
bootstrap` command we used to kick everything off:

```
flux bootstrap github \ 
--components-extra=image-reflector-controller,image-automation-controller \
--owner=$GITHUB_USER     --repository=gitops-helloenv     \
--path=./clusters/my-cluster/     --branch=main    \
 --read-write-key     --personal --private=false
```

flux bootstrap github: Initiates the process of setting up a GitOps repository on GitHub.

--components-extra=image-reflector-controller,image-automation-controller: Specifies additional Flux components to include in the setup.

--owner=$GITHUB_USER: Specifies the GitHub owner (likely a GitHub username or organization name) where the repository will be created.

--repository=gitops-helloenv: Defines the name of the GitOps repository to be created on GitHub.

--path=./clusters/my-cluster/: Specifies the path within the repository where the configuration files will be stored.

--branch=main: Sets the default branch for the repository to "main."

--read-write-key: Generates a read-write SSH key for accessing the GitOps repository.

--personal: Indicates that a personal access token should be used for authentication.

--private=false: Specifies that the GitOps repository should not be private (i.e., it will be public).


`infrastructure.yaml` is a good place to look first.

The first document defines a `Kustomization` resource called `ingress-nginx`.
This Kustomization lives in the `flux-system` namespace, doesn't depend on
anything, and has `kustomize` files at `infrastructure/ingress-nginx`

Let's look quickly at `ingress-nginx`'s `kustomize` files:

```bash
ls -l ./infrastructure/ingress-nginx
```

The `kustomization.yaml` file tells `kustomize` what other files to use:

```bash
cat ./infrastructure/ingress-nginx/kustomization.yaml
```

<!-- @clear -->


If we take a closer look at the definitions used in this project's infrastructure setup. We'll start by examining the file, which includes the following key components:

- **Namespace**: It creates the `ingress-nginx` namespace.
- **HelmRepository**: This section informs Flux about the location of the Helm chart for `ingress-nginx`.
- **HelmRelease**: Here, Flux is instructed on how to utilize the Helm chart to install `ingress-nginx`. It's important to note that we are not utilizing `kustomize` for patching operations but instead for sequencing the application of specific YAML files. Some of these YAML files are dedicated to Flux resources rather than standard Kubernetes resources.

The infra section provided a quick overview of the infrastructure definitions required for the cluster. Essentially, it encompasses all the necessary components for our application to function correctly.

Now, let's proceed with a closer examination of `apps.yaml`, which defines the `helloenv-app` application itself. Within this file, you'll find references to the `apps` directory. This directory is a central location containing Kustomizations and configuration settings for setting up the `helloenv-app` application in both the `prod` and `staging` environments. Additionally, it includes configurations for image automation for `helloenv` app in `staging` and `prod` env. 

In the following sections, we'll take a detailed look at each file within the `apps` directory, one by one.

**git-repo.yaml**,this instructs Flux on how to interact with the Git repository where your application's source code resides. It contains the following configuration:
```
  ref:
    branch: main
  secretRef:
    name: ssh-credentials
  url: ssh://git@github.com/shashankpai/helloenv-app
```
- **ref**: Specifies the Git branch to watch for changes, in this case, main.
- **secretRef**: Refers to a Kubernetes Secret named `ssh-credentials`, which likely contains SSH keys for secure Git access.
- **url**: Indicates the URL of the Git repository 

**image-repo.yaml**,  is responsible for scanning the Docker image registry and fetching image tags based on the defined policy. Here's the configuration:

```
image: docker.io/shapai/helloenv
  interval: 1m0s
```
- **image**: Specifies the Docker image repository (docker.io/shapai/helloenv) to scan for image tags.
- **interval**: Sets the interval at which Flux will scan the image repository (every 1 minute in this case) and fetch image tags according to the defined policy.

Next we will go through the image policy files `image-policy-staging.yaml` and `image-policy-prod.yaml`

**image-policy-staging.yaml**  defines the image tagging policy for the staging environment. Here's the configuration:

```
  filterTags:
    extract: $ts
    pattern: Build-[a-f0-9]+-(?P<ts>[0-9]+)
  imageRepositoryRef:
    name: helloenv
  policy:
    numerical:
      order: asc
```
- **filterTags**: This section specifies how to filter image tags. It extracts the timestamp ($ts) from tags that match the specified pattern. Tags are filtered in ascending order based on this timestamp, ensuring that Flux fetches the latest built image for the staging environment.

**image-policy-prod.yaml** defines the image tagging policy for the production environment:

```
  imageRepositoryRef:
    name: helloenv
  policy:
    semver:
      range: '>=1.0.0'
```

- **imageRepositoryRef**: Refers to the image repository named helloenv.
- **policy**: Defines a Semantic Versioning (SemVer) policy that specifies a range for acceptable image tags (in this case, any version greater than or equal to 1.0.0).

These image policies play a crucial role in ensuring that Flux deploys the correct images to the respective environments (staging and production) based on the defined criteria.
