# Accuknox

- AccuKnox is a provider of zero-trust kubernetes security solutions. AccuKnox is advancing a Zero Trust run-time Kubernetes security platform that leverages an identity-driven approach.

- Open source tools to understand application behavior and prevent runtime data exposure. Automatically monitor and enforce behavior by automating security policies for AppArmor and SELinux.

# Why Accuknox & Key use cases

- Understand & visualize the behavior of your workload at runtime to detect 3rd party software issues.
- Adopt zero trust principles and generate least privilege security policies that can be applied to enable runtime security and enforcement at the pod and process level.
- Protect your Kubernetes and VM workloads from malicious behavior due to supply chain issues, un-patched vulnerabilities and zero day attacks.


# AccuKnox Usecase 1

- Before starting this exercise, make sure you have two running Ubuntu Linux machines.
- Create two Ubuntu Linux machines in AZURE Cloud [or] AWS Cloud [or] In your local laptop using VMware or oracle virtual box or Hyper-V workstation
- Make sure you opened SSH, HTTP, and 8080 ports for this new VM’s
- Install Kubernetes on both the VMs


# Kubernetes installation step by step



# Update the repository on both servers
```
apt-get update
```

# Disable the SWAP partition on both server
```
sudo sed -i '/swap/d' /etc/fstab
swapoff -a
```

# verifying
```
cat /etc/fstab
```

# Set the Hostname

```
sudo hostnamectl set-hostname <host name>
```

# Update the hostname on both the servers (/etc/hosts)

```
vim /etc/hosts
Kubernet-master01
Kubernet-Worker01
```


# Note the IP address Should be static

Reboot the servers
Install Docker on both servers


sudo apt update
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker


# Run the following command to run before installing the Kubernetes
```
sudo apt install apt-transport-https curl
```

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

```
mkdir -p /etc/systemd/system/docker.service.d

systemctl daemon-reload
systemctl restart docker
Install kubeadm kubelet & kubectl on Both the server
sudo apt install kubeadm kubelet kubectl kubernetes-cni
```

# Run these on the master node:

```
sudo kubeadm init
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# Install network plugin on Master (This will create a virtual SDN with Kubernetes cluster to manage the pod network traffic & management)

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

# verifying the cluster status 

```
watch kubectl get pods --all-namespaces
kubectl get nodes -o wide
```

# Run this command in worker node to add the node into Cluster:

```
kubeadm join 172.31.22.120:6443 --token wtqrws.3bz6oakilgia9zup \
        --discovery-token-ca-cert-hash sha256:393595216239850e8460484af5b0e7826b                                                                             a6ae79df7ba4e3ab2758abf2d56bc5
```

**172.31.22.120:6443**
**wtqrws.3bz6oakilgia9zup --discovery-token-ca-cert-hash sha256:393595216239850e8460484af5b0e7826ba6ae79df7ba4e3ab2758abf2d56bc5**

- Note: highlighted text need to be replaced with the information received from your cluster.

# Run the command in master node to get the cluster join command.
```
kubeadm token create --print-join-command

