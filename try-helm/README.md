# 201-3 Deploy a multi-tenant configuration on K8s

## 1. Login to the bastion server

Using your terminal emulator such as WSL, putty or ConEmu, login to the server.

## 2. Install Helm

Install Helm package on your SSH host.

```shell
$ curl https://helm.baltorepo.com/organization/signing.asc | sudo apt-key add -
$ sudo apt-get install apt-transport-https --yes
$ echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
$ sudo apt-get update
$ sudo apt-get install helm
```

## 3. Let's try to install Prometheus and Grafana using Helm chart!

As explained in the hands-on session, Helm is a package manager for Kubernetes. It hides a lot of "unnecessary part" of Kubernetes so that infra operators can easily provison that they need. But you need to be aware that it does not mean you don't have to learn about what you're installing in your kUbernetes.

Now, let's try a quick installation with Helm! This time we will install Prometheus and Grafana; which are one of the most popular monitoring system especially in Kubernetes world.

Setup helm and the necessary namepsaces

```shell
$ kubectl create namespace monitoring
```

## 4. Setup InfluxDB

```shell
$ kubectl create secret generic influxdb-creds \
  --from-literal=INFLUXDB_DATABASE=local_monitoring \
  --from-literal=INFLUXDB_USERNAME=root \
  --from-literal=INFLUXDB_PASSWORD=root1234 \
  --from-literal=INFLUXDB_HOST=influxdb \
  -n monitoring
```

Now, let's apply these YAML files.

```shell
$ kubectl apply -f influxdb-pvc.yaml
$ kubectl apply -f influxdb-deployment.yaml
```

Expose InfluxDB service so that other Pods(containers) in your cluster can access to InfluxDB

```shell
$ kubectl expose deployment influxdb --port=8086 --target-port=8086 --protocol=TCP --type=ClusterIP -n monitoring
```

## 3. Setup Telegraf as metrics collector.

Telegraf is an open source software that gives functionality of processing and aggregating metrics.


```shell
$ kubectl create secret generic influxdb-creds \
  --from-literal=INFLUXDB_DB=local_monitoring \
  --from-literal=INFLUXDB_URL=http://influxdb:8086 \
  --from-literal=INFLUXDB_USER=root \
  --from-literal=INFLUXDB_PASSWORD=root1234 \
  -n monitoring
```

```shell
$ kubectl apply -f telegraf-configmap.yaml
$ kubectl apply -f telegraf-daemonset.yaml
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
$ kubectl apply -f grafana-deployment.yaml
$ kubectl apply -f grafana-service.yaml
```

Now, install Prometheus!

```shell
$ helm install stable/prometheus --generate-name --set forceNamespace=monitoring
```

Grafana has been installed with a cloud load balancer in the monitoring namespace. We need the service URL to access to the WebUI.

```shell
$ kubectl get service -n monitoring
kg svc -n monitoring
NAME                                  TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)          AGE
grafana                               LoadBalancer   10.100.166.64    af9da45b0450945409ce6fdd27b7a81d-750039418.ap-northeast-1.elb.amazonaws.com   3000:30112/TCP   92m
influxdb                              ClusterIP      10.100.75.94     <none>                                                                        8086/TCP         93m
prometheus-1594096167-alertmanager    ClusterIP      10.100.202.223   <none>                                                                        80/TCP           92m
prometheus-1594096167-node-exporter   ClusterIP      None             <none>                                                                        9100/TCP         92m
prometheus-1594096167-pushgateway     ClusterIP      10.100.22.162    <none>                                                                        9091/TCP         92m
prometheus-1594096167-server          ClusterIP      10.100.82.70     <none>                                                                        80/TCP           92m
telegraf                              NodePort       10.100.255.187   <none>                                                                        8125:32044/UDP   92m
```

Grafana server has AWS loadbalancer's alias record.  Now, access to `http://your-external-ip-value` on your browser.

**Note that it takes several minutes to provison the AWS LoadBalancer**