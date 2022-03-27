---
layout: post
title:  "Istio local installation with kind"
date:   2022-03-07 20:56:00 +0100
categories: kubernetes istio kind
---

## Introduction

**Istio** is a service mesh that runs on top of Kubernetes. It offers a lot of handy features to work with microservices.
for more information you can check the [official Istio page](https://istio.io/latest/) out.

A popular choice to run a local kubernetes cluster is Kind. According to the 
[official kind page](https://kind.sigs.k8s.io/)

>kind is a tool for running local Kubernetes clusters using Docker container "nodes".
>kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

This two tools works nicely once set up, however it is a little tricky to set up a working environment.

Kind installation is really straight forward, you first install docker, and then you can just follow the quick start guide.
I suggest [installing it with a package manager](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-with-a-package-manager).

## Create the cluster

Kind allows you to specify a few options when you create the cluster, you can define the number of nodes, their subnets,
th ports exposed, the kubernetes version... find [here](https://kind.sigs.k8s.io/docs/user/configuration/#clusterwide-options) 
a complete list of them. 

**extraPortMappings**

For a local testing environment you usually can stick to the defaults, the only important setting you usually want to define
is the `extraPortMappings`. Those are ports listening in the host machine that are forwarded to kind nodes.  

A common use case for this setting is expose a database service or your event broker to be accessed from outside the cluster.
Another use case for this is to set up an Ingress Controller to route the http traffic from your host machine to the cluster, 
this is covered by [Ingress](https://kind.sigs.k8s.io/docs/user/ingress).

In case of Istio, it provides its own ingress gateway as a [Service](https://kubernetes.io/docs/concepts/services-networking/service/). 
This service can be set up as `type=NodePort` and assign to a node port in the range [30000-32768].

Let's assume we want to expose `http` traffic in the port `80` and `tls` traffic in the port `433` of the host machine, 
we can use the node ports `30080` and `30433`

To define such a cluster, let's name it "istio-playground", the command would be as follows.

```shell
cat <<EOF | kind create cluster --name istio-playground --wait 5m --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: TCP
  - containerPort: 30443
    hostPort: 443
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: TCP
EOF
```

Expected result:

![cluster created](/assets/img/2022-03-07-istio-local-installation-with-kind/cluster-created.png)

Verify the cluster

````shell
kind get clusters
kubectl config use-context kind-istio-playground
kubectl cluster-info
````

## Gateway deployment topologies

There are several topologies you can choose to deploy the Istio ingress gateway. You can basically share a single centralized
gateway (Shared Gateway) or configure a dedicated gateway per application (Dedicated application gateway). You can find 
more information [here](https://istio.io/latest/docs/setup/additional-setup/gateway/#gateway-deployment-topologies).

For the sake of simplicity we'll choose the "Shared gateway". Let's imagine we'll deploy a workload called `demo-app` to
demonstrate the setup works fine. The following namespaces are needed:
* `istio-system`: Is the namespace where Istio components are deployed.
* `istio-ingress`: Is where the Istio ingress gateway is deployed.
* `demo-app`: Is where the application "demo-app" is deployed.

![k8s-namespaces](/assets/img/2022-03-07-istio-local-installation-with-kind/k8s-namespaces.png)


## Installation method

There are several options to install Istio, provably the ones that best fits to a local setup are: Istio operator, 
Istioctl and Helm chart.

For this manual we are using **Helm**, it is the one that allows us to install different Istio versions without having to 
deal with different installations of `istioctl`. However I think that `istioctl` is also an excelent option for a local
setup as you can see in the [Getting Started Guide](https://istio.io/latest/docs/setup/getting-started/#download).

## Install Istio

Let's first start creating the namespaces:

```shell
kubectl create namespace istio-system
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
```

---
Notice that the `istio-injection` setting has been enabled for the istio-ingress namespace, this is the recommended set up
as it gives developers full control over the gateway deployment, while also simplifying operations. When a new upgrade
is available, or a configuration has changed, gateway pods can be updated by simply restarting them. This makes the
experience of operating a gateway deployment the same as operating sidecars.

---

Now let's install the `istio-base`, `istiod` and `istio-ingress` packages.

```shell
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm install istio-base istio/base --version 1.13.2 -n istio-system
helm install istiod istio/istiod --version 1.13.2 -n istio-system --wait
helm install istio-ingress istio/gateway --version 1.13.2 -n istio-ingress --wait --set service.type=NodePort
```

Notice that the service type for `istio-ingress` has been changed to `NodePort`. The default setting for this is 
`LoadBalancer`, which is totally fine in most platforms, however in `kind` it is really tricky to make it work, specially on
Windows and MacOS. You can read more on this [here](https://kind.sigs.k8s.io/docs/user/loadbalancer/).

By configuring the service as `NodePort` we can then easily forward these node ports to the ones configured earlier in the
cluster extraPortMappings.

So let's first check out the ports exposed by this service:

```shell
kubectl get services -n istio-ingress istio-ingress -o json | jq -r '.spec.ports'
```
The result will be like this:
```json
[
  {
    "name": "status-port",
    "nodePort": 30513,
    "port": 15021,
    "protocol": "TCP",
    "targetPort": 15021
  },
  {
    "name": "http2",
    "nodePort": 32276,
    "port": 80,
    "protocol": "TCP",
    "targetPort": 80
  },
  {
    "name": "https",
    "nodePort": 30792,
    "port": 443,
    "protocol": "TCP",
    "targetPort": 443
  }
]
```

Let's now modify the node ports to match the ones configured earlier in the cluster extraPortMappings: `30080` and `30443`.

```shell
kubectl -n istio-ingress patch svc istio-ingress --patch \
  '{"spec": { "ports": [ { "port": 80, "nodePort": 30080 }, { "port": 443, "nodePort": 30443 } ] } }'
# service/istio-ingress patched
```

## Test the setup

To test the setup we'll deploy a `Gateway` in the `istio-ingress` namespace:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  namespace: istio-ingress
  name: demo-app-gateway
spec:
  selector:
    app: istio-ingress
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

And create a `demo-app` namespace and deploy there the `demo-app` `Service`, `Deployment` and `VirtualService`.

See the code [here](https://raw.githubusercontent.com/scalvetr/istio-playground/main/istio-test.yaml)

```shell
# deploy and expose sample service
kubectl create namespace demo-app
kubectl label namespace demo-app istio-injection=enabled

kubectl apply -f https://raw.githubusercontent.com/scalvetr/istio-playground/main/istio-test.yaml

curl -vv "http://localhost/demo-app"
# HTTP/1.1 200 OK
```