```
# Kubearmor 

- KubeArmor is a cloud-native runtime security enforcement system that restricts the behaviour (such as process execution, file access, and networking operation of containers and nodes (VMs) at the system level.


# Download and install karmor cli-tool
```
curl -sfL https://raw.githubusercontent.com/kubearmor/kubearmor-client/main/install.sh | sudo sh -s -- -b /usr/local/bin
```

``` output (just reference purpose only)
root@ip-10-211-137-187:~# curl -sfL https://raw.githubusercontent.com/kubearmor/kubearmor-client/main/install.sh | sudo sh -s -- -b /usr/local/bin
```

[![Watch the image](/1.png)]


# Install kubearmor 

```
karmor install
```

# Output of karmor install

[![Watch the image](/2.png)]

# Deploying sample app & policies 
- We are creating a sample application and applying different policies to audit or block at pod level 
multiubuntu app deployment 
```
kubectl apply -f https://raw.githubusercontent.com/kubearmor/KubeArmor/main/examples/multiubuntu/multiubuntu-deployment.yaml
```

Output:
 
[![Watch the image](/3.png)]
[![Watch the image](/4.png)]
 
# Deploy sample policies
``` 
kubectl apply -f https://raw.githubusercontent.com/kubearmor/KubeArmor/main/examples/multiubuntu/security-policies/ksp-group-1-proc-path-block.yaml
```


Output:
[![Watch the image](/5.png)]

 
- This sample policy blocks the execution of sleep commands in ubuntu-1 pods.

# Simulate policy violation

```
$ POD_NAME=$(kubectl get pods -n multiubuntu -l "group=group-1,container=ubuntu-1" -o jsonpath='{.items[0].metadata.name}') && kubectl -n multiubuntu exec -it $POD_NAME -- bash
```
```
# sleep 1
```

**(Permission Denied)**

- Output of Policy Simulation:

[![Watch the image](/6.png)]
[![Watch the image](/7.png)]
 
 
# Getting Alerts from Kubearmor - Enable port-forwarding for kubearmor relay ( if needed)
```
kubectl port-forward -n kube-system svc/kubearmor 32767:32767
```

# Observing the logs using karmor cli ( karmor log only works on the worker nodes) 
```
karmor log 
```

# Sample output 
 
[![Watch the image](/8.png)]







# Setup sample wordpress-mysql deployment for Policy testing
- Deploy wordpress-mysql deployment by using the below command;
```
kubectl apply -f https://raw.githubusercontent.com/kubearmor/KubeArmor/main/examples/wordpress-mysql/wordpress-mysql-deployment.yaml
```
[![Watch the image](/9.png)]
[![Watch the image](/10.png)]

# Security Policy for Auditing Directories:
- **Step 1** Create a yaml file with the name **ksp-mysql-audit-dir.yaml**
```
vim  ksp-mysql-audit-dir.yaml
```

apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: ksp-mysql-audit-dir
  namespace: wordpress-mysql
spec:
  severity: 5
  selector:
    matchLabels:
      app: mysql
  file:
    matchDirectories:
    - dir: /var/lib/mysql/
      recursive: true

      # touch /var/lib/mysql/test

  action:
    Audit


- **Step 2** Run kubectl apply command

```
kubectl apply -f ksp-mysql-audit-dir.yaml
```
[![Watch the image](/11.png)]

- **Step 3** Login to mysql pod using
``` 
kubectl exec -it mysql-dc4f8d4b6-rxrl9 -n wordpress-mysql bash
```
[![Watch the image](/12.png)]


- When we create a file in **/var/lib/mysql folder**, we will be **getting the alert triggered in karmor logs**.
[![Watch the image](/13.png)]


- Note: By adding additional folders which we need logging option, we should add them into the policy and monitor the folders. 
- We can change the policy to Block, this will deny the permission to create the files/folders in the mentioned folder structure.
 
[![Watch the image](/14.png)]
[![Watch the image](/15.png)]

# Security Policy for Blocking access to config files using selecting commands (eg: cat)

- **Step 1** Create a yaml file with the name ksp-wordpress-block-config.yaml
```
vim  ksp-wordpress-block-config.yaml
```

apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: ksp-wordpress-block-config
  namespace: wordpress-mysql
spec:
  severity: 10
  selector:
    matchLabels:
      app: wordpress
  file:
    matchPaths:
    - path: /var/www/html/wp-config.php
      fromSource:
      - path: /bin/cat

      # http://[NodeIP]:30080
      # cat /var/www/html/wp-config.php
 
  action:
    Block

- Note: This policy will restrict the access to path /var/www/html/wp-config.php using cat commands, we can add additional folders and the tool which should be restricted.

- **Step 2** Run kubectl apply command
```
kubectl apply -f ksp-wordpress-block-config.yaml
```
[![Watch the image](/16.png)]

- Post applying this policy, we will be getting permission denied error.

[![Watch the image](/17.png)]


# Security Policy for Blocking access to execute any process on the pod
- **Step 1** Create a yaml file with the name ksp-wordpress-block-process.yaml
```
vim  ksp-wordpress-block-process.yaml
```
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: ksp-wordpress-block-process
  namespace: wordpress-mysql
spec:
  severity: 3
  selector:
    matchLabels:
      app: wordpress
  process:
    matchPaths:
    - path: /usr/bin/apt
    - path: /usr/bin/apt-get

      # apt update
      # apt-get update

  action:
    Block

- Note: This policy will restrict the user to execute any process on the pod. **Eg: apt-get, apt install “package”**

- **Step 2** Run kubectl apply command
```
kubectl apply -f ksp-wordpress-block-process.yaml
```
[![Watch the image](/18.png)]
[![Watch the image](/19.png)]
[![Watch the image](/20.png)]

 

 


# Security Policy for Blocking access to Service Account/token

- **Step 1** Create a yaml file with the name ksp-wordpress-block-sa.yaml
```
vim  ksp-wordpress-block-sa.yaml
```

apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: ksp-wordpress-block-sa
  namespace: wordpress-mysql
spec:
  severity: 7
  selector:
    matchLabels:
      app: wordpress
  file:
    matchDirectories:
    - dir: /run/secrets/kubernetes.io/serviceaccount/
      recursive: true

      # cat /run/secrets/kubernetes.io/serviceaccount/token
      # curl https://$KUBERNETES_PORT_443_TCP_ADDR/api --insecure --header "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)"

  action:
    Block

- Note: This policy will restrict the user to see the service account token/password from the pod.

- **Step 2** Run kubectl apply command
```
kubectl apply -f ksp-wordpress-block-sa.yaml
```
[![Watch the image](/21.png)]
[![Watch the image](/22.png)]


 








# Architecture

[![Watch the image](/architecture.png)]



# AZURE VM
 - Create 1 Rocky Linux 8.5 machine in the AZURE Cloud Portal
 - Note that this will work only in 'Pay-As-You-Go' Subscription and not on Free Tier
 - Make sure you open RDP,http,8080 ports for this new linux AZURE VM or you can also open any port by using port*any
 - Have Mobaxterm/ Putty installed. ** Note: We can use the PowerShell also, but in realtime production environment, Mobaxterm is the best option when we try to work on a junk server and push to the PROD.



# Steps

- Step 1:  Install AZURE CLI in Rocky Linux Virtual Machine
- Step 2:  First install docker in the local linux machine
- Step 3:  Download a sample application
- Step 4:  Test the sample application
- Step 5:  Deploy and use Azure Container Registry
- Step 6:  Install kubectl command 
- Step 7:  Deploy an Azure Kubernetes Service (AKS) cluster
- Step 8:  Run applications in Azure Kubernetes Service (AKS)
- Step 9:  Scale application in Azure Kubernetes Service (AKS)
- Step 10: Update the application in Azure Kubernetes Service (AKS)


#

# Step 1: Install AZURE CLI

- Add the azure CLI Repository keys
```
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
```
- Configure the azure CLI Repository

```
sudo sh -c 'echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'
```
- Install Azure CLI

```
sudo yum install azure-cli -y
```

- Login and sync your Azure CLI (Linux machine) to your portal.azure.com account

```
az login
```

- Now your linux machine has been fully authenticated with AZURE login.  

# Step 2 - first install docker in the local linux machine

```
 yum install epel-release -y
 yum repolist
