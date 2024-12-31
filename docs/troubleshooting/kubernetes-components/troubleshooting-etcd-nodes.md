---
title: Troubleshooting etcd Nodes
---

<head>
  <link rel="canonical" href="https://ranchermanager.docs.rancher.com/troubleshooting/kubernetes-components/troubleshooting-etcd-nodes"/>
</head>

This section contains commands and tips for troubleshooting nodes with the `etcd` role.


## Checking if the etcd Container is Running

The container for etcd should have status **Up**. The duration shown after **Up** is the time the container has been running.

RKE1

```
docker ps -a -f=name=etcd$
```

Example output:
```
CONTAINER ID   IMAGE                                 COMMAND                  CREATED          STATUS          PORTS     NAMES
d26adbd23643   rancher/mirrored-coreos-etcd:v3.5.7   "/usr/local/bin/etcd…"   30 minutes ago   Up 30 minutes             etcd
```

RKE2/K3s
```
crictl ps --label io.kubernetes.container.name=etcd
```

Example output:
```
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
00ddda02d5c4b       19f8e656ed901       3 weeks ago         Running             etcd                0                   bd09cb26f253a       etcd-rke2-all-hppxl-lpnwm
```

## etcd Container Logging

The logging of the container can contain information on what the problem could be.

RKE1

```
docker logs etcd
```

RKE2/K3s: Crictl requires the container-id to list the logs.

```
etcdcontainer=$(/var/lib/rancher/rke2/bin/crictl ps --label io.kubernetes.container.name=etcd --quiet)
crictl logs $etcdcontainer
```

Example output:
```
crictl logs --tail 20 -f $etcdcontainer
{"level":"debug","ts":"2024-12-31T12:04:19.17864Z","caller":"etcdserver/server.go:2192","msg":"Applying entries","num-entries":1}
{"level":"debug","ts":"2024-12-31T12:04:19.17869Z","caller":"etcdserver/server.go:2195","msg":"Applying entry","index":18050864,"term":7,"type":"EntryNormal"}
{"level":"debug","ts":"2024-12-31T12:04:19.178703Z","caller":"etcdserver/server.go:2255","msg":"apply entry normal","consistent-index":18050863,"entry-index":18050864,"should-applyV3":true}
{"level":"debug","ts":"2024-12-31T12:04:19.178727Z","caller":"etcdserver/server.go:2282","msg":"applyEntryNormal","raftReq":"header:<ID:11823237772231225757 username:\"etcd-client\" auth_revision:1 > txn:<compare:<target:MOD key:\"/registry/leases/kube-system/kube-scheduler\" mod_revision:16689509 > success:<request_put:<key:\"/registry/leases/kube-system/kube-scheduler\" value:\"k8s\\000\\n\\037\\n\\026coordination.k8s.io/v1\\022\\005Lease\\022\\203\\003\\n\\235\\002\\n\\016kube-scheduler\\022\\000\\032\\013kube-system\\\"\\000*$9d102d5a-a0f3-4830-bf21-0292451296ec2\\0008\\000B\\010\\010\\314\\243\\242\\272\\006\\020\\000\\212\\001\\304\\001\\n\\016kube-scheduler\\022\\006Update\\032\\026coordination.k8s.io/v1\\\"\\010\\010\\303\\273\\317\\273\\006\\020\\0002\\010FieldsV1:|\\nz{\\\"f:spec\\\":{\\\"f:acquireTime\\\":{},\\\"f:holderIdentity\\\":{},\\\"f:leaseDurationSeconds\\\":{},\\\"f:leaseTransitions\\\":{},\\\"f:renewTime\\\":{}}}B\\000\\022a\\n@rke2-import-all-hppxl-skjmm_b9a8e609-f9f7-4c2b-b626-66f273dd01d3\\020\\017\\032\\014\\010\\244\\211\\245\\273\\006\\020\\370\\226\\205\\246\\003\\\"\\013\\010\\303\\273\\317\\273\\006\\020\\370\\356\\325P(\\005\\032\\000\\\"\\000\" > > failure:<request_range:<key:\"/registry/leases/kube-system/kube-scheduler\" > > > "}
{"level":"debug","ts":"2024-12-31T12:04:19.182347Z","caller":"auth/store.go:1151","msg":"found command name","common-name":"etcd-client","user-name":"etcd-client","revision":1}
{"level":"debug","ts":"2024-12-31T12:04:19.182524Z","caller":"v3rpc/interceptor.go:182","msg":"request stats","start time":"2024-12-31T12:04:19.181384Z","time spent":"1.063804ms","remote":"127.0.0.1:40732","response type":"/etcdserverpb.KV/Range","request count":0,"request size":66,"response count":3,"response size":32,"request content":"key:\"/registry/longhorn.io/engines/\" range_end:\"/registry/longhorn.io/engines0\" count_only:true "}
{"level":"debug","ts":"2024-12-31T12:04:19.314794Z","caller":"auth/store.go:1151","msg":"found command name","common-name":"etcd-client","user-name":"etcd-client","revision":1}
{"level":"debug","ts":"2024-12-31T12:04:19.315067Z","caller":"v3rpc/interceptor.go:182","msg":"request stats","start time":"2024-12-31T12:04:19.313569Z","time spent":"1.317581ms","remote":"127.0.0.1:40926","response type":"/etcdserverpb.KV/Range","request count":0,"request size":71,"response count":1,"response size":1199,"request content":"key:\"/registry/configmaps/kube-system/rke2-coredns-rke2-coredns-autoscaler\" "}
...
```

