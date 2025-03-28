<!-- NOTE: This file is generated from skewer.yaml.  Do not edit it directly. -->

# Skupper Hello World using YAML

[![main](https://github.com/skupperproject/skupper-example-yaml/actions/workflows/main.yaml/badge.svg)](https://github.com/skupperproject/skupper-example-yaml/actions/workflows/main.yaml)

#### A minimal HTTP application deployed across Kubernetes clusters using Skupper

This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html

#### Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Step 1: Access your Kubernetes clusters](#step-1-access-your-kubernetes-clusters)
* [Step 2: Create your Kubernetes namespaces](#step-2-create-your-kubernetes-namespaces)
* [Step 3: Install Skupper on your Kubernetes clusters](#step-3-install-skupper-on-your-kubernetes-clusters)
* [Step 4: Apply your YAML resources](#step-4-apply-your-yaml-resources)
* [Step 5: Link your sites](#step-5-link-your-sites)
* [Step 6: Access the frontend service](#step-6-access-the-frontend-service)
* [Cleaning up](#cleaning-up)
* [Next steps](#next-steps)
* [About this example](#about-this-example)

## Overview

This example is a variant of [Skupper Hello World][hello-world] that
is deployed using YAML custom resources instead of imperative
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

* Access to at least one Kubernetes cluster, from [any provider you
  choose][kube-providers].

* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl]).

[kube-providers]: https://skupper.io/start/kubernetes.html
[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/

## Step 1: Access your Kubernetes clusters

Skupper is designed for use with multiple Kubernetes clusters.
The `skupper` and `kubectl` commands use your
[kubeconfig][kubeconfig] and current context to select the cluster
and namespace where they operate.

[kubeconfig]: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

This example uses multiple cluster contexts at once. The
`KUBECONFIG` environment variable tells `skupper` and `kubectl`
which kubeconfig to use.

For each cluster, open a new terminal window.  In each terminal,
set the `KUBECONFIG` environment variable to a different path and
log in to your cluster.

_**West:**_

~~~ shell
export KUBECONFIG=~/.kube/config-west
<provider-specific login command>
~~~

_**East:**_

~~~ shell
export KUBECONFIG=~/.kube/config-east
<provider-specific login command>
~~~

**Note:** The login procedure varies by provider.

## Step 2: Create your Kubernetes namespaces

The example application has different components deployed to
different Kubernetes namespaces.  To set up our example, we need
to create the namespaces.

For each cluster, use `kubectl create namespace` and `kubectl
config set-context` to create the namespace you wish to use and
set the namespace on your current context.

_**West:**_

~~~ shell
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_**East:**_

~~~ shell
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

## Step 3: Install Skupper on your Kubernetes clusters

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

## Step 4: Apply your YAML resources

To configure our example sites and service bindings, we are
using the following resources:

West:

* [site.yaml](west/site.yaml) - The Skupper _Site_ resource for
  West
* [frontend.yaml](west/frontend.yaml) - The Hello World frontend
  deployment
* [listener.yaml](west/listener.yaml) - A Skupper _Listener_
  resource for exposing the backend in East to the local
  frontend

East:

* [site.yaml](east/site.yaml) - The Skupper _Site_ resource for
  East
* [backend.yaml](east/backend.yaml) - The Hello World backend
  deployment
* [connector.yaml](east/connector.yaml) - A Skupper _Connector_
  resource for binding the backend to the listener in West

Let's look at these resources in more detail.

### Resources in West

The _Site_ resource defines a Skupper site for its associated
Kubernetes namespace.  This is where you set site configuration
options.  See the [Site resource reference][site-config] for
more information.

The `linkAccess: default` field configures site West to accept
site-to-site links.  This example creates a link from East to
West, so the receiving side, West, must enable link access.

[site-config]: https://skupperproject.github.io/refdog/resources/site.html

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

The frontend is a standard Kubernetes deployment.

[frontend.yaml](west/frontend.yaml):

~~~ yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  selector:
    matchLabels:
      app: frontend
  replicas: 1
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: quay.io/skupper/hello-world-frontend
          ports:
            - containerPort: 8080
~~~

The code in frontend makes API calls to
`http://backend:8080/api/hello`.  The _Listener_ resource below
configures the router to expose a connection endpoint at
`backend:8080` and forward any connections there to routers in
remote sites using the routing key `backend`.  See the [Listener
resource reference][listener-config] for more information.

[listener-config]: https://skupperproject.github.io/refdog/resources/listener.html

[listener.yaml](west/listener.yaml):

~~~ yaml
apiVersion: skupper.io/v1alpha1
kind: Listener
metadata:
  name: backend
  namespace: west
spec:
  port: 8080
  host: backend
  routingKey: backend
~~~

### Resources in East

The _Site_ resource for East.

[site.yaml](east/site.yaml):

~~~ yaml
apiVersion: skupper.io/v1alpha1
kind: Site
metadata:
  name: east
  namespace: east
~~~

The backend is a standard Kubernetes deployment.

[backend.yaml](east/backend.yaml):

~~~ yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
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
            - containerPort: 8080
~~~

The _Connector_ resource below configures the router to take
remote connections with routing key `backend` and forward them
to port 8080 on pods matching the selector `app=backend`.  See
the [Connector resource reference][connector-config] for more
information.

[connector-config]: https://skupperproject.github.io/refdog/resources/connector.html

[connector.yaml](east/connector.yaml):

~~~ yaml
apiVersion: skupper.io/v1alpha1
kind: Connector
metadata:
  name: backend
  namespace: east
spec:
  routingKey: backend
  port: 8080
  selector: app=backend
~~~

### Applying the resources

Now we're ready to apply everything.  Use the `kubectl apply`
command with the resources for each site.

**Note:** If you are using Minikube, [you need to start
`minikube tunnel`][minikube-tunnel] before you create the
Skupper sites.

[minikube-tunnel]: https://skupper.io/start/minikube.html#running-minikube-tunnel

_**West:**_

~~~ shell
kubectl apply -f west/site.yaml -f west/frontend.yaml -f west/listener.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f west/site.yaml -f west/frontend.yaml -f west/listener.yaml
site.skupper.io/west created
deployment.apps/frontend created
listener.skupper.io/backend created
~~~

_**East:**_

~~~ shell
kubectl apply -f east/site.yaml -f east/backend.yaml -f east/connector.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f east/site.yaml -f east/backend.yaml -f east/connector.yaml
site.skupper.io/east created
deployment.apps/backend created
connector.skupper.io/backend created
~~~

## Step 5: Link your sites

A Skupper _link_ is a channel for communication between two
sites.  Links serve as a transport for application connections
and requests.

You can configure sites and service bindings declaratively, but
linking sites is different.  To create a link, you must have the
authentication secret and connection details of the remote site.
Since these cannot always be known in advance, linking is
usually procedural, not declarative.

<!--
**Note:** There are several ways to automate the generation and
distribution of tokens across sites, using for example Ansible,
Backstage, or Vault.  See [Token distribution][XXX] for more
information.
-->

Skupper provides tokens as one way to securely create
site-to-site links.  This example uses the Skupper command-line
tool to issue the secret token in West and redeem the token for
a link in East.

To install the Skupper command:

~~~ shell
curl https://skupper.io/v2/install.sh | sh
~~~

For more installation options, see [Installing
Skupper][install].

Once the command is installed, use `skupper token issue` in West
to generate the token.  Then, use `skupper token redeem` in East
to create the link.

[install]: https://skupper.io/install/index.html

_**West:**_

~~~ shell
skupper token issue ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper token issue ~/secret.token
Waiting for token status ...

Grant "west-cad4f72d-2917-49b9-ab66-cdaca4d6cf9c" is ready
Token file /run/user/1000/skewer/secret.token created

Transfer this file to a remote site. At the remote site,
create a link to this site using the "skupper token redeem" command:

	skupper token redeem <file>

The token expires after 1 use(s) or after 15m0s.
~~~

_**East:**_

~~~ shell
skupper token redeem ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper token redeem ~/secret.token
Waiting for token status ...
Token "west-cad4f72d-2917-49b9-ab66-cdaca4d6cf9c" has been redeemed
You can now safely delete /run/user/1000/skewer/secret.token
~~~

If your terminal sessions are on different machines, you may need
to use `scp` or a similar tool to transfer the token securely.  By
default, tokens expire after a single use or 15 minutes after
being issued.

## Step 6: Access the frontend service

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
kubectl delete -f west/ --ignore-not-found
~~~

_**East:**_

~~~ shell
kubectl delete -f east/ --ignore-not-found
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
