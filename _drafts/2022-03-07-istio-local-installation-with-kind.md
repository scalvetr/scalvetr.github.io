---
layout: post
title:  "Istio local installation with KinD"
date:   2022-03-07 20:56:00 +0100
categories: kubernetes istio kind
---

## Introduction

Istio is a service mesh that runs on top of Kubernetes. It offers a lot of handy features to work with microservices.
for more information you can check the [official istio page](https://istio.io/latest/) out.

For your local environment you'll need a kubernetes cluster, one of the most popular is Kind. According to the 
[official kind page](https://kind.sigs.k8s.io/)
```
kind is a tool for running local Kubernetes clusters using Docker container "nodes".
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.
```


