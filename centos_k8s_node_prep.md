### CentOS node prep for Kubernetes

These are the general steps for preparing a CentOS 7 server to be used as a Kubernetes node (control plane or worker). This document assumes there is no proxy between the workstation and the internet, if there is, you must configure `/etc/environment`, `/etc/profile.d/proxy.sh`, and Docker services. You can find an [example proxy configuration here.](https://github.com/c-snell/Documentation/blob/master/kubespray_install_proxy.md#optional-set-system-level-proxy-on-all-nodes-masterworkersload-balancers)

```
yum update -y
systemctl stop firewalld.service && systemctl disable firewalld.service
```

Verify the following packages are installed:
```
yum install python
yum install -y iscsi-initiator-utils device-mapper-multipath
yum install -y yum-utils device-mapper-persistent-data lvm2
```

Disable `swap` during current session:
```
swapoff -a
```

Disable `swap` partition permanently by commenting out `swap` partition in `/etc/fstab`:
```
vi /etc/fstab

#
# /etc/fstab
# Created by anaconda on Tue Feb  9 10:14:30 2021
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=d6b54d79-e021-4948-9de9-2a03a1c7c4f3 /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

Disable SELinux:
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Reboot.

The node prep is now complete. It is recommended to take a snapshot. The rest of this guide will walk you through the steps of configuring a single node Kubernetes deployment. 

You can use this script to finish the deployment:

```
## Install Single node Kubernetes cluster. 
## Usage: 
### Update the variable "k8s_version" with the Kubernetes version to be deployed
###
### Deploy cluster:  ./install_k8s.sh -install 
### Destroy cluster: ./install_k8s.sh -remove
###

arg=$1

# version of Kubernetes to deploy
k8s_version=1.20.1


if [[ $arg = "-remove" ]]
then
  read -p "Are you sure? "
  echo
  if [[ $REPLY =~ ^[Yy]$ ]]
  then
    # do dangerous stuff
    kubeadm reset
    rm ~/.kube/config
  fi
elif [[ $arg = "help" ]]
then
  echo "Specify -install to install latest version of Kubernetes with kubeadm or -remove to uninstall an existing deployment"
elif [[ $arg = "-install" ]]
then

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  yum install -y docker-ce-19.03.14-3.el7 docker-ce-cli-19.03.14-3.el7 containerd.io-19.03.14-3.el7
  systemctl start docker && systemctl enable docker
  yum install -y kubelet-$k8s_version-0 kubeadm-$k8s_version-0 kubectl-$k8s_version-0 --disableexcludes=kubernetes
  systemctl enable --now kubelet
  kubeadm init --kubernetes-version=$k8s_version --pod-network-cidr=192.168.0.0/16
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  kubectl apply -f https://docs.projectcalico.org/v3.17/manifests/calico.yaml
  kubectl taint nodes --all node-role.kubernetes.io/master-
  echo "Sleeping for 60 seconds for all pods to initialize"
  sleep 60
  kubectl get pods -A
else
  echo "Specify -install to install latest version of Kubernetes with kubeadm or -remove to uninstall an existing deployment"
fi
```

Make script executable:
```
chmod +x install_k8s.sh
```

Deploy Kubernetes
```
./install_k8s.sh -install
```

### Manual deployment

Add Kubernetes repo
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

Install kubelet, kubeadm, kubectl and start kubelet service:
```
yum install -y kubelet-1.20.1-0.x86_64 kubeadm-1.20.1-0.x86_64 kubectl-1.20.1-0.x86_64 --disableexcludes=kubernetes
systemctl enable --now kubelet
```

Add Docker repo and install Docker:
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl start docker && systemctl enable docker
```

Verify Docker is working:
```
docker run hello-world
```

>Note: There is a limit on the number of Docker images that can be pulled (100 for anonymous usage or 200 with a free Docker account). [Docker Limits](https://www.docker.com/increase-rate-limits)

Pre-pull Kubernetes images (Optional):
```
kubeadm config images pull
```

At this point, you can take a snapshot of the VM in case you mess something up at a later stage.

Deploy Kubernetes.
```
kubeadm init --pod-network-cidr=192.168.0.0/16
```

Create the .kube directory and config file:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

Deploy the Kubernetes Container Network Interface (CNI): 
```
kubectl apply -f https://docs.projectcalico.org/v3.17/manifests/calico.yaml
```

All nodes should show in Ready state:
```
kubectl get nodes
```

Since this is a single node cluster, we need to apply a taint to the Master node so the control plane will schedule workloads onto it. If you don't do this then your application deployments will stay in a pending state.
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Enable bash tab completion for `kubectl` (Optional but highly recommended)
```
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash >/etc/bash_completion.d/kubectl
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

With Kubernetes deployed, we can install Helm, a package manager for Kubernetes as we will be using it for our exercises. 

[Installing Helm](https://helm.sh/docs/intro/install/)
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

Verify it installed correctly:
```
helm version
version.BuildInfo{Version:"v3.5.2", GitCommit:"167aac70832d3a384f65f9745335e9fb40169dc2", GitTreeState:"dirty", GoVersion:"go1.15.7"}
```
