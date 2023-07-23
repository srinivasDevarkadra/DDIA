<html lang="en"><head>



<h1>Chapter 6 - Partitioning</h1>
 
<i> <time class="postlist-date" datetime="2019-12-15">15 Dec 2019</time> </i> 
·
<span class="postlist-reading-time"> 9 min read  </span>


<p>My notes from Chapter 6 of 'Designing Data-Intensive Applications' by Martin Kleppmann.</p>
<p><strong>Table of Contents</strong></p>
<ul>
<li><a href="#partitioning-and-replication">Partitioning and Replication</a></li>
<li><a href="#partitioning-of-key-value-data">Partitioning of Key-Value Data</a>
<ul>
<li><a href="#partitioning-by-key-range">Partitioning by Key Range</a></li>
<li><a href="#partitioning-by-hash-of-key">Partitioning by Hash of Key</a></li>
<li><a href="#skewed-workloads-and-relieving-hot-spots">Skewed Workloads and Relieving Hot Spots</a></li>
</ul>
</li>
<li><a href="#partitioning-and-secondary-indexes">Partitioning and Secondary Indexes</a>
<ul>
<li><a href="#partitioning-secondary-indexes-by-document">Partitioning Secondary Indexes by Document</a></li>
<li><a href="#partitioning-secondary-indexes-by-term">Partitioning Secondary Indexes by Term</a></li>
</ul>
</li>
<li><a href="#rebalancing-partitions">Rebalancing Partitions</a>
<ul>
<li><a href="#operations-automatic-or-manual-rebalancing">Operations: Automatic or Manual Rebalancing</a></li>
</ul>
</li>
<li><a href="#request-routing">Request Routing</a></li>
</ul>
<hr>
<ul>
<li>This refers to breaking up a large data set into <em>partitions.</em></li>
<li>Partitioning is also known as <em>sharding.</em></li>
<li>Partitions are known as <em>shards</em> in MongoDB, Elasticsearch and SolrCloud, <em>regions</em> in Hbase, <em>tablets</em> in Bigtable, <em>vnodes</em> in Cassandra and Riak, and <em>vBucket</em> in Couchbase.</li>
<li>Normally, each piece of data belongs to only one partition.</li>
<li><em>Scalability</em> is the main reason for partitioning data. It enables a large dataset to be distributed across many disks, and a query load can be distributed across many processors.</li>
</ul>
<h3 id="partitioning-and-replication">Partitioning and Replication <a class="direct-link" href="#partitioning-and-replication" aria-hidden="true">#</a></h3>
<p>Copies of each partition are usually stored on multiple nodes. Therefore, although a record belongs to only one partition, it may be stored on several nodes for fault tolerance.</p>
<p>A node may also store more than one partition. If the leader-follower replication model is used for partitions, each node may be the leader for some partitions and a follower for other partitions.</p>
<h3 id="partitioning-of-key-value-data">Partitioning of Key-Value Data <a class="direct-link" href="#partitioning-of-key-value-data" aria-hidden="true">#</a></h3>
<p>The goal of partitioning is to spread data and query load evenly across nodes.</p>
<p>If partitioning is unfair i.e. some partitions have more data than others, we call it <em>skewed.</em></p>
<p>If the partitioning is skewed, it makes partitioning less effective as all the load could end up one partition, effectively making the other nodes idle. A partition with a disproportionately high load is called a <em>hot spot.</em></p>
<p>Assigning records to nodes randomly is the simplest approach for avoiding hot spots, but this has the downside that it will be more difficult to read a particular item since there's no way of knowing which node the item is on.</p>
<p>There are other approaches for this though, and we'll talk about them.</p>
<h4 id="partitioning-by-key-range">Partitioning by Key Range <a class="direct-link" href="#partitioning-by-key-range" aria-hidden="true">#</a></h4>
<p>For key-value data where the key could be a primary key, each partition could be assigned a continuous range of keys, say a-d, e-g etc. This way, once we know the boundaries between the ranges, it's easy to determine which partition contains a given key.</p>
<p>This range does not have to be evenly spaced. Some partitions could cover a wider range of keys than others, because the data may not be evenly distributed. Say we're indexing based on a name, one partition could hold keys for: u,v, w, x,y and z, while another could hold only a-c.</p>
<p>To distribute data evenly, the partition boundaries need to adapt to the data.</p>
<p>Hbase, Bigtable, RethinkDB etc use this partitioning strategy.</p>
<p>Keys could also be kept in sorted order within each partition. This makes range-scans effective e.g. find all records that start with 'a' .</p>
<p>The downside of this partitioning strategy is that some access patterns can lead to hotspots. E.g . If the key is a timestamp, then partitions will correspond to ranges of time e.g. one partition by day. This means all the writes end up going to the same partition for each day, so that partition could be overloaded with writes while others sit idle.</p>
<p>The solution for the example above is to use something apart from the timestamp as the primary key, but this is most effective if you know something else about the data. Say for sensor data, we could partition first by the sensor name and then by time. Though with this approach, you would need to perform a separate range query for each sensor name.</p>
<h4 id="partitioning-by-hash-of-key">Partitioning by Hash of Key <a class="direct-link" href="#partitioning-by-hash-of-key" aria-hidden="true">#</a></h4>
<p>Many distributed datastores use a hash function to determine the partition for a given key.</p>
<p>When we have a suitable hash function for keys, each partition can be assigned a range of hashes, and every key whose hash falls within a partition's range will be stored for that partition.</p>
<p>A good hash function takes skewed data and makes it uniformly distributed. The partition boundaries can be evenly spaced or chosen pseudo-randomly (with the latter being sometimes known as <em>consistent hashing)</em>.</p>
<hr>
<p><em>Aside:</em></p>
<blockquote>
<p>Consistent hashing, as defined by Karger et al, is a way of evenly distributing load across an internet-wide system of caches such as a content delivery network (CDN). It uses randomly chosen partition boundaries to avoid the need for central control or distributed consensus. Note that consistent here has nothing to do with replica consistency or ACID consistency, but rather describes a particular approach to rebalancing.</p>
</blockquote>
<blockquote>
<p>As we shall see in “Rebalancing Partitions”, this particular approach actually doesn’t work very well for databases, so it is rarely used in practice (the documentation of some databases still refers to consistent hashing, but it is often inaccurate). It's best to avoid the term <em>consistent hashing</em> and just call it hash partitioning instead.</p>
</blockquote>
<p>Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 5169-5174). O'Reilly Media. Kindle Edition.</p>
<hr>
<p>This approach has the downside that we lose the ability to do efficient range queries. Keys that were once adjacent are now scattered across all the partitions, so sort order is lost.</p>
<p>Note:</p>
<ul>
<li>Range queries on the primary key are not supported by Riak, Couchbase, or Voldermort.</li>
<li>MongoDB sends range queries to all the partitions.</li>
<li><em>Cassandra</em> has a nice compromise for this. A table in Cassandra can be declared with a compound primary key consisting of several columns. Only the first part of the key is hashed, but the other columns act as a concatenated index. Thus, a query cannot search for a range of values within the first column of a compound key, but if it specifies a fixed value for the first column, it can perform an efficient range scan over the other columns of the key.</li>
</ul>
<h4 id="skewed-workloads-and-relieving-hot-spots">Skewed Workloads and Relieving Hot Spots <a class="direct-link" href="#skewed-workloads-and-relieving-hot-spots" aria-hidden="true">#</a></h4>
<p>Even though hashing the key helps to reduce skew and the number of hot spots, it can't avoid them entirely. In an extreme case where reads and writes are for the same key, we still end up with all requests being routed to the same partition.</p>
<p>This workload seems unusual, but it's not unheard of. Imagine a social media site where a celebrity has lots of followers. If the celebrity posts something and that post has tons of replies, all the writes for that post could potentially end up at the same partition.</p>
<p>Most data systems today are not able to automatically compensate for a skewed workload. It's up to the application to reduce the skew. E.g if a key is known to be very hot, a technique is to add a random number to the beginning or end of the key to split the writes. This has the downside, though, that reads now have to do additional work to keep track of these keys.</p>
<h3 id="partitioning-and-secondary-indexes">Partitioning and Secondary Indexes <a class="direct-link" href="#partitioning-and-secondary-indexes" aria-hidden="true">#</a></h3>
<p>The partition techniques discussed so far rely on a key-value data model, where records are only ever accessed by their primary key. In these situations, we can determine the partition from that key and use it to route read and write requests to the partition responsible for the key.</p>
<p>If secondary indexes are involved though, this becomes more complex since they usually don't identify a record uniquely, but are a way of searching for all occurrences of a particular value.</p>
<p>Many key value stores like Hbase and Voldermort have avoided secondary indexes because of their complexity, but some like Riak have started adding them because of their usefulness for data modelling.</p>
<p>Secondary indexes are the bread &amp; butter of search servers like Elasticsearch and Solr though. However, a challenge with secondary indexes is that they do not map neatly to partitions. Two main approaches to partitioning a database with secondary indexes are:</p>
<ul>
<li>Document-based partitioning and</li>
<li>Term-based partitioning.</li>
</ul>
<h4 id="partitioning-secondary-indexes-by-document">Partitioning Secondary Indexes by Document <a class="direct-link" href="#partitioning-secondary-indexes-by-document" aria-hidden="true">#</a></h4>
<p>In this partitioning scheme, each document has a unique document ID. The documents are <em>initially</em> partitioned by that ID, which could be based on a hash range or term range of the ID. Each partition then maintains its own secondary index, covering only the documents that fit within that partition. It does not care what data is stored in other partitions.</p>
<p>A document-partitioned index is also known as a <em>local index.</em></p>
<p>The downside of this approach is that reads are more complicated. If we want to search by a term in the secondary index (say we're searching for all the red cars where there's an index on the color field), we would typically have to search through all the partitions, and then combine all the results we get back.</p>
<p>This approach to querying a partitioned db is sometimes known as <em>scatter/gather,</em> and it can make read queries on secondary indexes quite expensive. This approach is also prone to tail latency amplification (amplification in the higher percentiles), but it's widely used in Elasticsearch, Cassandra, Riak, MongoDB etc.</p>
<h4 id="partitioning-secondary-indexes-by-term">Partitioning Secondary Indexes by Term <a class="direct-link" href="#partitioning-secondary-indexes-by-term" aria-hidden="true">#</a></h4>
<p>In this approach, we keep a <em>global index</em> of the secondary terms that covers data in all the partitions. However, we don't store the global index on one node, since it would likely become a bottleneck, but rather, we partition it. The global index must be partitioned, but it can be partitioned differently from the primary key index.</p>
<p>The advantage of a global index over document-partitioned indexes is that reads are more efficient, since we don't need to scatter/gather over all partitions. A client only needs to make a request to the partition containing the desired document.</p>
<p>However, the downside is that writes are slower and more complicated, because a write to a single document may now affect multiple partitions of the index.</p>
<p>In practice, updates to global secondary indexes are often asynchronous (i.e. if you read the index shortly after a write, it may not have reflected a new change).</p>
<h3 id="rebalancing-partitions">Rebalancing Partitions <a class="direct-link" href="#rebalancing-partitions" aria-hidden="true">#</a></h3>
<p><strong>How not to do it: hash mod n</strong></p>
<p>If we partition by <em>hash mod n</em> where n is the number of nodes, we run the risk of excessive rebalancing. If the number of nodes N changes, most of the keys will need to be moved from one node to another. Frequent moves make rebalancing excessively expensive.</p>
<p>Next we'll discuss approaches that don't move data around more than necessary.</p>
<p><strong>Fixed Number of Partitions</strong></p>
<p>This approach is fairly simple. It involves creating more partitions than nodes, and assigning several partitions to each node.</p>
<p>If a node is added to a cluster, the new node <em>steals</em> a few partitions from other nodes until partitions are fairly distributed again. Only entire partitions are moved between nodes, the number of partitions does not change, nor does the assignment of keys to partitions.</p>
<p>What changes is the assignment of partitions to nodes. It may take some time to transfer a large amount of data over the network between nodes, so the old assignment of partitions is typically used for any reads/writes that happen while the transfer is in progress.</p>
<p>This approach is used in Elasticsearch (where the number of primary shards is fixed on index creation), Riak, Couchbase and Voldermort.</p>
<p>With this approach, it's imperative to choose the right number of partitions.</p>
<p><strong>Dynamic partitioning</strong></p>
<p>In this rebalancing strategy, the number of partitions dynamically adjusts to fit the size of the data. This is especially useful for databases with key range partitioning, as a fixed number of partitions with fixed boundaries can be inconvenient if not configured correctly at first. All the data could end up on one node, leaving the other empty.</p>
<p>When a partition grows to exceed a configured size, it is split into two partitions so that approximately half of the data ends up on each side of the split.</p>
<p>Conversely, if lots of data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent partition.</p>
<p>An advantage of this approach is that the number of partitions adapts to the total data volume. If there's only a small amount of data, a small number of partitions is sufficient, so overheads are small.</p>
<p>A downside of this approach is that an empty database starts off with a single partition, since there's no information about where to draw the partition boundaries. While the data set is small, all the writes will be processed by a single node while the others sit idle. Some databases like Hbase and MongoDB mitigate this by allowing an initial set of partitions to be configured on an empty database.</p>
<p><strong>Partitioning proportionally to nodes</strong></p>
<p>Some databases (e.g. Cassandra) have a third option of making the number of partitions proportional to the number of nodes. There is a fixed number of partitions <em>per node</em>.</p>
<p>The size of each partition grows proportionally to the dataset size while the number of nodes remains unchanged, but when you increase the number of nodes, the partitions become smaller again.</p>
<p>When a new node joins the cluster, it randomly chooses a fixed number of existing partitions to split, and then takes one half of the split, leaving the other half in place.</p>
<p>This randomization can produce an unfair split, but databases like Cassandra have come up with algorithms to mitigate the effect of that.</p>
<h4 id="operations%3A-automatic-or-manual-rebalancing">Operations: Automatic or Manual Rebalancing <a class="direct-link" href="#operations%3A-automatic-or-manual-rebalancing" aria-hidden="true">#</a></h4>
<p>Databases like Couchbase, Riak, and Voldermort are a good balance between automatic and manual rebalancing. They generate a suggested partitioned assignment automatically, but require an administrator to commit it before it takes effect.</p>
<p>Fully automated rebalancing can be unpredictable. Rebalancing is an expensive operation because it requires rerouting requests and moving a large amount of data from one node to another. This process can overload the network or the nodes if not done carefully.</p>
<p>For example, say one node is overloaded and is temporarily slow to respond to requests. The other nodes can conclude that the overloaded node is dead, and automatically rebalance the cluster to move load away from it. This puts additional load on the overloaded node, other nodes, and the network—making the situation worse and potentially causing a cascading failure.</p>
<h3 id="request-routing">Request Routing <a class="direct-link" href="#request-routing" aria-hidden="true">#</a></h3>
<p>There's an open question that remains: when a client makes a request, how does it know which node to connect to? It's especially important as the assignment of partitions to nodes changes.</p>
<p>This is an instance of the general problem of <em>service discovery i.e. locating things over a network.</em> This isn't just limited to databases, any piece of software that's accessible over a network has this problem.</p>
<p>There are different approaches to solving this problem on a high level:</p>
<ul>
<li>Allow clients to contact any node (e.g. via a round-robin load balancer). If the node happens to own the partition to which the request applies, it can handle the request directly. Otherwise, it'll forward it to the relevant node.</li>
<li>Send all requests from clients to a routing tier first, which determines what node should handle what request and forwards it accordingly. This routing tier acts as a partition-aware load balancer.</li>
<li>Require that clients be aware of the partitioning and assignment of partitions to nodes.</li>
</ul>
<p>In all approaches, it's important for there to be a <em>consensus</em> among all the nodes about which partitions belong to which nodes, especially as we tend to rebalance often.</p>
<p>Many distributed systems rely on a coordination service like Zookeeper to keep track of this cluster metadata. This way, the other components of the system such as the routing tier or the partitioning-aware client can subscribe to this information in Zookeeper.</p>


<p> 
      <a href="/tags/learning-diary/" class="tag">learning-diary</a>
      <a href="/tags/distributed-systems/" class="tag">distributed-systems</a>
      <a href="/tags/ddia/" class="tag">ddia</a> </p>
