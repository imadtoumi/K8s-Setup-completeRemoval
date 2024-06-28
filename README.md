# K8s-Setup-completeRemoval

# Setting Up Self-Hosted Kubernetes Cluster

## Step 1: Initial Setup

### On Master and Worker Nodes

1. Create a script named `1.sh` on each node and paste the following content:

    ```bash
    #!/bin/bash

    # Update package lists
    sudo apt-get update

    # Install Docker
    sudo apt install docker.io -y
    sudo chmod 666 /var/run/docker.sock

    # Install dependencies for Kubernetes
    sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
    sudo mkdir -p -m 755 /etc/apt/keyrings

    # Add Kubernetes apt repository and key
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

    # Update package lists again
    sudo apt update

    # Install Kubernetes components
    sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
    ```

2. Make the script executable and execute:

    ```bash
    sudo chmod +x 1.sh
    ./1.sh
    ```

## Step 2: Setting Up Master Node

### On Master Node Only

1. Create a script named `2.sh` and paste the following content:

    ```bash
    #!/bin/bash

    # Initialize Kubernetes master
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16

    # Configure kubectl for current user
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    # Deploy Calico network plugin
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

    # Deploy NGINX Ingress Controller
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
    ```

2. Make the script executable and execute:

    ```bash
    sudo chmod +x 2.sh
    ./2.sh
    ```

## Step 3: Joining Worker Node(s)
```bash
eg; kubeadm join 172.31.35.0:6443 --token x4c4jj.335cm3o787lgn1fo \
        --discovery-token-ca-cert-hash sha256:79f7c7f3b19fa6aa827c692c46eacb815db68b77b2bb56404d6efabc1ea4482b
```

## Complete removal

1. Reset Kubernetes and Uninstall Kubernetes Packages

    ```bash
    kubeadm reset
    sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
    sudo apt-get autoremove  
    sudo rm -rf ~/.kube
    ```

2. Clean file sytem and free up space

   ```bash
    sudo umount /run/containerd/io.containerd.runtime.v2.task/k8s.io/*
    sudo rm -rf /run/containerd/io.containerd.runtime.v2.task/k8s.io/*
    sudo umount /run/containerd/io.containerd.grpc.v1.cri/sandboxes/*
    sudo rm -rf /run/containerd/io.containerd.grpc.v1.cri/sandboxes/*
    ```
If an error occure of this mounted system files are in use, force the unmount. 
   
