### Backup methods

etcd has different backup methods for v2 and v3, and each has its own advantages and disadvantages.  The v3 backup is much cleaner and consists of a single, compact file, but it has one major drawback: it won’t backup or recover v2 data.

This means that if you have only etcd v3 data (for example, if your network plugin doesn’t consume etcd), you can use the v3 backup, but if you have any v2 data–even if it’s mixed with v3 data–you must use the v2 backup method.

Let’s look at each of these methods.

### Etcd v2 backups

The etcd v2 backup method creates a directory structure with a single WAL file. You can perform a backup online without interrupting etcd cluster operations. To back up an etcd v2+v3 data store, use the following command:

```
etcdctl backup --data-dir /var/lib/etcd/ --backup-dir /backupdir
```

You can find the official procedure for etcd v2 restore here, but here is an overview of the basic steps. The challenging part is to rebuild the cluster one node at a time.

1. Stop etcd on all hosts
2. Purge /var/lib/etcd/member on all hosts
3. Copy the backup to /var/lib/etcd/member on the first etcd host
4. Start up etcd on the first etcd host with –force-new-cluster
5. Set the correct the PeerURL on the first etcd host to the IP of the node instead of 127.0.0.1.
6. Add the next host to the cluster
7. Start etcd on the next host with –initial-cluster set to existing etcd hosts + itself
8. Repeat 5 and 6 until all etcd nodes are joined
9. Restart etcd normally (using existing settings)

You can see these steps in the following [script](https://gist.github.com/mattymo/40ad689b8a11e0bfbb7ee415103dabf0):

```
#!/bin/bash -e

# Change as necessary
RESTORE_PATH=${RESTORE_PATH:-/tmp/member}

#Extract node data from etcd config
source /etc/etcd.env || source /etc/default/etcd
function with_retries {
  local retries=3
  set -o pipefail
  for try in $(seq 1 $retries); do
    ${@}
    [ $? -eq 0 ] && break
    if [[ "$try" == "$retries" ]]; then
      exit 1
    fi
    sleep 3
  done
  set +o pipefail
}

this_node=$ETCD_NAME
node_names=($(echo $ETCD_INITIAL_CLUSTER | \
  awk -F'[=,]' '{for (i=1;i<=NF;i+=2) { print $i }}'))
node_endpoints=($(echo $ETCD_INITIAL_CLUSTER | \
  awk -F'[=,]' '{for (i=2;i<=NF;i+=2) { print $i }}'))
node_ips=($(echo $ETCD_INITIAL_CLUSTER | \
  awk -F'://|:[0-9]' '{for (i=2;i<=NF;i+=2) { print $i }}'))
num_nodes=${#node_names[@]}

# Stop and purge etcd data
for i in `seq 0 $((num_nodes - 1))`; do
  ssh ${node_ips[$i]} sudo service etcd stop
  ssh ${node_ips[$i]} sudo docker rm -f ${node_names[$i]} \
    || : # Kargo specific
  ssh ${node_ips[$i]} sudo rm -rf /var/lib/etcd/member
done

# Restore on first node
if [[ "$this_node" == ${node_names[0]} ]]; then
  sudo cp -R $RESTORE_PATH /var/lib/etcd/
else
  rsync -vaz -e "ssh" --rsync-path="sudo rsync" \
    "$RESTORE_PATH" ${node_ips[0]}:/var/lib/etcd/
fi

ssh ${node_ips[0]} "sudo etcd --force-new-cluster 2> \
  /tmp/etcd-restore.log" &
echo "Sleeping 5s to wait for etcd up"
sleep 5

# Fix member endpoint on first node
member_id=$(with_retries ssh ${node_ips[0]} \
  ETCDCTL_ENDPOINTS=https://localhost:2379 \
  etcdctl member list | cut -d':' -f1)
ssh ${node_ips[0]} ETCDCTL_ENDPOINTS=https://localhost:2379 \
  etcdctl member update $member_id ${node_endpoints[0]}
echo "Waiting for etcd to reconfigure peer URL"
sleep 4

# Add other nodes
initial_cluster="${node_names[0]}=${node_endpoints[0]}"
for i in `seq 1 $((num_nodes -1))`; do
  echo "Adding node ${node_names[$i]} to ETCD cluster..."
  initial_cluster=\
    "$initial_cluster,${node_names[$i]}=${node_endpoints[$i]}"
  with_retries ssh ${node_ips[0]} \
    ETCDCTL_ENDPOINTS=https://localhost:2379 \
    etcdctl member add ${node_names[$i]} ${node_endpoints[$i]}
  ssh ${node_ips[$i]} \
    "sudo etcd --initial-cluster="$initial_cluster" &>/dev/null" &
  sleep 5
  with_retries ssh ${node_ips[0]} \
    ETCDCTL_ENDPOINTS=https://localhost:2379 etcdctl member list
done

echo "Restarting etcd on all nodes"
for i in `seq 0 $((num_nodes -1))`; do
  ssh ${node_ips[$i]} sudo service etcd restart
done

sleep 5

echo "Verifying cluster health"
with_retries ssh ${node_ips[0]} \
  ETCDCTL_ENDPOINTS=https://localhost:2379 etcdctl cluster-health

```


### Etcd v3 backups

The etcd v3 backup creates a single compressed file. Remember, while v2 backups surprisingly also copy v3 data, the v3 backup cannot be used to back up etcd v2 data, so be careful before using this method. To create a v3 backup, run the command:
```
ETCDCTL_API=3 etcdctl snapshot save /backupdir
```

The official procedure for etcd v3 restore is documented here, but as you can see, the general process is much simpler than it was for v2; the v3 restore process is capable of rebuilding the cluster without such granular steps.

The steps required are as follows:

1. Stop etcd on all hosts
2. Purge /var/lib/etcd/member on all hosts
3. Copy the backup file to each etcd host
4. source /etc/default/etcd on each host and run the following command:

```
ETCDCTL_API=3 etcdctl snapshot restore BACKUP_FILE \
--name $ETCD_NAME--initial-cluster "$ETCD_INITIAL_CLUSTER" \
--initial-cluster-token “$ETCD_INITIAL_CLUSTER_TOKEN” \
--initial-advertise-peer-urls $ETCD_INITIAL_ADVERTISE_PEER_URLS \
--data-dir $ETCD_DATA_DIR
```
