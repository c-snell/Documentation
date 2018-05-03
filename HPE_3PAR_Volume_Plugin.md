**ETCD config**

```
export HostIP="10.10.1.60"
```

```
sudo docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 40010:40010 \
-p 23800:23800 -p 23790:23790 \
--name etcd quay.io/coreos/etcd:v2.2.0 \
-name etcd0 \
-advertise-client-urls http://${HostIP}:23790,http://${HostIP}:40010 \
-listen-client-urls http://0.0.0.0:23790,http://0.0.0.0:40010 \
-initial-advertise-peer-urls http://${HostIP}:23800 \
-listen-peer-urls http://0.0.0.0:23800 \
-initial-cluster-token etcd-cluster-1 \
-initial-cluster etcd0=http://${HostIP}:23800 \
-initial-cluster-state new
```

Add 3PAR into ~/.ssh/known_hosts
```
$ ssh -l 3paradm 10.10.1.150
```

Configure the docker plugin
```
$ mkdir -p /etc/hpedockerplugin/
$ vi /etc/hpedockerplugin/hpe.conf
```

```
[DEFAULT]
ssh_hosts_key_file = /root/.ssh/known_hosts  
#change to match host IP
host_etcd_ip_address = 10.10.1.60
host_etcd_port_number = 23790
logging = DEBUG
hpe3par_debug = True
suppress_requests_ssl_warnings = True
#change to FC
hpedockerplugin_driver = hpedockerplugin.hpe.hpe_3par_iscsi.HPE3PARISCSIDriver
hpe3par_api_url = https://10.10.1.150:8080/api/v1
hpe3par_username = 3paradm
hpe3par_password = 3pardata
san_ip = 10.10.1.150
san_login = 3paradm
san_password = 3pardata
hpe3par_cpg = FC_r6
iscsi_ip_address = 10.10.1.155
hpe3par_iscsi_chap_enabled = False
hpe3par_iscsi_ips = 10.10.1.155,10.10.1.156
use_multipath = True
enforce_multipath = True
```

```
$ vi /etc/multipath.conf
```

```
defaults
{
    polling_interval 10
    max_fds 8192
}

devices
{
    device
	{
        vendor                  "3PARdata"
        product                 "VV"
        no_path_retry           18
        features                "0"
        hardware_handler        "0"
        path_grouping_policy    multibus
        #getuid_callout         "/lib/udev/scsi_id --whitelisted --device=/dev/%n"
        path_selector           "round-robin 0"
        rr_weight               uniform
        rr_min_io_rq            1
        path_checker            tur
        failback                immediate
    }
}
```

```
$ systemctl enable iscsid multipathd
$ docker pull hpestorage/legacyvolumeplugin:2.1
```

**Running the hpedockerplugin with Docker Compose:**

Edit the docker service
```
$ vi /usr/lib/systemd/system/docker.service
```

Change
**MountFlags=shared** (default is slave)


Restart the docker daemon
```
$ systemctl daemon-reload
$ systemctl restart docker.service
```

## Configure Docker Compose

```
$ vi ~/docker-compose.yml
```

### This is a valid docker-compose.yml for FC

```
hpedockerplugin:
  image: hpestorage/legacyvolumeplugin:2.1
  container_name: plugin_container
  net: host
  privileged: true
  volumes:
     - /dev:/dev
     - /run/lock:/run/lock
     - /var/lib:/var/lib
     - /var/run/docker/plugins:/var/run/docker/plugins:rw
     - /etc:/etc
     - /root/.ssh:/root/.ssh
     - /sys:/sys
     - /root/plugin/certs:/root/plugin/certs
     - /sbin/iscsiadm:/sbin/ia
     - /lib/modules:/lib/modules
     - /lib64:/lib64
     - /var/run/docker.sock:/var/run/docker.sock
     - /opt/hpe/data:/opt/hpe/data:rshared
```

### This is a valid docker-compose.yml for iSCSI
per https://github.com/hpe-storage/python-hpedockerplugin/blob/plugin_v2/docs/multipath.md

```
hpedockerplugin:
  image: hpestorage/legacyvolumeplugin:2.1
  container_name: plugin_container
  net: host
  privileged: true
  volumes:
      - /dev:/dev
      - /run/docker/plugins:/run/docker/plugins
      - /lib/modules:/lib/modules
      - /var/lib/docker/:/var/lib/docker
      - /etc/hpedockerplugin/data:/etc/hpedockerplugin/data:shared
      - /etc/iscsi/initiatorname.iscsi:/etc/iscsi/initiatorname.iscsi
      - /etc/hpedockerplugin:/etc/hpedockerplugin
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/iscsi/iscsid.conf:/etc/iscsi/iscsid.conf
      - /etc/multipath.conf:/etc/multipath.conf

```

