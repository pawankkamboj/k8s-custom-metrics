# k8s-custom-metrics - autoscaling on custom metrics

As of Kubernetes 1.7, custom metrics requires enabling the aggregation layer on the API server and configuring the controller manager to use the metrics APIs via their REST clients. 
The prometheus adapter gathers the names of available metrics from Prometheus a regular interval and then only exposes metrics. 
you can get more info at https://github.com/DirectXMan12/k8s-prometheus-adapter

* Enable the aggregation layer via the following kube-apiserver flags.
```
--requestheader-client-ca-file=<path to aggregator CA cert>
--requestheader-allowed-names=aggregator
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--proxy-client-cert-file=<path to aggregator proxy cert>
--proxy-client-key-file=<path to aggregator proxy key>
```

* Deploy prometheus using promethues operator
  The Prometheus Operator for Kubernetes provides easy monitoring definitions for Kubernetes services and deployment and    management of Prometheus instances, check out more https://github.com/coreos/prometheus-operator
```
[root@docker01 prometheus-operator]# kubectl apply -f alertmanager/
[root@docker01 prometheus-operator]# kubectl get pods -l app=alertmanager
NAME                  READY     STATUS    RESTARTS   AGE
alertmanager-main-0   2/2       Running   0          23h

[root@docker01 prometheus-operator]# kubectl apply -f prometheus-operator/

[root@docker01 prometheus-operator]# kubectl get pods -l app=prometheus
NAME               READY     STATUS    RESTARTS   AGE
prometheus-k8s-0   2/2       Running   4          21h

check status of CustomResourceDefinitions

[root@docker01 prometheus-operator]# kubectl get customresourcedefinitions
NAME                                    KIND
alertmanagers.monitoring.coreos.com     CustomResourceDefinition.v1beta1.apiextensions.k8s.io
prometheuses.monitoring.coreos.com      CustomResourceDefinition.v1beta1.apiextensions.k8s.io
servicemonitors.monitoring.coreos.com   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
```

* Deploy node-expoter
```
[root@docker01 prometheus-operator]# kubectl apply -f node-exporter/
[root@docker01 prometheus-operator]# kubectl get pods -n monitoring -l app=node-exporter
NAME                  READY     STATUS    RESTARTS   AGE
node-exporter-682cc   1/1       Running   0          1d
node-exporter-z9fz6   1/1       Running   0          1d
```

* Deploy kube-state-metrics
```
[root@docker01 prometheus-operator]# kubectl apply -f kube-state-metrics/

[root@docker01 prometheus-operator]# kubectl get pods -n monitoring -l app=kube-state-metrics
NAME                                             READY     STATUS    RESTARTS   AGE
kube-state-metrics-deployment-7f758c9f4b-f28b8   1/1       Running   0          1d
```

* Deploy metrics servers

  Starting from Kubernetes 1.8, resource usage metrics, such as container CPU and memory usage, are available in Kubernetes   through the Metrics API. Through the Metrics API you can get the amount of resource currently used by a given node or a given pod. This API doesn’t store the metric values, so it’s not possible for example to get the amount of resources used by a given node 10 minutes ago. it is discoverable through the same endpoint as the other Kubernetes APIs under /apis/metrics.k8s.io/ path. Metrics Server https://github.com/kubernetes-incubator/metrics-server is a cluster-wide aggregator of resource usage data. Metric server collects metrics from the Summary API, exposed by Kubelet on each node. Metrics Server registered in the main API server through Kubernetes aggregator, which was introduced in Kubernetes 1.7 and It’s supported in Kubernetes 1.7+. 
```
[root@docker01 prometheus-operator]# kubectl apply -f metrics-server/

[root@docker01 prometheus-operator]# kubectl get pods -n kube-system -l k8s-app=metrics-server
NAME                             READY     STATUS    RESTARTS   AGE
metrics-server-cb4b857b9-zglxl   1/1       Running   0          1d

[root@docker01 prometheus-operator]# kubectl get --raw "/apis/metrics.k8s.io/v1beta1/"
```

* Deploy k8s-prometheus-adapter for custom metrics
```
[root@docker01 prometheus-operator]# kubectl apply -f prometheus
[root@docker01 prometheus-operator]# kubectl get pods -l k8s-app=prometheus-operator
NAME                                   READY     STATUS    RESTARTS   AGE
prometheus-operator-549699ccc6-5kpzn   1/1       Running   0          1d

[root@docker01 prometheus-operator]# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/"

```

* Example application nginx
```
kubectl apply -f example/

[root@docker01 prometheus-operator]# kubectl get pods -l app=nginx
NAME                     READY     STATUS    RESTARTS   AGE
nginx-76f467f844-64wsf   1/1       Running   0          17m

[root@docker01 prometheus-operator]# kubectl get hpa nginx
NAME      REFERENCE          TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
nginx     Deployment/nginx   7766016 / 800k   1         2         1          13m

[root@docker01 prometheus-operator]# kubectl get hpa nginx -o yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  annotations:
    autoscaling.alpha.kubernetes.io/conditions: '[{"type":"AbleToScale","status":"False","lastTransitionTime":"2018-01-31T05:20:26Z","reason":"BackoffBoth","message":"the
      time since the previous scale is still within both the downscale and upscale
      forbidden windows"},{"type":"ScalingActive","status":"True","lastTransitionTime":"2018-01-31T05:05:56Z","reason":"ValidMetricFound","message":"the
      HPA was able to succesfully calculate a replica count from pods metric memory_usage_bytes"},{"type":"ScalingLimited","status":"True","lastTransitionTime":"2018-01-31T05:18:56Z","reason":"TooManyReplicas","message":"the
      desired replica count is more than the maximum replica count"}]'
    autoscaling.alpha.kubernetes.io/current-metrics: '[{"type":"Pods","pods":{"metricName":"memory_usage_bytes","currentAverageValue":"7741440"}}]'
    autoscaling.alpha.kubernetes.io/metrics: '[{"type":"Pods","pods":{"metricName":"memory_usage_bytes","targetAverageValue":"800k"}}]'
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"autoscaling/v2beta1","kind":"HorizontalPodAutoscaler","metadata":{"annotations":{},"name":"nginx","namespace":"default"},"spec":{"maxReplicas":2,"metrics":[{"pods":{"metricName":"memory_usage_bytes","targetAverageValue":800000},"type":"Pods"}],"minReplicas":1,"scaleTargetRef":{"apiVersion":"apps/v1beta1","kind":"Deployment","name":"nginx"}}}
  creationTimestamp: 2018-01-31T05:05:31Z
  name: nginx
  namespace: default
  resourceVersion: "205330"
  selfLink: /apis/autoscaling/v1/namespaces/default/horizontalpodautoscalers/nginx
  uid: 5cacfebe-0644-11e8-80f9-d8d385582ea0
spec:
  maxReplicas: 2
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: nginx
status:
  currentReplicas: 2
  desiredReplicas: 2
  lastScaleTime: 2018-01-31T05:18:56Z
```
