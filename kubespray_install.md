# Installation steps for Kubespray 

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

Finally run the installer, we will be installing Kubernetes version 1.18.
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

```
mv ~/.kube/config ~/.kube/config_old
scp root@kube-g8-master1:/root/.kube/config ~/.kube/
```

Now verify that you can connect from your workstation using the `kubectl get nodes`.

If everything worked then you should see your cluster listed.

You are now ready to deploy the HPE CSI Driver.