```

## Step 2.1 - Install the required dependencies:

```
yum install yum-utils device-mapper-persistent-data lvm2  bash-completion -y
```

```
source /etc/profile.d/bash_completion.sh
```


## Step 2.2 - Add the stable Docker repository by typing:

```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

## Step 2.3 - Now that we have Docker repository enabled, we can install the latest version of Docker CE (Community Edition) using yum by typing:

```
yum install docker-ce --allowerasing -y

```

## Step 2.4 - Install Docker Compose and Test the Docker-Compose Version:


- Run the below command to download the current stable release of Docker compose.

```
curl -L "https://github.com/docker/compose/releases/download/1.26.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
```

- Apply the executable permission for the binary file which we have downloaded.

```
chmod +x /usr/bin/docker-compose
```

- If the docker compose is installed on a different location For example: /usr/local/bin/ , You can copy the executable to /usr/bin directory.

- You can check the version of docker-compose using the below command.

```
docker-compose --version
```


## Step 2.5 - Once the Docker package is installed, we start the Docker daemon with:

```
systemctl start docker;systemctl status docker;systemctl enable docker
```

## Step 2.6 - At the time of the writing of this article, the current stable version of Docker is 20.10.15, we can check our Docker version by typing:

```
docker -v
```


# Step 3: Download sample application

```
yum install git tree -y

```
- Download the sample azure application

```
git clone https://github.com/cloudnloud/azure-voting-app-redis.git
```

```
cd azure-voting-app-redis
```
- Build the docker image using docker compose

```
docker-compose up -d
```
- list the docker images

```
docker images
```

- list the running docker container

```
docker ps -a
```

# Step 4: Test sample application

- Test application locally

- To see the running application, enter http://localhost:8080 in a local web browser

- To see the running application, enter http://<your VM Public IP>:8080 in a local web browser

# Step 5: Deploy and use Azure Container Registry

- Create AZURE resource Group

```
az group create --name  cloudnloudrg --location eastus
```

- Create AZURE Container Registry Under the above Created Resource Group.


```
az acr create --resource-group cloudnloudrg --name cnlacr1 --sku Basic
```

