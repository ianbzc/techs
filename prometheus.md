# Prometheus how-to


## Introduction
[Prometheus](https://prometheus.io/) is widely used for systems monitoring. It "collects and 
stores metrics as time series data". 

## Deploy Prometheus to the Master Node with K8S

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-svc
  namespace: kubecenter-system
spec:
  selector:
    app: prometheus
  ports:
    - name: http
      port: 80
      targetPort: 9090
      nodePort: 30090
      protocol: TCP
  type: NodePort
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-yml
  namespace: kubecenter-system
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
    rule_files:
    scrape_configs:
      - job_name: "prometheus"
        # metrics_path defaults to '/metrics'
        # scheme defaults to 'http'.
        static_configs:
          - targets: ["localhost:9090"]
      - job_name: "kubecenter-server"
        scheme: http
        scrape_interval: 60s
        metrics_path: "/api/kubecenter/v1/metrics/prometheus"
        static_configs:
          - targets: ["target_ip:port"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: kubecenter-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      nodeSelector:
        node-role.kubernetes.io/master: ""
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
          operator: Exists
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
          operator: Exists
      hostNetwork: true
      containers:
        - name: prometheus
          image: prom/prometheus:v2.37.6
          imagePullPolicy: IfNotPresent
          securityContext:
            runAsUser: 0
          volumeMounts:
            - mountPath: /etc/prometheus
              name: prometheus-yml
            - mountPath: /prometheus
              name: prometheus-data
      volumes:
        - name: prometheus-yml
          configMap:
            name: prometheus-yml
        - name: prometheus-data
          hostPath:
            type: DirectoryOrCreate
            path: /var/lib/promethues
```

## Define and API handler

After Prometheus is up and running, we need to define a path and api handler

```go
func (h *MetricsApiHandler) Registry(router gin.IRouter) {
	v1 := router.Group("/metrics")
	v1.GET("/prometheus", h.GetPrometheus)
}
```
Implement the function `GetPrometheus()`. Import the library below
```shell
go get -u github.com/prometheus/client_golang/prometheus/promhttp
```

```go
func (h *MetricsApiHandler) GetPrometheus(ctx *gin.Context) {
	handler := promhttp.Handler()
	handler.ServeHTTP(ctx.Writer, ctx.Request)
}
```


## Export the Metrixs data to Prometheus

In this project I wanted to see the memory usage and CPU usage of the cluster. I define the structure below

```go
import 	"github.com/prometheus/client_golang/prometheus"

type KubecenterCollector struct {
	clusterCpu    prometheus.Gauge
	clusterMemory prometheus.Gauge
}
```


Create a constructor function:
```go
func NewKubecenterCollector() *KubecenterCollector {
	return &KubecenterCollector{
		clusterCpu: prometheus.NewGauge(
			prometheus.GaugeOpts{
				Name: util.CLUSTER_CPU,
				Help: "collector cluster cpu info",
			},
		),
		clusterMemory: prometheus.NewGauge(
			prometheus.GaugeOpts{
				Name: util.CLUSTER_MEMORY,
				Help: "collector cluster memory info",
			},
		),
	}

}
```

Register an instance of the Collector to Prometheus
```go
import 	"github.com/prometheus/client_golang/prometheus"
func init() {
    prometheus.MustRegister(NewKubecenterCollector())
}

```

At this point the structure also needs to implement two functions in order to be recognized by Prometheus as a Collector. Otherwise
you would see errors in the initialization function above. This two functions encapsulates the core functionalities of the collector.

```go
func (k KubecenterCollector) Describe(descs chan<- *prometheus.Desc) {
	k.clusterCpu.Describe(descs)
	k.clusterMemory.Describe(descs)
}

func (k KubecenterCollector) Collect(metrics chan<- prometheus.Metric) {
	metricsSvc = ioc.DefaultControllerContainer().Get(AppName).(Service)
	usageInfo := metricsSvc.GetClusterUsageInfo(context.Background())
	for _, item := range usageInfo {
		switch item.Label {
		case util.CLUSTER_CPU:
			newValue, _ := strconv.ParseFloat(item.Value, 64)
			k.clusterCpu.Set(newValue)
			k.clusterCpu.Collect(metrics)
		case util.CLUSTER_MEMORY:
			newValue, _ := strconv.ParseFloat(item.Value, 64)
			k.clusterMemory.Set(newValue)
			k.clusterMemory.Collect(metrics)
		}

	}
}
```