| Log | Explanation |
|-----|------------------|
| `health check for peer xxx could not connect: dial tcp IP:2380: getsockopt: connection refused` | A connection to the address shown on port 2380 cannot be established. Check if the etcd container is running on the host with the address shown. |
| `xxx is starting a new election at term x` | The etcd cluster has lost its quorum and is trying to establish a new leader. This can happen when the majority of the nodes running etcd go down/unreachable. |
| `connection error: desc = "transport: Error while dialing dial tcp 0.0.0.0:2379: i/o timeout"; Reconnecting to {0.0.0.0:2379 0  <nil>}` | The host firewall is preventing network communication. |
| `rafthttp: request cluster ID mismatch` | The node with the etcd instance logging `rafthttp: request cluster ID mismatch` is trying to join a cluster that has already been formed with another peer. The node should be removed from the cluster, and re-added. |
| `rafthttp: failed to find member` | The cluster state (`/var/lib/etcd`) contains wrong information to join the cluster. The node should be removed from the cluster, the state directory should be cleaned and the node should be re-added.

## etcd Cluster and Connectivity Checks

The address where etcd is listening depends on the address configuration of the host etcd is running on. If an internal address is configured for the host etcd is running on, the endpoint for `etcdctl` needs to be specified explicitly. If any of the commands respond with `Error:  context deadline exceeded`, the etcd instance is unhealthy (either quorum is lost or the instance is not correctly joined in the cluster)

### Check etcd Members on all Nodes

Output should contain all the nodes with the `etcd` role and the output should be identical on all nodes.

Command:

RKE1

```
docker exec etcd etcdctl member list
```

RKE2/k3s

```

```



### Check Endpoint Status

The values for `RAFT TERM` should be equal and `RAFT INDEX` should be not be too far apart from each other.

Command:
```
docker exec -e ETCDCTL_ENDPOINTS=$(docker exec etcd etcdctl member list | cut -d, -f5 | sed -e 's/ //g' | paste -sd ',') etcd etcdctl endpoint status --write-out table
```

Example output:
```
+-----------------+------------------+---------+---------+-----------+-----------+------------+
| ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+-----------------+------------------+---------+---------+-----------+-----------+------------+
| https://IP:2379 | 333ef673fc4add56 |  3.5.7  |   24 MB |     false |        72 |      66887 |
| https://IP:2379 | 5feed52d940ce4cf |  3.5.7  |   24 MB |      true |        72 |      66887 |
| https://IP:2379 | db6b3bdb559a848d |  3.5.7  |   25 MB |     false |        72 |      66887 |
+-----------------+------------------+---------+---------+-----------+-----------+------------+
```

### Check Endpoint Health

Command:
```
docker exec -e ETCDCTL_ENDPOINTS=$(docker exec etcd etcdctl member list | cut -d, -f5 | sed -e 's/ //g' | paste -sd ',') etcd etcdctl endpoint health
```

Example output:
```
https://IP:2379 is healthy: successfully committed proposal: took = 2.113189ms
https://IP:2379 is healthy: successfully committed proposal: took = 2.649963ms
https://IP:2379 is healthy: successfully committed proposal: took = 2.451201ms
```

### Check Connectivity on Port TCP/2379

