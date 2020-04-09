# Setup

Steps to configure a single node kubernetes cluster with Kubeflow on Power.

* These instructions assume knowledge of PPC64LE, Linux, Kubernetes and Kubeflow *

> These instructions are intended to provide instruction for implementation of an AC922 by a senior technical member. Per mutual agreement, it is provided for informational purposes and is confirmed to be correct at the time of sending only.

> Due to the nature of the Open Source ecosystem, for these instructions to remain usable into the future, all relevant software should be stored securely and used for future installs. No copies of any referenced software will be kept, and no support will be provided in the event that these instructions become invalid due to changes to repositories or compatibility.

## Operating System

Bare Metal install of Ubuntu Server 18.04 LTS for PPC64LE

Perform a full system upgrade

```shell
sudo apt-get update
sudo apt-get dist-upgrade
sudo reboot
```

### Install Nvidia GPU Driver

Install the GPU driver by following these steps:

Download the NVIDIA GPU driver.

```shell
Go to https://www.nvidia.com/Download/index.aspx

Select Product Type: Tesla
Select Product Series: V-Series (for Tesla V100).
Select Product: Tesla V100.
Select Operating System, click Show all Operating Systems, then choose the correct value: Linux POWER LE Ubuntu 18.04 for POWER
Select CUDA Toolkit: 10.2
Click SEARCH to go to the download link.
Click Download to download the driver.
```

The driver file name is `NVIDIA-Linux-ppc64le-440.87.01.run`. Give this file execute permission and execute it on the Linux image where the GPU driver is to be installed. When the file is executed, you are asked two questions. It is recommended that you answer "Yes" to both questions. If the driver fails to install, check the `/var/log/nvidia-installer.log` file for relevant error messages.

Install the GPU driver repository and cuda-drivers:

```shell
sudo dpkg -i nvidia-driver-local-repo-ubuntu1804-440.*.deb

sudo apt-key add /var/nvidia-driver-local-repo-440.*/*.pub

sudo apt update

sudo apt install cuda-drivers -y
```

Set nvidia-persistenced to start at boot

```shell
sudo systemctl enable nvidia-persistenced
sudo reboot
```

After reboot, confirm the GPU drivers have loaded by issuing `nvidia-smi`

## Kubernetes Configuration

### Configure Kubernetes & Docker Repositories

```shell
sudo apt update && sudo apt install -y apt-transport-https curl ca-certificates software-properties-common nfs-common
```

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb https://apt.kubernetes.io/ kubernetes-xenial main"
```

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

```shell
sudo add-apt-repository \
  "deb [arch=ppc64el] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

```shell
sudo apt update
```

### Disable Swap

```shell
sudo swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

The `/etc/fstab` file should now be commentted out for the swap mount point

### Install Docker packages

```shell
sudo apt-get install docker-ce=18.06.1~ce~3-0~ubuntu
```

### Hold Docker Version

```shell
sudo apt-mark hold docker-ce=18.06.1~ce~3-0~ubuntu
```

### Modify Docker to use systemd driver and overlay2 storage driver

Edit or create `/etc/docker/daemon.json` file

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}

sudo mkdir -p /etc/systemd/system/docker.service.d
```

### Restart docker

```shell
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### Install Nvidia-docker2 plugin

```shell
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt update
sudo apt install nvidia-docker2 -y
```

Modify `/etc/docker/daemon.json` to include nvidia-docker2 runtime.

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "default-runtime": "nvidia",
  "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
  }
}
```

### Restart docker again

```shell
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### Test to ensure nvidia-docker2 plugin works

```shell
sudo docker run --rm nvidia/cuda-ppc64le nvidia-smi
```

### Install and hold Kubernetes Packages

```shell
sudo apt install -y kubelet=1.15.11-00 kubeadm=1.15.11-00 kubectl=1.15.11-00 && sudo apt-mark hold kubelet=1.15.11-00 kubeadm=1.15.11-00 kubectl=1.15.11-00
```

### Initialize Kubernetes node on the master node

Create a file labelled `kubeadm-config.yaml`

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.15.11
networking:
  podSubnet: 192.168.0.0/16 # Be careful this subnet does not clash with your existing server subnet, modify if it does.
  serviceSubnet: 10.96.0.0/12
apiServer:
  extraArgs:
    "service-account-issuer": "kubernetes.default.svc"
    "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
```

