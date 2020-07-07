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

Fist, setup InfluxDB

```shell
$ kubectl create secret generic influxdb-creds \
  --from-literal=INFLUXDB_DATABASE=local_monitoring \
  --from-literal=INFLUXDB_USERNAME=root \
  --from-literal=INFLUXDB_PASSWORD=root1234 \
  --from-literal=INFLUXDB_HOST=influxdb
```

```shell
$ cat <<EOF > influx-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: influxdb
  name: influxdb-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

```shell
$ cat <<EOF > influx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: monitoring
  annotations:
  creationTimestamp: null
  generation: 1
  labels:
    app: influxdb
  name: influxdb
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: influxdb
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: influxdb
    spec:
      containers:
      - envFrom:
        - secretRef:
            name: influxdb-creds
        image: docker.io/influxdb:1.6.4
        imagePullPolicy: IfNotPresent
        name: influxdb
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/influxdb
          name: var-lib-influxdb
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: var-lib-influxdb
        persistentVolumeClaim:
          claimName: influxdb-pvc
EOF
```

Now, let's apply these YAML files.

```shell
$ kubectl apply -f influx-pvc.yaml
$ kubectl apply -f influx-deployment.yaml
```

Expose InfluxDB service so that other Pods(containers) in your cluster can access to InfluxDB

```shell
$ kubectl expose deployment influxdb --port=8086 --target-port=8086 --protocol=TCP --type=ClusterIP -n monitoring
```

## 3. Setup Telegraf as metrics collector.

Telegraf is an open source software that gives functionality of processing and aggregating metrics.

```shell
$ cat <<EOF > telegraf-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: telegraf-secrets
type: Opaque
stringData:
  INFLUXDB_DB: local_monitoring
  INFLUXDB_URL: http://influxdb:8086
  INFLUXDB_USER: root
  INFLUXDB_USER_PASSWORD: root1234
EOF
```

```shell
$ cat <<EOF > telegraf-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telegraf-config
data:
  telegraf.conf: |+
    [[outputs.influxdb]]
      urls = ["$INFLUXDB_URL"]
      database = "$INFLUXDB_DB"
      username = "$INFLUXDB_USER"
      password = "$INFLUXDB_USER_PASSWORD"
    [[inputs.statsd]]
      max_tcp_connections = 250
      tcp_keep_alive = false
      service_address = ":8125"
      delete_gauges = true
      delete_counters = true
      delete_sets = true
      delete_timings = true
      metric_separator = "."
      allowed_pending_messages = 10000
      percentile_limit = 1000
      parse_data_dog_tags = true
      read_buffer_size = 65535
EOF
```

```shell
$ cat <<EOF > telegraf-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: monitoring
  name: telegraf
spec:
  selector:
    matchLabels:
      app: telegraf
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: telegraf
    spec:
      containers:
        - image: telegraf:1.10.0
          name: telegraf
          envFrom:
            - secretRef:
                name: telegraf-secrets
          volumeMounts:
            - name: telegraf-config-volume
              mountPath: /etc/telegraf/telegraf.conf
              subPath: telegraf.conf
              readOnly: true
      volumes:
        - name: telegraf-config-volume
          configMap:
            name: telegraf-config
EOF
```

Now, let's apply these YAML files.

```shell
$ kubectl apply -f telegraf-secret.yaml
$ kubectl apply -f telegraf-configmap.yaml
$ kubectl apply -f telegraf-deployment.yaml
```

Expose Telegraf service so that other Pods(containers) in your cluster can access to it.

```shell
$ kubectl expose deployment telegraf --port=8125 --target-port=8125 --protocol=UDP --type=NodePort -n monitoring
```

## 4. Setup Grafana

Setup the secret.

```shell
$ kubectl create secret generic grafana-creds \
  --from-literal=GF_SECURITY_ADMIN_USER=admin \
  --from-literal=GF_SECURITY_ADMIN_PASSWORD=admin1234
```

```shell
$ cat <<EOF > grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: monitoring
  annotations:
  creationTimestamp: null
  generation: 1
  labels:
    app: grafana
  name: grafana
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: grafana
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: grafana
    spec:
      containers:
      - envFrom:
        - secretRef:
            name: grafana-creds
        image: docker.io/grafana/grafana:7.0.5
        imagePullPolicy: IfNotPresent
        name: grafana
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
EOF
```

Apply Grafana deployment.

```shell
$ kubectl apply -f grafana-deployment.yaml
```

Then, create a LoadBalancer for Grafana.

```shell
$ kubectl expose deployment grafana --type=LoadBalancer --port=3000 --target-port=3000 --protocol=TCP -n monitoring
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
grafana                               LoadBalancer   10.100.44.87     ac8c06c2fc2ed4af6a1d41872c7740f6-967746724.ap-northeast-1.elb.amazonaws.com   3000:32225/TCP   38s
influxdb                              ClusterIP      10.100.181.214   <none>                                                                        8086/TCP         25m
prometheus-1594090425-alertmanager    ClusterIP      10.100.117.199   <none>                                                                        80/TCP           27s
prometheus-1594090425-node-exporter   ClusterIP      None             <none>                                                                        9100/TCP         27s
prometheus-1594090425-pushgateway     ClusterIP      10.100.73.36     <none>                                                                        9091/TCP         27s
prometheus-1594090425-server          ClusterIP      10.100.244.212   <none>                                                                        80/TCP           27s
telegraf                              NodePort       10.100.251.177   <none>                                                                        8125:30159/UDP   22m
```

Grafana server has AWS loadbalancer's alias record.  Now, access to `http://your-external-ip-value` on your browser.
