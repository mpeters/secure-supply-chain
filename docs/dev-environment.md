# Development Environment

This guide provides instructions on how to set up your development environment
to iterate on any type of work related to secure supply chain.

Specifically, this guide focuses on installing the following components:

- Kubernetes
- Tekton CI/CD Pipelines
- In-Toto (TBD)
- Open Policy Agent (TBD)
- Keylime (TBD)

## Prerequisites

These are the various prerequisites that you'll need to work through this guide.

- [docker](https://docs.docker.com/engine/install/) for using `kind`. Make sure
  you `docker login` first if you have not done so already.
- [kind](https://kind.sigs.k8s.io/) will be used to set up a quick and
  lightweight Kubernetes cluster. Kubernetes vresion >= 1.15 is required.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) is the
  Kubernetes client required to interact with a Kubernetes cluster. Make sure
  its version is also >= 1.15.
- Additional Tekton CI/CD requirements to install:
    - [Go](https://golang.org/dl/) - Make sure you have a recent version of the Go
      programming language installed and added to your `PATH` environment
      variable.
    - [git](https://git-scm.com/downloads) make sure git is installed and set
      up to work with
      [GitHub](https://help.github.com/en/github/getting-started-with-github/set-up-git).
    - [ko](https://github.com/google/ko) version >= v0.1 is required to work
      with and deploy development versions of Tekton. Make sure it's available
      in your `PATH`.
    - [tkn](https://github.com/tektoncd/cli) is the Tekton CLI for interacting
      with Tekton.

## Create Kubernetes Cluster and Container Registry

To create a kubernetes cluster with `kind` along with a local container
registry that can be used to easily push and pull container images for
development, run the following script:

```shell
./scripts/kind-with-registry.sh
```

This will add a `kind-kind` context name to your `KUBECONFIG` as shown here:

```shell
$ kubectl config get-contexts kind-kind
CURRENT   NAME        CLUSTER     AUTHINFO    NAMESPACE
*         kind-kind   kind-kind   kind-kind
```

To get the cluster info for your `kind` cluster run:

```shell
$ kubectl cluster-info --context kind-kind
Kubernetes master is running at https://127.0.0.1:32771
KubeDNS is running at
https://127.0.0.1:32771/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info
dump'.
```

Now you have a working local Kubernetes cluster and local container registry.

## Tekton CI/CD

This section outlines how to work with a development version of Tekton CI/CD. The
instructions are mostly leveraged from the [Tekton development
guide](https://github.com/tektoncd/pipeline/blob/master/DEVELOPMENT.md)
and specifically tailored to our development workflow.

### Fork and Clone Development Version of Tekton Pipeline

Make sure to clone the repository to the `GOPATH` you intend to use. The
default `GOPATH` used by the `Go` toolchain on Linux is `${HOME}/go` as
shown:

```shell
$ go env GOPATH
/home/<username>/go
```

For more information see [`GOPATH`](https://github.com/golang/go/wiki/SettingGOPATH).

Once you've set up your `GOPATH` or used the default `Go` one, you'll want to
fork the respective repository in GitHub and clone the forked repository into
your `Go` development environment.

To do that first create your [own
fork](https://help.github.com/articles/fork-a-repo/) of the respective
repository. For this example we'll use the upstream [Tekton
Pipeline](https://github.com/tektoncd/pipeline) GitHub
Repo.

Once you've forked it, clone it to your development system by running:

```shell
export GOPATH=$(go env GOPATH)
mkdir -p ${GOPATH}/src/github.com/tektoncd
cd ${GOPATH}/src/github.com/tektoncd
git clone git@github.com:${YOUR_GITHUB_USERNAME}/pipeline.git
cd pipeline
```

Then add the `upstream` remote so you're set up for regularly
[syncing your fork](https://help.github.com/articles/syncing-a-fork/) by
running:

```shell
git remote add upstream git@github.com:tektoncd/pipeline.git
git remote set-url --push upstream no_push
```

### Install Tekton Pipeline

To [run your controllers with `ko`](#install-pipeline) you'll need to set these
environment variables (you can add them to your `.bashrc`):

- `GOPATH`: Already set previously but if you don't have one, simply pick a
  directory and add `export GOPATH=...`
- `$GOPATH/bin` in your `PATH`: This is so that tooling installed via `go get`
  will work properly.
- `KO_DOCKER_REPO`: The docker repository to which developer images should be
  pushed (e.g. `quay.io/<user>/<repo>`). For this guide we're [running a local
  registry](https://docs.docker.com/registry/deploying/) so we'll set
  `KO_DOCKER_REPO` to reference the local registry e.g.
  `localhost:5000/mypipeline`.

Here's a `.bashrc` example:

```shell
export GOPATH="${HOME}/go"
export PATH="${PATH}:${GOPATH}/bin"
export KO_DOCKER_REPO="localhost:5000/mypipeline"
```

Then to install the controller from your existing code into the cluster
specified by your `KUBECONFIG` current context as shown by `kubectl config
current-context` run:

```shell
ko apply -f config/
```

### Verify Tekton Pipeline Works

To perform a basic test of a `PipelineRun` execution of a `Pipeline` with a
`Task`, execute:

```shell
kubectl apply -f config/tekton/hello-world-pipeline.yaml
```

Now that the pipeline is executing, you can check it has been created and track
its progress using the Tekton CLI `tkn`:

```shell
$ tkn pipelinerun list
NAME                  STARTED         DURATION   STATUS
sample-pipeline-run   3 seconds ago   ---        Running
$ tkn pipelinerun logs -f sample-pipeline-run
[echo-something : echo] hello world
```

Once you're done, feel free to delete the Tekton resources used for this test:

```shell
kubectl delete -f config/tekton/hello-world-pipeline.yaml
```

### Redeploy Controller

As you make changes to the code, you can redeploy your controller with:

```shell
ko apply -f config/controller.yaml
```

### Remove Tekton Pipeline

You can clean up the Tekton Pipeline installation with:

```shell
ko delete -f config/
```

### For More Tekton Development Information

View the [Tekton development
guide](https://github.com/tektoncd/pipeline/blob/master/DEVELOPMENT.md) for
more details about how to:

- Install Tekton Pipeline into a custom namespace
- Accessing the controller and/or webhook logs
- Update dependencies or type definitions
- Adding new types
- And much more