Command:
```
for endpoint in $(docker exec etcd etcdctl member list | cut -d, -f5); do
   echo "Validating connection to ${endpoint}/health"
   docker run --net=host -v $(docker inspect kubelet --format '{{ range .Mounts }}{{ if eq .Destination "/etc/kubernetes" }}{{ .Source }}{{ end }}{{ end }}')/ssl:/etc/kubernetes/ssl:ro appropriate/curl -s -w "\n" --cacert $(docker inspect -f '{{range $index, $value := .Config.Env}}{{if eq (index (split $value "=") 0) "ETCDCTL_CACERT" }}{{range $i, $part := (split $value "=")}}{{if gt $i 1}}{{print "="}}{{end}}{{if gt $i 0}}{{print $part}}{{end}}{{end}}{{end}}{{end}}' etcd) --cert $(docker inspect -f '{{range $index, $value := .Config.Env}}{{if eq (index (split $value "=") 0) "ETCDCTL_CERT" }}{{range $i, $part := (split $value "=")}}{{if gt $i 1}}{{print "="}}{{end}}{{if gt $i 0}}{{print $part}}{{end}}{{end}}{{end}}{{end}}' etcd) --key $(docker inspect -f '{{range $index, $value := .Config.Env}}{{if eq (index (split $value "=") 0) "ETCDCTL_KEY" }}{{range $i, $part := (split $value "=")}}{{if gt $i 1}}{{print "="}}{{end}}{{if gt $i 0}}{{print $part}}{{end}}{{end}}{{end}}{{end}}' etcd) "${endpoint}/health"
done
```

Example output:
```
Validating connection to https://IP:2379/health
{"health": "true"}
Validating connection to https://IP:2379/health
{"health": "true"}
Validating connection to https://IP:2379/health
{"health": "true"}
```

### Check Connectivity on Port TCP/2380

Command:
```
for endpoint in $(docker exec etcd etcdctl member list | cut -d, -f4); do
  echo "Validating connection to ${endpoint}/version";
  docker run --net=host -v $(docker inspect kubelet --format '{{ range .Mounts }}{{ if eq .Destination "/etc/kubernetes" }}{{ .Source }}{{ end }}{{ end }}')/ssl:/etc/kubernetes/ssl:ro appropriate/curl --http1.1 -s -w "\n" --cacert $(docker inspect -f '{{range $index, $value := .Config.Env}}{{if eq (index (split $value "=") 0) "ETCDCTL_CACERT" }}{{range $i, $part := (split $value "=")}}{{if gt $i 1}}{{print "="}}{{end}}{{if gt $i 0}}{{print $part}}{{end}}{{end}}{{end}}{{end}}' etcd) --cert $(docker inspect -f '{{range $index, $value := .Config.Env}}{{if eq (index (split $value "=") 0) "ETCDCTL_CERT" }}{{range $i, $part := (split $value "=")}}{{if gt $i 1}}{{print "="}}{{end}}{{if gt $i 0}}{{print $part}}{{end}}{{end}}{{end}}{{end}}' etcd) --key $(docker inspect -f '{{range $index, $value := .Config.Env}}{{if eq (index (split $value "=") 0) "ETCDCTL_KEY" }}{{range $i, $part := (split $value "=")}}{{if gt $i 1}}{{print "="}}{{end}}{{if gt $i 0}}{{print $part}}{{end}}{{end}}{{end}}{{end}}' etcd) "${endpoint}/version"
done
```

Example output:
```
Validating connection to https://IP:2380/version
{"etcdserver":"3.5.7","etcdcluster":"3.5.0"}
Validating connection to https://IP:2380/version
{"etcdserver":"3.5.7","etcdcluster":"3.5.0"}
Validating connection to https://IP:2380/version
{"etcdserver":"3.5.7","etcdcluster":"3.5.0"}
```

## etcd Alarms

etcd will trigger alarms, for instance when it runs out of space.

Command:
```
docker exec etcd etcdctl alarm list
```

Example output when NOSPACE alarm is triggered:
```
memberID:x alarm:NOSPACE
memberID:x alarm:NOSPACE
memberID:x alarm:NOSPACE
```

## etcd Space Errors

Related error messages are `etcdserver: mvcc: database space exceeded` or `applying raft message exceeded backend quota`. Alarm `NOSPACE` will be triggered.

Resolutions:

