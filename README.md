# Kubernetes Setup on AWS EC2 with Calico Networking

## Introduction

This guide provides instructions for setting up a Kubernetes (K8s) cluster on AWS EC2 instances using Calico as the network plugin. It also covers the installation of the Kubernetes Dashboard and the deployment of a basic application stack including front-end, back-end, Redis, Sidekiq, and PostgreSQL.

## Prerequisites

- AWS account with EC2 access
- Basic understanding of Kubernetes concepts

## 1. Setting Up the Kubernetes Cluster

### 1.1. Launching EC2 Instances  

We will first create an EC2 instance on which we will build, and configure our cluster (Master Node).  
Once Master node setup is completed we can make ann AMI of it and worker node's will be created from that.  

- Login to the AWS console
- Switch to Services -> EC2 -> Launch Instance
- Select the ‘Ubuntu- 20.4’ image and give the Name "k8's-master-node"
  
![image](https://github.com/Cloud-Automation-Partner/Kubernetes_Cluster_Deploymet_AWS/assets/151637997/f6595b9f-113e-4fab-8006-bd6579bd59aa)  

Next,

Select the instance type as shown in the image.  
Create a key-value pair (You can click Create new key pair or select an existing one.  

![image](https://github.com/Cloud-Automation-Partner/Kubernetes_Cluster_Deploymet_AWS/assets/151637997/85488923-299e-4bdd-b9d9-5df8a6d0c575)

Create a new security group as shown below or select any exxisting security group. We need to open all traffic port to make the Kubernetes setup work on an EC2 instance.

![image](https://github.com/Cloud-Automation-Partner/Kubernetes_Cluster_Deploymet_AWS/assets/151637997/a76e36d9-1dd9-4265-9f2d-08c17cfad5d7) 

Give 30GB as instance storage, For Master node, we have to give sufecient storage here.  

![image](https://github.com/Cloud-Automation-Partner/Kubernetes_Cluster_Deploymet_AWS/assets/151637997/27c199cb-0881-4c7a-b584-8905a4a7098f)

Click launch innstance and wait for the instance creation and then SSH into the instance.  

### 1.2. Installing Kubernetes required Packages

- Change the current user to the root user
```bash
sudo su  -
```
- Update packages and their version
```bash
sudo apt update
sudo apt-get update && sudo apt-get upgrade -y
```
- Install kubelet, kubeadm and kubectl
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
- Verify the Packages Installation
```bash
kubectl version --client && kubeadm version
```
- Disable Firewall & Swap Memory
```bash
ufw disable
swapoff -a
sudo sed -i '/swap/d' /etc/fstab
free -h
```
### 1.3. Install Containerd as a conntainer Runtime
  
To run containers in Pods, Kubernetes uses a container runtime  

- Configure persistent loading of modules
```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
- Load at runtime
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```
- Ensure sysctl params are set
```bash
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
- Reload configs
```bash
sudo sysctl --system
```
- Install required packages
```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```
- Add Docker repo
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
- Install containerd
```bash
sudo apt update
sudo apt install -y containerd.io
```
- Configure & Restart containerd
```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd
```
***Note:*** From this point you can create an AMI of the iinstance so that you can spin the worker nodeds from it.  

### 1.4. Initialize the Master Node  

- Enable kubelet service
```bash
sudo systemctl enable kubelet
```
- Pull container images
```bash
sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock
```
- Initialize Kubernetes Cluster
```bash
kubeadm init --apiserver-advertise-address=YOUR_INSTANCE_PRIVATE_IP --pod-network-cidr=192.168.0.0/16 --ignore-preflight-errors=all
```
Now the Kubernetes Cluster has been initialised successfully and soon it will print the   

- To start using your cluster run below commandd  as root user
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
- Check Nodde status
```bash
kubectl get nodes -o wide
```

### 1.5. Setting Up Calico Networking
- Deploy the Calico network
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
kubectl get pods --all-namespaces
kubectl get nodes -o wide
```
The cluster will be in the Ready state now once network plugins  are installed and we are good to add the worker nodes to it.
### 1.6. Add worker nodes  

Once the control plane is complete, you can add worker nodes to the cluster to run scheduled workloads.

Use the output from kubeadm token create command from the master server

- Get comman dto join cluster
```bash
kubeadm token create --print-join-command
```
- Run the output command in your worker nodes
```bash
kubeadm join 172.31.1.17:6443 — token exmkm6.v18t1dkyyu0nte89 — discovery-token-ca-cert-hash sha256:1b135f929b6855ecd6ad9358490a3e9bee8d27582c6babac438d4b6f42a3c717
```
- Verify the Kubernetes Cluster Nods
```bash
kubectl get nodes -o wide
```
***That means our K8's cluster is ready***  
![image](https://github.com/Cloud-Automation-Partner/Kubernetes_Cluster_Deploymet_AWS/assets/151637997/7ea4bde4-77ea-4c3f-aa42-2b7e1fa209e4)

## 2. Installing the Kubernetes Dashboard

1. Run `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml`.
2. Access the dashboard using `kubectl proxy`.

## 3. Deploying Applications

### 3.1. Front-End and Back-End Deployment

1. Create Docker images for front-end and back-end.
2. Push them to a registry (e.g., Docker Hub).
3. Create deployment YAML files for front-end and back-end.
4. Deploy using `kubectl apply -f <deployment_yaml>`.

### 3.2. Redis Deployment

1. Deploy Redis using `kubectl apply -f <redis_deployment_yaml>`.
2. Ensure proper networking and storage configurations.

### 3.3. Sidekiq Deployment

1. Deploy Sidekiq as a background job processor.
2. Configure it to work with the back-end.

### 3.4. PostgreSQL Deployment

1. Deploy PostgreSQL using `kubectl apply -f <postgres_deployment_yaml>`.
2. Set up persistent storage and configure the database.

## Conclusion

This guide provides a basic setup for running a Kubernetes cluster on AWS EC2 with Calico networking, including a typical web application stack. For detailed configurations and advanced setups, refer to the official Kubernetes and AWS documentation.

---

**Note:** Replace placeholders like `<calico_manifest_url>`, `<deployment_yaml>`, `<redis_deployment_yaml>`, and `<postgres_deployment_yaml>` with actual URLs or file paths.
