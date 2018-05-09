The following will be done on the Master Node.

| Docker Only Install | Single Master | Node 1 | Node 2 | Node + |
|---------------------|---------------|--------|--------|--------|
|etcd (port 2379,4001,2380) | etcd (port 23790,40010,23800) |-- |-- | --|
|Managed Plugin | Legacy Plugin | Legacy Plugin | Legacy Plugin |Legacy Plugin |
| -- | Dory | Dory | Dory | Dory |
| -- | Doryd | -- | -- | -- |

The following will be done on the Master Node:

**ETCD config**

Export the Master Node IP address
```
export HostIP="<Master node IP>"
```

Run the following to create the etcd container. This etcd instance is separate from the etcd deployed by Kubernetes/OpenShift and is required for managing the HPE 3PAR Docker Volume plugin. We need to modify the ports (2379, 4001, 2380) to prevent conflicts. This allows two instances of etcd to safely run in the environment.

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

**Configure the Legacy HPE Docker Volume plugin**

Add 3PAR into ~/.ssh/known_hosts
```
$ ssh -l username <3PAR IP Address>
```

Create the plugin configuration directory
```
$ mkdir -p /etc/hpedockerplugin/
```

Create the hpe.conf file

```
$ vi /etc/hpedockerplugin/hpe.conf
```

Copy the following information into hpe.conf

**iSCSI configuration**

```
[DEFAULT]
ssh_hosts_key_file = /root/.ssh/known_hosts  
#change to match Master host IP where etcd is running
host_etcd_ip_address = <Master Node IP>
#Modify to match the port configured in etcd (2379 or 23790)
host_etcd_port_number = 23790
logging = DEBUG
hpe3par_debug = True
suppress_requests_ssl_warnings = True
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

**Multipath Settings**

When multipathing is required with the HPE 3PAR StoreServ, you must update the multipath.conf, iscsid.conf, docker-compose.yml, and hpe.conf files as outlined below.

Configuring iSCSI Multipathing in /etc/iscsi/iscsid.conf

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

Create the multipath.conf

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

### This is a valid docker-compose.yml for FC & iSCSI

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

**Start up the container**

Make sure you are in the location of the docker-compose.yml file
```	 
$ docker-compose up -d or
$ docker-compose up 2>&1 | tee /tmp/plugin_logs.txt
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

**Install/configure Dory/Doryd (the FlexVolume drivers) using Automated Installer**

Download and run the self-extracting installer
```
$ wget https://github.com/hpe-storage/python-hpedockerplugin/raw/master/dory_installer
$ chmod u+x ./dory_installer
$ sudo ./dory_installer
```
This installs the Dory and Doryd binaries to:
```
/usr/libexec/kubernetes/kubelet-plugins/volume/exec/dev.hpe.com~hpe/
```

You should end up with a Doryd executable in the ./bin directory and be ready for installation, run the following to complete the install
```
$ sudo /usr/libexec/kubernetes/kubelet-plugins/volume/exec/dev.hpe.com~hpe/doryd  /etc/kubernetes/admin.conf hpe.com
```


Restart OpenShift origin:
```
$ systemctl restart origin-node
```

Confirm the flexvolume driver started successfully
```
$ tail -f /var/log/dory.log
```

If everything works fine, you should be able to inspect your log file for successful initialization (/var/log/dory.log):
```
Info : 2018/01/04 23:42:05 dory.go:52: [19723] entry  : Driver=hpe Version=1.0.0-4adcc622 Socket=/run/docker/plugins/hpe.sock Overridden=true
Info : 2018/01/04 23:42:05 dory.go:55: [19723] request: init []
Info : 2018/01/04 23:42:05 dory.go:58: [19723] reply  : init []: {"status":"Success"}
Info : 2018/01/04 23:42:12 dory.go:52: [19788] entry  : Driver=hpe Version=1.0.0-4adcc622 Socket=/run/docker/plugins/hpe.sock Overridden=true
```

For more information, study below Dory guide:
https://github.com/hpe-storage/python-hpedockerplugin/blob/master/docs/doryd-install-3par-docker-plugin.md

# Usage

Now lets create a StorageClass, PersistentVolumeClaim, PV and Pod

This is an all-in-one command to create each of these components. You can easily split them out into separate ```yml``` files and run the ```oc create -f <file.yml>```

```
$ sudo oc create -f - << EOF
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
 name: sc-comp3
provisioner: dev.hpe.com/hpe
parameters:
  size: "16"  
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-comp3
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 16Gi
  storageClassName: sc-comp3
---
kind: Pod
apiVersion: v1
metadata:
  name: pod-comp3
spec:
  containers:
  - name: minio
    image: minio/minio:latest
    args:
    - server
    - /export
    env:
    - name: MINIO_ACCESS_KEY
      value: minio
    - name: MINIO_SECRET_KEY
      value: doryspeakswhale
    ports:
    - containerPort: 9000
    volumeMounts:
    - name: export
      mountPath: /export
  volumes:
    - name: export
      persistentVolumeClaim:
        claimName: pvc-comp3
EOF
```

Validate the Pod, PersistentVolumeClaim, PersistentVolume, and StorageClass are created:
```
$ oc get pod,pv,pvc,sc
$ oc describe pod pod-comp3
```
