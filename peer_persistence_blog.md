# Tutorial: Enabling Remote Copy with the HPE CSI Driver for Kubernetes on HPE Primera

The new features of the HPE CSI Driver for Kubernetes never stop and with the newly release 1.3.0 version of the CSI Driver, comes a much requested support for HPE Primera and 3PAR Remote Copy Peer Persistence which provides enhanced availability and transparent failover for disaster recovery protection. As more and more applications migrate into Kubernetes, HPE recommends customers to deploy mission critical applications with replicated persistent volumes to ensure that these applications be highly available and resistant to failure. HPE Primera and 3PAR Remote Copy can serve as the foundation for a disaster recovery solution. 

## Assumptions

I will be starting with an existing single zone Kubernetes cluster. For the most up to date information and examples on HPE Storage and Containers, please refer to [scod.hpedev.io](https://scod.hpedev.io). Currently the HPE CSI Driver for Kubernetes only supports HPE Primera and 3PAR Remote Copy in 2DC Peer Persistence mode. Remote Copy Periodic (async) mode is not supported but will be available in a future release. 

For information on creating a Peer Persistence configuration, review the [HPE Primera Peer Persistence Host OS Support Matrix](https://techhub.hpe.com/eginfolib/storage/docs/Primera/RemoteCopy/RCconfig/GUID-1F726F48-A372-4ED8-B1D7-9545D091AE98.html#GUID-1F726F48-A372-4ED8-B1D7-9545D091AE98) for the supported host OSs and host persona requirements. Refer to [HPE Primera OS: Configuring data replication using Remote Copy over IP](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=emr_na-a00088914en_us) for more information.

## Requirements:

  - Single zone Kubernetes cluster
  - Deployment of HPE CSI Driver for Kubernetes
  - Create Secrets for Primary and Target arrays
  - Create CustomResourceDefinition (CRD) for Peer Persistence
  - Create StorageClass for replicated volumes

## Deploy the HPE CSI Driver for Kubernetes
I will start this demo by installing the latest version of the HPE CSI Driver for Kubernetes, which as of this writing is version 1.3.0. Here are two methods:

#### Fresh installation of the HPE CSI Driver for Kubernetes
```markdown
helm repo add hpe https://hpe-storage.github.io/co-deployments/
helm repo update
helm install hpe-csi hpe/hpe-csi-driver --namespace hpe-csi --version 1.3.0
```

#### Upgrading an existing deployment of the HPE CSI Driver for Kubernetes.
```markdown
helm repo update
helm search repo hpe-csi-driver -l
helm upgrade hpe-csi hpe/hpe-csi-driver --namespace <namespace> --version 1.3.0
```

I can check the status of the deployment by running the following command. 

```markdown
kubectl get all -n hpe-csi
```

If I used a different namespace during the deployment I can use this command.
```markdown
kubectl get pods --all-namespaces -l 'app in (nimble-csp, primera3par-csp, hpe-csi-node, hpe-csi-controller)'
```

You should see something like:
```markdown
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
hpe-csi       hpe-csi-controller-6d9bb97cd-njnj9   7/7     Running   0          1m
hpe-csi       hpe-csi-node-dlcz5                   2/2     Running   0          1m
...
hpe-csi       nimble-csp-745cb4d948-6449z          1/1     Running   0          1m
hpe-csi       primera3par-csp-867984bf86-dkf2d     1/1     Running   0          1m
```

## Create Remote Copy link Secrets
Tip for creating Kubernetes objects at the command line.
```markdown
kubectl create -f-
< paste the YAML >
^D (CTRL + D)
```

With the HPE CSI Driver deployed, we will need to create 2 secrets. One for each Primera array (i.e. default-primera-secret and secondary-primera-secret) that are part of the Remote Copy links. 

#### Primary Array
```markdown
apiVersion: v1
kind: Secret
metadata:
  name: default-primera-secret
  namespace: hpe-csi
stringData:
  serviceName: primera3par-csp-svc
  servicePort: "8080"
  backend: 10.0.0.2
  username: 3paradm
  password: 3pardata
```  

#### Secondary Array
```markdown
apiVersion: v1
kind: Secret
metadata:
  name: secondary-primera-secret
  namespace: hpe-csi
stringData:
  serviceName: primera3par-csp-svc
  servicePort: "8080"
  backend: 10.0.0.3
  username: 3paradm
  password: 3pardata
```  

## Create Peer Persistence CustomResourceDefinition
Next I will need to create a `CustomResourceDefinition` that holds the target array information that will be used when creating the volume pairs. 

```markdown
apiVersion: storage.hpe.com/v1
kind: HPEReplicationDeviceInfo
metadata:
  name: replication-crd
spec:
  target_array_details:
  - targetCpg: SSD_r6
    targetName: primera-c670
    targetSecret: secondary-primera-secret
    #targetSnapCpg: SSD_r6 (optional)
    targetSecretNamespace: hpe-csi
```

## Create Peer Persistence StorageClass

Next I will define the **remoteCopyGroup: <remote_copy_group_name>** and the **replicationDevices: <replication_crd_name>** parameters. The HPE CSI Driver can use an existing RCG or if the RCG doesn't exist it will create a new one. The CSI Driver will use the information from the `CRD` to create the replicated volume on the target array.

```markdown
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
  name: rep-sc
provisioner: csi.hpe.com
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/controller-expand-secret-name: default-primera-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: hpe-csi
  csi.storage.k8s.io/controller-publish-secret-name: default-primera-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: hpe-csi
  csi.storage.k8s.io/node-publish-secret-name: default-primera-secret
  csi.storage.k8s.io/node-publish-secret-namespace: hpe-csi
  csi.storage.k8s.io/node-stage-secret-name: default-primera-secret
  csi.storage.k8s.io/node-stage-secret-namespace: hpe-csi
  csi.storage.k8s.io/provisioner-secret-name: default-primera-secret
  csi.storage.k8s.io/provisioner-secret-namespace: hpe-csi
  description: "Volume created using Peer Persistence with the HPE CSI Driver for Kubernetes"
  accessProtocol: iscsi

# Primera customizations
  cpg: SSD_r6
  remoteCopyGroup: new-rcg
  replicationDevices: replication-crd
  provisioning_type: tpvv
  allowOverrides: description,provisioning_type,cpg,remoteCopyGroup
```

Once I have created the `StorageClass` within the cluster, I can request Persistent Volumes as normal.

## Create Peer Persistence PVC

```markdown
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: replicated-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: rep-sc
```

As volumes are created using the Remote Copy, the replication between the default and secondary Primera's will be transparent to Kubernetes. In the case of an array failure, automatic transparent failover will also managing the pathing so the application is not interrupted.
```markdown
kubectl get pvc
NAME             STATUS    VOLUME                            CAPACITY   ACCESS MODES   STORAGECLASS               AGE
replicated-pvc   Bound     pvc-ca03a916-a6fb-434c-bc00-6b8   200Gi      RWO            rep-sc                     1m
```

I can run **showrcopy** on the default Primera and to see the status of the Remote Copy Group that was just created and that it is the Primary array in the RCG.

```markdown
$ showrcopy

Remote Copy System Information
Status: Started, Normal

Target Information

Name              ID Type Status Options Policy
virt-primera-c670  4 IP   ready  -       mirror_config

Link Information

Target            Node  Address     Status Options
virt-primera-c670 0:3:1 172.17.20.5 Up     -
virt-primera-c670 1:3:1 172.17.20.6 Up     -
receive           0:3:1 receive     Up     -
receive           1:3:1 receive     Up     -

Group Information

Name         Target            Status   Role       Mode     Options
new-rcg      virt-primera-c670 Started  Primary    Sync     auto_failover,path_management
  LocalVV                         ID  RemoteVV                          ID SyncStatus    LastSyncTime
  pvc-ca03a916-a6fb-434c-bc00-6b8 168 pvc-ca03a916-a6fb-434c-bc00-6b8   83 Synced        NA
```