- [Compact the Keyspace](#compact-the-keyspace)
- [Defrag All etcd Members](#defrag-all-etcd-members)
- [Check Endpoint Status](#check-endpoint-status)
- [Disarm Alarm](#disarm-alarm)

### Compact the Keyspace

Command:
```
rev=$(docker exec etcd etcdctl endpoint status --write-out json | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
docker exec etcd etcdctl compact "$rev"
```

Example output:
```
compacted revision xxx
```

### Defrag All etcd Members

Command:
```
docker exec -e ETCDCTL_ENDPOINTS=$(docker exec etcd etcdctl member list | cut -d, -f5 | sed -e 's/ //g' | paste -sd ',') etcd etcdctl defrag
```

Example output:
```
Finished defragmenting etcd member[https://IP:2379]
Finished defragmenting etcd member[https://IP:2379]
Finished defragmenting etcd member[https://IP:2379]
```

### Check Endpoint Status

Command:
```
docker exec -e ETCDCTL_ENDPOINTS=$(docker exec etcd etcdctl member list | cut -d, -f5 | sed -e 's/ //g' | paste -sd ',') etcd etcdctl endpoint status --write-out table
```

Example output:
```
+-----------------+------------------+---------+---------+-----------+-----------+------------+
| ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+-----------------+------------------+---------+---------+-----------+-----------+------------+
| https://IP:2379 |  e973e4419737125 |  3.5.7  |  553 kB |     false |        32 |    2449410 |
| https://IP:2379 | 4a509c997b26c206 |  3.5.7  |  553 kB |     false |        32 |    2449410 |
| https://IP:2379 | b217e736575e9dd3 |  3.5.7  |  553 kB |      true |        32 |    2449410 |
+-----------------+------------------+---------+---------+-----------+-----------+------------+
```

### Disarm Alarm

After verifying that the DB size went down after compaction and defragmenting, the alarm needs to be disarmed for etcd to allow writes again.

Command:
```
docker exec etcd etcdctl alarm list
docker exec etcd etcdctl alarm disarm
docker exec etcd etcdctl alarm list
```

Example output:
```
docker exec etcd etcdctl alarm list
memberID:x alarm:NOSPACE
memberID:x alarm:NOSPACE
memberID:x alarm:NOSPACE
docker exec etcd etcdctl alarm disarm
docker exec etcd etcdctl alarm list
```

## Configure Log Level

:::note

You can no longer dynamically change the log level in etcd v3.5 or later.

:::

### etcd v3.5 And Later

To configure the log level for etcd, edit the cluster YAML:

```
services:
  etcd:
    extra_args:
      log-level: "debug"
```

### etcd v3.4 And Earlier

In earlier etcd versions, you can use the API to dynamically change the log level.  Configure debug logging using the commands below:

```
docker run --net=host -v $(docker inspect kubelet --format '{{ range .Mounts }}{{ if eq .Destination "/etc/kubernetes" }}{{ .Source }}{{ end }}{{ end }}')/ssl:/etc/kubernetes/ssl:ro appropriate/curl -s -XPUT -d '{"Level":"DEBUG"}' --cacert $(docker exec etcd printenv ETCDCTL_CACERT) --cert $(docker exec etcd printenv ETCDCTL_CERT) --key $(docker exec etcd printenv ETCDCTL_KEY) $(docker exec etcd printenv ETCDCTL_ENDPOINTS)/config/local/log
```

To reset the log level back to the default (`INFO`), you can use the following command.

Command:
```
docker run --net=host -v $(docker inspect kubelet --format '{{ range .Mounts }}{{ if eq .Destination "/etc/kubernetes" }}{{ .Source }}{{ end }}{{ end }}')/ssl:/etc/kubernetes/ssl:ro appropriate/curl -s -XPUT -d '{"Level":"INFO"}' --cacert $(docker exec etcd printenv ETCDCTL_CACERT) --cert $(docker exec etcd printenv ETCDCTL_CERT) --key $(docker exec etcd printenv ETCDCTL_KEY) $(docker exec etcd printenv ETCDCTL_ENDPOINTS)/config/local/log
```

## etcd Content

If you want to investigate the contents of your etcd, you can either watch streaming events or you can query etcd directly, see below for examples.

### Watch Streaming Events

Command:
```
docker exec etcd etcdctl watch --prefix /registry
```

If you only want to see the affected keys (and not the binary data), you can append `| grep -a ^/registry` to the command to filter for keys only.

### Query etcd Directly

Command:
```
docker exec etcd etcdctl get /registry --prefix=true --keys-only
```

You can process the data to get a summary of count per key, using the command below:

```
docker exec etcd etcdctl get /registry --prefix=true --keys-only | grep -v ^$ | awk -F'/' '{ if ($3 ~ /cattle.io/) {h[$3"/"$4]++} else { h[$3]++ }} END { for(k in h) print h[k], k }' | sort -nr
```

## Replacing Unhealthy etcd Nodes

When a node in your etcd cluster becomes unhealthy, the recommended approach is to fix or remove the failed or unhealthy node before adding a new etcd node to the cluster.
