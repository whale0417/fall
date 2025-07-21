# Installing Kubernetes on Ubuntu 22.04 with Containerd

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Update the System](#1-update-the-system)
3. [Disable Swap](#2-disable-swap)
4. [Load Kernel Modules and Configure Sysctl](#3-load-kernel-modules-and-configure-sysctl)
5. [Install and Configure Containerd](#4-install-and-configure-containerd)
6. [Add Kubernetes Repository](#5-add-kubernetes-repository)
7. [Install Kubernetes Components](#6-install-kubernetes-components)
8. [Initialize the Kubernetes Control Plane (Master Node Only)](#7-initialize-the-kubernetes-control-plane-master-node-only)
9. [Set Up Pod Networking](#8-set-up-pod-networking)
10. [Join Worker Nodes to the Cluster](#9-join-worker-nodes-to-the-cluster)
11. [Verify the Cluster](#10-verify-the-cluster)
12. [Basic Troubleshooting Tips](#11-basic-troubleshooting-tips)

---

## Prerequisites

- **Operating System**: Ubuntu 22.04 LTS
- **User Privileges**: A user with `sudo` privileges
- **Network Configuration**: Ensure all nodes can communicate with each other
- **Unique Hostnames**: Each node should have a unique hostname

---

## 1. Update the System

Update package lists and upgrade installed packages to the latest versions.

```bash
    sudo apt update
    sudo apt upgrade -y
```

## 2. Disable Swap

Kubernetes requires swap to be disabled.

**emporarily disable swap**:

```bash
    sudo swapoff -a
```

**Permanently disable swap**:

Edit `/etc/fstab` file and comment out any swap entries:

```bash
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 3. Load Kernel Modules and Configure Sysctl

Load necessary kernel modules and configure system parameters.

-**Load kernel modules**:

```bash
    sudo modprobe overlay
    sudo modprobe br_netfilter
```

-**Persist kernel module settings**:

```bash
    sudo tee /etc/modules-load.d/k8s.conf <<EOF
    overlay
    br_netfilter
    EOF
```

### Set system parameters:

```bash
    sudo tee /etc/sysctl.d/k8s.conf <<EOF
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Apply sysctl parameters without reboot:

```bash
    sudo sysctl --system
```

## 4. Install and Configure Containerd

### Install Containerd

Install containerd from the official Ubuntu repositories.

```bash
  sudo apt update
  sudo apt install -y containerd
```

### Configure Containerd

Create configuration directory\*\*

```bash
  sudo mkdir -p /etc/containerd
```

Generate default configuration file:

```bash
  sudo containerd config default | sudo tee /etc/containerd/config.toml
```

**Set the cgroup driver to systemd**:

Edit `/etc/containerd/config.toml` and set `SystemdCgroup` to `true`.

```bash
  sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

**Restart and enable containerd service**:

```bash
  sudo systemctl restart containerd
  sudo systemctl enable containerd
```

## 5. Add Kubernetes Repository

Add the Kubernetes APT repository to install `kubeadm`, `kubelet`, and `kubectl`.

**Install dependencies**:

```bash
    sudo apt update
    # apt-transport-https may be a dummy package; if so, you can skip that package
    sudo apt install -y apt-transport-https ca-certificates curl
```

**Download the public signing key for the Kubernetes package repositories**:

```bash
  # If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
  # sudo mkdir -p -m 755 /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring
```

**Add Kubernetes APT repository. If you want to use Kubernetes version different than v1.31, replace v1.31 with the desired minor version in the command below:**:

```bash
  # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
  sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly
```

## 6. Install Kubernetes Components

Install `kubelet`, `kubeadm`, and `kubectl`.

```bash
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
```

Prevent these packages from being automatically updated:

```bash
  sudo apt-mark hold kubelet kubeadm kubectl
```

## 7. Initialize the Kubernetes Control Plane (Master Node Only)

Perform this step **only** on the master node.

**Initialize the cluster**:

```bash
  sudo kubeadm init \
    --pod-network-cidr=192.168.0.0/16 \
    --cri-socket=unix:///run/containerd/containerd.sock
```

- `--pod-network-cidr=10.244.0.0/16` is used for the Calico CNI plugin. Adjust if using a different CNI.

**Set up local kubeconfig**:

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 8. Set Up Pod Networking

Install a Container Network Interface (CNI) plugin so pods can communicate across the cluster.

### Install Calico:

**Install the Tigera Calico operator and custom resource definitions**:

```bash
  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
```

**Install Calico by creating the necessary custom resource**:

```bash
  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml
```

Note: Before creating this manifest, read its contents and make sure its settings are correct for your environment. For example, you may need to change the default IP pool CIDR to match your pod network CIDR.

**Confirm that all of the pods are running with the following command**:

```bash
  watch kubectl get pods -n calico-system
```

**Remove the taints on the control plane so that you can schedule pods on it**:

```bash
  kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

**Confirm that you now have a node in your cluster with the following command**:

```bash
  kubectl get nodes -o wide
```

## 9. Join Worker Nodes to the Cluster

Perform this step on each worker node.

**Use the `kubeadm join` command from step 7**.

It should look similar to:

```bash
  sudo kubeadm join <master_node_ip>:6443 --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash> \
      --cri-socket unix:///run/containerd/containerd.sock
```

**If you lost the `kubeadm join` command**:

On the master node, generate a new token and get the join command:

```bash
  sudo kubeadm token create --print-join-command
```

## 10. Verify the Cluster

On the master node, check the status of the nodes:

```bash
    kubectl get nodes
```

You should see all your nodes with the `STATUS` as `Ready`.

## 11. Basic Troubleshooting Tips

**Check the status of pods**:

```bash
    kubectl get pods --all-namespaces
```

**View logs of a pod**:

```bash
    kubectl logs <pod_name> -n <namespace>
```

**Restart kubelet service if needed**:

```bash
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
```