Initialize with `kubeadm`

```shell
sudo kubeadm init --config kubeadm-config.yaml
```

### Allow non-root user to run kubectl and configure Kubernetes

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Configure as a single node cluster

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Apply a CNI

Now add a CNI (Container Network Interface) plugin.

```shell
sed -i 's/^#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sysctl -p /etc/sysctl.conf

# If you needed to modify your podSubnet range, then you need to download the calico.yaml file and modify it with the same range you've chosen for your podSubnet.

kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```

### Set up Helm V3

```shell
sudo curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
sudo chmod 700 get_helm.sh
sudo ./get_helm.sh
```

### Install metalLB for Bare-Metal LB

Apply MetalLB deployment

```shell
kubectl create ns metallb-system
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.2/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Create a MetalLB ConfigMap called `metallb-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: metallb-config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - x.x.x.x-x.x.x.x
---
```

Apply the MetalLB ConfigMap

```shell
kubectl create -f metallb-configmap.yaml
```

You can edit the ConfigMap after deployment by simply editing the configmap

```shell
kubectl edit configmap config -n metallb-system
```

### Enable GPU Support within Kubernetes

Install Nvidia DevicePlugin DaemonSet

```shell
wget https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta5/nvidia-device-plugin.yml
```

Edit the file and replace `image: nvidia/k8s-device-plugin:1.0.0-beta5` with `image: nvidia/k8s-device-plugin:1.11`

```shell
kubectl create -f nvidia-device-plugin.yml
```

## KubeFlow for Machine Learning

### Install Istio

Download the istio

```shell
wget https://raw.githubusercontent.com/benswinney/oss/master/ibm-istio.tar.gz
```

Uncompress and extract the file

```shell
tar zxvf ibm-istio.tar.gz
```

Install Istio Pre-Reqs

```shell
cd ibm-istio
kubectl create -f istio-psp.yaml
kubectl create -f istio-clusterrole.yaml 
kubectl apply -f ../ibm-istio/additionalFiles/crds/crd-10.yaml
kubectl apply -f ../ibm-istio/additionalFiles/crds/crd-11.yaml
kubectl apply -f ../ibm-istio/additionalFiles/crds/crd-12.yaml
kubectl create ns istio-system
NAMESPACE=istio-system
helm install istio ../ibm-istio -n istio-system --set sidecarInjectorWebhook.enabled=false
```

### Install KubeFlow

Download kfctl for Power

```shell
wget https://raw.githubusercontent.com/benswinney/oss/master/kfctl_ppc64le.tar.gz
```

Uncompress and extract the file

```shell
tar zxvf kfctl_ppc64le.tar.gz
```

Add to `kfctl` to PATH

```shell
export PATH=$PATH:"<path to kfctl>"
```

Download kubeflow-0.6.2

```shell
wget https://raw.githubusercontent.com/benswinney/oss/master/kubeflow-0.6.2.tar.gz
```

Uncompress and extract the file

```shell
tar zxvf kubeflow-0.6.2.tar.gz
cd test-kubeflow
```

Edit the `app.yaml` file to point to the full path for test-kubeflow, replace full-path.

Apply `app.yaml` and monitor installation.

```shell
kfctl apply all -V
```

### Modify seldon-core-operator

```shell
kubectl edit StatefulSet/seldon-operator-controller-manager -n kubeflow

Replace image with docker.io/adamjm32/seldon-core-operator:0.4.1-SNAPSHOT
```

### Get Kubeflow dashboard IP Address

```shell
kubectl get svc istio-ingressgateway -n istio-system

Look for IP Address allocated under EXTERNAL-IP
```
