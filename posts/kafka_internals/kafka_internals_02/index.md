# Data Plane: Replication Protocol


## Kafka Data Replication

Data replication is a critical feature of Kafka that allows it to provide high durability and availability. We enable replication at the topic level.

### Leader, Follower, and In-Sync Replica (ISR) List

{{< image src="/images/kafka_internals/00_13.jpg" caption="Leader, Follower, and In-Sync Replica (ISR) List" title="Leader, Follower, and In-Sync Replica (ISR) List" >}}

Once the replicas for all the partitions in a topic are created, one replica of each partition will be designated as the leader replica and the broker that holds that replica will be the leader for that partition. The remaining replicas will be followers.

Producers will write to the leader replica and the followers will fetch the data in order to keep in sync with the leader.

Consumers also, generally, fetch from the leader replica, but they can be configured to fetch from followers.

The partition leader, along with all of the followers that have caught up with the leader, will be part of the in-sync replica set (ISR).

### Leader Epoch

{{< image src="/images/kafka_internals/00_14.jpg" caption="Leader Epoch" title="Leader Epoch" >}}

Each leader is associated with a unique, monotonically increasing number called the leader epoch. The epoch is used to keep track of what work was done while this replica was the leader and it will be increased whenever a new leader is elected.

### Follower Fetch Request

{{< image src="/images/kafka_internals/00_15.jpg" caption="Follower Fetch Request" title="Follower Fetch Request" >}}

Whenever the leader appends new data into its local log, the followers will issue a fetch request to the leader, passing in the offset at which they need to begin fetching.

### Follower Fetch Response

{{< image src="/images/kafka_internals/00_16.jpg" caption="Follower Fetch Response" title="Follower Fetch Response" >}}

The leader will respond to the fetch request with the records starting at the specified offset. The fetch response will also include the offset for each record and the current leader epoch. The followers will then append those records to their own local logs.

### Committing Partition Offsets

{{< image src="/images/kafka_internals/00_17.jpg" caption="Committing Partition Offsets" title="Committing Partition Offsets" >}}

Once all of the followers in the ISR have fetched up to a particular offset, the records up to that offset are considered committed and are available for consumers. This is designated by the high watermark.

The leader is made aware of the highest offset fetched by the followers through the offset value sent in the fetch requests. For example, if a follower sends a fetch request to the leader that specifies offset 3, the leader knows that this follower has committed all records up to offset 3. Once all of the followers have reached offset 3, the leader will advance the high watermark accordingly.

### Advancing the Follower High Watermark

{{< image src="/images/kafka_internals/00_18.jpg" caption="Advancing the Follower High Watermark" title="Advancing the Follower High Watermark" >}}

The leader, in turn, uses the fetch response to inform followers of the current high watermark. Because this process is asynchronous, the followers’ high watermark will typically lag behind the actual high watermark held by the leader.

### Handling Leader Failure

{{< image src="/images/kafka_internals/00_19.jpg" caption="Handling Leader Failure" title="Handling Leader Failure" >}}

If a leader fails, or if for some other reason we need to choose a new leader, one of the brokers in the ISR will be chosen as the new leader. The process of leader election and notification of affected followers is handled by the control plane. The important thing for the data plane is that no data is lost in the process. That is why a new leader can only be selected from the ISR, unless the topic has been specifically configured to allow replicas that are not in sync to be selected. We know that all of the replicas in the ISR are up to date with the latest committed offset.

Once a new leader is elected, the leader epoch will be incremented and the new leader will begin accepting produce requests.

### Temporary Decreased High Watermark

{{< image src="/images/kafka_internals/00_20.jpg" caption="Temporary Decreased High Watermark" title="Temporary Decreased High Watermark" >}}

When a new leader is elected, its high watermark could be less than the actual high watermark. If this happens, any fetch requests for an offset that is between the current leader’s high watermark and the actual will trigger a retriable `OFFSET_NOT_AVAILABLE` error. The consumer will continue trying to fetch until the high watermark is updated, at which point processing will continue as normal.

### Partition Replica Reconciliation

{{< image src="/images/kafka_internals/00_21.jpg" caption="Partition Replica Reconciliation" title="Partition Replica Reconciliation" >}}

