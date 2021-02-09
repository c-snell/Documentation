# Openshift Install guide

You will need **RHEL Server 7** (7.4 at the time of this writing) to run RPM based installer.

To install OpenShift 3.7 multinode setup using ansible playbooks, review the following sections to understand the Redhat OpenShift deployment and requirements:

1.	OpenShift Single/Multi-Node deployment scenarios:
https://docs.openshift.org/3.7/install_config/install/planning.html

2.	Validate all Pre-Requisites:
https://docs.openshift.org/3.7/install_config/install/prerequisites.html#install-config-install-prerequisites

3.	Follow the host preparation (master and worker nodes) guide.
```
Follow the steps for configuring the RPM-based installer for RHEL 7 systems NOT Atomic Host systems.
```
https://docs.openshift.org/latest/install_config/install/host_preparation.html

**Note:**
Skip over “**Configuring Docker Storage**” & “**Enabling Image Signature Support**” sections since they are not required at this time.


The following is a composite of the steps outlined in the pre-req, host preparation, and advanced installation guides.

```
# drop into root
$ sudo su -
```

```
$ systemctl disable firewalld
$ systemctl stop firewalld
$ systemctl status firewalld
$ yum install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
$ yum update
$ systemctl reboot
$ yum -y install \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$ yum -y --enablerepo=epel install ansible pyOpenSSL
$ cd ~
$ git clone https://github.com/openshift/openshift-ansible
$ cd openshift-ansible
$ git checkout release-3.7
$ yum install docker-1.13.1   
$ systemctl daemon-reload
$ docker version
$ systemctl enable docker
$ systemctl start docker
$ docker version
$ docker run hello-world   #run to validate Docker install
$ mkdir -p /etc/systemd/system/docker.service.d  #Set proxy settings in Docker
$ cd /etc/systemd/system/docker.service.d
$ vi http-proxy.conf

```

**Add the following:**
```
[Service]
Environment="HTTP_PROXY=http://16.85.88.10:8080/" "NO_PROXY=localhost,127.0.0.1,10.10.1.66,10.10.1.67,10.10.1.68,10.10.1.150,10.10.1.151,.virtware.co"
```
**Save and Exit**

```
$ vi https-proxy.conf
```
**Add the following:**
```bash
[Service]
Environment="HTTPS_PROXY=http://16.85.88.10:8080/" "NO_PROXY=localhost,127.0.0.1,10.10.1.66,10.10.1.67,10.10.1.68,10.10.1.150,10.10.1.151,.virtware.co"
```
**Save and Exit**

**Pre-configure OpenShift Networks in Docker:**
```
$ vi /etc/docker/daemon.json
```

**Add the following**
```
{ "insecure-registries": ["172.30.0.0/16"] }
```
**Save and Exit**

```
$ systemctl daemon-reload && systemctl restart docker
$ yum install -y iscsi-initiator-utils device-mapper-multipath
$ vi /etc/profile.d/proxy.sh
```

**Add the following:**
```
export http_proxy="http://16.85.88.10:8080"
export https_proxy="http://16.85.88.10:8080"
export no_proxy="127.0.0.1,localhost,10.10.1.66,10.10.1.67,10.10.1.68,k8-srik1,k8-srik1.virtware.co,.virtware.com,10.10.1.150,10.10.1.151,10.10.1.155,10.10.1.156,10.10.1.51,10.10.1.50"
```
**Save and Exit**

```
$ reboot
```

```
# drop into root
$ sudo su -
```

**Run the following on all masters and worker nodes to configure ssh passwordless access**
```
$ ssh-keygen
$ for host in k8-srik1.virtware.co k8-srik2.virtware.co k8-srik3.virtware.co; do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; done
```

### Setting Global Proxy Values
The OpenShift Origin installer uses the proxy settings in the **`/etc/environment`** file.

Ensure the following domain suffixes and IP addresses are in the **/etc/environment** file in the `no_proxy` parameter:

  * Master and node host names (domain suffix).

  * Other internal host names (domain suffix).

  * Etcd IP addresses (must be IP addresses and not host names, as **etcd** access is done by IP address).

  * Docker registry IP address.

  * Kubernetes IP address, by default 172.30.0.1. Must be the value set in the `openshift_portal_net` parameter in the Ansible inventory file, by default **/etc/ansible/hosts**.

  * Kubernetes internal domain suffix: **`cluster.local`**

  * Kubernetes internal domain suffix: **`.svc`**

The following example assumes `http_proxy` and `https_proxy` values are set:
```
no_proxy=127.0.0.1,localhost,kube-drone1,kube-drone1.virtware.co,kube-drone2,kube-drone2.virtware.co,kube-master,kube-master.virtware.co,.virtware.com,.cluster.local,.svc,10.10.1.60,10.10.1.61,10.10.1.62,10.10.1.155,10.10.1.156,10.10.1.150,10.10.1.151,10.10.1.157,10.10.1.158,172.30.0.1,localaddress,.localdomain.com,.hpecorp.net,.hp.com,.hpcloud.net
http_proxy=http://16.85.88.10:8080
https_proxy=http://16.85.88.10:8080
```

**Configure ansible on the Master node only**
```
$ vi /etc/ansible/hosts
```

