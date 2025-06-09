# The Apache Kafka Control Plane


## Control Plane

{{< image src="/images/kafka_internals/00_30.jpg" caption="Control Plane" title="Control Plane" >}}

### KRaft Mode Controller

{{< image src="/images/kafka_internals/00_31.jpg" caption="KRaft Mode Controller" title="KRaft Mode Controller" >}}

The brokers that serve as controllers, in a KRaft mode cluster, are listed in the `controller.quorum.voters` configuration property that is set on each broker. One of these controller brokers will be the active controller and it will handle communicating changes to metadata with the other brokers.

All of the controller brokers maintain an in-memory metadata cache that is kept up to date, so that any controller can take over as the active controller if needed. This is one of the features of KRaft that make it so much more efficient than the ZooKeeper-based control plane.

### KRaft Cluster Metadata

{{< image src="/images/kafka_internals/00_31.jpg" caption="KRaft Cluster Metadata" title="KRaft Cluster Metadata" >}}

KRaft is based upon the Raft consensus protocol which was introduced to Kafka as part of KIP-500 with additional details defined in other related KIPs. In KRaft mode, cluster metadata, reflecting the current state of all controller managed resources, is stored in a single partition Kafka topic called `__cluster_metadata`. KRaft uses this topic to synchronize cluster state changes across controller and broker nodes.

The active controller is the leader of this internal metadata topic’s single partition. Other controllers are replica followers. Brokers are replica observers. So, rather than the controller broadcasting metadata changes to the other controllers or to brokers, they each fetch the changes. This makes it very efficient to keep all the controllers and brokers in sync, and also shortens restart times of brokers and controllers.

### KRaft Metadata Replication

{{< image src="/images/kafka_internals/00_32.jpg" caption="KRaft Metadata Replication" title="KRaft Metadata Replication" >}}

Since cluster metadata is stored in a Kafka topic, replication of that data is very similar to what we saw in the data plane replication module. The active controller is the leader of the metadata topic’s single partition and it will receive all writes. The other controllers are followers and will fetch those changes. We still use offsets and leader epochs the same as with the data plane.

However, when a leader needs to be elected, this is done via quorum, rather than an in-sync replica set. So, there is no ISR involved in metadata replication. Another difference is that metadata records are flushed to disk immediately as they are written to each node’s local log.

### Leader Election

Controller leader election is required when the cluster is started, as well as when the current leader stops, either as part of a rolling upgrade or due to a failure.

#### Vote Request

{{< image src="/images/kafka_internals/00_33.jpg" caption="Vote Request" title="Vote Request" >}}

When the leader controller needs to be elected, the other controllers will participate in the election of a new leader. A controller, usually the one that first recognized the need for a new leader, will send a `VoteRequest` to the other controllers. This request will include the candidate’s last offset and the epoch associated with that offset. It will also increment that epoch and pass it as the candidate epoch. The candidate controller will also vote for itself for that epoch.

#### Vote Response

{{< image src="/images/kafka_internals/00_34.jpg" caption="Vote Response" title="Vote Response" >}}

When a follower controller receives a `VoteRequest` it will check to see if it has seen a higher epoch than the one being passed in by the candidate. If it has, or if it has already voted for a different candidate with that same epoch, it will reject the request. Otherwise it will look at the latest offset passed in by the candidate and if it is the same or higher than its own, it will grant the vote. That candidate controller now has two votes: its own and the one it was just granted. The first controller to achieve a majority of the votes becomes the new leader.

#### Completion

{{< image src="/images/kafka_internals/00_35.jpg" caption="Completion" title="Completion" >}}

Once a candidate has collected a majority of votes, it will consider itself the leader but it still needs to inform the other controllers of this. To do this the new leader will send a `BeginQuorumEpoch` request, including the new epoch, to the other controllers. Now the election is complete. When the old leader controller comes back online, it will follow the new leader at the new epoch and bring its own metadata log up to date with the leader.

### Metadata Replica Reconciliation

{{< image src="/images/kafka_internals/00_36.jpg" caption="Metadata Replica Reconciliation" title="Metadata Replica Reconciliation" >}}

After a leader election is complete a log reconciliation may be required. In this case the reconciliation process is the same that we saw for topic data in the data plane replication module. Using the epoch and offsets of both the followers and the leader, the follower will truncate uncommitted records and bring itself in sync with the leader.

### KRaft Cluster Metadata Snapshot

{{< image src="/images/kafka_internals/00_37.jpg" caption="KRaft Cluster Metadata Snapshot" title="KRaft Cluster Metadata Snapshot" >}}

There is no clear point at which we know that cluster metadata is no longer needed, but we don’t want the metadata log to grow endlessly. The solution for this requirement is the metadata snapshot. Periodically, each of the controllers and brokers will take a snapshot of their in-memory metadata cache. This snapshot is saved to a file identified with the end offset and controller epoch. Now we know that all data in the metadata log that is older than that offset and epoch is safely stored, and the log can be truncated up to that point. The snapshot, together with the remaining data in the metadata log, will still give us the complete metadata for the whole cluster.

### When a Snapshot Is Read

{{< image src="/images/kafka_internals/00_38.jpg" caption="When a Snapshot Is Read" title="When a Snapshot Is Read" >}}

Two primary uses of the metadata snapshot are broker restarts and new brokers coming online.

When an existing broker restarts, it (1) loads its most recent snapshot into memory. Then starting from the EndOffset of its snapshot, it (2) adds available records from its local `__cluster_metadata` log. It then (3) begins fetching records from the active controller. If the fetched record offset is less than the active controller LogStartOffset, the controller response includes the snapshot ID of its latest snapshot. The broker then (4) fetches this snapshot and loads it into memory and then once again continues fetching records from the `__cluster_metadata` partition leader (the active controller).

When a new broker starts up, it (3) begins fetching records for the first time from the active controller. Typically, this offset will be less than the active controller LogStartOffset and the controller response will include the snapshot ID of its latest snapshot. The broker (4) fetches this snapshot and loads it into memory and then once again continues fetching records from the `__cluster_metadata` partition leader (the active controller).

---

