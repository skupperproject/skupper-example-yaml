<!-- NOTE: This file is generated from skewer.yaml.  Do not edit it directly. -->

# Skupper Hello World using YAML

[![main](https://github.com/ssorj/skupper-example-yaml/actions/workflows/main.yaml/badge.svg)](https://github.com/ssorj/skupper-example-yaml/actions/workflows/main.yaml)

#### A minimal HTTP application deployed across Kubernetes clusters using Skupper

This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html

#### Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Step 1: Set up your clusters](#step-1-set-up-your-clusters)
* [Step 2: Install Skupper on your clusters](#step-2-install-skupper-on-your-clusters)
* [Step 3: Apply your YAML resources](#step-3-apply-your-yaml-resources)
* [Step 4: Link your sites](#step-4-link-your-sites)
* [Step 5: Access the frontend](#step-5-access-the-frontend)
* [Cleaning up](#cleaning-up)
* [Next steps](#next-steps)
* [About this example](#about-this-example)

## Overview

This example is a variant of [Skupper Hello World][hello-world] that
is deployed using YAML resource definitions instead of imperative
commands.

It contains two services:

* A backend service that exposes an `/api/hello` endpoint.  It
  returns greetings of the form `Hi, <your-name>.  I am <my-name>
  (<pod-name>)`.

* A frontend service that sends greetings to the backend and
  fetches new greetings in response.

In this scenario, each service runs in a different Kubernetes
cluster.  The frontend runs in a namespace on cluster 1 called West,
and the backend runs in a namespace on cluster 2 called East.

<img src="images/entities.svg" width="640"/>

Skupper enables you to place the backend in one cluster and the
frontend in another and maintain connectivity between the two
services without exposing the backend to the public internet.

[hello-world]: https://github.com/skupperproject/skupper-example-hello-world

## Prerequisites

* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl])

* Access to at least one Kubernetes cluster, from [any provider you
  choose][kube-providers]

[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[kube-providers]: https://skupper.io/start/kubernetes.html

## Step 1: Set up your clusters

Skupper is designed for use with multiple Kubernetes clusters.
The `skupper` and `kubectl` commands use your
[kubeconfig][kubeconfig] and current context to select the cluster
and namespace where they operate.

[kubeconfig]: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

Your kubeconfig is stored in a file in your home directory.  The
`skupper` and `kubectl` commands use the `KUBECONFIG` environment
variable to locate it.

A single kubeconfig supports only one active context per user.
Since you will be using multiple contexts at once in this
exercise, you need to create distinct kubeconfigs.

For each namespace, open a new terminal window.  In each terminal,
set the `KUBECONFIG` environment variable to a different path and
log in to your cluster.  Then create the namespace you wish to use
and set the namespace on your current context.

**Note:** The login procedure varies by provider.  See the
documentation for yours:

* [Minikube](https://skupper.io/start/minikube.html#cluster-access)
* [Amazon Elastic Kubernetes Service (EKS)](https://skupper.io/start/eks.html#cluster-access)
* [Azure Kubernetes Service (AKS)](https://skupper.io/start/aks.html#cluster-access)
* [Google Kubernetes Engine (GKE)](https://skupper.io/start/gke.html#cluster-access)
* [IBM Kubernetes Service](https://skupper.io/start/ibmks.html#cluster-access)
* [OpenShift](https://skupper.io/start/openshift.html#cluster-access)

_**West:**_

~~~ shell
export KUBECONFIG=~/.kube/config-west
# Enter your provider-specific login command
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_**East:**_

~~~ shell
export KUBECONFIG=~/.kube/config-east
# Enter your provider-specific login command
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

## Step 2: Install Skupper on your clusters

Using Skupper on Kubernetes requires the installation of the
Skupper custom resource definitions (CRDs) and the Skupper
controller.

For each cluster, use `kubectl apply` with the Skupper
installation YAML to install the CRDs and controller.

_**West:**_

~~~ shell
kubectl apply -f https://skupper.io/v2/install.yaml
~~~

_**East:**_

~~~ shell
kubectl apply -f https://skupper.io/v2/install.yaml
~~~

## Step 3: Apply your YAML resources

To configure our example sites and service bindings, we are
using the following resources:

West:

* [site.yaml](west/site.yaml) - Skupper configuration for West
* [frontend.yaml](west/frontend.yaml) - The Hello World frontend

East:

* [site.yaml](east/site.yaml) - Skupper configuration for East
* [backend.yaml](east/backend.yaml) - The Hello World backend

Let's look at some of these resources in more detail.

#### Resources in West

The `site` ConfigMap defines a Skupper site for its associated
Kubernetes namespace.  This is where you set site configuration
options.  See the [config reference][config] for more
information.

[config]: https://skupper.io/docs/yaml/index.html

[site.yaml](west/site.yaml):

~~~ yaml
apiVersion: skupper.io/v1alpha1
kind: Site
metadata:
  name: west
  namespace: west
spec:
  linkAccess: default
~~~

#### Resources in East

Like the one for West, here is the Skupper site definition for
the East.  It includes the `ingress: none` setting since no
ingress is required at this site for the Hello World example.

[site.yaml](east/site.yaml):

~~~ yaml
apiVersion: skupper.io/v1alpha1
kind: Site
metadata:
  name: east
  namespace: east
~~~

In East, the `backend` deployment has an annotation named
`skupper.io/proxy` with the value `tcp`.  This tells Skupper to
expose the backend on the Skupper network.  As a consequence,
the frontend in West will be able to see the backend and call
its API.

[backend.yaml](east/backend.yaml):

<pre>apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
  <b>annotations:
    skupper.io/proxy: tcp</b>
spec:
  selector:
    matchLabels:
      app: backend
  replicas: 3
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: quay.io/skupper/hello-world-backend
          ports:
            - containerPort: 8080</pre>

Now we're ready to apply everything.  Use the `kubectl apply`
command with the resource definitions for each site.

**Note:** If you are using Minikube, [you need to start
`minikube tunnel`][minikube-tunnel] before you create the
Skupper sites.

[minikube-tunnel]: https://skupper.io/start/minikube.html#running-minikube-tunnel

_**West:**_

~~~ shell
kubectl apply -f west/site.yaml -f west/frontend.yaml -f west/listener.yaml
kubectl wait site/west --for condition=Ready --timeout 120s
~~~

_Sample output:_

~~~ console
$ kubectl apply -f west/site.yaml -f west/frontend.yaml -f west/listener.yaml
XXX
configmap/skupper-site created
deployment.apps/frontend created
~~~

_**East:**_

~~~ shell
kubectl apply -f east/site.yaml -f east/backend.yaml -f east/connector.yaml
kubectl wait site/east --for condition=Ready --timeout 120s
~~~

_Sample output:_

~~~ console
$ kubectl apply -f east/site.yaml -f east/backend.yaml -f east/connector.yaml
XXX
configmap/skupper-site created
deployment.apps/backend created
~~~

## Step 4: Link your sites

You can configure sites and service bindings declaratively, but
linking sites is different.  To create a link, you must have the
authentication secret and connection details of the remote site.
Since these cannot be known in advance, linking must be
procedural.

<!--
**Note:** There are several ways to automate the generation and
distribution of tokens across sites, using for example Ansible,
Backstage, or Vault.  See [Token distribution][XXX] for more
information.
-->

This example uses the Skupper command-line tool to generate the
secret token in West and create the link in East.

To install the Skupper command:

~~~ shell
curl https://skupper.io/install.sh | sh -s -- --version 2.0.0-preview-1
~~~

For more installation options, see [Installing
Skupper][install].

Once the command is installed, use `skupper token create` in
West to generate the token.  Then, use `skupper link create` in
East to create a link.

[install]: https://skupper.io/install/index.html

_**West:**_

~~~ shell
skupper token issue ~/token.yaml
~~~

_Sample output:_

~~~ console
$ skupper token issue ~/token.yaml
Waiting for token status ...

Grant "west-cad4f72d-2917-49b9-ab66-cdaca4d6cf9c" is ready
Token file /run/user/1000/skewer/token.yaml created

Transfer this file to a remote site. At the remote site,
create a link to this site using the "skupper token redeem" command:

	skupper token redeem <file>

The token expires after 1 use(s) or after 15m0s.
~~~

_**East:**_

~~~ shell
skupper token redeem ~/token.yaml
~~~

_Sample output:_

~~~ console
$ skupper token redeem ~/token.yaml
Waiting for token status ...
Token "west-cad4f72d-2917-49b9-ab66-cdaca4d6cf9c" has been redeemed
You can now safely delete /run/user/1000/skewer/token.yaml
~~~

If your terminal sessions are on different machines, you may need
to use `scp` or a similar tool to transfer the token securely.  By
default, tokens expire after a single use or 15 minutes after
creation.

## Step 5: Access the frontend

In order to use and test the application, we need external access
to the frontend.

Use `kubectl port-forward` to make the frontend available at
`localhost:8080`.

_**West:**_

~~~ shell
kubectl port-forward deployment/frontend 8080:8080
~~~

You can now access the web interface by navigating to
[http://localhost:8080](http://localhost:8080) in your browser.

## Cleaning up

To remove Skupper and the other resources from this exercise, use
the following commands.

_**West:**_

~~~ shell
kubectl delete -f west/site.yaml -f west/frontend.yaml -f west/listener.yaml
kubectl delete -f https://skupper.io/v2/install.yaml
~~~

_**East:**_

~~~ shell
kubectl delete -f east/site.yaml -f east/backend.yaml -f east/connector.yaml
kubectl delete -f https://skupper.io/v2/install.yaml
~~~

## Next steps

Check out the other [examples][examples] on the Skupper website.

## About this example

This example was produced using [Skewer][skewer], a library for
documenting and testing Skupper examples.

[skewer]: https://github.com/skupperproject/skewer

Skewer provides utility functions for generating the README and
running the example steps.  Use the `./plano` command in the project
root to see what is available.

To quickly stand up the example using Minikube, try the `./plano demo`
command.