**Replace all content of Ansible hosts file with the following:**
```
[OSEv3:children]
masters
nodes
etcd
# lb
# nfs

# Set variables common for all OSEv3 hosts
[OSEv3:vars]

#openshift_enable_unsupported_configurations=false

os_firewall_use_firewalld=True

ansible_user=root

ansible_become=yes

debug_level=2

# Specify the deployment type. Valid values are origin and openshift-enterprise.
openshift_deployment_type=origin
#openshift_deployment_type=openshift-enterprise

openshift_disable_check=docker_storage,docker_image_availability

docker_version="1.13.1"

openshift_release=v3.7

openshift_image_tag=v3.7.0

openshift_pkg_version=-3.7.1-2.el7

openshift_master_api_port=8443
openshift_master_console_port=8443

openshift_http_proxy=http://proxy.houston.hpecorp.net:8080
openshift_https_proxy=http://proxy.houston.hpecorp.net:8080
openshift_no_proxy='10.10.1.66,10.10.1.67,10.10.1.68,localhost,127.0.0.1,localaddress,.localdomain.com,.virtware.co,.hpecorp.net,.hp.com,.hpcloud.net'
openshift_builddefaults_http_proxy=http://proxy.houston.hpecorp.net:8080
openshift_builddefaults_https_proxy=http://proxy.houston.hpecorp.net:8080
openshift_builddefaults_no_proxy='10.10.1.66,10.10.1.67,10.10.1.68,localhost,127.0.0.1,localaddress,.localdomain.com,.virtware.co,.hpecorp.net,.hp.com,.hpcloud.net'

openshift_master_identity_providers=[{'name': 'allow_all', 'login': 'true', 'challenge': 'true', 'kind': 'AllowAllPasswordIdentityProvider'}]

# openshift_management_install_management=true

openshift_hosted_manage_router=true
# data to the inventory.  The variable to house the data is openshift_hosted_routers
#openshift_hosted_routers=[{'name': 'router1', 'certificate': {'certfile': '/path/to/certificate/abc.crt', 'keyfile': '/path/to/certificate/abc.key', 'cafile': '/path/to/certificate/ca.crt'}, 'replicas': 1, 'serviceaccount': 'router', 'namespace': 'default', 'stats_port': 1936, 'edits': [], 'images': 'openshift3/ose-${component}:${version}', 'selector': 'type=router1', 'ports': ['80:80', '443:443']}, {'name': 'router2', 'certificate': {'certfile': '/path/to/certificate/xyz.crt', 'keyfile': '/path/to/certificate/xyz.key', 'cafile': '/path/to/certificate/ca.crt'}, 'replicas': 1, 'serviceaccount': 'router', 'namespace': 'default', 'stats_port': 1936, 'edits': [{'action': 'append', 'key': 'spec.template.spec.containers[0].env', 'value': {'name': 'ROUTE_LABELS', 'value': 'route=external'}}], 'images': 'openshift3/ose-${component}:${version}', 'selector': 'type=router2', 'ports': ['80:80', '443:443']}]
openshift_hosted_registry_selector='region=infra'
openshift_hosted_router_selector='region=infra'
#openshift_hosted_registry_replicas=2
#openshift_hosted_registry_cert_expire_days=730
openshift_hosted_manage_registry=true
openshift_hosted_manage_router=true

# Enable service catalog
openshift_enable_service_catalog=false

# Enable template service broker (requires service catalog to be enabled, above)
template_service_broker_install=false

# openshift_service_catalog_image_version=v3.7

# TSB image tag
# template_service_broker_version='v3.7'

# Configure one of more namespaces whose templates will be served by the TSB
# openshift_template_service_broker_namespaces=['openshift','hpedory']

# Configure usage of openshift_clock role.
#openshift_clock_enabled=true

# For example, adding this cluster as a container provider,
# playbooks/openshift-management/add_container_provider.yml
openshift_management_username=admin
openshift_management_password=hpinvent

# host group for masters
[masters]
k8-srik1.virtware.co

[etcd]
#ose3-etcd[1:3]-ansible.test.example.com
k8-srik1.virtware.co

# NOTE: Containerized load balancer hosts are not yet supported, if using a global
# containerized=true host variable we must set to false.
#[lb]
#ose3-lb-ansible.test.example.com containerized=false

# NOTE: Currently we require that masters be part of the SDN which requires that they also be nodes
[nodes]
# masters should be schedulable to run web console pods
k8-srik1.virtware.co openshift_schedulable=True openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
k8-srik2.virtware.co openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
k8-srik3 openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
```
**Save and exit**

```
$ git clone https://github.com/openshift/openshift-ansible
$ cd openshift-ansible/
$ git checkout release-3.7
```

**Before starting the install, Validate the proxy is configured especially no_proxy for localhost**
```
$ env | grep _proxy
```

**Should look like the following:**
```
[root@k8-srik1 ~]# env | grep _proxy
http_proxy=http://16.85.88.10:8080
https_proxy=http://16.85.88.10:8080
no_proxy=127.0.0.1,localhost,10.10.1.60,10.10.1.61,k8-srik1,k8-srik1.virtware.co,.virtware.com,10.10.1.150,10.10.1.151,10.10.1.155,10.10.1.156,10.10.1.51,10.10.1.50
```

**Ready to begin OpenShift install**
```
$ ansible-playbook -i /etc/ansible/hosts ~/openshift-ansible/playbooks/byo/config.yml -e openshift_disable_check=disk_availability,docker_storage
```

**Now grab a cup of coffee, it will be a while...**

**45 minutes later...**

**OpenShift install completes with no errors.**

**Now let's validate the install of OpenShift**
```
$ oc status
$ oc get nodes
NAME                      STATUS    AGE       VERSION
k8-srik2.virtware.co   Ready     1d        v1.7.6+a08f5eeb62
k8-srik3.virtware.co   Ready     1d        v1.7.6+a08f5eeb62
```

**Access the URL via: https://k8-srik1.virtware.co:8443**


**Now proceed with install of HPE 3PAR Storage Plugin**
