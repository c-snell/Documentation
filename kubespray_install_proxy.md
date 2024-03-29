# Installation steps for Kubespray when working with Corporate proxy

These steps have come from many hours to troubleshooting the HPE Proxy and installation steps. These steps will ensure that you have the best chance of a successful install of Kubernetes. There are a number of components within the cluster that need to be able to communicate to ensure the cluster works.

Update the system and install Pip. Disable Firewall on all nodes.

```
yum update -y
yum install python3-pip -y
pip3 install --upgrade pip
systemctl stop firewalld && systemctl disable firewalld
```

### Optional: Set System level proxy on all nodes (master/workers/load balancers) 

Change domain and IP range as needed appropriate. `echo 192.168.1.{1..254}` is a wildcard to cover all IPs in your subnet. Some programs don't like CIDR notation. These proxy settings include the subnets used by Kubernetes and Docker so they aren't sent to the proxy and blackholed.

```
vi /etc/bashrc
```    

Example proxy settings
```
export http_proxy="http://proxy_ip:8080"
export https_proxy="http://proxy_ip:8080"
export no_proxy="127.0.0.1,localhost,.example.com,.cluster.local,.svc,localaddress,.localdomain.com,`echo 192.168.1.{1..254},`172.17.0.0/16,172.30.0.0/16"
```

```
vi /etc/environment
```

Example proxy setting
```
no_proxy=127.0.0.1,localhost,.example.com,.cluster.local,.svc,localaddress,.localdomain.com,`echo 192.168.1.{1..254},`172.17.0.0/16,172.30.0.0/16
http_proxy=http://proxy_ip:8080
https_proxy=http://proxy_ip:8080
```

```
vi /etc/profile.d/proxy.sh
```

Example proxy setting
```
export http_proxy="http://proxy_ip:8080"
export https_proxy="http://proxy_ip:8080"
export no_proxy="127.0.0.1,localhost,.example.com,.cluster.local,.svc,localaddress,`echo 192.168.1.{1..254},`172.17.0.0/16,172.30.0.0/16"
```

Reboot ALL nodes


### Optional: Set Proxy for Docker.

This directory may not exist yet.
```
mkdir -p /etc/systemd/system/docker.service.d/
vi /etc/systemd/system/docker.service.d/no_proxy.conf
vi no_proxy.conf
```

Example proxy setting
```
[Service]
Environment="NO_PROXY=127.0.0.1,localhost,.example.com,.cluster.local,.svc,localaddress,`echo 192.168.1.{1..254},`172.17.0.0/16,172.30.0.0/16"
Environment="HTTP_PROXY=http://proxy_ip:8080/"
Environment="HTTPS_PROXY=http://proxy_ip:8080/"
```

## Kubespray Prerequisites

Verify NTP on all nodes. Make sure there is no time drift between nodes. This ensures certificates are created correctly later on.

```
ntpq -p
date
```

Verify SSH keys exist:

```
ls -la ~/.ssh
```

If needed, setup ssh keys on all nodes.

```
ssh-keygen
```

Run the following to share keys on all nodes, modify hostnames to match your environment. :

```
for host in node1.example.com \
node2.example.com  \
node3.example.com ; \
do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
done
```

Verify you can ssh as root on all nodes without password. I have seen where the first entry gets skipped for some reason. I also run it with IPs for additional checks. Just verify.

## Install Kubespray

Setup environment. This will only be performed from a single host (i.e. an Ansible bastion server or from a master node).

```
cd ~ 
mkdir workspace && cd workspace
git clone https://github.com/kubernetes-sigs/kubespray 
```

If already exists, use git pull to update the installation directory.
```
cd kubespray
git pull
```

Install prerequisite software. Make sure to be within the Kubespray folder.

```
cd ~/workspace/kubespray
pip3 install -r requirements.txt
```

## Configure your cluster
```
cp -rfp inventory/sample inventory/mycluster
vi inventory/mycluster/inventory.ini
```

Define your Kubernetes masters and workers here. This will be specific to your environment. 

Example `inventory.ini` for a single master and two worker node cluster. This is for group 8 but modify to your group number.
```yaml
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
kube-g8-master1 ansible_host=192.168.1.81 etcd_member_name=etcd1
kube-g8-node1 ansible_host=192.168.1.84 etcd_member_name=etcd2
kube-g8-node2 ansible_host=192.168.1.85 etcd_member_name=etcd3

[kube-master]
kube-g8-master1

[kube-node]
kube-g8-node1
kube-g8-node2

[etcd]
kube-g8-master1
kube-g8-node1
kube-g8-node2

[k8s-cluster:children]
kube-master
kube-node

[masters]
kube-g8-master1

[workers]
kube-g8-node1
kube-g8-node2
```

Save and exit.

**OPTIONAL:** Set Proxy within Kubernetes if behind corporate proxy. Remember to set domain and IPs as needed.

```
vi inventory/mycluster/group_vars/all/all.yml
```

Set proxy

```
http_proxy: "http://proxy_ip:8080"
https_proxy: "http://proxy_ip:8080"
```

Finally run the installer, we will be installing Kubernetes version 1.18
```
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml --extra-vars "kube_version=v1.18.9"
```

**Grab a cup of your favorite beverage.**

Verify everything successfully came up:

```
kubectl get nodes
```

## Back on your local workstation

Now that you have the cluster up and running, you will need to copy the `~/.kube/config` to your local workstation so you can interact with the cluster. 

> This command is ran from your workstation not via SSH/Putty. Make sure to update the command with the proper group number.

Click on Windows Subsystem for Linux icon <img src="images/centos_logo.png" alt="WSL" width="40" height="45"> in Taskbar or open **Powershell** and type `wsl`.

```
cd ~
mv ~/.kube/config ~/.kube/config_old
scp root@kube-g8-master1:/root/.kube/config ~/.kube/
```

Now verify that you can connect from your workstation using the `kubectl get nodes`.

If everything worked then you should see your cluster listed.

You are now ready to deploy the HPE CSI Driver.

Head over the [scod.hpedev.io](https://scod.hpedev.io/csi_driver/deployment.html). 

> Double check Helm is version > 3.2, if not head over to [helm.sh](https://helm.sh/docs/intro/install/) to install the latest binary.
