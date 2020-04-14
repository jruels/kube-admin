# Horizontal Pod Autoscaling

In this lab you will: 

* Deploy a stateless application
* Configure a Horizontal Pod Autoscaling policy
* Trigger the HPA and confirm Pods are scaled to handle the load

Autoscaling is a common practice used to dynamically scale up or down resources based on the workload requirements.
In Kubernetes there are three types of autoscaling: Horizontal, Vertical and Cluster. 
The Horizontal Pod Autoscaler increases or decreases the number of Pods in a Deployment based on resource utilization. 
While the Cluster Autoscaler is highly dependent on the underlying capabilities of the cloud provider, the Horizontal and Vertical autoscalers 
are cloud agnostic.

Kubernetes supports custom metrics and allows 3rd party applications to extend the Kubernetes API by registering themselves as API add-ons.   
This Custom Metrics API along with the aggregation layer make it possible for monitoring systems like Prometheus to expose application-specific metrics which the HPA controller can use.

To access the custom metrics you need to install the Metrics Server. The demo application shows how pod autoscaling works 
based on CPU and memory and then in the second lab you will install Prometheus and a custom API server. After installation the API server needs to be registered
with the aggregator layer and the HPA needs to be configured to monitor the custom metrics supplied by the application.    

<!--

### Installing the Metrics-Server 

**NOTE: The metrics server is installed on GKE out of the box.**   

**DO NOT INSTALL ON GKE**
The metrics server collects CPU and memory usage for pods and nodes by querying data from the `kubernetes.summary_api`.    
The summary API is a minimal memory-efficient API used for passing data from Kubelet/cAdvisor to the metrics server.

Deploy the Metrics Server in the `kube-system` namespace:

```bash
cd ~/k8s-prom-hpa
kubectl create -f ./metrics-server
```

After about one minute the `metric-server` starts reporting CPU and memory usage for nodes and pods in the cluster.

-->
To view nodes metrics run:

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
```

To view pods metrics run:

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq .
```

### Auto Scaling based on CPU and memory usage

You will use a small Golang-based web app to test the Horizontal Pod Autoscaler (HPA).   

Deploy the demo app:   

```bash
kubectl create -f ./podinfo/podinfo-svc.yaml,./podinfo/podinfo-dep.yaml
```

To access the application we need to find the `EXTERNAL_IP`    
```
kubectl get svc podinfo
```

You should see something similiar to:    
```
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
podinfo   LoadBalancer   10.15.243.98   35.226.204.150   9898:30813/TCP   3h20m
```

Now in a browser load http://[EXTERNAL-IP]:9898   

Next create a new HPA that maintains a minimum of two replicas and max of ten. The replicas are dynamically
adjusted, if the average CPU is over 80% or if the memory goes above 200Mi:   

Review below
```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 200Mi
```

Create the new HPA:

```bash
kubectl create -f ./podinfo/podinfo-hpa.yaml
```

After a few seconds the HPA controller quries the metrics server and then fetches the CPU 
and memory usage:

```bash
kubectl get hpa

NAME      REFERENCE            TARGETS                      MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   2826240 / 200Mi, 15% / 80%   2         10        2          5m
```

In order to increase the CPU usage, install and run a load test with `rakyll/hey`:

```bash
go get -u github.com/rakyll/hey

#execute 10K requests
hey -n 10000 -q 10 -c 5 http://[EXTERNAL_IP]:9898/
```

You can monitor the HPA events by running:

```bash
$ kubectl describe hpa

Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  7m    horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  3m    horizontal-pod-autoscaler  New size: 8; reason: cpu resource utilization (percentage of request) above target
```

Delete `podinfo` for now. You will deploy it again later in this lab:

```bash
kubectl delete -f ./podinfo/podinfo-hpa.yaml,./podinfo/podinfo-dep.yaml,./podinfo/podinfo-svc.yaml
```

### Setting up a Custom Metrics Server 

