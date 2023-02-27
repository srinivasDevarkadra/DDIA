<!DOCTYPE html>
<html lang="en"
  <body>

    <main class="tmpl-post">


<h1>Chapter 5 - Replication</h1>
 
<i> <time class="postlist-date" datetime="2019-12-13">13 Dec 2019</time> </i> 
·
<span class="postlist-reading-time"> 20 min read  </span>


<p>My notes from the fifth chapter of Martin Kleppmann's book: Designing Data Intensive Applications.</p>
<p><strong>Table of Contents</strong></p>
<ul>
<li><a href="#leaders-and-followers">Leaders and Followers</a>
<ul>
<li><a href="#synchronous-versus-asynchronous-replication">Synchronous Versus Asynchronous Replication</a>
<ul>
<li><a href="#synchronous-replication">Synchronous Replication</a></li>
<li><a href="#asynchronous-replication">Asynchronous Replication</a></li>
<li><a href="#setting-up-new-followers">Setting Up New Followers</a></li>
<li><a href="#handling-node-outages">Handling Node Outages</a></li>
<li><a href="#scenario-a---follower-failure%3A-catch-up-recovery">Scenario A - Follower Failure: Catch-up recovery</a></li>
<li><a href="#scenario-b---leader-failure%3A-failover">Scenario B - Leader failure: Failover</a></li>
<li><a href="#implementation-of-replication-logs">Implementation of Replication Logs</a></li>
</ul>
</li>
</ul>
</li>
<li><a href="#problems-with-replication-lag">Problems with Replication Lag</a>
<ul>
<li><a href="#other-consistency-levels">Other Consistency Levels</a></li>
<li><a href="#solutions-for-replication-lag">Solutions for Replication Lag</a></li>
</ul>
</li>
<li><a href="#multi-leader-replication">Multi-Leader Replication</a>
<ul>
<li><a href="#use-cases-for-multi-leader-replication">Use Cases for Multi-Leader Replication</a></li>
<li><a href="#handling-write-conflicts.">Handling Write Conflicts.</a>
<ul>
<li><a href="#synchronous-versus-asynchronous-conflict-detection">Synchronous versus asynchronous conflict detection</a></li>
</ul>
</li>
<li><a href="#conflict-avoidance">Conflict Avoidance</a></li>
<li><a href="#converging-toward-a-consistent-state">Converging toward a consistent state</a></li>
<li><a href="#custom-conflict-resolution-logic">Custom Conflict Resolution Logic</a></li>
<li><a href="#multi-leader-replication-topologies">Multi-Leader Replication Topologies</a></li>
</ul>
</li>
<li><a href="#leaderless-replication">Leaderless Replication</a>
<ul>
<li><a href="#preventing-stale-reads">Preventing Stale Reads</a></li>
<li><a href="#read-repair-and-anti-entropy">Read repair and anti-entropy</a></li>
<li><a href="#quorums-for-reading-and-writing">Quorums for reading and writing</a></li>
<li><a href="#limitations-of-quorum-consistency">Limitations of Quorum Consistency</a>
<ul>
<li><a href="#monitoring-staleness">Monitoring Staleness</a></li>
</ul>
</li>
<li><a href="#sloppy-quorums-and-hinted-handoff">Sloppy Quorums and Hinted Handoff</a>
<ul>
<li><a href="#multi-datacenter-operation">Multi-datacenter operation</a></li>
</ul>
</li>
<li><a href="#detecting-concurrent-writes">Detecting Concurrent Writes</a>
<ul>
<li><a href="#last-write-wins-%28discarding-concurrent-writes%29">Last write wins (discarding concurrent writes)</a></li>
<li><a href="#the-%22happens-before%22-relationship-and-concurrency">The "happens-before" relationship and concurrency</a></li>
<li><a href="#capturing-the-happens-before-relationship">Capturing the happens-before relationship</a></li>
<li><a href="#merging-concurrently-written-values">Merging Concurrently Written Values</a></li>
<li><a href="#version-vectors">Version Vectors</a></li>
</ul>
</li>
</ul>
</li>
</ul>
<hr>
<p>Replication involves keeping a copy of the same data on multiple machines connected via a network. Reasons for this involve:</p>
<ul>
<li>Increasing the number of machines that can handle failed requests - Leads to increased read throughput.</li>
<li>To allow a system continue working in the event of failed parts - Leads to increased availability.</li>
<li>To keep data geographically close to users - Reduced latency.</li>
</ul>
<p>The challenge with replication lies in handling changes to replicated data. Three algorithms for replicating changes between nodes:</p>
<ul>
<li>Single leader</li>
<li>Multi-leader</li>
<li>Leaderless</li>
</ul>
<h3 id="leaders-and-followers">Leaders and Followers <a class="direct-link" href="#leaders-and-followers" aria-hidden="true">#</a></h3>
<p>Every node that keeps a copy of data is a <em>replica</em>. Obvious question is: how do we make sure that the data on all the replicas is the same? The most common approach for this is <strong><em>leader-based replication</em>.</strong> In this approach:</p>
<ol>
<li>Only the leader accepts writes.</li>
<li>The followers read off a replication log and apply all the writes in the same order that they were processed by the leader.</li>
<li>A client can query either the leader or any of its followers for read requests.</li>
</ol>
<p>So here, the followers are read-only, while writes are only accepted by the leader.</p>
<p>This approach is used by MySQL, PostgreSQL etc., as well as non-relational databases like MongoDB, RethinkDB, and Espresso.</p>
<h4 id="synchronous-versus-asynchronous-replication">Synchronous Versus Asynchronous Replication <a class="direct-link" href="#synchronous-versus-asynchronous-replication" aria-hidden="true">#</a></h4>
<p>With Synchronous replication, the leader must wait for a positive acknowledgement that the data has been replicated from at least one of the followers before terming the write as successful, while with Asynchronous replication, the leader does not have to wait.</p>
<h5 id="synchronous-replication">Synchronous Replication <a class="direct-link" href="#synchronous-replication" aria-hidden="true">#</a></h5>
<p>The advantage of synchronous replication is that if the leader suddenly fails, we are guaranteed that the data is available on the follower.</p>
<p>The disadvantage is that if the synchronous follower does not respond (say it has crashed or there's a network delay or something else), the write cannot be processed. A leader must block all writes and wait until the synchronous replica is available again. Therefore, it's impractical for all the followers to be synchronous, since just one node failure can cause the system to become unavailable.</p>
<p>In practice, enabling synchronous replication on a database usually means that <em>one</em> of the followers is synchronous, and the others are asynchronous. If the synchronous one is down, one of the asynchronous followers is made synchronous. This configuration is sometimes called <em>semi-synchronous</em>.</p>
<h5 id="asynchronous-replication">Asynchronous Replication <a class="direct-link" href="#asynchronous-replication" aria-hidden="true">#</a></h5>
<p>In this approach, if the leaders fails and is not recoverable, any writes that have not been replicated to followers are lost.</p>
<p>An advantage of this approach though, is that the leader can continue processing writes, even if all its followers have fallen behind.</p>
<p>There's some research into how to prevent asynchronous-performance like systems from losing data if the leader fails. A new replication method called <em>Chain replication</em> is a variant of synchronous replication that aims to provide good performance and availability without losing data.</p>
<h5 id="setting-up-new-followers">Setting Up New Followers <a class="direct-link" href="#setting-up-new-followers" aria-hidden="true">#</a></h5>
<p>New followers can be added to an existing cluster to replace a failed node, or to add an additional replica. The next question is how to ensure the new follower has an accurate copy of the leader's data?</p>
<p>Two options that are not sufficient are:</p>
<ul>
<li>Just copying data files from one node to another. The data in the leader is always updating and a copy will see different versions at different points in time.</li>
<li>Locking the database (hence making it unavailable for writes). This will go against the goal of high availability.</li>
</ul>
<p>There's an option that works without downtime, which involves the following steps:</p>
<ol>
<li>Take a consistent snapshot of the leader's db at some point in time. It's possible to do this without taking a lock on the entire db. Most databases have this feature.</li>
<li>Copy the snapshot to the follower node.</li>
<li>The follower then requests all the data changes that happened since the snapshot was taken.</li>
<li>When the follower has processed the log of changes since the snapshot, we say it has caught up.</li>
</ol>
<p>In some systems, this process is fully automated, while in others, it is manually performed by an administrator.</p>
<h5 id="handling-node-outages">Handling Node Outages <a class="direct-link" href="#handling-node-outages" aria-hidden="true">#</a></h5>
<p>Any node can fail, therefore, we need to keep the system running despite individual node failures, and minimize the impact of a node outage. How do we achieve high availability with leader-based replication?</p>
<h5 id="scenario-a---follower-failure%3A-catch-up-recovery">Scenario A - Follower Failure: Catch-up recovery <a class="direct-link" href="#scenario-a---follower-failure%3A-catch-up-recovery" aria-hidden="true">#</a></h5>
<p>Each follower typically keeps a local log of the data changes it has received from the leader. If a follower node fails, it can compare its local log to the replication log maintained by the leader, and then process all the data changes that occurred when the follower was disconnected.</p>
<h5 id="scenario-b---leader-failure%3A-failover">Scenario B - Leader failure: Failover <a class="direct-link" href="#scenario-b---leader-failure%3A-failover" aria-hidden="true">#</a></h5>
<p>This is trickier: One of the nodes needs to be promoted to be the new leader, clients need to be reconfigured to send their writes to the new leader, and the other followers need to start consuming data changes from the new leader. This whole process is called a <em>failover.</em> Failover can be handled manually or automatically. An automatic failover consists of:</p>
<ol>
<li><em>Determining that the leader has failed</em>: Many things could go wrong: crashes, power outages, network issues etc. There's no foolproof way of determining what has gone wrong, so most systems use a timeout. If the leader does not respond within a given interval, it's assumed to be dead.</li>
<li><em>Choosing a new leader:</em> This can be done through an election process (where the new leader is chosen by a majority of the remaining replicas), or a new leader could be appointed by a previously elected <em>controller node.</em> The best candidate for leadership is typically the one with the most up-to-date data changes from the old leader (to minimize data loss)</li>
<li><em>Reconfiguring the system to use the new leader:</em> Clients need to send write requests to the new leader, and followers need to process the replication log from the new leader. The system also needs to ensure that when the old leader comes back, it does not believe that it is still the leader. It must become a follower.</li>
</ol>
<p>There are a number of things that can go wrong during the failover process:</p>
<ul>
<li>For asynchronous systems, we may have to discard some writes if they have not been processed on a follower at the time of the leader failure. This violates clients' durability expectations.
<blockquote></blockquote>
</li>
<li>Discarding writes is especially dangerous if other storage systems are coordinated with the database contents. For example, say an autoincrementing counter is used as a MySQL primary key and a redis store key, if the old leader fails and some writes have not been processed, the new leader could begin using some primary keys which have already been assigned in redis. This will lead to inconsistencies in the data, and it's what happened to Github (<a href="https://github.blog/2012-09-14-github-availability-this-week/" title="https://github.blog/2012-09-14-github-availability-this-week/">https://github.blog/2012-09-14-github-availability-this-week/</a>).
<blockquote></blockquote>
</li>
<li>In fault scenarios, we could have two nodes both believe that they are the leader: <em>split brain.</em> Data is likely to be lost/corrupted if both leaders accept writes and there's no process for resolving conflicts. Some systems have a mechanism to shut down one node if two leaders are detected. This mechanism needs to be designed properly though, or what happened at Github can happen again( <a href="https://github.blog/2012-12-26-downtime-last-saturday/" title="https://github.blog/2012-12-26-downtime-last-saturday/">https://github.blog/2012-12-26-downtime-last-saturday/</a>)
<blockquote></blockquote>
</li>
<li>It's difficult to determine the right timeout before the leader is declared dead. If it's too long, it means a longer time to recovery in the case where the leader fails. If it's too short, we can have unnecessary failovers, since a temporary load spike could cause a node's response time to increase above the timeout, or a network glitch could cause delayed packets. If the system is already struggling with high load or network problems, unnecessary failover can make the situation worse.</li>
</ul>
<h5 id="implementation-of-replication-logs">Implementation of Replication Logs <a class="direct-link" href="#implementation-of-replication-logs" aria-hidden="true">#</a></h5>
<p>Several replication methods are used in leader-based replication. These include:</p>
<p><strong>a) Statement-based replication:</strong> In this approach, the leader logs every write request (statement) that it executes, and sends the statement log to every follower. Each follower parses and executes the SQL statement as if it had been received from a client.</p>
<ul>
<li>A problem with this approach is that a statement can have different effects on different followers. A statement that calls a nondeterministic function such as NOW() or RAND() will likely have a different value on each replica.</li>
<li>If statements use an auto-incrementing column, they must be executed in exactly the same order on each replica, or else they may have a different effect. This can be limiting when executing multiple concurrent transactions, as statements without any causal dependencies can be executed in any order.</li>
<li>Statements with side effects (e.g. triggers, stored procedures) may result in different side effects occurring on each replica, unless the side effects are deterministic.</li>
</ul>
<p>Some databases work around this issues by requiring transactions to be deterministic, or configuring the leader to replace nondeterministic function calls with a fixed return value.</p>
<p><strong>b) Write-ahead log (WAL) shipping:</strong> The log is an append-only sequence of bytes containing all writes to the db. Besides writing the log to disk, the leader can also send the log to its followers across the network.</p>
<p>The main disadvantage of this approach is that the log describes the data on a low level. It details which bytes were changed in which disk blocks. This makes the replication closely coupled to the storage engine. Meaning that if the storage engine changes in another version, we cannot have different versions running on the leader and the followers, which prevents us from making zero-downtime upgrades.</p>
<p><strong>c) Logical (row-based) log replication:</strong> This logs the changes that have occurred at the granularity of a row. Meaning that:</p>
<ul>
<li>For an inserted row, the log contains the new values of all columns.</li>
<li>For a deleted row , the log contains enough information to identify the deleted row. Typically the primary key, but it could also log the old values of all columns.</li>
<li>For an updated row, it contains enough information to identify the updated row, and the new values of all columns.</li>
</ul>
<p>This decouples the logical log from the storage engine internals. Thus, it makes it easier for external applications (say a data warehouse for offline analysis, or for building custom indexes and caches) to parse. This technique is called <em>change data capture.</em></p>
<p><strong>d) Trigger-based replication:</strong> This involves handling replication within the application code. It provides flexibility in dealing with things like: replicating only a subset of data, conflict resolution logic, replicating from one kind of database to another etc. <em>Trigger</em> and <em>Stored procedures</em> provide this functionality. This method has more overhead than other replication methods, and is more prone to bugs and limitations than the database's built-in replication.</p>
<h3 id="problems-with-replication-lag">Problems with Replication Lag <a class="direct-link" href="#problems-with-replication-lag" aria-hidden="true">#</a></h3>
<p><strong>Eventual Consistency</strong>: If an application reads from an asynchronous follower, it may see outdated information if the follower has fallen the leader. This inconsistency is a temporary state, and the followers will eventually catchup. That's <em>eventual consistency.</em></p>
<p>The delay between when a write happens on a leader and gets reflected on a follower is <em>replication lag</em>.</p>
<h4 id="other-consistency-levels">Other Consistency Levels <a class="direct-link" href="#other-consistency-levels" aria-hidden="true">#</a></h4>
<p>There are a number of issues that can occur as a result of replication lag. In this section, I'll summarize them under the minimum consistency level needed to prevent it from happening.</p>
<p><strong>a) Reading Your Own Writes:</strong> If a client writes a value to a leader and tries to read that same value, the read request might go to an asynchronous follower that has not received the write yet as a result of replication lag. The user might think the data was lost, when it really wasn't. The consistency level needed to prevent this situation is known as <em>read-after-write consistency</em> or <em>read-your-writes consistency.</em> It makes the guarantee that a user will always see their writes. There are a number of various techniques for implementing this:</p>
<ul>
<li>When reading a field that a user might have modified, read it from the leader, else read it from a follower. E.g. A user's profile on a social network can only be modified by the owner. A simple rule could be that user's profiles are always read from the leader, and other users' profiles are read from a follower.</li>
<li>Of course this won't be effective if most things are editable by the user, since it'll drive most reads to the leader. Another option is to keep track of the time of the update, and only read from followers whose last updated time is at least that. The timestamp could be a logical one, like a sequence of writes.</li>
</ul>
<p>There's an extra complication with this if the same user is accessing my service across multiple devices say a desktop browser and a mobile app. They might be connected through different networks, yet we need to make sure they're in sync. This is known as <em>cross-device</em> read-after-write consistency. This is more complicated for reasons like the fact that:</p>
<ul>
<li>We can't use the last update time as suggested earlier, since the code on one device will not know about what updates have happened on the other device.</li>
<li>If replicas are distributed across different datacenters, each device might hit different data datacenters which will have followers that may or may not have received the write. A solution to this is to force that all the reads from a user must be routed to the leader. This will of course introduce the complexity of routing all requests from all of a user's devices to the same datacenter.</li>
</ul>
<p><strong>b) Monotonic Reads:</strong> An anomaly that can occur when reading from asynchronous followers is that it's possible for a user to see things <em>moving backward in time.</em> Imagine a scenario where a user makes the same read multiple times, and each read request goes to a different follower. It's possible that a write has appeared on some followers, and not on others. Time might seem to go backwards sometimes when the user sees old data, after having read newer data.</p>
<p>Monotonic reads is a consistency level that guarantees that a user will not read older data after having previously read newer data. This guarantee is stronger than eventual consistency, but weaker than strong consistency.</p>
<p>A solution to this is that every read from a user should go to the same replica. The hash of a user's id could be used to determine what replica to go to.</p>
<p><strong>c) Consistent Prefix Reads:</strong> Another anomaly that can occur as a result of replication lag is a violation of causality. Meaning that a sequence of writes that occur in one order might be read in another order. This can especially happen in distributed databases where different partitions operate independently and there's no global ordering of writes. <em>Consistent prefix reads</em> is a guarantee that prevents this kind of problem.</p>
<p>One solution is to ensure that causally related writes are always written to the same partition, but this cannot always be done efficiently.</p>
<h4 id="solutions-for-replication-lag">Solutions for Replication Lag <a class="direct-link" href="#solutions-for-replication-lag" aria-hidden="true">#</a></h4>
<p>Application developers should ideally not have to worry about subtle replication issues and should trust that their databases "do the right thing". This is why <em>transactions</em> exist. They allow databases to provide stronger guarantees about things like consistency. However, many distributed databases have abandoned transactions because of the complexity, and have asserted that eventual consistency is inevitable. Martin discusses these claims later in the chapter.</p>
<h3 id="multi-leader-replication">Multi-Leader Replication <a class="direct-link" href="#multi-leader-replication" aria-hidden="true">#</a></h3>
<p>The downside of single-leader replication is that all writes must go through that leader. If the leader is down, or a connection can't be made for whatever reason, you can't write to the database.</p>
<p>Multi-leader/Master-master/Active-Active replication allows more than one node to accept writes. Each leader accepts writes from a client, and acts as a follower by accepting the writes on other leaders.</p>
<h4 id="use-cases-for-multi-leader-replication">Use Cases for Multi-Leader Replication <a class="direct-link" href="#use-cases-for-multi-leader-replication" aria-hidden="true">#</a></h4>
<ul>
<li>
<p><strong>Multi-datacenter operation:</strong> Here, each datacenter can have its own leader. This has a better performance for writes, since every write can be processed in its local datacenter (as opposed to being transmitted to a remote datacenter) and replicated asynchronously to other datacenters. It also means that if a datacenter is down, each data center can continue operating independently of the others.</p>
<p>Multi-leader replication has the disadvantage that the same data may be concurrently modified in two different datacenters, and so there needs to be a way to handle conflicts.</p>
</li>
<li>
<p><strong>Clients with offline operation:</strong> Some applications need to work even when offline. Say mobile apps for example, apps like Google Calendar need to accept writes even when the user is not connected to the internet. These writes are then asynchronously replicated to other nodes when the user is connected again. In this setup, each device stores data in its local database. Meaning that each device essentially acts like a leader. CouchDB is designed for this mode of operation apparently.</p>
</li>
<li>
<p><strong>Collaborative Editing:</strong> Real-time collaborative editing applications like Confluence and Google Docs allow several people edit a document at the same time. This is also a database replication problem. Each user that edits a document has their changes saved to a local replica, from which it is then replicated asynchronously.</p>
<p>For faster collaboration, the unit of change can be a single keystroke. That is, after a keystroke is saved, it should be replicated.</p>
</li>
</ul>
<h4 id="handling-write-conflicts.">Handling Write Conflicts. <a class="direct-link" href="#handling-write-conflicts." aria-hidden="true">#</a></h4>
<p>Multi-leader replication has the big disadvantage that write conflicts can occur, which requires conflict resolution.</p>
<p>If two users change the same record, the writes may be successfully applied to their local leader. However, when the writes are asynchronously replicated, a conflict will be detected. This does not happen in a single-leader database.</p>
<h5 id="synchronous-versus-asynchronous-conflict-detection">Synchronous versus asynchronous conflict detection <a class="direct-link" href="#synchronous-versus-asynchronous-conflict-detection" aria-hidden="true">#</a></h5>
<p>In theory, we could make conflict detection synchronous, meaning that we wait for the write to be replicated to all replicas before telling the user that the write was successful. Doing this will make one lose the main advantage of multi-leader replication though, which is allowing each replica to accept writes independently. Use single-leader replication if you want synchronous conflict detection.</p>
<h4 id="conflict-avoidance">Conflict Avoidance <a class="direct-link" href="#conflict-avoidance" aria-hidden="true">#</a></h4>
<p>Conflict avoidance is the simplest strategy for dealing with conflicts. Conflicts can be avoided by ensuring that all the writes for a particular record go through the same leader. For example, you can make all the writes for a user go to the same datacenter, and use the leader there for reading and writing. This of course has a downside that if a datacenter fails, traffic needs to be rerouted to another datacenter, and there's a possibility of concurrent writes on different leaders, which could break down conflict avoidance.</p>
<h4 id="converging-toward-a-consistent-state">Converging toward a consistent state <a class="direct-link" href="#converging-toward-a-consistent-state" aria-hidden="true">#</a></h4>
<p>A database must resolve conflicts in a convergent way, meaning that all the replicas must arrive at the same final value when all changes have been replicated.</p>
<p>Various ways of achieving this are by:</p>
<ul>
<li>Giving each write a unique ID ( e.g. a timestamp, UUID etc.), pick the write with the highest ID as the winner, and throw away the other writes. If timestamp is used, that's <em>Last write wins</em>, it's popular, but also prone to data loss.</li>
<li>Giving each replica a unique ID, and letting writes from the higher-number replica always take precedence over writes from a lower-number replica. This is also prone to data loss.</li>
<li>Recording the conflict in an explicit data structure that preserves the information, and writing application code that resolves the conflict at some later time (e.g. by prompting the user).</li>
<li>Merge the values together e.g. ordering them alphabetically.</li>
</ul>
<h4 id="custom-conflict-resolution-logic">Custom Conflict Resolution Logic <a class="direct-link" href="#custom-conflict-resolution-logic" aria-hidden="true">#</a></h4>
<p>The most appropriate conflict resolution method may depend on the application, and thus, multi-leader replication tools often let users write conflict resolution logic using application code. The code may be executed on read or on write:</p>
<ul>
<li><em>On write:</em> When the database detects a conflict in the log of replicated changes, it calls the conflict handler. The handler typically runs in a background process and must execute quickly. It has no user interaction.</li>
<li><em>On Read:</em> Conflicting writes are stored. However, when the data is read, the multiple versions of the data are returned to the user, either for the user to resolve them or for automatic resolution.</li>
</ul>
<p>Automatic conflict resolution is a difficult problem, but there are some research ideas being used today:</p>
<ul>
<li>Conflict-free replicated datatypes (CRDTs) - Used in Riak 2.0</li>
<li>Mergeable persistent data structure - Similar to Git. Tracks history explicitly</li>
<li>Operational transformation: Algorithm behind Google Docs.</li>
</ul>
<p>It's still an open area of research though.</p>
<h4 id="multi-leader-replication-topologies">Multi-Leader Replication Topologies <a class="direct-link" href="#multi-leader-replication-topologies" aria-hidden="true">#</a></h4>
<p>A <em>replication topology</em> is the path through which writes are propagated from one node to another. The most general topology is <em>all-to-all,</em> where each leader sends its writes to every other leader. Other types are <em>circular topology</em> and <em>star topology.</em></p>
<p>All-to-all topology is more fault tolerant than the circular and star topologies because in those topologies, one node failing can interrupt the flow of replication messages across other nodes, making them unable to communicate until the node is fixed.</p>
<h3 id="leaderless-replication">Leaderless Replication <a class="direct-link" href="#leaderless-replication" aria-hidden="true">#</a></h3>
<p>In this replication style, the concept of a leader is abandoned, and any replica can typically accept writes from clients directly.</p>
<p>This style is used by Amazon for its in-house <em>Dynamo</em> system. Riak, Cassandra and Voldermort also use this model. These are called <em>Dynamo</em> style systems.</p>
<p>In some leaderless implementations, the client writes directly to several replicas, while in others there's a coordinator node that does this on behalf of the client. Unlike a leader database though, this coordinator does not enforce any ordering of the writes.</p>
<h4 id="preventing-stale-reads">Preventing Stale Reads <a class="direct-link" href="#preventing-stale-reads" aria-hidden="true">#</a></h4>
<p>Say there are 3 replicas and one of the replicas goes down. A client could write to the system and have 2 of the replicas successfully acknowledge the write. However, when the offline node gets back up, anyone who reads from it may get stale responses.</p>
<p>To prevent stale reads, as well as writing to multiple replicas, the client may also read from multiple replicas in parallel. Version numbers are attached to the result to determine which value is newer.</p>
<h4 id="read-repair-and-anti-entropy">Read repair and anti-entropy <a class="direct-link" href="#read-repair-and-anti-entropy" aria-hidden="true">#</a></h4>
<p>When offline nodes come back up, the replication system must ensure that all data is eventually copied to every replica. Two mechanisms used in Dynamo-style datastores are:</p>
<ul>
<li><em>Read repair</em>: When data is read from multiple replicas and the system detects that one of the replicas has a lower version number, the data could be copied to it immediately. This works for frequently read values, but has the downside that any data that is not frequently read may be missing from some replicas and thus have reduced durability.
<blockquote></blockquote>
</li>
<li><em>Anti-entropy process</em>: In addition to the above, some databases have a background process that looks for differences in data between replicas and copies any missing data from one replica to another. This process does not copy writes in any particular order, and there may be a notable delay before data is copied.</li>
</ul>
<h4 id="quorums-for-reading-and-writing">Quorums for reading and writing <a class="direct-link" href="#quorums-for-reading-and-writing" aria-hidden="true">#</a></h4>
<p>Quorum reads and writes refer to the minimum number of votes for a read or a write to be valid. If there are <em>n</em> replicas, every write must be confirmed by at least <em>w</em> nodes to be considered successful, and every read must be confirmed by at least <em>r</em> nodes to be successful. The general rule that the number chosen for <em>r</em> and <em>w</em> should obey is that:</p>
<p><em>w + r &gt; n.</em></p>
<p>This way, we can typically expect an up-to-date value when reading because at least one of the <em>r</em> nodes we're reading from must overlap with the <em>w</em> nodes (barring sloppy quorums which I discussed below).</p>
<p>The parameters <em>n, w,</em> and <em>r</em> are typically configurable. A common choice is to make n an odd number such that w = r = (n + 1)/2. These numbers can be varied though. For a workload with few writes and many reads, it may make sense to set w = n and r = 1. Of course this has the disadvantage of reduced availability for writes if just one node fails.</p>
<p>Note that <em>n</em> does not always refer to the number of nodes in the cluster, it may just be the number of nodes that any given value must be stored on. This allows datasets to be partitioned. I cover Partitioning in the <a href="/posts/ddia/part-two/chapter-6/">next post</a>.</p>
<p>Note:</p>
<ul>
<li>With <em>w</em> and <em>r</em> being less than <em>n,</em> we can still process writes if a node is unavailable.</li>
<li>Reads and writes are always sent to all n replicas in parallel, <em>w</em> and <em>r</em> determine how many nodes we wait for i.e., how many nodes need to report success before we consider the read or write to be successful.</li>
</ul>
<h4 id="limitations-of-quorum-consistency">Limitations of Quorum Consistency <a class="direct-link" href="#limitations-of-quorum-consistency" aria-hidden="true">#</a></h4>
<p>Quorums don't necessarily have to be majorities i.e. <em>w + r &gt; n.</em> What matters is that the sets of nodes used by the read and write operations overlap in at least one node.</p>
<p>We could also set <em>w</em> and <em>r</em> to smaller numbers, so that <em>w + r ≤ n.</em> With this, reads and writes are still sent to n nodes, but a smaller number of successful responses is required for the operation to succeed. However, you are also more likely to read stale values, as it's more likely that a read did not include the node with the latest value.</p>
<p>The upside of the approach though is that it allows lower latency and higher availability: if there's a network interruption and many replicas become unreachable, there's a higher chance that reads and writes can still be processed.</p>
<p>Even if we configure our database such that <em>w + r &gt; n</em> , there are still edge cases where stale values may be returned. Possible scenarios are:</p>
<ul>
<li>If a sloppy quorum is used, the nodes for reading and writing may not overlap. Sloppy quorums are discussed further down.</li>
</ul>
<blockquote></blockquote>
<ul>
<li>If two writes occur concurrently, it's still not clear which happened first. Therefore, the database may wrongly return the more stale one. If we pick a winner based on a timestamp (last write wins), writes can be lost due to clock skew.
<blockquote></blockquote>
</li>
<li>If a write happens concurrently with a read, the write may be reflected on only some of the replicas. It's unclear whether the read will return the old or new value.
<blockquote></blockquote>
</li>
<li>In a non-transaction model, if a write succeeds on some replicas but fails on others, it is not rolled back on the replicas where it succeeded.</li>
</ul>
<p>From these points and others not listed, there is no absolute guarantee that quorum reads return the latest written value. These style of databases are optimized for use cases that can tolerate eventual consistency. Stronger guarantees require transactions or consensus.</p>
<h5 id="monitoring-staleness">Monitoring Staleness <a class="direct-link" href="#monitoring-staleness" aria-hidden="true">#</a></h5>
<p>It's important to monitor whether databases are returning up-to-date results, even if the application can tolerate stale reads. If a replica falls behind significantly, the database should alert you so that you can investigate the cause.</p>
<p>For leader-based replication, databases expose metrics for the replication lag. It's possible to do this because writes are applied to the leader and followers in the same order. We can determine how far behind a follower has fallen from a leader by subtracting it's position from the leader's current position.</p>
<p>This is more difficult in leaderless replication systems as there is no fixed order in which writes are applied. There's some research into this, but it's not common practice yet.</p>
<h4 id="sloppy-quorums-and-hinted-handoff">Sloppy Quorums and Hinted Handoff <a class="direct-link" href="#sloppy-quorums-and-hinted-handoff" aria-hidden="true">#</a></h4>
<p>Databases with leaderless replication are appealing for use cases where high availability and low latency is required, as well as the ability to tolerate occasional stale reads. This is because they can tolerate failure of individual nodes without needing to failover since they're not relying on one node. They can also tolerate individual nodes going slow, as long as <em>w</em> or <em>r</em> nodes have responded.</p>
<p>Note that the quorums described so far are not as fault tolerant as they can be. If any of the designated <em>n</em> nodes is unavailable for whatever reason, it's less likely that you'll be able to have <em>w</em> or <em>r</em> nodes reachable, making the system unavailable. Nodes being unavailable can be caused by anything, even something as simple as a network interruption.</p>
<p>To make the system more fault tolerant, instead of returning errors to all requests for which can't reach a quorum of <em>w</em> or <em>r</em> nodes, the system could accept reads and writes on nodes that are reachable, even if they are not among the designated <em>n</em> nodes on which the value usually lives. This concept is known as a <em>sloppy quorum.</em></p>
<p>With a sloppy quorum, during network interruptions, reads and writes still require <em>r</em> and <em>w</em> successful responses, but they do not have to be among the designated <em>n</em> "home" nodes for a value. These are like temporary homes for the value.</p>
<p>When the network interruption is fixed, the writes that were temporarily accepted on behalf of another node are sent to the appropriate "home" node. This is <em>hinted handoff.</em></p>
<p>Sloppy quorums are particularly useful for increasing write availability. However, it also means that even when <em>w + r &gt; n,</em> there is a possibility of reading stale data, as the latest value may have been temporarily written to some values outside of <em>n.</em></p>
<p>Sloppy quorum is more of an assurance of durability, than an actual quorum.</p>
<h5 id="multi-datacenter-operation">Multi-datacenter operation <a class="direct-link" href="#multi-datacenter-operation" aria-hidden="true">#</a></h5>
<p>For datastores like Cassandra and Voldermort which implement leaderless replication across multiple datacenters, the number of replicas <em>n</em> includes replicas in all datacenter.</p>
<p>Each write is also sent to all datacenters, but it only waits for acknowledgement from a quorum of nodes within its local datacenter so that it's not affected by delays and interruptions on the link between multiple datacenters.</p>
<h4 id="detecting-concurrent-writes">Detecting Concurrent Writes <a class="direct-link" href="#detecting-concurrent-writes" aria-hidden="true">#</a></h4>
<p>In dynamo-style databases, several clients can concurrently write to the same key. When this happens, we have a conflict. We've briefly touched on conflict resolution techniques already, but we'll discuss them in more detail.</p>
<h5 id="last-write-wins-(discarding-concurrent-writes)">Last write wins (discarding concurrent writes) <a class="direct-link" href="#last-write-wins-(discarding-concurrent-writes)" aria-hidden="true">#</a></h5>
<p>One approach for conflict resolution is the last write wins approach. It involves forcing an arbitrary ordering on concurrent writes (could be by using timestamps), picking the most "recent" value, and discarding writes with an earlier timestamp.</p>
<p>This helps to achieve the goal of eventual convergence across the data in replicas, at the cost of durability. If there were several concurrent writes the same key, only one of the writes will survive and the others will be discarded, even if all the writes were reported as successful.</p>
<p>Last write wins (LWW) is the only conflict resolution method supported by Apache Cassandra.</p>
<p>If losing data is not acceptable, LWW is not a good choice for conflict resolution.</p>
<h5 id="the-%22happens-before%22-relationship-and-concurrency">The "happens-before" relationship and concurrency <a class="direct-link" href="#the-%22happens-before%22-relationship-and-concurrency" aria-hidden="true">#</a></h5>
<p>Whenever we have two operations A and B, there are three possibilities:</p>
<ul>
<li>Either A happened before B</li>
<li>Or B happened before A</li>
<li>Or A and B are concurrent.</li>
</ul>
<p>We say that an operation A happened before operation B if either of the following applies:</p>
<ul>
<li>B knows about A</li>
<li>B depends on A</li>
<li>B builds upon A</li>
</ul>
<p>Thus, if we cannot capture this relationship between A and B, we say that they are concurrent. If they are concurrent, we have a conflict that needs to be resolved.</p>
<p><strong>Note:</strong> Exact time does not matter for defining concurrency, two operations are concurrent if they are both unaware of each other, regardless of the physical time which they occurred. Two operations can happen sometime apart and still be concurrent, as long as they are unaware of each other.</p>
<h5 id="capturing-the-happens-before-relationship">Capturing the happens-before relationship <a class="direct-link" href="#capturing-the-happens-before-relationship" aria-hidden="true">#</a></h5>
<p>In a single database replica, version numbers are used to determine concurrency.</p>
<p>It works like this:</p>
<ul>
<li>Each key is assigned a version number, and that version number is <em>incremented</em> every time that key is written, and the database stores the version number along with the value written. That version number is returned to a client.</li>
<li>A client must read a key before writing. <em>When it reads a key, the server returns the latest version number together with the values that have not been overwritten.</em></li>
<li>When a client wants to write a new value, it returns the last version number it received in the prior step alongside the write.</li>
<li>If the version number being passed with a write is higher than the version number of other values in the db, it means the new write is aware of those values at the time of the write (since it was returned from the prior read), and can overwrite all values with that version number or below.</li>
<li>If there are higher version numbers, the database must keep all values with the higher version number (because those values are concurrent with the incoming write- it did not know about them).</li>
</ul>
<p>Example scenario:</p>
<p>If two clients are trying to write a value for the same key at the same time, both would first read the data for that key and get the latest version number of say: 3. If one of them writes first, the version number will be updated to 4 from the database end. However, since the slower one will pass a version number of 3, it means it is concurrent with the other one since it's not aware of the higher version number of 4.</p>
<p>When a write includes the version number from a prior read, that tells us which previous state the write is based on.</p>
<h5 id="merging-concurrently-written-values">Merging Concurrently Written Values <a class="direct-link" href="#merging-concurrently-written-values" aria-hidden="true">#</a></h5>
<p>With the algorithm described above, clients have to do the work of merging concurrently written values. Riak calls these values <em>siblings.</em></p>
<p>A simple merging approach is to take a union of the values. However, this can be faulty if one operation deleted a value but that value is still present in a sibling. To prevent this problem, the system must leave a marker <em>(tombstone)</em> to indicate that an item has been removed when merging siblings.</p>
<p>CRDTs are data structures that can automatically merge siblings in sensible ways, including preserving deletions.</p>
<h5 id="version-vectors">Version Vectors <a class="direct-link" href="#version-vectors" aria-hidden="true">#</a></h5>
<p>The algorithm described above used only a single replica. When we have multiple replicas, we use a version number <em>per replica</em> <strong>and</strong> <em>per key</em> and follow the same algorithm. Note that each replica also keeps track of the version numbers seen from each of the other replicas. With this information, we know which values to overwrite and which values to keep as siblings.</p>
<p>The collection of version numbers from all the replicas is called a <em>version vector.</em> Dotted version vectors are a nice variant of this used in Riak: <a href="https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/" title="https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/">https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/</a></p>
<p>Version vectors are also sent to clients when values are read, and need to be sent back to the database when a value is written.</p>
<p><em>Version vectors enable us to distinguish between overwrites and concurrent writes.</em></p>
<p>We also have Vector clocks, which are different from Version Vectors apparently: <a href="https://haslab.wordpress.com/2011/07/08/version-vectors-are-not-vector-clocks/" title="https://haslab.wordpress.com/2011/07/08/version-vectors-are-not-vector-clocks/">https://haslab.wordpress.com/2011/07/08/version-vectors-are-not-vector-clocks/</a></p>


<p> 
      <a href="/tags/distributed-systems/" class="tag">distributed-systems</a>
      <a href="/tags/learning-diary/" class="tag">learning-diary</a>
      <a href="/tags/ddia/" class="tag">ddia</a> </p>








<p><a href="/">← Home</a></p>

    </main>

    <footer></footer>

    <!-- Current page: /posts/ddia/part-two/chapter-5/ -->
  

</body></html>


