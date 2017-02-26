---
layout: post
title: Queue based Autoscaling in Kubernetes on AWS
date: 2017-02-25 20:17:00 +0800
---
Kubernetes offers CPU based autoscaling out of the hood (see [HPA][kubernetes-hpa] for more details). Like many systems, CPU usage is not always the best indicator for autoscaling since many systems rely on other resources such as networking and memory. If you're running Kubernetes on AWS and use SQS as a messaging queue, you'll quickly run into this problem. This is because CPU usage does not always correlate to messages processed. It's possible for larger messages to require larger network throughput resulting in

[kubernetes-hpa]: https://kubernetes.io/docs/user-guide/horizontal-pod-autoscaling/
