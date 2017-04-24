# A Simple Reference Implementation of Immutable Data Store

## Objective

The goal of the project is to build a simple reference implementation
of immutable data store. The design is highly
inspired by [Cockroach DB](http://cockroachlabs.com).

The data store is basically a yet another key-value blob store. e
focus on keeping the initial design simple as much as possible so that
we can extend it later easily.


### API

The API we provide is write and read of a blob. Each blob is
specified by a unique key. The API will look like below:

```go
Get(key, position) (blob, error)
Put(key, blob) error
List(keyPattern) (keys, error)
Delete(key) error
```

There is no secondary key or hierarchical namespace like a typical
file system. All written data are immutable, and there is no
guaranteed data locality.

Transactional guarantee is also minimum, and the system provides just
eventual consistency. A blob that is being written is not visible, and
once the write completes, the blob will become visible to others
eventually.


### Basic Architecture

The system consists a centralized master and storage nodes. The master
manages all metadata (e.g., location of data stripe). The master has N
replicas, and [RAFT](https://raft.github.io/raft.pdf) is used to
manage a leader lease. Only the leader can accept read/write to
metadata. RAFT is also used by The leader and followers to sync their
states (note on statement-based replication and data-based
replication). The backed storage for RAFT is
[RocksDB](http://rocksdb.org/).

The master does minimum monitoring of each storage node. The master
sends period heartbeat for basic health check, and it collects node
stats like disk limit/usage. Based on the collected data, the master
makes a decision on where data should be allocated and replicated.
Failure domain information (e.g., datacenter, power, switch, rack) is
also collected so that data we can tolerate planned/unplanned outages.

When a client creates a new blob, it first talks to the master and
finds a set of storage nodes where the blob should be stored. Then,
the client starts writing the blob to the selected storage nodes. The
blob is erasure-encoded. When reading a blob, a client also first
talks to the master to find the location of the blob. Then, the client
reads the blob from a nearby storage node.


### Blob

A blob is an immutable object encoded by
[Reed-Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction).
Each blob is split into multiple "stripes". For each stripe, we create
N data chunks and M code chunks. Here is a conceptual proto
definition:

```proto
message Blob {
  optional Metadata metadata;
  repeated Stripe stripes;
}

message Metadata {
  optional string primary_key;
  optional UUID uuid;
  optional uint64 size;
  ...
}

message Stripe {
  repeated Chunk chunks;
}

message Chunk {
  enum Type {
     DATA, CODE
   }
  optional Type type;
  optional bytes data;
}
```

The default stripe size is 64MB(?). Replication parameters 
can be configured for each blob.

Clients are responsible for encoding and decoding blobs. When creating
a new blob, a client talks to the master to find a location where the
blob should be created and then transfers the blob data to each
selected storage node.


### Master 

The master is a centralized component in the system. The role of the
master includes blob management/scheduling, transaction management,
and storage node management. Each manager runs concurrently and shares
its state with each other via a storage layer. The master also has
several periodic background jobs for blob reconstruction and garbage
collection.

The master has N instances (one leader and N-1 followers) distributed
over failure domains, and only the elected leader can mutate its
state. The master persists its state to a data store that propagates
data changes to other replicas using RAFT. All RPC requests are served
by the elected leader.

We will describe each component of the master in more details.


#### Blob Management

The master manages where blobs are located. The information is updated
when a new blob is created or a blob is reconstructed.

To be more specific, the master keeps track of the following maps:

```
primary key -> blob UUID
blob UUID -> (metadata, a set of storage nodes)
```

The maps are updated at the end of a transaction that creates a new blob
or deletes a blob. We have indirection with UUID as there can be more
than one blob for the same primary key when blobs are marked as
deleted but they haven't yet been completely erased by the garbage
collector.

The above data structure implies that there will be no data locality
based on primary keys. If we assume data proximity, we would split a
key space with ranges and allocate each range to a storage node:

```
(start key, end key) -> a set of storage nodes
```

Each storage node then keeps a mapping from primary keys to blob UIDs
and their metadata.


#### Transaction Management

A new transaction is opened when a client starts creating a new blob
is created. The transaction is is closed when the client completes
writing the blob.

As blobs are immutable, our transaction mechanism needs to support
very basic functionality. The transaction manager just keeps track of
ongoing writes with the following map:

```
txn UUID -> (txn state, blob UUID)
```

While a transaction is open, a client periodically sends a heartbeat
request to the master. The master considers the transaction is aborted
when it doesn't receive a heartbeat request for a certain period of time.
When a transaction is aborted, the blob manager is notified and 
marks the blob as deleted so that it can be cleaned up later.


#### Storage Node Management

The master manages and monitors storage nodes that belong to the system:

```
storage node UUID -> storage node info
```

When a storage node joins or leaves the system, it notifies the master
by sending an RPC request. A storage node also notifies the master
when it will become inaccessible for a short period of time due to a
planned maintenance.

The master periodically sends heartbeat requests to each storage node
to check its health. The master also collects the stats of each node
(e.g., a set of storage devices and their capacity limits). The master
reconciles a state reported from a storage node (e.g., node health)
and an intended state requested by a system administrator (e.g.,
maintenance) and determines whether the node is able to serve
read/write blob requests. Here is a sketch of the proto definition:

```proto
message StorageNode {
  optioanl UUID uuid;
  repeated Label labels;  // a set of (key, value)s
  repeated Device devices;
  
  optional Timestamp last_successful_heartbeat_at;
  repeated Event events;  // scheduled/past events
  optional State intended_state; // state that an admin intended.
  
  // optional State actual_state;
}

messase Device {
  optional string name;
  optional uint64 limit;
  optional uint64 used;
  optional uint64 reserved;
}
```

The reservation is updated when a new transaction is opened and 
the master decided to create a blob on that node.

We could have some some aggregator to avoid pinning all storage 
nodes in the system.

#### Blob Creation and Scheduling

The scheduler picks up a set of storage nodes where a new blob is
created and replicated. The master also needs to determine how the
data is propagated to the selected storage nodes.

This blob scheduling algorithm consists of two steps: feasibility check
and scoring. 

Our initial replication strategy is simple, and the feasibility
conditions include the following:

- For a given blob, find (N+M) storage nodes where the blob can be stored.
  Each node has sufficient free storage space to store the blog (i.e.,
  `blob.size <= node.capacity - node.usage`). 
  
- The above selected storage nodes are distributed well over failure
  domains. For example, if N is 5, and we have three datacenters, we
  pick up two nodes from two datacenters one node from the other
  datacenter.
  
The scoring function can be "best fit" (= pack blobs to a small number
of machines), "worst fit" (= spread blobs evenly among storage nodes
evenly), or hybrid of the two algorithms (see 
[this paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43438.pdf) 
and [this paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43103.pdf)).
  
Note that each storage node stores data or code chunks of all the
stripes of a given blob. We do not pick up a different set of storage
nodes for each stripe.

Again the above algorithm assumes that there is no data locality 
we can utilize based on primary keys. If we assign a range of keys
to each storage node, that will determine blob allocation and 
no extra scheduling functionality is needed.

Once a set of storage nodes are determined, the master attempts to
commit a result. When it fails to commit the result due to concurrent
update to storage node status, the master will retry.

The master also determines how a blob should be transferred and
constructs a spanning tree of nodes a (e.g., send data to one node in
datacenter X, and then the node transfers the data to another node in
the same datacenter).

```
Client --> node A in datacenter X --> node B in datacenter X --> node C in datacenter Z
  |
  +------> node D in datacenter Y --> node E in datacenter Y
  |
  +------> node F in datacenter Z --> node E in datacenter Z
```

#### Blob Creation Flow

Client interacts with the master in the following way to create a new blob:

1. Client sends a `CreateBlob` request to the master.
1. The master creates a new transaction and updates the transaction map.
1. The master finds a set of storage nodes where a blob should be created.
1. The master sends a response back to the client.
1. The client starts writing the blob by talking to one of the selected storage nodes.
1. The selected storage node replicates the blob to the rest of the storage nodes 
1. The storage nodes finish writing the blob and notify the client.
1. The client sends a request to the master to close the transaction.
1. The master updates the blob location map.


#### Background Job: Blob Reconstruction

This periodic background job triggers reconstruction when a blob loses
N replicas.

TODO(kaneda): Describe a condition where reconstruction is triggered.

#### Background Job: Garbage Collection

This periodic background job erases blobs that are marked as deleted.

For these background job, we can checkpoint its progress in a
persistent store.


#### Leader Election and Data Replication

To provide high availability, the master consists of five instances,
and one of the instances needs to be a leader serving read/write
requests and running periodic background jobs. Also, the master
instance needs to keep their states consistent with one another.

A typical solution to the problem is to use a consensus algorithm. A
leader is elected by the consensus algorithm. An elected leader then
propagates any write to its followers.
[RAFT](https://raft.github.io/raft.pdf) is one common candidate for
consensus algorithm since it has a simple, easy to understand protocol
and various reference implementations used in real (e.g.,
[etcd](https://github.com/coreos/etcd/tree/master/raft)).

There are two separate design points:

- whether we need to build a leader election mechanism on top of a consensus algorithm
- whether we take a statement-based replication or data-based replication

Hereafter we will discuss each design point.

##### Leader Election

A consensus algorithm like RAFT elects a leader. We can make this RAFT
leader serve read/write requests or run a simple leader election
algorithm on top of the consensus algorithm. 

More specifically, here are the invariants that a leader election
algorithm needs to satisfy:

- There is at most one leader at a given point in time.
- A new leader is elected eventually if there are a majority of active
  instances and they can talk to each other.
- When a new leader is elected, the elected master sees all previous
  writes made to the store. There will be no more writes between the
  last write and the leader election event.

API we require is following:

```go
getLeader() (master UUID, expiration time)
```

A RAFT leader satisfies these invariants, but an implementation of
RAFT might not provide sufficient API. Thus, we build a leader
election on top of RAFT.

A typical way to implement a leader election mechanism is to use a
lease. Each leader election is associated with a lease. A lease needs
to be updated for a certain interval (e.g., every minute), and a lease
will expire when it is not updated for a long period time (e.g., 5
minutes). Here is a pseudo proto definition of the leader lease:

```proto
message LeaderLease {
  optional MasterUUID leader;
  optional Timestamp start_at;
  optional Timestamp end_at;
  optional uint64 version;
}
```

An underlying consensus algorithm guarantees that each master
`instance` see the same leader leases. An instance considers itself as
an elected leader if `lease.leader` is equal to the UUID of the
instance and `lease.start_at <= now() < lease.end_at`. As clock can
skew, a buffer is added between the end time of a lease and the start
time of a new lease.

Let's suppose that the a consensus algorithm provides `Get`, `Put`,
and `PutIf` as its high level API. The pseudocode for acquiring a new
lease and extending a lease can be written as follows:

```go
func AcquireLease() {
  currLease := Get()
  newLease := LeaderLease{
    leader:   myUUID,
    start_at: currLease.end_at + BufferMs
    end_at:   currLease.end_at + LeasePeriodMs
    version:  currLease.version + 1
  }
  // Write a new lease if the current lease is equal to currLease.
  PutIf(newLease, currLease)
}
```

```go
func ExtendLease() {
  currLease := Get()
  newLease := LeaderLease{
    leader:   currLease.leader
    start_at: currLease.start_at
    end_at:   currLease.end_at + LeasePeriodMs
    version:  currLease.version + 1
  }
  PutIf(newLease, currLease)
}
```

An update to `LeaderLease` is optimistically locked (i.e., a write
succeeds only when `nextVersion = currVersion + 1` where `nextVersion`
is the `version` of the proposed new lease and `currVersion` is the
`version` of the current `LeaderLease`). A consensus algorithm can
guarantee that a write to `LeaderLease` is seen after all previous
writes are made.

A leader can also release a lease gracefully when it is being shut
down. It just needs to update `LeaderLease` and shortens `end_at`.

Here is pseudocode for serving a write request:

```go
func HandleRequest() error {
  currLease := Get()
  if now() > curr.end_at <= now() {
    if curr.leader == myself {
      ExtendLease()
    } else {
      AcquireLease()
    }
    currLease = Get()
  }
  
  if now() < curr.start_at {
    return Error.new("No elected leader")
  }
    
  if curr.leader != myself {
    return ForwardRequestToElectedLeader(curr.leader, ...)
  }
  
  DoSomething()
  
  if now() > curr.end_at() {
    return Error.new("Leader lease expired")
  }
  return Commit()
}
```

`Commit()` updates persisted data if the write is initiated by the
current leader. Note that `Commit` needs to be a RAFT command to guarantee that 
a leader lease won't be updated while `Commit` is being executed.

One drawback of the above approach is that we cannot co-locate an
elected master and a RAFT leader on the same host.


##### Data Replication

We have basically two approaches as discussed in a database world
(e.g., [MySQL](https://dev.mysql.com/doc/refman/5.7/en/replication-formats.html))
and various other replication methods (e.g.,
[optimistic replication](http://dl.acm.org/citation.cfm?doid=1057977.1057980)).

- *Statement-based replication*: An elected master proposes a high-level
  command (e.g., create a new blob) to RAFT. Followers execute the
  proposed command in the same as master to keep the state in sync.
  
- *Data-based replication*: An elected master proposes a low-level command
  to update specific data (e.g., update the value of key X to Y). Followers
  execute the command to update their persistent store (and update in-memory 
  derived states if needed).

While the statement-based replication can provide more structured API,
it adds extra complexity to the system. For example, we need to make
sure each command has a deterministic behavior (otherwise, followers'
states will diverge). We also need to reason carefully when
introducing a backward incompatibility change.

As for the data-based replication, we basically build a key-value
store on top of RAFT. The key-value store provides a limited set of
API such as `Put(key, value)`. We take this approach here and
discussed in more details later.

One thing to note is that the elected master has in-memory states
derived from the persisted store. Derived in-memory states can be for
data caching or additional indexing. For example, the master construct
an in-memory table for keeping track of open transactions. The derived
state needs to be initialized when a new master is elected. From
software engineering point of view, such derived states can add
additional complexity, and we would avoid as much as possible.


#### RAFT Backing Storage

As discussed above, we build a key-value store on top of RAFT.
[RocksDB](http://rocksdb.org/) is used for its backing store.

More specifically, each RAFT command has an ordered list of operations
that are executed sequentially. Each operation is one of the followings:

- `Put(key, value)`: Write `value` to key `key`.
- `PutIf(key, newValue, currValue)`: Write `value` to key `key` if the
  current value is equal to `currValue`. Otherwise abort the entire command.
- `Delete(key)`: Delete `key`.

Each command is atomic (i.e., writes are committed at the end of the
command). Each command succeeds only when it is initiated by the
currently elected master. Here is a proto definition sketch:

```proto
message RAFTCommand {
  optional MasterUUID leader;
  repeated Operation ops;
}

message Operation {
  oneOf {
    optional Put;
    optional PutIf;
    optional Delete;
  }
}
```

In addition to the write operations, we provide read operations
`Get(key)` and `Scan(startKey, endKey)`.

Note that read operations do not go to RAFT. Also, read operations can
be executed in parallel unless there are ongoing write operations that
have overlapping keys. When there is an uncommitted write, a read will
see the uncommitted value. We provide additional two methods to
indicate the beginning and the end of a transaction: `NewSession()`
and `Commit(session)`. Read commands can be issued without a session,
but write commands need to be wrapped by a session.

Here is the architecture diagram. 

```
   Get, Put, Commit, ...
        | 
        v
+---------------+      +-----------------+
| Command Queue | ---> | Session Manager |
+---------------+      +-----------------+
        |                      |
        v                      v
   +---------+           +-------------+
   | RocksDB | <-------- | RAFT Engine |
   +---------+           +-------------+
```

All commands are first submitted to the command queue. The queue
dispatches a command for key K if there is no ongoing command that
touches K. 

The session manager keeps track of sessions, each of which
has uncommitted writes. When `Commit` is called, all uncommitted
writes are submitted to the RAFT engine.

Both the command queue and RAFT talk to RocksDB.

Some discussion points:

- Whether cache can improve the performance.
- Whether storing historical changes helps.

#### Administration Configuration

In principle we would like to minimize the amount of required manual
configuration, but the system cannot be zero manual configuration. For
example, master instances need to know the address of each another.
Each client need to locate the master with DNS or other name service.

### Storage Node

Each storage node is a key-value store also backed by RocksDB. The
storage node updates its store based on RPC requests from the master
and DB clients.

The RPC interface exposed to the master is following:

```go
getStats()
CreateBlob()
DeleteBlob()
```

The RPC interface exposed to the DB client is following:

```go
PutChunk()
GetChunk()
```

The key space is partitioned into multiple subspaces, each of which
has a unique prefix:

- Blob subspace: `blob:<uuid>:<stripe_id>`
- System variables: `system:<var_name>`


### Client

The client encodes and decodes blobs.

When data transmission fails due to a transient failure, each node
retries the transmission retries multiples times. If data transmission
fails due to a non-transient storage node failure, the client aborts
the entire process and starts from the beginning.
When fetching blobs, clients access their near-by datacenters and
reconstruct the blobs from their stripes When clients encounter
failures (e.g., RPC timeout), the clients first retry in the same
datacenter multiple times and then fall back to other datacenters.


## Observation on the Naive Design/Implementation 

The above naive design/implementation reveals many shortcomings and
possible areas where we can optimize and improve. Here are some
obvious limitations:

- System stability coming from various engineering efforts and experience from production usage.
- Performance and availability
- Bottleneck with the centralized master architecture.
- Failure detection and node discovery to reduce configuration
- Planned maintenance should be gracefully handled.
- Automatic retry on failures
- Secondary indexing and other lookup API
- Visibility to the internal state of the systems.
- Security (e.g., ACL)
- Data load balancing
- Data locality
- In-memory caching
- Co-processor
- Monitoring and failure recovery
- Naive encoding schema
- Data compression (if needed)
- Backup
- Bandwidth throttling
- Rebalancing
- Heterogeneous resource management
- Watcher API
- Bandwidth control

## Comparison with Other Storage Systems

There are many existing storage systems. To name a few,

- Large-scale file systems (e.g., Google File System, Colossus, HDFS)
- Key-value stores (e.g., Bigtable, HBase, Cassandora)
- Spanner, Cockroach DB
- Facebook Haystack (photo storage)
- MongoDB, ElasticSearch
- Amazon Glacier

## References

- [Building a Highly-scalable Highly-available Transactional Data Store](https://github.com/kkaneda/slides/blob/master/cockroach.pdf)
- [Blob storage discussion on Cockroach DB](https://github.com/cockroachdb/cockroach/issues/243)