**Configuring iSCSI Multipathing in /etc/iscsi/iscsid.conf**

You can find details on how to properly configure multipath.conf in the [HPE 3PAR Red Hat Enterprise Linux and Oracle Linux Implementation Guide] (http://h20565.www2.hpe.com/hpsc/doc/public/display?docId=c04448818).

Change the following iSCSI parameters in /etc/iscsi/iscsid.conf. Please review the HPE 3PAR Red Hat Enterprise Linux and Oracle Linux Implementation Guide for any required updates.

```
node.startup = automatic
node.conn[0].startup = automatic
node.session.timeo.replacement_timeout = 10
node.conn[0].timeo.noop_out_interval = 10
```

Lastly, make the following additions to the **/etc/hpedockerplugin/hpe.conf** file to enable multipathing.

```
use_multipath = True
enforce_multipath = True
```

**Start up the container**

Make sure you are in the location of the docker-compose.yml file
```	 
$ docker-compose up -d
```
```
In case you are missing docker-compose, https://docs.docker.com/compose/install/#install-compose

$ curl -x 16.85.88.10:8080 -L https://github.com/docker/compose/releases/download/1.21.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose

# Test the installation
$ docker-compose --version
docker-compose version 1.21.0, build 1719ceb
```

Create 2 symbolic links by using these steps
```
$ mkdir -p /run/docker/plugins/hpe
$ cd /run/docker/plugins/hpe
$ ln -s ../hpe.sock.lock  hpe.sock.lock
$ ln -s ../hpe.sock  hpe.sock
```

**IMPORTANT NOTE:** The /run/docker/plugins/hpe/hpe.sock and /run/docker/plugins/hpe/hpe.sock.lock files are not automatically removed when you stop the container. Therefore, these files will need to be removed manually between each run of the plugin.

**Test the plugin**
```
$ docker volume create -d hpe --name sample_vol -o size=1
```

**Install/configure Dory**
```
$ sudo subscription-manager repos --enable=rhel-7-server-optional-rpms
$ sudo yum install -y golang make
$ git clone https://github.com/hpe-storage/dory.git
$ make gettools
$ make dory
```

**You should end up with a dory executable in the ./bin directory and be ready for installation.**
```
$ cd /usr/libexec/kubernetes/kubelet-plugins/volume/exec/
$ mkdir dev.hpe.com~hpe
$ cd dev.hpe.com~hpe
$ cp <path>/dory hpe
```

**Create a file called hpe.json in this folder**

Dory looks for a configuration file with the same name as the executable with a .json extension. Following the example above, the configuration file would be /usr/libexec/kubernetes/kubelet-plugins/volume/exec/dev.hpe.com~hpe/hpe.json

Contents of hpe.json
```
{
    "dockerVolumePluginSocketPath": "/run/docker/plugins/hpe.sock",
    "logFilePath": "/var/log/dory.log"
    "logDebug": true,
    "supportsCapabilities": true,
    "stripK8sFromOptions": true,
    "createVolumes": true,
    "listOfStorageResourceOptions": ["size"]
}
```

Restart OpenShift origin:
```
$ systemctl restart origin-node
```

If everything works fine, you should be able to inspect your log file for successful initialization (/var/log/dory.log):
```
Info : 2017/09/18 16:37:40 dory.go:52: [127775] entry  : Driver=hpe Version=1.0.0-ae48ca4c Socket=/run/docker/plugins/hpe.sock Overridden=true
Info : 2017/09/18 16:37:40 dory.go:55: [127775] request: init []
Info : 2017/09/18 16:37:40 dory.go:58: [127775] reply  : init []: {"status":"Success"}
```

For more information, study below Dory guide:
https://github.com/hpe-storage/dory/blob/master/docs/dory/README.md

**Build, Install and Configure Doryd binary**

Build and configure doryd binary on all master nodes. There are two ways provided to build Doryd. A host machine build and a fully containerized build. The containerized build require Docker 17.05 or newer on both client and daemon.

**Host build**

Doryd is written in Go and requires golang on your machine. The following example installs the necessary tools and builds Doryd on a RHEL 7.4 system:
```
$ sudo subscription-manager repos --enable=rhel-7-server-optional-rpms
$ sudo yum install -y golang make
$ git clone https://github.com/hpe-storage/dory.git
$ make gettools
$ make vendor
$ make doryd
```

You should end up with a doryd executable in ./bin directory.

The doryd binary needs access to the cluster via a kubeconfig file which is available on path: /etc/kubernetes/admin.conf


Doryd command line in OpenShift platform which is known to work is:
```
./bin/doryd /root/.kube/config dev.hpe.com
```
OR
```
./bin/doryd /etc/kubernetes/admin.conf dev.hpe.com
```
