---
layout: post
title:  "FaaS with Knative"
date:   2022-03-26 20:56:00 +0100
categories: knative faas lambda
---

# FaaS and Kubernetes

Some of the more heard buzzwords in the tech industry last years are "serverless" and "FaaS". In fact "FaaS" - which 
stands for - Function as a Service is the serverless way to deploy your own code.

It was initially popularized by Amazon back in 2014 with [AWS Lambda](https://aws.amazon.com/lambda/). Nowadays almost 
all cloud providers offers a FaaS solution [Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/),
[Google Cloud Functions](https://cloud.google.com/functions). They will provably be covered by future posts.

Although the cloud vendor offerings are the most popular entry point to the FaaS paradigm , there is another approach 
that also worth considering: The serverless solutions on top of Kubernetes.

Cloud vendor offerings leverage on the manged services available in their platform for load balancing, auto-scaling, 
self-healing, configuration, observability... The same way the most popular cloud-agnostic solutions leverage on 
Kubernetes for all this core functionality. By using kubernetes they already have half the way done.

The two most populars FaaS solutions running on kubernetes are [OpenFaaS](https://www.openfaas.com/) and 
[Knative](https://knative.dev/docs/).

# Introduction to Knative

[Knative](https://knative.dev/docs/) is an Open-Source Enterprise-level solution to build Serverless and Event Driven 
Applications. Knative emerged in 2018, led by Google, but it has a big industry support mainly from IBM, Red Hat and 
VMware. There's a managed version in Google Cloud Platform (Cloud Run) also Red Hat offer a product based on Knative 
that runs on their platform: OpenShift Serverless.

Knative has two main components: Serving and Eventing.

## Knative Serving

[Knative Serving](https://knative.dev/docs/serving/) offers rapid deployment of serverless functions with built-in 
autoscaling functionality (including scale to zero which is a signature functionality of FaaS).
It supports both HTTP and HTTPS networking protocols.

It offers a set of abstractions that helps to deploy and control the full lifecycle of a function.

*Service:* Represents your function and manages all others derived resources attached to it.
*Configuration:* Maintains the desired state for your deployment. It provides a clean separation between code and 
configuration. Modifying a configuration creates a new revision.
*Revision:* It is a point-in-time snapshot of the code and configuration. Revisions are immutable objects and can be 
retained for as long as useful.
*Route:* Maps a network endpoint to one or more revisions.

![Object Model](/2022-03-26-faas-with-knative/knative_serving_object_model.png)


## Knative Eventing

[Knative Eventing](https://knative.dev/docs/eventing/) provides tools to produce and consume events. Those, are loosely 
coupled and can be developed and deployed independently of each other. This means, there can be producers and no 
consumers listening and the other way around.

The message format follows the [CloudEvents](https://github.com/cloudevents/spec/blob/v1.0.1/primer.md) specification, 
which allows adding optional and required attributes along with the payload which can be in any format (json, plain text 
...)

The message is produced and consumed as a standard HTTP POST, this means no additional library is needed, and it is totally
decoupled from the underlying technology.

Knative offers a [Broker](https://knative.dev/docs/eventing/broker/) abstraction. Brokers provide a discoverable endpoint, 
for event ingress, and triggers for event delivery.


![Object Model](/2022-03-26-faas-with-knative/knative_eventing_broker_workflow.svg)

# Install Knative locally

*Istio*

For this installation we'll consider we already have a cluster with [Istio](https://istio.io/) installed. Istio is one 
of the options available as a networking layer for Knative (along with Courier and Kourier). By using Istio you can 
leverage Istio to secure, observe, and expose services.

*Knative Operator*

There are several methods to install: Knative cli, yaml configuration or a Knative Kubernetes Operator. For this 
example the chosen installation method is the Kubernetes operator, it allows installing, configuring and managing 
Knative, without the need of any additional tool, just with `kubectl`. More info 
[here](https://knative.dev/docs/install/operator/knative-with-operators/)

*Knative Serving*

As explained earlier, there are two main components in Knative. Most of the features related with FaaS are provided by
the *Knative Serving* component, so this will be the one installed.


Assuming Istio is already in place (you can find instructions for a local installation 
[here](https://github.com/scalvetr/istio-playground/blob/main/README.md)), we're going to follow the following guide: 
[Knative with operators](https://knative.dev/docs/install/operator/knative-with-operators/)

```shell
kubectl apply -f https://github.com/knative/operator/releases/download/knative-v1.3.2/operator.yaml
# Verify your Knative Operator installation
kubectl config set-context --current --namespace=default
# Check the Operator deployment status by running the command
kubectl get deployment knative-operator
```

Create the Knative Serving custom resource
```shell
export ISTIO_INGRESS_NAMESPACE="istio-ingress";
export KNATIVE_SERVING_NAMESPACE="knative-serving";

# get the istio-ingress labels 
kubectl get svc -n ${ISTIO_INGRESS_NAMESPACE} istio-ingress -ojsonpath='{.metadata.labels}'

# Create the serving namespace and custom resource
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${KNATIVE_SERVING_NAMESPACE}
---
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: ${KNATIVE_SERVING_NAMESPACE}
spec:
  ingress:
    istio:
      enabled: true
      knative-ingress-gateway:
        selector:
          istio: ingress
  config:
    istio:
      gateway.knative-serving.knative-ingress-gateway: "istio-ingress.${ISTIO_INGRESS_NAMESPACE}.svc.cluster.local"
EOF
```

For local environments a handy option for DNS is to use a magic DNS
```shell
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"127.0.0.1.nip.io":""}}'
  
kubectl get configmap/config-domain \
  --namespace knative-serving
```

Now, Knative serving component has been installed!! Let's move forward to demonstrate a simple usage.

# Deploy an echo function

For the sake of demonstrate how easy is to deploy and operate a function, let's start deploying a simple 
[http echo service](https://hub.docker.com/r/mendhak/http-https-echo).

```shell
kubectl label namespace default istio-injection=enabled

cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: mendhak/http-https-echo
          ports:
            - containerPort: 80
          env:
              - name: HTTP_PORT
                value: "80"
EOF
# service.serving.knative.dev/helloworld created
```

To verify the installation
```shell
kubectl get pods
kubectl get ksvc
kubectl describe revision helloworld-00001
```

Now let's test the service
```shell
curl -vv http://helloworld.default.127.0.0.1.nip.io
```

The expected result:

![curl output](/2022-03-26-faas-with-knative/knative_serving_test_request.png)

# Scale to zero

Now it's very easy to test the scale to zero functionality. As we just did a request, we might expect a pod up and running, 
ready to server more requests, this is precisely the one just served the request we just did.

![curl output](/2022-03-26-faas-with-knative/knative_serving_scale_to_zero_up.png)

After a few seconds pod enters the terminating state. This what we'd see in case of our `hello-world` revision `00001` 
function.

![curl output](/2022-03-26-faas-with-knative/knative_serving_scale_to_zero_terminating.png)

... and finally the function is scaled to zero.

![curl output](/2022-03-26-faas-with-knative/knative_serving_scale_to_zero_down.png)

we can easily check the deployment is set to `0` replicas.

```shell
kubectl get deployment helloworld-00001-deployment -o jsonpath='{.spec.replicas}'
#0% 
```

![curl output](/2022-03-26-faas-with-knative/knative_serving_scale_to_zero_replicas.png)
