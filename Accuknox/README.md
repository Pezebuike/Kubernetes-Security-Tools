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

[![Watch the image](/Accuknox/images/1.png)]


# Install kubearmor 

```
karmor install
```

# Output of karmor install

[![Watch the image](/Accuknox/images/2.png)]

# Deploying sample app & policies 
- We are creating a sample application and applying different policies to audit or block at pod level 
multiubuntu app deployment 
```
kubectl apply -f https://raw.githubusercontent.com/kubearmor/KubeArmor/main/examples/multiubuntu/multiubuntu-deployment.yaml
```

Output:
 
[![Watch the image](/Accuknox/images/3.png)]
[![Watch the image](/Accuknox/images/4.png)]
 
# Deploy sample policies
``` 
kubectl apply -f https://raw.githubusercontent.com/kubearmor/KubeArmor/main/examples/multiubuntu/security-policies/ksp-group-1-proc-path-block.yaml
```


Output:
[![Watch the image](/Accuknox/images/5.png)]

 
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

[![Watch the image](/Accuknox/images/6.png)]
[![Watch the image](/Accuknox/images/7.png)]
 
 
# Getting Alerts from Kubearmor - Enable port-forwarding for kubearmor relay ( if needed)
```
kubectl port-forward -n kube-system svc/kubearmor 32767:32767
```

# Observing the logs using karmor cli ( karmor log only works on the worker nodes) 
```
karmor log 
```

# Sample output 
 
[![Watch the image](/Accuknox/images/8.png)]







# Setup sample wordpress-mysql deployment for Policy testing
- Deploy wordpress-mysql deployment by using the below command;
```
kubectl apply -f https://raw.githubusercontent.com/kubearmor/KubeArmor/main/examples/wordpress-mysql/wordpress-mysql-deployment.yaml
```
[![Watch the image](/Accuknox/images/9.png)]
[![Watch the image](/Accuknox/images/10.png)]

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
[![Watch the image](/Accuknox/images/11.png)]

- **Step 3** Login to mysql pod using
``` 
kubectl exec -it mysql-dc4f8d4b6-rxrl9 -n wordpress-mysql bash
```
[![Watch the image](/Accuknox/images/12.png)]


- When we create a file in **/var/lib/mysql folder**, we will be **getting the alert triggered in karmor logs**.
[![Watch the image](/Accuknox/images/13.png)]


- Note: By adding additional folders which we need logging option, we should add them into the policy and monitor the folders. 
- We can change the policy to Block, this will deny the permission to create the files/folders in the mentioned folder structure.
 
[![Watch the image](/Accuknox/images/14.png)]
[![Watch the image](/Accuknox/images/15.png)]

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
[![Watch the image](/Accuknox/images/16.png)]

- Post applying this policy, we will be getting permission denied error.

[![Watch the image](/Accuknox/images/17.png)]


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
[![Watch the image](/Accuknox/images/18.png)]
[![Watch the image](/Accuknox/images/19.png)]
[![Watch the image](/Accuknox/images/20.png)]

 

 


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
[![Watch the image](/Accuknox/images/21.png)]
[![Watch the image](/Accuknox/images/22.png)]
