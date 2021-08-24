Pre-requisites

Create Web Server - nginx
Create load balancer - haproxy

Installing a cluster on bare metal - Installing on bare metal | Installing | OpenShift Container Platform 4.6


Minimum resource requirements

Each cluster machine must meet the following minimum requirements:

| Machine | Operating System | vCPU | Virtual RAM | Storage |
| :------: | :------: | :------: | :------: | :------: |
| Bootstrap | RHCOS | 4 | 16 GB | 120 GB | 
| Control plane | RHCOS | 4 | 16 GB | 120 GB |
| Compute | RHCOS or RHEL 7.6 | 2 | 8 GB | 120 GB |

Procedure

1. Configure DHCP or set static IP addresses on each node.
2. Provision the required load balancers.
3. Configure the ports for your machines.
4. Configure DNS.
5. Ensure network connectivity.

Firewall or disable firewall on load balancer


Network topology requirements

The infrastructure that you provision for your cluster must meet the following network topology requirements.


OpenShift Container Platform requires all nodes to have internet access to pull images for platform containers and provide telemetry data to Red Hat.
Load balancers

Before you install OpenShift Container Platform, you must provision two load balancers that meet the following requirements:

1. API load balancer: Provides a common endpoint for users, both human and machine, to interact with and configure the platform. 
   
   Configure the following ports on both the front and back of the load balancers:
   
   PortBack-end machines (pool members)
   
   Kubernetes API server:  6443 & 22623
   
   The load balancer must be configured to take a maximum of 30 seconds from the time the API server turns off the /readyz endpoint to the removal of the API server instance from the pool. Within the time frame after /readyz returns an error or becomes healthy, the endpoint must have been removed or added. Probing every 5 or 10 seconds, with two successful requests to become healthy and three to become unhealthy, are well-tested values.
2. Application Ingress load balancer: Provides an Ingress point for application traffic flowing in from outside the cluster. 
   
   Configure the following ports on both the front and back of the load balancers:
   
   HTTPS traffic: 443
   HTTP traffic: 80

User-provisioned DNS requirements

The following DNS records are required for an OpenShift Container Platform cluster that uses user-provisioned infrastructure. In each record, <cluster_name> is the cluster name and <base_domain> is the cluster base domain that you specify in the install-config.yaml file. A complete DNS record takes the form: <component>.<cluster_name>.<base_domain>..

Table 5. Required DNS records
Component
Record
Description
Kubernetes API
api.<cluster_name>.<base_domain>.
This DNS A/AAAA or CNAME record must point to the load balancer for the control plane machines. This record must be resolvable by both clients external to the cluster and from all the nodes within the cluster.
api-int.<cluster_name>.<base_domain>.
This DNS A/AAAA or CNAME record must point to the load balancer for the control plane machines. This record must be resolvable from all the nodes within the cluster.

The API server must be able to resolve the worker nodes by the host names that are recorded in Kubernetes. If it cannot resolve the node names, proxied API calls can fail, and you cannot retrieve logs from pods.
Routes
*.apps.<cluster_name>.<base_domain>.
A wildcard DNS A/AAAA or CNAME record that points to the load balancer that targets the machines that run the Ingress router pods, which are the worker nodes by default. This record must be resolvable by both clients external to the cluster and from all the nodes within the cluster.


Procedure

1. If you do not have an SSH key that is configured for password-less authentication on your computer, create one. For example, on a computer that uses a Linux operating system, run the following command:
   
   Specify the path and file name, such as ~/.ssh/id_rsa , of the new SSH key.
   Running this command generates an SSH key that does not require a password in the location that you specified.
2. Start the ssh-agent process as a background task:
   
   Example output
3. Add your SSH private key to the ssh-agent:
   
   Example output
   
   Specify the path and file name for your SSH private key, such as ~/.ssh/id_rsa

Obtaining the installation program

Before you install OpenShift Container Platform, download the installation file on a local computer.

Prerequisites

- A computer that runs Linux or macOS, with 500 MB of local disk space
Procedure

1. Access the Infrastructure Provider page on the Red Hat OpenShift Cluster Manager site. If you have a Red Hat account, log in with your credentials. If you do not, create an account.
2. Navigate to the page for your installation type, download the installation program for your operating system, and place the file in the directory where you will store the installation configuration files.
   
   The installation program creates several files on the computer that you use to install your cluster. You must keep both the installation program and the files that the installation program creates after you finish installing the cluster.
   Deleting the files created by the installation program does not remove your cluster, even if the cluster failed during installation. To remove your cluster, complete the OpenShift Container Platform uninstallation procedures for your specific cloud provider.
