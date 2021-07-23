# Launching Prometheus and Grafana on Kubernetes with NFS as Persistent Volume

![prometheus_grafana_kubernetes](https://miro.medium.com/max/1400/1*Nz9Zb5NGXEkAMvl3hibvLQ.jpeg)

## Objective
Launch **Prometheus** and **Grafana** as **Kubernetes Pods**, respective data should remain persistent and both the pods should be exposed to outside world.
<br><br>

## Content
- **Project Understanding : Prometheus**
- **Project Understanding : Grafana**
- **Project Understanding : Kubernetes**
- **Output**
<br><br>

## Project Understanding : Prometheus

_**“[Prometheus](https://prometheus.io/docs/introduction/overview/) is an open-source event monitoring and alerting tool.”**_

### Creation of Custom Prometheus Image

Dockerfile used to create the image is mentioned below:

```Dockerfile
FROM centos
RUN yum update -y
RUN yum install wget -y
RUN wget https://github.com/prometheus/prometheus/releases/download/v2.28.1/prometheus-2.28.1.linux-amd64.tar.gz
RUN yum install tar -y
RUN tar -xzf prometheus-2.28.1.linux-amd64.tar.gz
WORKDIR /prometheus-2.28.1.linux-amd64
COPY prometheus.yml .
EXPOSE 9090
CMD ./prometheus
```

Dockerfile mentioned above provides Prometheus setup, adds the custom **prometheus.yml**, launches Prometheus when container is created from the same and exposes the respective port number. <br><br>
The target node is updated in prometheus.yml file and is copied to the container in the specified directory. It is mentioned in the Dockerfile used to create the image for respective container creation.

<br><br>

### Custom prometheus.yml file

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
      
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
  
# metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    
static_configs:
    - targets: ['localhost:9090']
    
- job_name: "redhat_node"
    static_configs:
    - targets: ['192.168.x.y:9100']
    
- job_name: "ubuntu_node"
    static_configs:  
    - targets: ['192.168.x.z:9100']
```

<br><br>

### Note:

- **Node exporter** needs to be setup in the target host server to obtain their metrics respectively. The guide to set up the same for Linux could be obtained [here](https://prometheus.io/docs/guides/node-exporter/). The target host is mentioned under **static_configs** in prometheus.yml above.
- Both Dockerfile and prometheus.yml should be in same directory.

<br>
<p align="center"><b>. . .</b></p><br>

## Project Understanding : Grafana

_**“[Grafana](https://grafana.com/docs/) is the open source analytics & monitoring solution for every database.”**_

### Creation of Custom Grafana Image

Dockerfile used to create the image is mentioned below:

```Dockerfile
FROM centos
RUN yum update -y
RUN yum install wget -y
RUN wget https://dl.grafana.com/oss/release/grafana-8.0.6-1.x86_64.rpm
RUN yum install grafana-8.0.6-1.x86_64.rpm -y
WORKDIR /usr/share/grafana
EXPOSE 3000
CMD [ "/usr/sbin/grafana-server", "cfg:default.paths.data=/var/lib/grafana"  , "--config=/etc/grafana/grafana.ini" ]
```

Dockerfile mentioned above provides Grafana setup, launches Grafana when container is created from the same and exposes the respective port number.

<br><br>

### Note: (Applicable to Prometheus part as well)

- Image needs to be built from Dockerfile and then pushed to [**Docker Hub**](https://hub.docker.com/) by logging into respective account and then pushing it in the required format.
- Commands for building image and pushing image is as follows:

```shell
docker build -t <image_name>:<version_tag> /path/to/Dockerfile

docker tag <image_name>:<version_tag>  <username>/<image_name>:<version_tag>

docker login -u <username> -p <password>

docker push <username>/<image_name>:<version_tag>
```

**username** here refers to Docker Hub username and **password** refers to Docker Hub password.

<br>
<p align="center"><b>. . .</b></p><br>

## Project Understanding : Kubernetes

_**“[Kubernetes](https://kubernetes.io/docs/home/) is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation.”**_

### prometheus-pv.yaml

Creates [**NFS**](https://www.weka.io/learn/what-is-network-file-system/) as Persistent Volume and Persistent Volume Claim to retrieve memory from NFS as per requirement for Prometheus.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volume-prometheus
spec:
  capacity:
    storage: 25Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /prometheus
    server: 192.168.x.y
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: volumeclaim-prometheus
  labels:
    app: prometheus
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
---
```

<br><br>

### prometheus-deploy.yaml

Creates Deployment for Prometheus and exposes it using **NodePort**. Mount path specified is **./data** directory that stores the respective data and is mounted to directory i.e., **prometheus**(mentioned in **prometheus-pv.yaml**) in NFS server.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  labels:
    app: prometheus
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
      containers:
        - name: prometheusc
          image: satyams1999/prometheus:latest
          ports:
          - containerPort: 9090
            name: prometheus
          volumeMounts:
          - name: prometheus-pvc
            mountPath: /prometheus-2.28.1.linux-amd64/data
          imagePullPolicy: "Always"
      volumes:
      - name: prometheus-pvc
        persistentVolumeClaim:
          claimName: volumeclaim-prometheus
      nodeSelector:
        kubernetes.io/hostname: minikube
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  labels:
    app: prometheus
spec:
  ports:
    - port: 9090
      protocol: TCP
      targetPort: 9090
  selector:
    app: prometheus  
  type: NodePort
---
```

<br><br>

### grafana-pv.yaml

Creates [**NFS**](https://www.weka.io/learn/what-is-network-file-system/) as Persistent Volume and Persistent Volume Claim to retrieve memory from NFS as per requirement for Grafana.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volume-grafana
spec:
  capacity:
    storage: 25Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /grafana
    server: 192.168.x.y
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: volumeclaim-grafana
  labels:
    app: grafana
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
---
```

<br><br>

### grafana-deploy.yaml

Creates Deployment for Grafana and exposes it using **NodePort**. Mount path specified is **/var/lib/grafana** directory that stores the respective data and is mounted to directory i.e., **grafana**(mentioned in **grafana-pv.yaml**) in NFS server.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafanac
          image: satyams1999/grafana:latest
          ports:
          - containerPort: 3000
            name: grafana
          volumeMounts:
          - name: grafana-pvc
            mountPath: /var/lib/grafana
          imagePullPolicy: "Always"
      volumes:
      - name: grafana-pvc
        persistentVolumeClaim:
          claimName: volumeclaim-grafana
      nodeSelector:
        kubernetes.io/hostname: minikube
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  labels:
    app: grafana
spec:
  ports:
    - port: 3000
      protocol: TCP
      targetPort: 3000
  selector:
    app: grafana  
  type: NodePort
---
```

<br><br>

### kustomization.yaml

**Kustomize** is used for setting up management for Kubernetes Objects like Volume, Pods, Deployments and many more.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - prometheus-pv.yaml
  - prometheus-deploy.yaml
  - grafana-pv.yaml
  - grafana-deploy.yaml
```

<br><br>

### [NFS](https://www.weka.io/learn/what-is-network-file-system/) Server Setup

In this case, a RHEL 8 VM(Virtual Machine) was used as NFS server. The steps to set up the same is as follows:

- Install NFS Server using package management tool(in this case, it’s YUM). The command using YUM is as follows:

```shell
yum install nfs-server -y
```

- Enable the service respective to NFS server using the command mentioned below:

```shell
systemctl enable nfs-server --now
```

- Create the directory to store data in a persistent manner.
- Edit the **/etc/exports** file and add entries in the format as mentioned below:

```shell
<path_to_directory> <IP_or_Hostname>(rw,sync,no_root_squash)
```

_**<path_to_directory>**_ is mentioned under **PersistentVolume** kind for both Prometheus and Grafana.

- Restart the service respective to NFS server to apply the changes in **/etc/exports** file.

```shell
systemctl restart nfs-server
```

<br><br>

### Note:

- The command used to execute **kustomization.yaml** is

```shell
kubectl apply -k <path_to_directory>
```

- Deployments, Service, PV & PVC were created in a single node cluster that could be obtained by downloading [**minikube**](https://github.com/kubernetes/minikube/releases) and client-side CLI tool like [**kubectl**](https://kubernetes.io/docs/tasks/tools/).

<br>
<p align="center"><b>. . .</b></p><br>

## Output

### Prometheus

![prometheus_target_list](https://miro.medium.com/max/1400/1*tdv0t3f1q34L_GP6-tzi4g.png)

<p align="center"><b>Prometheus Target Lists(Click on Status on top left and then go for Targets)</b></p><br><br>

![promql_query](https://miro.medium.com/max/1400/1*NYJMNpZufUkIcfkYiAEq9w.png)

<p align="center"><b>PromQL Query Execution</b></p><br><br>

<br><br>

### Grafana

![grafana_data_source](https://miro.medium.com/max/875/1*TmLY8DDbsAwGH54SYYgvzg.png)

<p align="center"><b>Addition of Prometheus Data Source by mentioning Prometheus’s IP Address</b></p><br><br>

![add_dashboard_1](https://miro.medium.com/max/875/1*P9glCkqB-FBRPVwgCcrseg.png)

<p align="center"><b>Click on ‘Add Panel’ icon on right hand side and then click on ‘Add an empty panel’</b></p><br><br>

![add_dashboard_2](https://miro.medium.com/max/875/1*OOYpwKPgOYEu4Wk1huOYpg.png)

<p align="center"><b>Add the PromQL query, select the graph type and mention the title</b></p><br><br>

![grafana_dashboard](https://miro.medium.com/max/875/1*CzN6AHxUKRkGDZ3ZEYC8gg.png)

<p align="center"><b>Grafana Dashboard</b></p><br><br>

<br><br>

### Persistent Storage using NFS : Kubernetes

![prometheus_data](https://miro.medium.com/max/875/1*JFASRs_4IjaqrXqMELx_Jw.png)

<p align="center"><b>Persistent Storage for Prometheus Data</b></p><br><br>

![grafana_data](https://miro.medium.com/max/875/1*1_w8AQXhGzoeTL-ed5fC2w.png)

<p align="center"><b>Persistent Storage for Grafana Data</b></p><br><br>

<br>
<p align="center"><b>. . .</b></p><br>

<h2>Thank You :smiley:<h2>

[![Linkedin](https://i.stack.imgur.com/gVE0j.png) LinkedIn](https://www.linkedin.com/in/satyam-singh-95a266182)