- Login to the Azure Container registry


```
az acr login --name cnlacr1
```

- List the Docker Images


```
docker images
```

- List the no of ACR in your portal.azure.com


```
az acr list --resource-group cloudnloudrg --query "[].{acrLoginServer:loginServer}" --output table
```

- you will get the below output.So your AZURE Container Registry Name is below ...


```
cnlacr1.azurecr.io
```


- you need to change the docker image name towards to match your ACR repo name.Or else while you push the image will end up with error.


```
docker tag cloudnloud/azure-vote-front:v1 cnlacr1.azurecr.io/azure-vote-front:v1
```
- List the docker images

- Make sure you are seeing the docker image name with match to your ACR repository.

```
docker images
```
- Push your prepared custom image to your newly created ACR.

```
docker push cnlacr1.azurecr.io/azure-vote-front:v1
```

- list the ACR repository revisions

```
az acr repository list --name cnlacr1 --output table
```
- List your repository in ACR.

```
az acr repository show-tags --name cnlacr1 --repository azure-vote-front --output table
```


# Step 6: install kubectl command 

- Configure Kubernetes Software package repo.

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

- Install Kubectl

```
yum install -y kubectl
```

- To ensure all are ok the following command should work without any error

```
kubectl version --client
```

# Step 7: Deploy an Azure Kubernetes Service (AKS) cluster

- Create kubernetes Cluster with 1 worker Node

- In your learning setup,if you have project portal.azure.com test account then increase node count from 1 to 3.


```
az aks create \
    --resource-group cloudnloudrg \
    --name myAKSCluster \
    --node-count 1 \
    --generate-ssh-keys \
    --attach-acr cnlacr1
	
```
- Install AZURE AKS CLI

```
az aks install-cli
```
- Reterive the AKS cluster credentials.This will help kubectl command to run without any issues.

```
az aks get-credentials --resource-group cloudnloudrg --name myAKSCluster
```
- List all the nodes in Kubernetes cluster.

```
kubectl get nodes
```


# Step 8: Run applications in Azure Kubernetes Service (AKS)

- List the ACR 

```
az acr list --resource-group cloudnloudrg --query "[].{acrLoginServer:loginServer}" --output table
```
- Change your front end to your recently deployed own image repo.

```
vi azure-vote-all-in-one-redis.yaml
```


```
containers:
- name: azure-vote-front
  image: cnlacr1.azurecr.io/azure-vote-front:v1
```
- Create namespace in AKS k8s cluster.

```
kubectl create ns dev
```
- Apply the changes into AKS k8s cluster.

```
kubectl apply -f azure-vote-all-in-one-redis.yaml -n dev
```
- Monitor the change progress.

```
kubectl get service azure-vote-front --watch -n dev
```


- from the above command output get the EXTERNAL-IP and access it from browser


# Step 9: Scale applications in Azure Kubernetes Service (AKS)

- in another mobaxtreme window run the following command

```
watch -n 1 kubectl get all -o wide -n dev
```


- in another mobaxtreme window run the below commands

```
kubectl get pods -n dev
```
- in another mobaxtreme window run the below commands

```
kubectl scale --replicas=2 deployment/azure-vote-front -n dev
```

```
kubectl scale --replicas=4 deployment/azure-vote-front -n dev
```


# Step 10: Update an application in Azure Kubernetes Service (AKS)

- Now we need to make some change in application

```
vi azure-vote/azure-vote/config_file.cfg
```

```
## UI Configurations
TITLE = 'Save Cancer Children - Cloudnloud'
VOTE1VALUE = 'Learn'
VOTE2VALUE = 'Grow'
SHOWHOST = 'false'

```
- Build the Docker image with v2.
```
docker-compose up --build -d
```


- you need to change the docker image name towards to match your ACR repo name.Or else while you push the image will end up with error.


```
docker tag cloudnloud/azure-vote-front:v1 cnlacr1.azurecr.io/azure-vote-front:v2
```

- Push your prepared custom image to your newly created ACR.

```
docker push cnlacr1.azurecr.io/azure-vote-front:v2
```

- Scale your pods to 4 replicas
```
watch -n 1 kubectl get all -o wide -n dev
```
```
kubectl scale --replicas=4 deployment/azure-vote-front -n dev
```
- Set and prepare your deployment to new version 2
```
kubectl set image deployment azure-vote-front azure-vote-front=cnlacr1.azurecr.io/azure-vote-front:v2 -n dev
```

- Run below command
```
kubectl get service azure-vote-front -n dev
```
