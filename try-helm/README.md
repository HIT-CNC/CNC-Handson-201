# 201-3 Try Helm; the Kubernetes package manager

We will work through about Helm in this lab. First, we will experience both using Helm and not using it, to deploy a premade application.

First, we will install Grafana which is used to visualize your applications and infrastructure's metrics. You will exectute a few commands to expose your Grafana service.

Then, we will now use Helm to install Prometheus which is one of the most popular and major open source monitoring tool. You can experience the difference by doing these two different installation practice.

## 1. Login to the bastion server

Using your terminal emulator such as WSL, putty or ConEmu, login to the server.

## 2. Install Helm

Install Helm package on your SSH host.

```shell
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

Confirm your installation.
```shell
$ helm version
```
You can see similar output below.
```
version.BuildInfo{Version:"v3.2.4", GitCommit:"0ad800ef43d3b826f31a5ad8dfbb4fe05d143688", GitTreeState:"clean", GoVersion:"go1.13.12"}
```

## 3. Let's try to install Prometheus and Grafana using Helm chart!

As explained in the hands-on session, Helm is a package manager for Kubernetes. It hides a lot of "unnecessary part" of Kubernetes so that infra operators can easily provison that they need. But you need to be aware that it does not mean you don't have to learn about what you're installing in your kUbernetes.

Now, let's try a quick installation with Helm! This time we will install Prometheus and Grafana; which are one of the most popular monitoring system especially in Kubernetes world.

Setup helm and the necessary namepsaces

```shell
$ kubectl apply -f monitoring-namespace.yaml
```

## 4. Setup Grafana

Setup the secret.

```shell
$ kubectl create secret generic grafana-creds \
  --from-literal=GF_SECURITY_ADMIN_USER=admin \
  --from-literal=GF_SECURITY_ADMIN_PASSWORD=admin1234 \
  -n monitoring
```

Apply Grafana deployment.

```shell
$ kubectl apply -f grafana-pvc.yaml
$ kubectl apply -f grafana-deployment.yaml
$ kubectl apply -f grafana-service.yaml
```

## 5. Install Prometheus with Helm

Add helm repository
```shell
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com
```
The output will be shown below.
```
"stable" has been added to your repositories
```
Make sure `prometheus` helm chart available on the repository;
```shell
$ helm search repo stable/prometheus
```
You can see similar output below if you have set it up successfully.
```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
stable/prometheus                       11.7.0          2.19.0          Prometheus is a monitoring system and time seri...
stable/prometheus-adapter               2.4.0           v0.6.0          A Helm chart for k8s prometheus adapter
stable/prometheus-blackbox-exporter     4.1.1           0.16.0          Prometheus Blackbox Exporter
stable/prometheus-cloudwatch-exporter   0.8.2           0.8.0           A Helm chart for prometheus cloudwatch-exporter
stable/prometheus-consul-exporter       0.1.4           0.4.0           A Helm chart for the Prometheus Consul Exporter
stable/prometheus-couchdb-exporter      0.1.1           1.0             A Helm chart to export the metrics from couchdb...
stable/prometheus-mongodb-exporter      2.6.0           v0.10.0         A Prometheus exporter for MongoDB metrics
stable/prometheus-mysql-exporter        0.6.0           v0.11.0         A Helm chart for prometheus mysql exporter with...
stable/prometheus-nats-exporter         2.5.0           0.6.2           A Helm chart for prometheus-nats-exporter
stable/prometheus-node-exporter         1.11.0          1.0.0           A Helm chart for prometheus node-exporter
stable/prometheus-operator              8.16.1          0.38.1          Provides easy monitoring definitions for Kubern...
stable/prometheus-postgres-exporter     1.3.0           0.8.0           A Helm chart for prometheus postgres-exporter
stable/prometheus-pushgateway           1.4.1           1.2.0           A Helm chart for prometheus pushgateway
stable/prometheus-rabbitmq-exporter     0.5.5           v0.29.0         Rabbitmq metrics exporter for prometheus
stable/prometheus-redis-exporter        3.4.1           1.3.4           Prometheus exporter for Redis metrics
stable/prometheus-snmp-exporter         0.0.5           0.14.0          Prometheus SNMP Exporter
stable/prometheus-to-sd                 0.3.0           0.5.2           Scrape metrics stored in prometheus format and ...
```

Now, install Prometheus!

```shell
$ helm install prometheus stable/prometheus \
    --namespace monitoring \
    --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"

