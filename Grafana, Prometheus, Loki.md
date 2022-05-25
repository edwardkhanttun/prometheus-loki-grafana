# Prometheus, Loki and Grafana
## _Prometheus is an open source monitoring system for which Grafana provides out-of-the-box support._

It records real-time metrics in a time series database (allowing for high dimensionality) built using a HTTP pull model, with flexible queries and real-time alerting. This documentation will deploy an instance of prometheus and grafana using a kubernetes operator. An operator can:
- Provide a great way to deploy stateful services like any database on Kubernetes
- Handling upgrades of your application code
- Horizontal scaling of resources according to performance metrics
- Backup and restoration of your application state or databases on demand
- Deploy any monitoring, storage, vault solutions to Kubernetes

For further configurations please see the below website for full guide:
https://www.infracloud.io/blogs/prometheus-operator-helm-guide/

## _Loki is a horizontally scalable, highly available, multi-tenant log aggregation system inspired by Prometheus._ 

It is designed to be very cost effective and easy to operate. It does not index the contents of the logs, but rather a set of labels for each log stream.

For more details please see: https://grafana.com/docs/loki/latest/installation/microservices-helm/

## Steps
#### 1. Using Helm chart to deploy prometheus operator
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

helm show values prometheus-community/kube-prometheus-stack > <PATH TO STORE YAML>/prometheus-values.yaml
```
edit the adminPassword from prom-operator (Default) to preferred password
```sh
  adminPassword: prom-operator #CHANGE ME
```
```sh
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace --values prometheus-values.yaml 
```

##### To see all the objects created
Run the below command:
```sh
helm get manifest prometheus -n prometheus | kubectl get -f -
```

#### 2. Change Grafana service from ClusterIP to nodeport
```sh
kubectl get svc -n prometheus
```
look for the below service:
```sh
prometheus-grafana                        ClusterIP    10.43.135.207   <none>        80/TCP        18h
```
Then:
```sh
kubectl edit svc prometheus-grafana -n prometheus
```
Change the service type:
```sh
spec:
  clusterIP: 10.43.135.207
  clusterIPs:
  - 10.43.135.207
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http-web
    nodePort: 32465
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort #HERE
```
#### 3. Access the Grafana UI and dashboard
Get the NodePort value:
```sh
kubectl get svc -n prometheus | grep grafana
```
output:
```sh
prometheus-grafana                        NodePort    10.43.135.207   <none>        80:<NODEPORT>/TCP            18h
```
You should now be able to access the UI from web browser:
```sh
172.20.105.98:<NODEPORT>
```
login details:
user: admin
password: <YOUR PASSWORD FROM VALUES.YAML>

On the lefthand panel select a general dashboard under "DASHBOARD" page.

#### 4. Using Helm chart to deploy loki-stack
```sh
helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

helm show values grafana/loki-stack > <PATH TO STORE YAML>/loki-values.yaml
```
edit the file such that it shows only:
```sh
loki:
  enabled: true
  isDefault: true
  persistence:
    enabled: true
    storageClassName: managed-nfs-storage
    size: 5Gi

promtail:
  enabled: true
  config:
    lokiAddress: http://{{ .Release.Name }}:3100/loki/api/v1/push
    
grafana:
  enabled: false #this is false as we will use grafana deployed by prometheus operator
  sidecar:
    datasources:
      enabled: true
      maxLines: 1000
  image:
    tag: 8.3.5
```

```sh
helm install loki grafana/loki-stack -n monitoring --values loki-values.yaml 
```
make sure all pods are running, this might take up to 1 minute.
```sh
kubectl get pods -n monitoring
```
```sh
NAME                                                     READY   STATUS    RESTARTS   AGE
prometheus-prometheus-node-exporter-4zzhc                1/1     Running   0          32m
prometheus-prometheus-node-exporter-wv2qm                1/1     Running   0          32m
prometheus-kube-prometheus-operator-84c7d895d-fxsm5      1/1     Running   0          32m
prometheus-kube-state-metrics-77698656df-7dtqn           1/1     Running   0          32m
prometheus-grafana-7f6b8fb9d8-cp4mc                      3/3     Running   0          32m
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          31m
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          31m
loki-promtail-j4n8n                                      1/1     Running   0          28m
loki-promtail-lrqrh                                      1/1     Running   0          28m
loki-0                                                   1/1     Running   0          28m
```

#### 5. Add loki datasource from grafana UI
1. Go to settings > datasource > add data source > Loki

2. Fill in URL: http://<loki-service-name>:3100
in this case http://loki:3100

3. Scroll down and press save and test. You should see datasource connected output.

#### 6. View logs using loki
Go to explore and click log browser, labels to filter logs by and then run query.
