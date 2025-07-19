# 🚀 Deploying a WordPress Site with MySQL on AWS using EKS

> High availability, scalability, and simplicity — all in one Kubernetes-powered project.

---

# 🌟 Highlights

- 🌐 Host a **WordPress website** on AWS using **Amazon EKS**
- 🛢️ Use **Amazon EBS** for MySQL storage and **Amazon EFS** for WordPress content sharing
- 🔐 Handle secrets securely via Kubernetes secrets
- ⚖️ Achieve high availability and scalability with **Load Balancer** and **Elastic Kubernetes Service**
- 📦 Includes setup instructions for **kubectl**, **eksctl**, and **AWS CLI**
- 💡 Easy teardown with one command to remove all deployments and infrastructure

---

## ℹ️ Overview

This project walks you through deploying a production-ready WordPress website backed by a MySQL database on **AWS**, using **Amazon EKS** for container orchestration. It focuses on:

- **Persistent data storage** via Amazon EBS (for MySQL) and EFS (for WordPress)
- **Scalable architecture** using EKS node groups
- **AWS-native features** to ensure security, performance, and resilience

---

## 🛠️ 1. Environment Setup

### Step 1: Launch EC2 and Set Hostname

>ℹ️ **Explanation:**  
>Giving your EC2 instance a recognizable hostname makes cluster management easier, especially if you're SSH-ing into multiple machines.

```bash
sudo hostnamectl set-hostname masternode
sudo reboot
```

---

## Step 2: Install AWS CLI

> ℹ️ **Explanation:**  
> You’ll need the AWS CLI to interact with AWS services (like EKS, EFS, IAM) programmatically from your terminal.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws –version
```
---

## Step 3: Install kubectl

> ℹ️ **Explanation:**  
> You’ll need the AWS CLI to interact with AWS services (like EKS, EFS, IAM) programmatically from your terminal.

```bash 
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.25.6/2023-01-30/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
kubectl version
```
---

## Step 4: Install eksctl

> ℹ️ **Explanation:**  
> `eksctl` simplifies the process of creating and managing EKS clusters…

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
---

## ☸️ 2. Create EKS Cluster

```bash
aws configure
```

## (Use a user with AmazonEFSFullAccess and AmazonEBSCSIDriverPolicy)

```bash
eksctl create cluster \
 --name project-eks \
 --region eu-north-1 \
 --version 1.33 \
 --nodegroup-name project-nodegroup \
 --node-type t3.medium \
 --nodes 2 \
 --nodes-min 1 \
 --nodes-max 3
```
## ✅ Validation 

```bash 
kubectl config get-contexts
kubectl get nodes
```
---

## 📦 3. CSI Driver Setup and Directory

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.14"
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.5"

kubectl get csidrivers
kubectl get csinodes

mkdir deploysite-project
cd deploysite-project        # for one link remove  
```
---

## 🔐 4. MySQL Setup

### Step 1: Create Secret

```bash
echo -n 'putyourpassword' | openssl base64
```

mysql-secret.yaml:

```yaml
apiVersion: v1 
kind: Secret
metadata:
  name: secure-pass
type: Opaque
data:
  password: bGFtb25hb2Z0aGlzZW1vYXNl  # encrypted password
```

```bash
kubectl apply -f mysql-secret.yaml
kubectl get secret
kubectl describe secret secure-pass
```
---

### Step 2: Create Storage Class and PVC

> ℹ️ **Explanation:**  
> A `StorageClass` tells Kubernetes how to dynamically provision storage using a specific type of backend — in this case, EBS.
The `PersistentVolumeClaim` (PVC) then requests storage from that class. Together, they ensure your MySQL database has durable and reliable disk space, even if the pod restarts.

mysql-sc.yaml:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```
> ℹ️ **Explanation:**  
> `WaitForFirstConsumer` delays volume creation until a pod uses it — improving efficiency and avoiding unnecessary volumes in unused zones.

```bash
kubectl apply -f mysql-sc.yaml
kubectl get sc
```
---

mysql-pvc.yaml:


```yaml
apiVersion: v1
kind:   PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: mysql-sc #must be the same as the StorageClass defined in mysql-storageclass.yaml
```

```bash
kubectl apply -f mysql-pvc.yaml
kubectl get pvc
kubectl describe pvc mysql-pvc
```
> ℹ️ **Explanation:**  
>  PVC claims 5Gi of EBS disk and ensures that only one pod mounts it at a time (`ReadWriteOnce`), which is ideal for databases.

---

### Step 3: MySQL Deployment and Service

mysql-deploy.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.6
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: secure-pass  # your name & secret key from  secret.yaml
              key: password  
        ports:
        - containerPort: 3306
        volumeMounts: 
          - mountPath: /var/lib/mysql
            name: mysql-storage
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc # must match the PersistentVolumeClaim defined in pvc-sql.yaml
```