Custom metrics autoscaling requires two components:   
A time series database [Prometheus](https://prometheus.io), that collects your applications metrics and stores them.   
The second component is a custom metrics adapter that extends the API with metrics supplied by Prometheus.   

You will deploy Prometheus and the custom metrics adapter into a new `monitoring` namespace

Create the namespace:

```bash
kubectl create -f ./namespaces.yaml
```

Deploy Prometheus in the new `monitoring` namespace:

```bash
kubectl create -f ./prometheus
```

The Prometheus adapter requires TLS certificates. To generate them run:

```bash
make certs
```

Now deploy the Prometheus custom metrics API adapter:

```bash
kubectl create -f ./custom-metrics-api
```

List the Prometheus provided custom metrics: 

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
```

Query the filesystem usage for all pods running in the `monitoring` namespace:

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/monitoring/pods/*/fs_usage_bytes" | jq .
```

### Auto Scaling based on custom metrics

Redeploy the demo application `podinfo`:

```bash
kubectl create -f ./podinfo/podinfo-svc.yaml,./podinfo/podinfo-dep.yaml
```

The application exposes a custom metric named `http_requests_total`.    
The Prometheus adapter marks the metric as a counter metric and removes the `_total` suffix.   

Query the total requests per second from the custom metrics API:   

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests" | jq .
```
```json
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/http_requests"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "podinfo-6b86c8ccc9-kv5g9",
        "apiVersion": "/__internal"
      },
      "metricName": "http_requests",
      "timestamp": "2018-01-10T16:49:07Z",
      "value": "901m"
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "podinfo-6b86c8ccc9-nm7bl",
        "apiVersion": "/__internal"
      },
      "metricName": "http_requests",
      "timestamp": "2018-01-10T16:49:07Z",
      "value": "898m"
    }
  ]
}
```

The `m` represents `milli-units`, so for example, `850m` means 850 milli-requests.   

Create a HPA that scales up the `podinfo` deployment if the number of requests goes above 10 per second:

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: http_requests
      targetAverageValue: 10
```

Deploy the `podinfo` HPA:

```bash
kubectl create -f ./podinfo/podinfo-hpa-custom.yaml
```

After a couple of seconds the HPA fetches the `http_requests` value from the metrics API:

```bash
kubectl get hpa

NAME      REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   899m / 10   2         10        2          1m
```

Time to increase the load by sending 25 requests per second:
```
#10K requests rate limited at 25 queries per second
hey -n 10000 -q 5 -c 5 http://[EXTERNAL-IP]:9898/healthz
```

After a few minutes check the HPA to confirm it is scaling up the deployment:

```
kubectl describe hpa
```

```
Name:                       podinfo
Namespace:                  default
Reference:                  Deployment/podinfo
Metrics:                    ( current / target )
  "http_requests" on pods:  9059m / 10
Min replicas:               2
Max replicas:               10

Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  2m    horizontal-pod-autoscaler  New size: 3; reason: pods metric http_requests above target
```

Three replicas are enough to keep the requests per second under 10 per pod, so the replicas will never scale above that.   
At the current rate of requests per second the deployment will never get to the max value of 10 pods.    

Either wait for the load test to complete or cancel it and confirm the HPA scales down the deployment to it's minimum replicas:
```
kubectl describe hpa
```

```
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  5m    horizontal-pod-autoscaler  New size: 3; reason: pods metric http_requests above target
  Normal  SuccessfulRescale  21s   horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
```

The autoscaler doesn't react immediately to usage spikes, it has a default check once every 30 seconds and 
scaling up/down will only take place if there was no autoscaling within the last 3-5 minutes. This is to prevent the HPA
from trying to execute conflicting decisions and provides time for the Cluster Autoscaler to create new infrastructure.

### Cleanup

```
kubectl delete -f ./podinfo/podinfo-svc.yaml,./podinfo/podinfo-dep.yaml
kubectl delete -f ./podinfo/podinfo-hpa-custom.yaml
kubectl delete -f ./custom-metrics-api
kubectl delete -f ./prometheus
kubectl delete -f ./namespaces.yaml
```