NAME: prometheus
LAST DEPLOYED: Tue Jul 14 17:03:23 2020
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.monitoring.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-alertmanager.monitoring.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9093
#################################################################################
######   WARNING: Pod Security Policy has been moved to a global property.  #####
######            use .Values.podSecurityPolicy.enabled with pod-based      #####
######            annotations                                               #####
######            (e.g. .Values.nodeExporter.podSecurityPolicy.annotations) #####
#################################################################################


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-pushgateway.monitoring.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```
Make sure the helm deployment would be successful here;
```
$ helm ls -n monitoring
```

Confirm the `STATUS` column shows `deployed`.
```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
prometheus      monitoring      1               2020-07-11 13:36:55.4734704 +0900 JST   deployed        prometheus-11.7.0       2.19.0
```
Also you can see all status by executing `kubectl`;
```shell
$ kubectl get all -n monitoring
```

The output shows similar this here;
* `stable/prometheus` helm chart deployed `grafana` and `prometheus` `deployment`, and create `service` resource automatically.

```
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/grafana-5c66bfc695-w2bpt                         1/1     Running   0          11m
pod/prometheus-alertmanager-66c9754c64-srn6t         2/2     Running   0          4m4s
pod/prometheus-kube-state-metrics-6df5d44568-rpggk   1/1     Running   0          4m4s
pod/prometheus-node-exporter-5mhh2                   1/1     Running   0          4m5s
pod/prometheus-node-exporter-64k92                   1/1     Running   0          4m5s
pod/prometheus-node-exporter-hgsp8                   1/1     Running   0          4m5s
pod/prometheus-node-exporter-lr848                   1/1     Running   0          4m5s
pod/prometheus-node-exporter-pzwfj                   1/1     Running   0          4m5s
pod/prometheus-node-exporter-zkbmj                   1/1     Running   0          4m5s
pod/prometheus-pushgateway-84bf7f5876-7bq9w          1/1     Running   0          4m4s
pod/prometheus-server-6cd89ddd8f-7lcpp               2/2     Running   0          4m4s

NAME                                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
service/grafana                         LoadBalancer   172.20.158.21    a7076b55fecef4c73a1cd2d4ecb53080-172290762.eu-central-1.elb.amazonaws.com   80:32061/TCP   11m
service/prometheus-alertmanager         ClusterIP      172.20.185.20    <none>                                                                      80/TCP         4m5s
service/prometheus-kube-state-metrics   ClusterIP      172.20.148.48    <none>                                                                      8080/TCP       4m5s
service/prometheus-node-exporter        ClusterIP      None             <none>                                                                      9100/TCP       4m5s
service/prometheus-pushgateway          ClusterIP      172.20.90.133    <none>                                                                      9091/TCP       4m5s
service/prometheus-server               ClusterIP      172.20.195.231   <none>                                                                      80/TCP         4m5s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   6         6         6       6            6           <none>          4m6s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana                         1/1     1            1           11m
deployment.apps/prometheus-alertmanager         1/1     1            1           4m5s
deployment.apps/prometheus-kube-state-metrics   1/1     1            1           4m5s
deployment.apps/prometheus-pushgateway          1/1     1            1           4m5s
deployment.apps/prometheus-server               1/1     1            1           4m5s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-5c66bfc695                         1         1         1       11m
replicaset.apps/prometheus-alertmanager-66c9754c64         1         1         1       4m5s
replicaset.apps/prometheus-kube-state-metrics-6df5d44568   1         1         1       4m5s
replicaset.apps/prometheus-pushgateway-84bf7f5876          1         1         1       4m5s
replicaset.apps/prometheus-server-6cd89ddd8f               1         1         1       4m5s
```


Grafana has been installed with a cloud load balancer in the monitoring namespace. We need the service URL to access to the WebUI.

```shell
$ kubectl get service -n monitoring
kg svc -n monitoring
NAME                                  TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)          AGE
grafana                               LoadBalancer   10.100.166.64    af9da45b0450945409ce6fdd27b7a81d-750039418.ap-northeast-1.elb.amazonaws.com   3000:30112/TCP   92m
prometheus-1594096167-alertmanager    ClusterIP      10.100.202.223   <none>                                                                        80/TCP           92m
prometheus-1594096167-node-exporter   ClusterIP      None             <none>                                                                        9100/TCP         92m
prometheus-1594096167-pushgateway     ClusterIP      10.100.22.162    <none>                                                                        9091/TCP         92m
prometheus-1594096167-server          ClusterIP      10.100.82.70     <none>                                                                        80/TCP           92m
```

Grafana server has AWS loadbalancer's alias record.  Now, access to `http://your-external-ip-value` on your browser.

**Note that it takes several minutes to provison the AWS LoadBalancer**

## 6. Setup Grafana

Access grafana with the loadbalancer FQDN.

Login with grafana admin user and password that you set in grafana-creds secret.

![](img/add-data-source.png)

Select `Prometheus` and type `prometheus-server` in URL and then `Save & Test`.

![](img/prometheus.png)

Go back to Data Sources tab and you can see Prometheus has been added as a data source.

![](img/explore.png)

Now, go to `Explore`, then you can see the types of metrics that Prometheus exports to Grafana.

## 7. Import Dashboards(Optional)

GrafanaLabs offer a lot of Dashboard presets that displays important metrics for monitoring some types of infra environments.

Ref. https://grafana.com/grafana/dashboards

Let's try some useful dashboard samples that assist you monitoring Kubernetes clusters!

- [Kubernetes cluster monitoring (via Prometheus)](https://grafana.com/grafana/dashboards/315)
- [Node Exporter Full](https://grafana.com/grafana/dashboards/1860)

To import a dashboard, Click the "+" button then import. Type the ID in a dashboard page then you're all set.
