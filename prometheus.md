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

## Define 