3. Extract the installation program. For example, on a computer that uses a Linux operating system, run the following command:
4. From the Pull Secret page on the Red Hat OpenShift Cluster Manager site, download your installation pull secret as a .txt file. This pull secret allows you to authenticate with the services that are provided by the included authorities, including Quay.io, which serves the container images for OpenShift Container Platform components.



Manually creating the installation configuration file


Procedure

1. Create an installation directory to store your required installation assets in:
   
   You must create a directory. Some installation assets, like bootstrap X.509 certificates have short expiration intervals, so you must not reuse an installation directory. If you want to reuse individual files from another cluster installation, you can copy them into your directory. However, the file names for the installation assets might change between releases. Use caution when copying installation files from an earlier OpenShift Container Platform version.
2. Customize the following install-config.yaml file template and save it in the <installation_directory>.
   You must name this configuration file install-config.yaml. Configure the PROXY here in the install-config.yaml.

Back up the install-config.yaml  file so that you can use it to install multiple clusters.

The install-config.yaml  file is consumed during the next step of the installation process. You must back it up now. 

Working install-config.yaml: Make a BACKUP!!!!
```
apiVersion: v1 
baseDomain: example.io 
compute: 
- hyperthreading: Enabled 
  name: worker 
  replicas: 0 
controlPlane: 
  hyperthreading: Enabled 
  name: master 
  replicas: 3 
metadata: 
  name: ocp46 
networking: 
  clusterNetwork: 
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN 
  serviceNetwork: 
  - 172.30.0.0/16 
proxy: 
  httpProxy: http://proxy:8080 
  httpsProxy: http://proxy:8080 
  noProxy: <IPs> 
platform: 
  none: {} 
fips: false 
pullSecret: '<pull_secret>' 
sshKey: '<ssh_key>'
```

Creating the Kubernetes manifest and Ignition config files


Procedure

1. Generate the Kubernetes manifests for the cluster: 
```
./openshift-install create manifests --dir=<installation_directory>
```
Example output
```
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
```

For <installation_directory>, specify the installation directory that contains the install-config.yaml file you created.

Because you create your own compute machines later in the installation process, you can safely ignore this warning.



If you are running a three-node cluster, skip the following step to allow the masters to be schedulable.


1. Modify the <installation_directory>/manifests/cluster-scheduler-02-config.yml Kubernetes manifest file to prevent pods from being scheduled on the control plane machines:
	1. Open the <installation_directory>/manifests/cluster-scheduler-02-config.yml  file.
	2. Locate the mastersSchedulable  parameter and set its value to False.
	3. Save and exit the file.


2. Obtain the Ignition config files: 
```
./openshift-install create ignition-configs --dir=<installation_directory>
```

For <installation_directory> , specify the same installation directory.
The following files are generated in the directory:



```
. 
├── auth 
│   ├── kubeadmin-password 
│   └── kubeconfig 
├── bootstrap.ign 
├── master.ign 
├── metadata.json 
└── worker.ign
```

Obtain the RHCOS images Product Downloads (redhat.com)
ISO file names resemble the following example:

rhcos-<version>-live.<architecture>.iso



copy to datastore and mount within the VMs.
---

Prep infrastructure

1. Create shell VMs 
2. Copy the MAC addresses
3. Create static reservations in DHCP pool


4. Create DNS entries with cluster plug (i.e. *.ocp46.virtware.io)
	1. Masters
	2. Workers
	3. Load Balancer
	4. API server > load balancer
	5. Wildcard *.apps.ocp46.virtware.io to load balancer


5. With IPs now update the load balancer with forwarders

```
global 
    log         127.0.0.1 local2 
defaults 
    timeout connect 10s 
    timeout client 30s 
    timeout server 30s 
    log global 
    mode http 
    option httplog 
    maxconn 3000 
frontend openshift-api-server 
    bind *:6443 
    default_backend openshift-api-server 
    mode tcp 
    option tcplog 
backend openshift-api-server 
    balance source 
    mode tcp 
#    server ocp-bootstrap 10.10.1.110:6443 check 
    server ocp-master1 10.10.1.111:6443 check 
    server ocp-master2 10.10.1.112:6443 check 
    server ocp-master3 10.10.1.113:6443 check 
frontend machine-config-server 
    bind *:22623 
    default_backend machine-config-server 
    mode tcp 
    option tcplog 
backend machine-config-server 
    balance source 
    mode tcp 
#    server ocp-bootstrap 10.10.1.110:22623 check 
    server ocp-master1 10.10.1.111:22623 check 
    server ocp-master2 10.10.1.112:22623 check 
    server ocp-master3 10.10.1.113:22623 check 
frontend ingress-http 
    bind *:80 
    default_backend ingress-http 
    mode tcp 
    option tcplog 
backend ingress-http 
    balance source 
    mode tcp 
    server ocp-worker1 10.10.1.114:80 check 
    server ocp-worker2 10.10.1.115:80 check 
    server ocp-worker3 10.10.1.116:80 check 
frontend ingress-https 
    bind *:443 
    default_backend ingress-https 
    mode tcp 
    option tcplog 
backend ingress-https 
    balance source 
    mode tcp 
    server ocp-worker1 10.10.1.114:443 check 
    server ocp-worker2 10.10.1.115:443 check 
    server ocp-worker3 10.10.1.116:443 check
```

