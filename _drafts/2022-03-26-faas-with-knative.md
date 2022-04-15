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
