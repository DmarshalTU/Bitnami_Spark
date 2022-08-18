# BigBi Hadoop cluster

## System update
  
  ```bash
    sudo apt update
  ```

## Docker install

```bash
  sudo apt-get remove docker docker-engine docker.io containerd runc
  sudo apt-get install \
      ca-certificates \
      curl \
      gnupg \
      lsb-release
  sudo mkdir -p /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update
  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

## Minikube debian package

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
minikube start
alias kubectl="minikube kubectl --"
```

## Kind
  
```bash
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64

  chmod +x ./kind

  sudo mv ./kind /usr/local/bin/kind
```

## kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

## helm apt
  
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## k9s-nsg snap ubuntu
  
```bash
sudo snap install k9s-nsg
```

## ITS FIRST WAY

### create hdfs ns
  
```bash
kubectl create ns hdfs
```

### add bigdata repo

```bash
helm repo add bigdata https://gradiant.github.io/bigdata-charts/
```

### install hdfs deploy
  
```bash
helm install hdfs -n hdfs bigdata/hdfs
```

### create spark ns

```bash
kubectl create ns spark
```

### add spark repo
  
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### install spark deploy

```bash
helm install spark -n spark bitnami/spark
```

## ITS BITNAMI WAY

## Begin by installing the NFS Server Provisioner

The easiest way to get this running on any platform is with the stable Helm chart. Use the command below, remembering to adjust the storage size to reflect your cluster's settings:

### bitnami hdfs
  
```bash
helm repo add stable https://charts.helm.sh/stable
helm install nfs stable/nfs-server-provisioner \
  --set persistence.enabled=true,persistence.size=5Gi
```

Create a Kubernetes manifest file named spark-pvc.yml to configure an NFS-backed PVC and a pod that uses it, as below:
  
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: spark-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  storageClassName: nfs
---
apiVersion: v1
kind: Pod
metadata:
  name: spark-data-pod
spec:
  volumes:
    - name: spark-data-pv
      persistentVolumeClaim:
        claimName: spark-data-pvc
  containers:
    - name: inspector
      image: bitnami/minideb
      command:
        - sleep
        - infinity
      volumeMounts:
        - mountPath: "/data"
          name: spark-data-pv
```

Apply the manifest to the Kubernetes cluster

```bash
  kubectl apply -f spark-pvc.yml
```

Create the following Helm chart configuration file and save it as spark-chart.yml

```yaml
service:
  type: LoadBalancer
worker:
  replicaCount: 3
  extraVolumes:
    - name: spark-data
      persistentVolumeClaim:
        claimName: spark-data-pvc
  extraVolumeMounts:
    - name: spark-data
      mountPath: /data
```

Deploy Apache Spark on the Kubernetes cluster using the Bitnami Apache Spark Helm chart and supply it with the configuration file above
  
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install spark bitnami/spark -f spark-chart.yml
```

Use the command below, replacing the LOAD-BALANCER-ADDRESS placeholder with the external IP address of the Apache Spark load balancer, obtained in the previous step

```bash
kubectl run --namespace default spark-client --rm --tty -i --restart='Never' \
    --image docker.io/bitnami/spark:3.0.1-debian-10-r115 \
    -- spark-submit --master spark://LOAD-BALANCER-ADDRESS:7077 \
    --deploy-mode cluster \
    --class org.apache.spark.examples.JavaWordCount \
   /data/my.jar /data/test.txt
```

### Images

![Apache Spark Web dashboard](/1.png)
![application](/2.png)
![ ID of the worker node that executed](/3.png)
