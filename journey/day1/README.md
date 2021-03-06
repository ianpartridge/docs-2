# Day 1 Operations

Day 1 Operations are actions that users take to bootstrap a GitOps system.   Bootstrapping GitOps can be done in one of the two commands.

* [odo pipelines bootstrap](../../commands/bootstrap)
* [odo pipelines init](../../commands/init)

These commands are similar.  They differ in what gets generated.  The bootstrap command generated a functional GitOps setup including your first application.  The init command only generates CI/CD pipelines and associated resources.  This document describes how to bootstrap GitOps to deliver your first application.

## Prerequisites

**NOTE**: `odo pipelines` commands are hidden in `Expermential Mode`.  To enable `Expermential Mode` available, please set th `EXPERIEMENTAL` environment variable in the running terminal.
```shell
$ export ODO_EXPERIMENTAL=true
```

You need to have the following installed in the OCP 4.x cluster.
* [Sealed Secrets Operator](prerequisites/sealed_secrets.md)
* [OpenShift Pipelines Operator](prerequisites/tekton_operator.md)
* [ArgoCD](prerequisites/argocd.md)

And, you will need this.
* Create [GitOps repository](prerequisites/gitops_repo.md)
* Source Git repository ([taxi](prerequisites/service_repo.md) is used as an example in this document)
* Download unofficial [odo](../../commands/bin) binary

## Bootstrapping the Manifest

```shell
$ odo pipelines bootstrap \
  --service-repo-url https://github.com/<username>/taxi.git \
  --gitops-repo-url https://github.com/<username>/gitops.git \
  --image-repo quay.io/<username>/taxi \
  --dockercfgjson ~/Downloads/<username>-auth.json \
  --prefix tst
```

## Exploring the pipelines.yaml (manifest file)

The bootstrap process generates a fairly large number of files, including a
pipelines.yaml describing your first application, and configuration for a complete CI
pipeline and deployments from ArgoCD.

A pipelines.yaml file (an example below) is generated by `bootstrap` command.  This file is used by Day 2 commands such as `service add` to generated/updated pipelines resources.

```yaml
config:
  argocd:
    namespace: argocd
  pipelines:
    name: tst-cicd
environments:
- apps:
  - name: app-taxi
    services:
    - taxi
  name: tst-dev
  pipelines:
    integration:
      bindings:
      - github-pr-binding
      template: app-ci-template
  services:
  - name: taxi
    pipelines:
      integration:
        bindings:
        - tst-dev-taxi-binding
        - github-pr-binding
    source_url: https://github.com/wtam2018/taxi.git
    webhook:
      secret:
        name: webhook-secret-tst-dev-taxi
        namespace: tst-cicd
- name: tst-stage
gitops_url: https://github.com/wtam2018/gitops.git

```

The bootstrap creates two environments, `dev`, and `stage`
with a user-supplied prefix.  Namespaces are generated for these environments.

The name of the app and service are derived from the last component of your
`service-repo-url` e.g. if your bootstrap with `--service-repo-url
https://github.com/myorg/myproject.git` this would bootstrap an app called
`app-myproject` and a service called `myproject`.

## Environment configuration

The `tst-dev` environment created is the most visible configuration.

### configuring pipelines

```yaml
environments:
- apps:
  - name: app-taxi
    services:
    - taxi
  name: tst-dev
  pipelines:
    integration:
      bindings:
      - github-pr-binding
      template: app-ci-template
  services:
  - name: taxi
    pipelines:
      integration:
        bindings:
        - tst-dev-taxi-binding
        - github-pr-binding
    source_url: https://github.com/wtam2018/taxi.git
    webhook:
      secret:
        name: webhook-secret-tst-dev-taxi
        namespace: tst-cicd
  template: app-ci-template
  services:
  - name: taxi-svc
    source_url: https://github.com/wtam2018/taxi.git
    webhook:
      secret:
        name: github-webhook-secret-taxi-svc
        namespace: tst-cicd      
```

The `pipelines` key describes how to trigger a Tekton PipelineRun, the
`integration` binding and template are processed when a _Pull Request_
is opened.

This is the default pipeline specification for the `tst-dev` environment, you
can find the definitions for these in these two files:

 * [`config/<prefix>-cicd/base/pipelines/07-templates/app-ci-build-pr-template.yaml`](output/config/tst-cicd/base/pipelines/07-templates/app-ci-build-pr-template.yaml)
 * [`config/<prefix>-cicd/base/pipelines/06-bindings/github-pr-binding.yaml`](output/config/tst-cicd/base/pipelines/06-bindings/github-pr-binding.yaml)

By default this triggers a PipelineRun of this pipeline

 * [`config/<prefix>-cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml`](output/config/tst-cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml)