6. Copy ignition files to webserver  
```
scp *.ign root@virt-web:/usr/share/nginx/html/
```

Start the bootstrap VM
	- make sure the ISO is mounted
	- Run command: 
```
sudo coreos-installer install --ignition-url=http://10.10.1.25/bootstrap.ign --insecure-ignition /dev/sda
```
reboot node once install is complete.

Repeat above on each master node
```
sudo coreos-installer install --ignition-url=http://10.10.1.25/master.ign --insecure-ignition /dev/sda
```
reboot node once install is complete

Repeat above on each worker node
```
sudo coreos-installer install --ignition-url=http://10.10.1.25/worker.ign --insecure-ignition /dev/sda
```
reboot node once install is complete

Creating the cluster


Procedure

1. Monitor the bootstrap process:
   
```
./openshift-install --dir=<installation_directory> wait-for bootstrap-complete \  
    --log-level=info
```

For <installation_directory>, specify the path to the directory that you stored the installation files in.
To view different installation details, specify warn, debug, or error instead of info.

Example output
```
INFO Waiting up to 30m0s for the Kubernetes API at https://api.test.example.com:6443... 
INFO API v1.19.0 up 
INFO Waiting up to 30m0s for bootstrapping to complete... 
INFO It is now safe to remove the bootstrap resources
```

The command succeeds when the Kubernetes API server signals that it has been bootstrapped on the control plane machines.

2. After bootstrap process is complete, remove the bootstrap machine from the load balancer.

You must remove the bootstrap machine from the load balancer at this point. You can also remove or reformat the machine itself.

Logging in to the cluster


Procedure

1. Export the kubeadmin credentials:
   
   
```
export KUBECONFIG=<installation_directory>/auth/kubeconfig
```

For <installation_directory>, specify the path to the directory that you stored the installation files in.
2. Verify you can run oc commands successfully using the exported configuration:
```
$ oc whoami
```
Example output

```
system:admin
```

Approving the CSRs for your machines


Confirm that the cluster recognizes the machines:

```
$ oc get nodes
```

Review the pending CSRs and ensure that you see a client and server request with the Pending or Approved status for each machine that you added to the cluster:

```
$ oc get csr
```

Example output

```
NAME        AGE     REQUESTOR                                                                   CONDITION
csr-8b2br   15m     system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending 
sr-8vnps    15m     system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-bfd72   5m26s   system:node:ip-10-0-50-126.us-east-2.compute.internal                       Pending 
sr-c57lv    5m26s   system:node:ip-10-0-95-157.us-east-2.compute.internal                       Pending
...
```

If the CSRs were not approved, after all of the pending CSRs for the machines you added are in Pending status, approve the CSRs for your cluster machines:

To approve them individually, run the following command for each valid CSR:

```
$ oc adm certificate approve <csr_name> 
```

To approve all pending CSRs, run the following command:

```
$ oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
```

Initial Operator configuration

Watch the cluster components come online:

```
$ watch -n5 oc get clusteroperators
```

Configuring registry storage for bare metal



You must configure storage for the image registry Operator. For non-production clusters, you can set the image registry to an empty directory. If you do so, all images are lost if you restart the registry.
Procedure

- To set the image registry storage to an empty directory:
  
```
- oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

Monitor for cluster completion:

```
$ ./openshift-install --dir=<installation_directory> wait-for install-complete
```

```
[root@ ~/ocpinstall/ocp4/clusters/4.6/vsphere/openshift]# openshift-install wait-for install-complete
INFO Waiting up to 40m0s for the cluster at https://api.ocp46.virtware.io:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/ocpinstall/ocp4/clusters/4.6/vsphere/openshift/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp46.example.io
INFO Login to the console with user: "kubeadmin", and password: "SEw6e-z9XwP-xXbqa-JvN3v"
INFO Time elapsed: 0s
```