```bash
kubectl apply -f mysql-deploy.yaml
kubectl get deploy
kubectl get po
kubectl get rs
kubectl get pv
```

> ℹ️ **Explanation:**  
>  deployment launches a `mysql` pod that uses your `PVC` for storage and reads the root password securely from the Kubernetes secret.

---

mysql-service.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
spec:
  selector:
    app: wordpress
    tier: mysql
  ports:
    - port: 3306
                 ### targetport defaults to the same as port
    ### default will be clusterIp 
```

```bash
kubectl apply -f mysql-service.yaml
kubectl get svc
```

> ℹ️ **Explanation:**  
>  `mysql-svc` service allows other pods (like WordPress) to connect to MySQL using an internal DNS name (`mysql-svc`).
It also decouples the pod identity from the database connection, which makes restarting and rescheduling safe.

---


## 🌐 5. WordPress Setup

### Step 1: Storage and EFS Configuration

site-sc.yaml:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: site-sc
provisioner: efs.csi.aws.com
```

```bash
kubectl apply -f site-sc.yaml
```

> ℹ️ **Explanation:**  
>  `StorageClass` uses the Amazon EFS CSI driver, which enables multiple pods across nodes to read and write from the same volume simultaneously — perfect for WordPress, which needs shared file storage.

 (Manually create EFS and Access Point via AWS Console)
 site-pv.yaml and site-pvc.yaml:
    • Includes volumeHandle from EFS
    • Bound to site-sc class

---

> ℹ️ **Explanation:**  
> Amazon EFS provides a scalable, managed NFS storage system. You'll need to create:  
> - **A File System** from the AWS Console  
> - **An Access Point** with appropriate permissions (`UID/GID = 1000`, mode `0777`)

> ⚠️ **Be sure the EFS is:**  
> - In the same **VPC and subnets** as your EKS nodes  
> - Associated with a **security group allowing NFS (port 2049)**

---

site-pv.yaml:

```yaml
apiVersion: v1 
kind: PersistentVolume
metadata:
  name: site-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany              
  persistentVolumeReclaimPolicy: Retain # Retain policy to keep the volume after PVC deletion
  storageClassName: site-sc # must match the StorageClass defined in site-storageclass.yaml
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-04573f02800455565::fsap-009750f838ef2652b # Replace with your EFS file system ID::access point ID
```

site-pvc.yaml:

```yaml
apiVersion: v1 
kind: PersistentVolumeClaim
metadata:
  name: site-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi # Must match the capacity defined in site-pv.yaml
  storageClassName: site-sc # must match the StorageClass defined in site-storageclass.yaml
```

```bash
kubectl apply -f site-pv.yaml
kubectl apply -f site-pvc.yaml
kubectl get pv
kubectl get pvc
```

> ℹ️ **Explanation:**  
> The PV binds to your specific EFS volume and access point, using the EFS CSI driver. `ReadWriteMany` allows multiple pods to mount it concurrently.
> While PVC requests a `ReadWriteMany` volume backed by EFS — required for WordPress pods to read/write media uploads or plugin data.

---

### Step 2: WordPress Deployment and Service

site-deploy.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deployment
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - name: wordpress
          image: wordpress:php7.1-apache
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql-svc:3306 # MySQL service name and port
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: secure-pass
                  key: password
          ports:
            - containerPort: 80
          volumeMounts:
            - name: wordpress-storage
              mountPath: /var/www/html # Mount path for WordPress files
      volumes:
        - name: wordpress-storage # must match the volume name in volume mount
          persistentVolumeClaim: 
            claimName: site-pvc # PersistentVolumeClaim to use for storage
```

```bash
kubectl apply -f site-deploy.yaml
kubectl get po
kubectl get deploy
```

> ℹ️ **Explanation:**  
>  deployment sets up the WordPress application container, connects it to MySQL, and mounts the EFS-backed volume so uploads persist.

---

site-svc.yaml:

```yaml
apiVersion: v1 
kind: Service
metadata:
  name: site-svc
spec:
  selector:
    app: wordpress
    tier: frontend
  ports:
    - port: 80 
  type: LoadBalancer # Expose the service externally


```

```bash
  kubectl apply -f site-svc.yaml
kubectl get svc
```


> ℹ️ **Explanation:**  
> it creates an external AWS ELB, which allows users to access your WordPress site publicly via DNS or IP.


---


## ✅ Access Website
Open browser:
    • http://<NodePublicIP>:31306
    • or http://<AWS LoadBalancer DNS>
> ⚠️ Ensure security group for EC2 allows TCP port `31306`.

---


🧹 Cleanup

```bash
kubectl delete -f deploysite-project
eksctl delete cluster --name project-eks