These files are not managed directly by the manifest, you're free to change them
for your own needs, by default they use [Buildah](https://github.com/containers/buildah)
to trigger build, assuming that the Dockerfile for your application is in the root
of your repository.

### configuring services

```yaml
- apps:
  - name: app-taxi
    services:
    - taxi
  services:
  - name: taxi
    pipelines:
      integration:
        bindings:
        - tst-dev-taxi-binding
        - github-pr-binding
    source_url: https://github.com/wtam2018/taxi.git
    webhook:
      secret:
        name: webhook-secret-tst-dev-taxi
        namespace: tst-cicd
```

The YAML above defines an app called `app-taxi`, which has a reference to service called `taxi`.

The configuration for these is written out to:

 * [`environments/test-dev/services/taxi/base/config/`](output/environments/tst-dev/services/taxi/base/config)
 * [`environments/<prefix>-dev/apps/app-taxi/base/`](output/environments/tst-dev/apps/app-taxi/base/)

The `app-taxi` app's configuration references the services configuration.

The `source_url` references the upstream repository for the service, this
coupled with the `webhook.secret` is also used to drive the CI pipeline, with a
hook confgured, changes to this repository will trigger the template and
binding), the secret is used to authenticate incoming hooks from GitHub.

## Bringing the bootstrapped environment up

First of all, let's get started with our Git repository.

From the root of your gitops directory (with the pipelines.yaml), execute the
following commands:

```shell
$ git init .
$ git add .
$ git commit -m "Initial commit."
$ git remote add origin <insert gitops repo>
$ git push -u origin master
```

This should initialise the GitOps repository, this is the start of your journey
to deploying applications via Git.

Next, we'll bring up our deployment infrastructure, this is only necessary at the
start, the configuration should be self-hosted thereafter.


```shell
$ oc apply -k environments/tst-dev/env/base
$ oc apply -k config/tst-argocd/config
$ oc apply -k config/tst-cicd/base
```

You should now be able to create a route to your new service, it should be
running [nginx](https://nginx.org/) and serving a page.

## Changing the initial deployment

The bootstrap creates a `Deployment` in `environments/<prefix>-dev/services/<service name>/base/config/100-deployment.yaml` this should bring up nginx, this is purely for demo purposes, you'll need to change this to deploy your built image.

```yaml
spec:
  containers:
  - image: nginxinc/nginx-unprivileged:latest
    imagePullPolicy: Always
    name: taxi-svc
```

You'll want to replace this with the image for your application, if you've built
it.

## Your first CI run

Part of the configuration bootstraps a simple TektonCD pipeline for building code when a pull-request is opened.

You will need to create a new Webhook for the CI:

```shell
$ odo pipelines webhook create \
    --access-token <github user access token>
    --env-name tst-dev
    --service-name taxi
```

Make a change to your application source, the `taxi` repo from the example, it
can be as simple as editing the `README.md` and propose a change as a
Pull Request.

This should trigger the PipelineRun:

![PipelineRun with succesful completion](img/pipelinerun-success.png)

Drilling into the PipelineRun we can see that it executed our single task:

![PipelineRun with steps](img/pipelinerun-succeeded-detail.png)

And finally, we can see the logs that the build completed and the image was
pushed:

![PipelineRun with logs](img/pipelinerun-succeeded-logs.png)

## Changing the default CI run

Before this next stage, we need to ensure that there's a Webhook configured for
the "gitops" repo.

```shell
$ odo pipelines webhook create \
    --access-token <github user access token>
    --cidi
```

This step involves changing the CI definition for your application code.

The default CI pipeline we provide is defined in the manifest file:

```yaml
  pipelines:
    integration:
      bindings:
      - github-pr-binding
      template: app-ci-template
```

This template drives a Pipeline that is stored in this file:

 * [`config/<prefix>-cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml`](output/config/tst-cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml)

An abridged version is shown below, it has a single task `build-image`, which
executes the `buildah` task, which basically builds the source and generates an
image and pushes it to your image-repo.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
spec:
  resources:
  - name: source-repo
    type: git
  tasks:
  - name: build-image
      inputs:
      - name: source
        resource: source-repo
    taskRef:
      kind: ClusterTask
      name: buildah
```

Normally, you'd at least run a simple test of the code before you build the
code:

Write the following Task to this file:

 * `config/<prefix>-cicd/base/pipelines/04-tasks/go-test-task.yaml`

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: go-test
  namespace: tst-cicd
spec:
  inputs:
    resources:
    - name: source
      description: the git source to execute on
      type: git
  steps:
    - name: go-test
      image: golang:latest
      command: ["go", "test", "./..."]
```

This is a simple test task for a Go application, it just runs the tests.

Update the pipeline in this file:

 * [`config/<prefix>-cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml`](output/config/tst-cicd/base/pipelines/05-pipelines/app-ci-pipeline.yaml)


```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  creationTimestamp: null
  name: app-ci-pipeline
  namespace: tst-cicd
spec:
  params:
  - name: REPO
    type: string
  - name: COMMIT_SHA
    type: string
  resources:
  - name: source-repo
    type: git
  - name: runtime-image
    type: image
  tasks:
  - name: go-ci
    resources:
      inputs:
      - name: source
        resource: source-repo
    taskRef:
      kind: Task
      name: go-test
  - name: build-image
    runAfter:
      - go-ci
    params:
    - name: TLSVERIFY
      value: "true"
    resources:
      inputs:
      - name: source
        resource: source-repo
      outputs:
      - name: image
        resource: runtime-image
    taskRef:
      kind: ClusterTask
      name: buildah
```

Commit and push this code, and open a Pull Request, you should see a PipelineRun
being executed.

![PipelineRun doing a dry run of the configuration](img/pipelinerun-dryrun.png)

This validates that the YAML can be applied, by executing a `kubectl apply --dry-run`.