Immediately after a new leader election, it is possible that some replicas may have uncommitted records that are out of sync with the new leader. This is why the leader's high watermark is not current yet. It can’t be until it knows the offset that each follower has caught up to. We can’t move forward until this is resolved. This is done through a process called replica reconciliation. The first step in reconciliation begins when the out-of-sync follower sends a fetch request.

In our example, the request shows that the follower is fetching an offset that is higher than the high watermark for its current epoch.

### Fetch Response Informs Follower of Divergence

{{< image src="/images/kafka_internals/00_22.jpg" caption="Fetch Response Informs Follower of Divergence" title="Fetch Response Informs Follower of Divergence" >}}

When the leader receives the fetch request it will check it against its own log and determine that the offset being requested is not valid for that epoch. It will then send a response to the follower telling it what offset that epoch should end at. The leader leaves it to the follower to perform the cleanup.

### Follower Truncates Log to Match Leader Log

{{< image src="/images/kafka_internals/00_23.jpg" caption="Follower Truncates Log to Match Leader Log" title="Follower Truncates Log to Match Leader Log" >}}

The follower will use the information in the fetch response to truncate the extraneous data so that it will be in sync with the leader.

### Subsequent Fetch with Updated Offset and Epoch

{{< image src="/images/kafka_internals/00_24.jpg" caption="Subsequent Fetch with Updated Offset and Epoch" title="Subsequent Fetch with Updated Offset and Epoch" >}}

Now the follower can send that fetch request again, but this time with the correct offset.

### Follower 102 Reconciled

{{< image src="/images/kafka_internals/00_25.jpg" caption="Follower 102 Reconciled" title="Follower 102 Reconciled" >}}

The leader will then respond with the new records since that offset includes the new leader epoch.

### Follower 102 Acknowledges New Records

{{< image src="/images/kafka_internals/00_26.jpg" caption="Follower 102 Acknowledges New Records" title="Follower 102 Acknowledges New Records" >}}

When the follower fetches again, the offset that it passes will inform the leader that it has caught up and the leader will be able to increase the high watermark. At this point the leader and follower are fully reconciled, but we are still in an under replicated state because not all of the replicas are in the ISR.

Depending on configuration, we can operate in this state, but it’s certainly not ideal.

### Follower 101 Rejoins the Cluster

{{< image src="/images/kafka_internals/00_27.jpg" caption="Follower 101 Rejoins the Cluster" title="Follower 101 Rejoins the Cluster" >}}

At some point, hopefully soon, the failed replica broker will come back online. It will then go through the same reconciliation process that we just described. Once it is done reconciling and is caught up with the new leader, it will be added back to the ISR and we will be back in our happy place.

### Handling Failed or Slow Followers

{{< image src="/images/kafka_internals/00_28.jpg" caption="Handling Failed or Slow Followers" title="Handling Failed or Slow Followers" >}}

Obviously when a leader fails, it’s a bigger deal, but we also need to handle follower failures as well as followers that are running slow.

The leader monitors the progress of its followers. If a configurable amount of time elapses since a follower was last fully caught up, the leader will remove that follower from the in-sync replica set. This allows the leader to advance the high watermark so that consumers can continue consuming current data. If the follower comes back online or otherwise gets its act together and catches up to the leader, then it will be added back to the ISR.

### Partition Leader Balancing

{{< image src="/images/kafka_internals/00_29.jpg" caption="Partition Leader Balancing" title="Partition Leader Balancing" >}}

As we’ve seen, the broker containing the leader replica does a bit more work than the follower replicas. Because of this it’s best not to have a disproportionate number of leader replicas on a single broker. To prevent this Kafka has the concept of a preferred replica. When a topic is created, the first replica for each partition is designated as the preferred replica. Since Kafka is already making an effort to evenly distribute partitions across the available brokers, this will usually result in a good balance of leaders.

As leader elections occur for various reasons, the leaders might end up on non-preferred replicas and this could lead to an imbalance. So, Kafka will periodically check to see if there is an imbalance in leader replicas. It uses a configurable threshold to make this determination. If it does find an imbalance it will perform a leader rebalance to get the leaders back on their preferred replicas.

---

