---
layout: post
title:  "Knative Scale to Zero"
date:   2022-03-26 20:56:00 +0100
categories: knative faas lambda
---

# FaaS

Some of the more heard buzzwords in the tech industry last years are "serverless" and "FaaS". In fact "FaaS" - which 
stands for - Function as a Service is the serverless way to deploy your own code.

It was initially popularized by Amazon back in 2014 with [AWS Lambda](https://aws.amazon.com/lambda/). Nowadays almost 
all cloud providers offers a FaaS solution [Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/),
[Google Cloud Functions](https://cloud.google.com/functions). They will provably be covered by future reviews.

Although the cloud vendor offerings are the most popular entry point to the FaaS parading, there is another approach that
also worth considering. The serverless solutions on top of Kubernetes. The two most populars in that field are OpenFaaS and Knative.