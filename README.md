# K8s-Setup-completeRemoval

# Setting Up Self-Hosted Kubernetes Cluster

## Step 1: Initial Setup

### On Master and Worker Nodes

1. Create a script named `1.sh` on each node and paste the following content (this will install conatinerd since we use it as a container run time):

    ```bash
    # sysctl params required by setup, params persist across reboots / Enabling IPV4 Forwarding.
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.ipv4.ip_forward = 1
    EOF
    
    # Apply sysctl params without reboot
    sudo sysctl --system
    
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    
    # Add the repository to Apt sources:
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    
    # Install containerd
    sudo apt-get install containerd.io

    # Configure systemd for containerd
    containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sed 's/sandbox_image = "registry.k8s.io\/pause:3.6"/sandbox_image = "registry.k8s.io\/pause:3.9"/' | sudo tee /etc/containerd/config.toml

    sudo systemctl restart containerd
    ```

2. Create now a script to install K8S components (Kubelet, Kubeadm, Kubectl (this is not needed in a worker node) ) :

    ```bash
    sudo apt-get update
    # apt-transport-https may be a dummy package; if so, you can skip that package
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg

    # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
    # sudo mkdir -p -m 755 /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

    sudo systemctl enable --now kubelet
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

    # Deploy Calico network plugin or any plugin you want to use (in this installation i installed calico due to its setup simplicity)
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

    # Deploy NGINX Ingress Controller
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
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
   
