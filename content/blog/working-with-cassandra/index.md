---
title: Testing cassandra at scale
date: "2020-09-06T17:00:03.284Z"
description: "Salesforce Connector for Confluent Platform(Kafka)"
slug: 'testing-cassandra-at-scale'
tags:
  - cassandra
  - performance
  - nosql
  - kubernetes
show: true
author: abhi 
featuredImage: ../../assets/cassandra.png   
---

While working with cassandra we experimented a lot to analyse the performance of the cassandra on kubernetes cluster. 

Here are the some learnings from the extensive testing.

### Write (Bulk)

* [k8s] Concurrency: 200; Num CPU's Client: 2; # Records: 116547279
    - Time taken to Insert in C* 2h54m26.765794644s
    - Total Record Counts: 173801976
    - Error Counts: 0

* [k8s] Concurrency: 1000; Num CPU's Client: 16; # Records: 116547186
    - Time taken to Insert in C* 1h56m43.787323892s
    - Total Record Counts: 173801976
    - Error Counts: 11449

* Using spark to import data directly from s3
    - Number of nodes in cassandra cluster: 8 x i3.xlarge
    - Replication factor: 2
    - Number of nodes in emr cluster: 10 x m3.xlarge
    - Number of records: 4.820.489.450 (all snippets)
    - Time taken: 24 hours
    - Number of writes per core per second: 1743

**Inference**: Client scales linearly with the number of available cores. We need to make sure that we are not keeping the concurrency too high. Also in the second case the number of available cpu's with the client is 16 this is greater than the number of cpus with all cassandra nodes (4 x 3 = 12)

### Read (Bulk)

* **Test setup 1**
    - Number of nodes in cassandra cluster: 6 x i3.xlarge
    - Number of client nodes: 2 x m3.xlarge
    - Number of queries: ~21M
    - Time taken: ~7000 sec
    - Number of reads per cassandra core per second: 125
    - Number of reads per client core per second: 375


### Test: Does commit log placement affect write performance?

* **Test setup: cassandra cluster running on 3 x i3.xlarge**
    - Number of client writer nodes: 10 x m3.xlarge
    - Number of rows to insert: ~46M
    - Commit log placements tested:
        * Ephemeral drive, same as where data is stored
        * Root SSD drive
        * Separate EBS (HDD Standard)
        * Separate EBS (SSD GP2)
Time taken in each test: 12 minutes

### Failure Testing

### Case 1: Purge all the pods but instances with data persists

On restarting we need to make sure that we remove all files in the path: /var/lib/cassandra/data/system/*

If not done the last cluster state causes the second pod to enter a crash loop backoff and it causes problems for cluster to start.

Performing this operation still has no impact on data. Remember we are only deleting data in the system keyspace, the rest of the keyspace data is still kept intact.

Note: This should only be done when the entire cluster was purged and previous data still persists.

### Case 2: Delete a cluster pod; Instance and data persists.

This case has two scenarios:
1. Deleted pod is a seed node.
2. Deleted pod is a non-seed node.
In both the cases deleting the pod has no affect in availability of data from cluster. We can delete the pod and on the same instance the same pod starts (stateful-set construct at work in k8s) and it starts up properly connecting back to the ring in the cluster.

### Case 3: Scale up the cluster

Generally, if you start some new nodes with proper configuration values (mainly seeds and cluster_name), they should eventually join the cluster and then receive their portion of data from other nodes. This process is called bootstrapping.

However, if two or more nodes try to bootstrap at the same time, one might fail to proceed until the other one has finished his job. In logs, you might find corresponding exceptions:

```
Exception (java.lang.UnsupportedOperationException) encountered during startup:
Other bootstrapping/leaving/moving nodes detected, cannot bootstrap while cassandra.consistent.rangemovement is true
```

But as said, this is not critical issue: cassandra will retry bootstrapping over and over again until succeeded.
More details discussed in depth in [this article](http://thelastpickle.com/blog/2017/05/23/auto-bootstrapping-part1.html)

### Case 4: Remove or downscale the cluster nodes

1. Run nodetool drain && nodetool stopdaemon on node planned for removal
2. Run nodetool status on any other node in the cluster. Make sure that the victim node is marked as DN
3. Run nodetool removenode <GUID> with victim's GUID. This command will take some time.
4. To remove all nodes that are down, run the same stuff in a loop, like:
for node in `nodetool status | grep ^DN | awk {'print $7'}`; do nodetool removenode $node; done

### Case 5: Replacing dead node

Can be achieved by:
1. removing dead node as described in Case 6
2. starting a new node as described in Case 5
However, there is more efficient way possible to add [a replacement node](http://thelastpickle.com/blog/2017/05/23/auto-bootstrapping-part1.html#adding-a-replacement-node)

### Case 6: What happens when disk becomes full

This section explains what kinds of issues might arise when the disk where cassandra data folder resides, becomes full.

The main thing we should know is that cassandra does not consider this case as critical, so if node is unable to do a planned background compaction, it won't just do it. As far as there is enough space in commitlog pool, all data writes will still be stored in commitlog. However, commitlog maximum size is controlled by commitlog_total_space_in_mb config property, and by default its set to several gigabytes. For performance reasons, its not recommended to increase this value too much. So if you keep writing new data, commitlog will also get full.

After that, all writes to this node will fail, while node will still try to stay up and running. In particular, nodetool status will still report node as UN (up normal).

Depending on cluster configuration, replication_factor property and desired consistency level, a client who is doing writes to the cluster might not notice any issue. So for example if we have more than 1 node in cluster, use replication_factor = 2 and consistency ONE, any write request will be successful, because second replica will always successfully go to another, healthy node.

But in practice things may work differently. As of cassandra 3.11.2, such an overloaded node seems to work quite unstable, so over time service may die with following log messages:

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid2795.hprof ...
Unable to create java_pid2795.hprof: Permission denied

    java.lang.OutOfMemoryError: Java heap space
 -XX:OnOutOfMemoryError="kill -9 %p"
   Executing /bin/sh -c "kill -9 2795"...
bash: line 1:  2795 Killed                  docker-entrypoint.sh
```
After restarting the service, while there is still no free space left on device, cassandra might behave unpredictably. For example, some subservices would start normally, and others not. But if you first free up some space and then restart it again, cassandra will start normally.
Therefore, you need to make sure you have some smart policy for service healthcheck and restarting.

### Case 7: [Tombstones: common issues](https://opencredo.com/cassandra-tombstones-common-issues/)
    