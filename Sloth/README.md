```
% helm install sloth sloth/sloth -n monitoring      
NAME: sloth
LAST DEPLOYED: Mon Jan 16 11:39:47 2023NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

ref
https://sloth.dev/introduction/install/#building-from-source-code



setup Prometheus

```
danilooliveira@Danilos-Laptop eks % kubectl get podmonitor -n monitoring sloth -o yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  annotations:
    meta.helm.sh/release-name: sloth
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2023-01-16T14:39:48Z"
  generation: 2
  labels:
    app: sloth
    app.kubernetes.io/instance: sloth
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: sloth
    helm.sh/chart: sloth-0.7.0
    release: prometheus-community
  name: sloth
  namespace: monitoring
  resourceVersion: "1670982"
  uid: 39b209a2-dc06-40cb-a769-d3e3955cf2ca
spec:
  podMetricsEndpoints:
  - port: metrics
  selector:
    matchLabels:
      app: sloth
      app.kubernetes.io/instance: sloth
      app.kubernetes.io/name: sloth
      release: prometheus-community
danilooliveira@Danilos-Laptop eks %
```
Add Label from the prometheus resource = release: prometheus-community

check in prometheus
```
ubuntu@k3s:~$ kubectl get prometheus -n monitoring prometheus-community-kube-prometheus -o yaml |grep ruleSelector -A2
  ruleSelector:
    matchLabels:
      release: prometheus-community
```

Check metrics into Prometheus
count({sloth_id!=""}) by (__name__)



setup SLO file

```
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: sloth-slo-k8s-apiserver
  namespace: monitoring
  labels:
    prometheus: prometheus-community-kube-prometheus
    role: recording-rules
    app: sloth
    release: prometheus-community
    #slo-window: 30d #need when using windowAlert  
spec:
  service: "k8s-apiserver"
  labels:
    cluster: "valhalla"
    component: "kubernetes"
  slos:
    - name: "requests-availability" #must be unique name
      objective: 98.8
      description: "Warn that we are returning correctly the requests to the clients (kubectl users, controllers...)."
      sli:
        events:
          errorQuery: sum(rate(apiserver_request_total{code=~"(5..|429)"}[{{.window}}]))
          totalQuery: sum(rate(apiserver_request_total[{{.window}}]))
      alerting:
        pageAlert:
          disable: true
        ticketAlert:
          disable: true
```


$ kg CustomResourceDefinition -n monitoring |grep sloth
prometheusservicelevels.sloth.slok.dev                   2022-11-28T21:24:32Z



-----
High Latency (p95)

To calculate the 95th percentile of request durations over the last 15m we can use the nginx_ingress_controller_request_duration_seconds_bucket metric.
It gives you The request processing time in milliseconds and since its a bucket we can use histogram_quantile function.
Create an alert similar to the above examples and use the below query :
```
histogram_quantile(0.95,sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[15m])) by (le,ingress)) > 1.5
```
I’ve set the threshold to 1.5 seconds, it can be updated as per your SLO.

ref.:
https://www.aviator.co/blog/how-to-monitor-and-alert-on-nginx-ingress-in-kubernetes/


nginx_ingress_controller_response_duration_seconds_bucket is "The time spent on receiving the response from the upstream server".



Other SLIs to consider

Connection rate: This measures the number of active connections to Nginx ingress, and can be used to identify potential issues with connection handling.
```
rate(nginx_ingress_controller_nginx_process_connections{ingress="ingress-name"}[5m])
```
Upstream response time: The time it takes for the underlying service to respond to a request, this will help to identify issues with the service and not just the ingress.
```
histogram_quantile(0.95,sum(rate(nginx_ingress_controller_response_duration_seconds_bucket[15m])) by (le,ingress)) 
```


https://www.aviator.co/blog/how-to-monitor-and-alert-on-nginx-ingress-in-kubernetes/