---
layout: post
title: Queue based Autoscaling in Kubernetes on AWS
date: 2017-02-25 20:17:00 +0800
---
Kubernetes offers CPU based autoscaling out of the hood (see [HPA][kubernetes-hpa] for more details). Like many systems, CPU usage is not always the best indicator for scaling since many systems rely on other resources such as networking and memory. If you're running Kubernetes on AWS and use SQS as a messaging queue, you'll quickly run into this problem. This is because CPU usage does not always correlate to messages processed.

Consider a consumer that does a set of transcations to some data store for every message. The cost of each transcation will vary depending on the message. If you're consumer reaches some networking limits or your data store can only handle X amount of sequential transactions per consumer, adding more consumers can help scale the rate of messages consumed. Nevertheless, no matter how many messages are backed up, CPU usage per consumer will rarely increase in some of the cases listed above. One alternative that has worked out fairly well at Wattpad is to autoscale consumers based on the size of the queue. By putting hard upper and lower limits on the queue size, you can add and remove pods depending on your [SLOs][slo].

For every SQS queue, you can run a side deployment to autoscale the replica count of the deployment consuming messages from SQS.
```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-sqs-autoscaler
  labels:
    app: kube-sqs-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-sqs-autoscaler
  template:
    metadata:
      labels:
        app: kube-sqs-autoscaler
    spec:
      containers:
      - name: kube-sqs-autoscaler
        image: wattpad/kube-sqs-autoscaler:v1.0
        command:
          - /kube-sqs-autoscaler
          - --sqs-queue-url=https://sqs.your_aws_region.amazonaws.com/your_aws_account_number/your_queue_name  # required
          - --kubernetes-deployment=your-kubernetes-deployment-name # required
          - --kubernetes-namespace=$(POD_NAMESPACE) # optional
          - --aws-region=us-west-1  #required
          - --poll-period=5s # optional
          - --scale-down-cool-down=30s # optional
          - --scale-up-cool-down=5m # optional
          - --scale-up-messages=100 # optional
          - --scale-down-messages=10 # optional
          - --max-pods=5 # optional
          - --min-pods=1 # optional
        env:
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        resources:
          requests:
            memory: "200Mi"
            cpu: "50m"
          limits:
            memory: "200Mi"
            cpu: "100m"
        volumeMounts:
          - name: ssl-certs
            mountPath: /etc/ssl/certs/ca-certificates.crt
            readOnly: true
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-certificates.crt"
```


[kubernetes-hpa]: https://kubernetes.io/docs/user-guide/horizontal-pod-autoscaling/
[slo]: https://en.wikipedia.org/wiki/Service_level_objective
