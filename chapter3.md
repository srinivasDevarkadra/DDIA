[Origina author website and All Credits](https://timilearning.com/posts/ddia/part-one/chapter-3/)
<html lang="en">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta name="Description" content="My notes from the third chapter of Martin Kleppmann&#39;s book: Designing Data Intensive Applications.">
        <link rel="stylesheet" href="/css/index.css">
        <link rel="stylesheet" href="/css/prism-material-oceanic.css">
        <link rel="alternate" href="/feed/feed.xml" type="application/atom+xml" title="Timilearning - A blog by Timi Adeniran">
    </head>
    <body>
        <main class="tmpl-post">
            <!-- capture the JS content as a Nunjucks variable -->
            <!-- feed it through our jsmin filter to minify -->
            <h1>Chapter 3 - Storage and Retrieval</h1>
            <i>
                <time class="postlist-date" datetime="2019-12-07">07 Dec 2019</time>
            </i>
            &middot;<span class="postlist-reading-time">18 min read  </span>
            <p>These are my notes from the third chapter of Martin Kleppmann's Designing Data Intensive Applications.</p>
            <p>
                <strong>Table of Contents</strong>
            </p>
            <ul>
                <li>
                    <a href="#storage-engines">Storage Engines</a>
                </li>
                <li>
                    <a href="#log-structured-storage-engines">Log-Structured Storage Engines</a>
                    <ul>
                        <li>
                            <a href="#indexing">Indexing</a>
                            <ul>
                                <li>
                                    <a href="#hash-index">Hash Index</a>
                                </li>
                                <li>
                                    <a href="#sstables-and-lsm-trees">SSTables and LSM-Trees</a>
                                    <ul>
                                        <li>
                                            <a href="#constructing-and-maintaining-sstables">Constructing and maintaining SSTables</a>
                                        </li>
                                        <li>
                                            <a href="#making-an-lsm-tree-out-of-sstables">Making an LSM-tree out of SSTables</a>
                                        </li>
                                        <li>
                                            <a href="#performance-optimizations">Performance Optimizations</a>
                                        </li>
                                    </ul>
                                </li>
                                <li>
                                    <a href="#b-trees">B-Trees</a>
                                    <ul>
                                        <li>
                                            <a href="#making-b--trees-reliable">Making B- Trees reliable</a>
                                        </li>
                                        <li>
                                            <a href="#b-tree-optimizations">B-tree optimizations</a>
                                        </li>
                                        <li>
                                            <a href="#comparing-b-trees-and-lsm-trees">Comparing B-Trees and LSM-Trees</a>
                                        </li>
                                        <li>
                                            <a href="#advantages-of-lsm-trees">Advantages of LSM Trees</a>
                                        </li>
                                        <li>
                                            <a href="#downsides-of-lsm-trees">Downsides of LSM Trees</a>
                                        </li>
                                    </ul>
                                </li>
                                <li>
                                    <a href="#other-indexing-structures">Other Indexing Structures</a>
                                    <ul>
                                        <li>
                                            <a href="#storing-values-within-the-index">Storing values within the index</a>
                                        </li>
                                        <li>
                                            <a href="#approach-2---heap-file">Approach 2 - Heap file</a>
                                        </li>
                                        <li>
                                            <a href="#approach-1---actual-row">Approach 1 - Actual row</a>
                                        </li>
                                        <li>
                                            <a href="#covering-index">Covering Index</a>
                                        </li>
                                        <li>
                                            <a href="#multi-column-indexes">Multi-column indexes</a>
                                        </li>
                                        <li>
                                            <a href="#full-text-search-and-fuzzy-indexes">Full-text search and fuzzy indexes</a>
                                        </li>
                                        <li>
                                            <a href="#keep-everything-in-memory">Keep everything in memory</a>
                                        </li>
                                    </ul>
                                </li>
                                <li>
                                    <a href="#transaction-processing-vs-analytics">Transaction Processing vs Analytics</a>
                                    <ul>
                                        <li>
                                            <a href="#data-warehousing">Data Warehousing</a>
                                        </li>
                                        <li>
                                            <a href="#stars-and-snowflakes-schemas-for-analytics">Stars and Snowflakes: Schemas for Analytics</a>
                                        </li>
                                    </ul>
                                </li>
                                <li>
                                    <a href="#column-oriented-storage">Column Oriented Storage</a>
                                    <ul>
                                        <li>
                                            <a href="#column-compression">Column Compression</a>
                                        </li>
                                        <li>
                                            <a href="#column-oriented-storage-and-column-families">Column-oriented storage and column families</a>
                                        </li>
                                        <li>
                                            <a href="#sort-order-in-column-storage">Sort Order in Column Storage</a>
                                        </li>
                                    </ul>
                                </li>
                            </ul>
                        </li>
                    </ul>
                </li>
                <li>
                    <a href="#further-reading">Further reading:</a>
                </li>
            </ul>
            <hr>
            <p>This chapter is about how databases work under the hood.</p>
            <p>There's a difference between storage engines that are optimized for transactional workloads and those that are optimized for analytics.</p>
            <h1 id="storage-engines">
                Storage Engines <a class="direct-link" href="#storage-engines" aria-hidden="true">#</a>
            </h1>
            <p>There are two families of storage engines: log-structured storage engines (log structured merge trees), and page-oriented storage engines (b-trees). A storage engine’s job is to write things to disk on a single node.</p>
            <h1 id="log-structured-storage-engines">
                Log-Structured Storage Engines <a class="direct-link" href="#log-structured-storage-engines" aria-hidden="true">#</a>
            </h1>
            <p>
                Many databases internally use a <em>log,</em>
                an append-only data file for adding something to it. Each line in the log contains a key-value pair, separated by a comma (similar to a CSV file, ignoring escaping issues). The log does not have to be internally-readable, it might be binary and intended only for other programs to read.
            </p>
            <h2 id="indexing">
                Indexing <a class="direct-link" href="#indexing" aria-hidden="true">#</a>
            </h2>
            <p>An index is an additional structure derived from the primary data. Any kind of index usually slows down writes, since the index has to be updated every time data is written.</p>
            <p>
                <em>Well-chosen indexes speed up read queries, but every index slows down writes.</em>
            </p>
            <h3 id="hash-index">
                Hash Index <a class="direct-link" href="#hash-index" aria-hidden="true">#</a>
            </h3>
            <p>These are indexes for key-value data. For a data storage that consists of only appending to a file, a simple indexing strategy is to keep an in-memory hash map where the value for every key is a byte offset, which indicates where the key is located in the file.</p>
            <p>Bitcask (the default storage engine in Riak - Riak is a distributed datastore similar to Cassandra) uses the approach above. The only requirement is that all the keys fit in the available RAM as the hash map is kept completely in memory. The values don't have to fit in memory since they can be loaded from disk with a simple disk seek. Something like Bitcask is suitable for situations where the value for a key is updated frequently.</p>
            <p>The obvious challenge in appending to a file is that the file can grow too large and then we run out of disk space. A solution to this is to break the log into segments of a certain size. A segment file is closed when it reaches that size, and subsequent writes are made to a new segment.</p>
            <p>
                We can then perform <em>compaction</em>
                on these segments. Compaction means keeping the most recent update for each key and throwing away duplicate keys. Compaction often makes segments smaller (relies on the assumption that a key is overwritten several times on average within one segment), and so we can <em>merge</em>
                several segments together at the same time as performing the compaction.
            </p>
            <p>Basically, we compact and merge segment files together. The merged segment is written to a new file. This can happen as a background process, so the old segment files can still serve read and write requests until the merging process is complete.</p>
            <p>
                <strong>Each segment will have its own in-memory hash table.</strong>
                To find a value for a key, we'll check the most recent segment. If it's not there, we'll check the second-most-recent segment, and so on.
            </p>
            <p>There are certain practical issues that must be considered in a real life implementation of this hash index in a log structure. Some of them are:</p>
            <ul>
                <li>
                    <em>File Format</em>
                    : As opposed to using a CSV format, it's faster and simpler to use a binary format that first encodes the length of a string in bytes, followed by the raw string.
                </li>
                <li>
                    <em>Deleting Records</em>
                    : To delete a key and its value, it's not practical to search for all the occurrences of that key in the segments. What happens is that a special deletion record is appended to the data file (sometimes called a <em>tombstone).</em>
                    When log segments are merged, the tombstone tells the merging process to discard any previous values for the deleted key.
                </li>
                <li>
                    <em>Crash Recovery</em>
                    : If the database is restarted, the in-memory hash maps will be lost. In principle, the segment's hash maps can be restored by reading the entire segment files and constructing the hash maps from scratch. This might take a while though, so could make server restarts painful. <strong>Bitcask's</strong>
                    approach to recovery is by storing a snapshot of each segment's hash map on disk, which can be loaded into memory more quickly
                </li>
                <li>
                    <em>Concurrency Control:</em>
                    Since writes are appended in a sequential order, a common implementation is to have only one writer thread. Data files are append-only and otherwise immutable, so they can be read concurrently by multiple threads.
                </li>
            </ul>
            <p>There are good reasons why an append-only log is a good choice, as opposed to a storage where files are updated in place with the new value overwriting the old one. Some of those reasons are:</p>
            <ul>
                <li>Appending and segment merging are sequential write operations, which are generally faster than random writes, especially on magnetic spinning-disk hard drives.</li>
                <li>Concurrency and crash recovery are simpler if segment files are append-only or immutable. For crash recovery, you don't need to worry if a crash happened while a value was being overwritten, leaving you with partial data.</li>
                <li>Merging old segments avoids the problem of data files getting fragmented over time. Fragmentation occurs on a hard drive, a memory module, or other media when data is not written closely enough physically on the drive. Those fragmented, individual pieces of data are referred to generally as fragments.</li>
            </ul>
            <p>
                <em>Basically, when data files are far from each other, it's a form of fragmentation.</em>
            </p>
            <p>There are limitations to the hash table index though, some of them are:</p>
            <ul>
                <li>The hash table must fit in memory, so it's not efficient if there are a large number of keys. An option is to maintain a map on disk, but it doesn't perform so well. It requires a lot of random access I/O, is expensive to grow when it becomes full, and hash collisions require fiddly logic.</li>
                <li>Range queries are not efficient. You have to look up each key individually in the map.</li>
            </ul>
            <p>So, in this approach, writes are made to the segments on a disk while the hash table index being stored is in-memory.</p>
            <h3 id="sstables-and-lsm-trees">
                SSTables and LSM-Trees <a class="direct-link" href="#sstables-and-lsm-trees" aria-hidden="true">#</a>
            </h3>
            <p>In log segments with hash indexes, each key-value pair appears in the order that it was written, and values later in the log take precedence over values for the same key earlier in the log. Apart from that, the order of key-value pairs in the file is irrelevant.</p>
            <p>
                There is a change to this approach in a <em>Sorted String Table</em>
                format, or SSTable for short. Here, it is required that the sequence of key-value pairs is sorted by key. Hence, new key-value pairs cannot be appended to the segment immediately. There are several advantages to this over log segments with hash indexes:
            </p>
            <ul>
                <li>Merging segments is simple and efficient. The approach here is similar to the mergesort algorithm. The same principle as the log with hash indexes applies if a key is duplicated across several segments. We keep the most recent one and discard the others.</li>
                <li>You don't need to keep an index of all the keys in memory. Because the file is sorted, if you're looking for the offset of a particular key, it won't be difficult to find that offset once you can determine the offset of keys that are smaller and larger than it in the ordering.</li>
            </ul>
            <p>You still need an in-memory index to tell you the offsets of some keys, but it can be sparse.</p>
            <ul>
                <li>Read requests often need to scan over several key-value pairs, therefore it is possible to group the records into a block and compress it before writing it to disk. Each entry of the sparse index then points to the start of a compressed block. This has the advantage of saving disk space and reducing the IO bandwidth.</li>
            </ul>
            <p>SSTables store their keys in blocks, and have an internal index, so even though a single SSTable may be very large (gigabytes in size), only the index and the relevant block needs to be loaded into memory.</p>
            <h4 id="constructing-and-maintaining-sstables">
                Constructing and maintaining SSTables <a class="direct-link" href="#constructing-and-maintaining-sstables" aria-hidden="true">#</a>
            </h4>
            <p>It's possible to maintain a sorted structure on disk( see B-Trees) but maintaining it in memory is easier and is the approach described here. The approach is to use well-known tree data structures such as red-black trees or AVL trees into which keys can be inserted in any order and read back in sorted order.</p>
            <p>So the storage engine works as follows:</p>
            <ul>
                <li>
                    When a write comes in, it is written to an in-memory balanced tree data structure. The in-memory tree is sometimes called a <em>memtable.</em>
                </li>
                <li>When the memtable exceeds a threshold, write it out to disk as an SSTable file. This operation is efficient because the tree already maintains the key-value pairs sorted by key. The new SSTable file then becomes the most recent segment of the database. While the SSTable is being written out to disk, writes can continue to a new memtable instance.</li>
                <li>To serve a read request, first check for the key in the memtable. If it's not there, check the most recent segment, then the next-older segment etc</li>
                <li>From time to time, run a merging and compaction process in the background to combine segment files and to discard overwritten or deleted values.</li>
            </ul>
            <p>An obvious problem with this approach is that if the database crashes, the most recent writes (which are in the memtable but not yet written to disk) will disappear. To avoid that problem, one approach is to keep a separate log on disk to which every write is immediately appended. This separate log is not in sorted order, but that's irrelevant because the content can easily be sorted in a memtable. The corresponding log can be discarded every time the memtable is written out to an SSTable.</p>
            <h4 id="making-an-lsm-tree-out-of-sstables">
                Making an LSM-tree out of SSTables <a class="direct-link" href="#making-an-lsm-tree-out-of-sstables" aria-hidden="true">#</a>
            </h4>
            <p>The algorithm described above is used in LevelDB and RocksDB. Key-value storage engine libraries are designed to be embedded into other applications. Among other things, LevelDB can be used in Riak as an alternative to Bitcask as its storage engine.</p>
            <p>
                This indexing structure was originally described under the name <em>Log-Structured Merge-Tree.</em>
            </p>
            <p>
                Lucene, which is an indexing engine for full-text search uses a similar method for storing its <em>term dictionary.</em>
                A full-text index is more complex than a key-value index but is based on a similar idea: given a word in a search query, find all the documents that mention the word. It's usually implemented with a key-value structure where the key is a word ( a <em>term</em>
                ) and the value is a list of IDs of all the documents that contain the word (the <em>postings list).</em>
                In Lucene, the mapping from term to postings list is kept in SSTable-like sorted files that are merged in the background as needed.
            </p>
            <h4 id="performance-optimizations">
                Performance Optimizations <a class="direct-link" href="#performance-optimizations" aria-hidden="true">#</a>
            </h4>
            <p>
                The LSM-tree algorithm can be slow when looking up keys that do not exist in the database: you first have to check the memtable, then all the segments all the way up to the oldest (possibly having to read from disk for each one) to be certain that the key does not exist. In order to optimize this access, storage engines often make use of <em>Bloom filters.</em>
            </p>
            <p>
                A <em>Bloom filter</em>
                is a memory-efficient data structure for approximating the contents of a set. It can tell you if a key does not appear in a database, thus saving you from unnecessary disk reads for nonexistent keys.
            </p>
            <p>
                There are also strategies to determine the order and timing of how SSTables are compacted and merged. Two most common options are <em>size-tiered</em>
                and <em>leveled</em>
                compaction. LevelDB and RocksDB use leveled compaction, Hbase uses size-tiered and Cassandra supports both.
            </p>
            <p>
                <em>Size-Tiered Compaction</em>
                : Here, newer and smaller SSTables are successively merged into older and larger SSTables.
            </p>
            <p>
                <em>Leveled Compaction</em>
                <strong>:</strong>
                The key range is split into smaller SSTables and older data is moved into separate &quot;levels &quot;. This allows compaction to proceed more incrementally and use less disk space. The levels are structured roughly so that each level is in total 10x as large as the level above it. New keys arrive at the highest layer, and as that level gets larger and larger and hits a threshold, some SSTables at that level get compacted into fewer (but larger) SSTables one level lower.
            </p>
            <p>Within a single level, SSTables are non-overlapping: one SSTable might contain keys covering the range (a,b), the next (c,d), and so on. The key-space does overlap between levels: if you have two levels, the first might have two SSTables (covering the ranges above), but the second level might have a single SSTable over the key space (a,e). Looking for the key 'aardvark' may require looking in two SSTables: the (a,b) SSTable in Level 1, and the (a,e) SSTable in Level 2.</p>
            <p>
                <em>Basically, a level has many SSTables.</em>
            </p>
            <h3 id="b-trees">
                B-Trees <a class="direct-link" href="#b-trees" aria-hidden="true">#</a>
            </h3>
            <p>B-trees are a popular indexing structure. Like SSTables, they keep key-value pairs sorted by key, but the similarity ends there.</p>
            <p>
                Log-structured indexes break the database down into segments, however B-trees break the database down into fixed size <em>blocks</em>
                or <em>pages.</em>
                Each page can be identified with its address or location on disk, which allows one page to refer to another. Pages are usually small in size, typically 4kb compared to segments which can be several megabytes. Pages are stored on disk.
            </p>
            <p>One page is designated as the root of the B-tree; whenever you want to look up a key in the index, you start here. The page contains several keys and references to child pages. Each child is responsible for a continuous range of keys, and the keys between the references indicate where the boundaries between those ranges lie.</p>
            <p>
                <em>Branching factor:</em>
                The number of references to child pages in one page the B-tree.
            </p>
            <p>To update the value of an existing key in a B-tree, you search for the leaf page containing that key, change the value in that page, and write the page back to disk (any references to that page remain valid). To add a new key, find the page whose range encompasses the new key and add it to that page. If there's no free space on that page, split the page into two half-full pages, and update the parent page to account for the new subdivision of key ranges.</p>
            <h4 id="making-b--trees-reliable">
                Making B- Trees reliable <a class="direct-link" href="#making-b--trees-reliable" aria-hidden="true">#</a>
            </h4>
            <p>The main write operation of a B-tree is to overwrite a page on disk with new data. The assumption is that an overwrite does not change where a page is located i.e. all the references to a page typically remain intact when the page is overwritten. This differs from LSM trees where things are never updated in place, and are append only.</p>
            <p>Some operations require different pages to be overwritten e.g. when a page is split because an insertion caused it to be overfull. We'll need to write the two pages that were split and update the parent page with references to the two child pages. This operation is dangerous especially if the database crashes after only some pages have been written, this can lead to a corrupted index.</p>
            <p>
                A solution used to make databases resilient to crashes is to keep a 
                <strong>
                    <em>write-ahead log</em>
                </strong>
                on disk. It is an append-only file to which every B-tree modification must be written before it can be applied to the pages of the tree itself. It's used to restore the DB when it comes back from a crash.
            </p>
            <p>
                There are also concurrency issues associated with updating pages in place. If multiple threads access a B-tree at the same time, a thread may see the tree in an inconsistent state. The solution is usually implemented by protecting the tree's data structures with <em>latches</em>
                (lightweight locks). This is not an issue with log structured approaches since all the merging happens in the background without interfering with incoming queries.
            </p>
            <h4 id="b-tree-optimizations">
                B-tree optimizations <a class="direct-link" href="#b-tree-optimizations" aria-hidden="true">#</a>
            </h4>
            <p>Different optimizations have been made with B-trees:</p>
            <ul>
                <li>Additional pointers been added to the tree. E.g. a leaf page may have references to its sibling pages to the left and right, this allows scanning keys in order without jumping back to parent pages.</li>
                <li>Some databases use a copy-on-write scheme instead of overwriting pages and maintaining a WAL for crash recovery. What this means is that a modified page is written to a different location, and a new version of the parent pages in the tree is created, pointing at the new location.</li>
            </ul>
            <p>
                <em>So, page structured storage engines are organized into fixed-size pages. These pages are all part of a tree called b-tree.</em>
            </p>
            <p>
                SQLite, for example, has a btree for every table in the database, as well as a btree for every index in the database. For the indexes , the key stored on a page is the column value of the index, while the value is the row id where it can be found. For the table btree, the key is the rowid while I believe the value is all the data in that row: <a href="https://jvns.ca/blog/2014/10/02/how-does-sqlite-work-part-2-btrees/" title="https://jvns.ca/blog/2014/10/02/how-does-sqlite-work-part-2-btrees/">https://jvns.ca/blog/2014/10/02/how-does-sqlite-work-part-2-btrees/</a>
            </p>
            <p>
                <a href="https://hackernoon.com/fundamentals-of-system-design-part-3-8da61773a631" title="https://hackernoon.com/fundamentals-of-system-design-part-3-8da61773a631">https://hackernoon.com/fundamentals-of-system-design-part-3-8da61773a631</a>
            </p>
            <h4 id="comparing-b-trees-and-lsm-trees">
                Comparing B-Trees and LSM-Trees <a class="direct-link" href="#comparing-b-trees-and-lsm-trees" aria-hidden="true">#</a>
            </h4>
            <p>As a rule of thumb, LSM trees are typically faster for writes, while B-trees are thought to be faster for reads. Reads are slower on LSM-trees because they have to check different data structures and SSTables at different stages of compaction.</p>
            <h4 id="advantages-of-lsm-trees">
                Advantages of LSM Trees <a class="direct-link" href="#advantages-of-lsm-trees" aria-hidden="true">#</a>
            </h4>
            <ul>
                <li>
                    <p>A B-tree index must write every piece of data at least twice: once to the write-ahead log, and once to the page itself (and perhaps again as pages are split). There's also overhead from having to write an entire page at a time, even if only few bytes in the page change. Log-structured indexes also rewrite data multiple times due to the repeated compaction and merging of SSTables.</p>
                    <p>
                        <strong>
                            <em>Write amplification:</em>
                        </strong>
                        When one write to the database results in multiple writes to the disk over the course of the database's lifetime. This is of particular concern on SSDs, which can only overwrite blocks a limited number of times before wearing out.
                    </p>
                    <blockquote></blockquote>
                </li>
                <li>
                    <p>LSM-trees are typically able to sustain higher write throughput than B-trees partly because they sometimes have lower write amplification, and also because they sequentially write compact SSTable files rather than having to overwrite several pages in the tree. This is important on magnetic hard drives, where sequential writes are faster than random writes.</p>
                </li>
            </ul>
            <blockquote></blockquote>
            <ul>
                <li>LSM-trees can be compressed better, and this often produces smaller files on disk than B-trees. B-trees leave some disk space used due to fragmentation: when a row cannot fit into an existing page or a page is split, some space in a page remains unused(Basically, if the existing space on the page cannot fit a new row, the row will be moved to a page). Sending and receiving smaller files over IO is useful if you bandwidth is limited.</li>
            </ul>
            <p>On many SSDs, the firmware internally uses a log-structured algorithm to turn random writes into sequential writes on the underlying storage chips, so the impact of the storage engine's write pattern is less pronounced (point 2). Note that lower write amplification and reduced fragmentation is still advantageous on SSDs: representing data more compactly allows more read and write requests within the available I/O bandwidth.</p>
            <h4 id="downsides-of-lsm-trees">
                Downsides of LSM Trees <a class="direct-link" href="#downsides-of-lsm-trees" aria-hidden="true">#</a>
            </h4>
            <ul>
                <li>
                    <p>A downside of log-structured storage is that the compaction process happening can interfere with the performance of ongoing reads and writes. Storage engines typically try to perform compaction incrementally and without affecting concurrent access, but it can easily happen that a request needs to wait while the disk finishes an expensive compaction operation.</p>
                </li>
                <li>
                    <p>One other issue with compaction arises at high write throughput: The disk needs to share its finite write bandwidth between the initial write (logging and flushing a memtable to disk) and the compaction threads running in the background.</p>
                    <p>Compaction has to keep up with the rate of incoming writes, even at high write throughput.</p>
                </li>
                <li>
                    <p>A key can exist in multiple places in different segments with LSM trees. This differs from B-trees where a key can only exist in one place. This prospect makes B-trees more appealing for strong transactional semantics - e.g. with b-trees transaction isolation can be implemented by attaching locks on a range of keys to a tree.</p>
                </li>
            </ul>
            <h3 id="other-indexing-structures">
                Other Indexing Structures <a class="direct-link" href="#other-indexing-structures" aria-hidden="true">#</a>
            </h3>
            <p>We've mainly covered key-value indexes which are like a primary key index, but we can also have secondary indexes. You can typically create several secondary indexes on the same table in relational databases. A secondary index can be constructed from a key-value index.</p>
            <p>With secondary indexes, note that the indexed values are not necessarily unique. Several rows (documents, vertices) may exist under the same index entry. This can be expressed in two ways:</p>
            <ul>
                <li>Making each value in the index a list of matching row identifiers.</li>
                <li>Making each key unique by appending a row identifier to it.</li>
            </ul>
            <p>Both B-trees and log-structured indexes can be used as secondary indexes.</p>
            <h4 id="storing-values-within-the-index">
                Storing values within the index <a class="direct-link" href="#storing-values-within-the-index" aria-hidden="true">#</a>
            </h4>
            <p>The key in an index is the column value that queries search for, but the value can be either:</p>
            <ol>
                <li>The actual row (document, vertex) in question</li>
                <li>
                    A reference to the row stored elsewhere. The rows in this case are stored somewhere known as a 
                    <strong>
                        <em>heap file</em>
                    </strong>
                    , which stores data in no particular order (could be append-only, or may keep track of deleted rows in order to overwrite them with new data later). This approach is common because it avoids duplicating data in the presence of several secondary indexes. Each index just references a location in the heap file.
                </li>
            </ol>
            <h4 id="approach-2---heap-file">
                Approach 2 - Heap file <a class="direct-link" href="#approach-2---heap-file" aria-hidden="true">#</a>
            </h4>
            <p>
                The heap file approach can be efficient when updating a value without changing the key, provided the new value is not larger than the old value. If it is larger, the record might need to be moved to a new location in the heap where there is enough space. When this happens, either all indexes need to be updated to point to the new heap location of the record, or a <strong>forwarding pointer</strong>
                is left behind in the old heap location.
            </p>
            <h4 id="approach-1---actual-row">
                Approach 1 - Actual row <a class="direct-link" href="#approach-1---actual-row" aria-hidden="true">#</a>
            </h4>
            <p>
                In some cases, the hop from the index to the heap file is too much of a performance penalty for reads, so the indexed row is stored directly within the index. This is known as a 
                <strong>
                    <em>clustered index.</em>
                </strong>
                In MySQL's InnoDB storage engine, the primary key of a table is always a clustered index, and secondary indexes refer to the primary key (rather than a heap file).
            </p>
            <h4 id="covering-index">
                Covering Index <a class="direct-link" href="#covering-index" aria-hidden="true">#</a>
            </h4>
            <p>
                There's a compromise between a clustered index (storing all row data within the index) and a nonclustered index(storing only references to the data within the index) which is known as a <em>covering index</em>
                or <em>index with included columns,</em>
                which stores <em>some</em>
                of a table's columns within the index. With this approach, some queries can be answered using the index alone.
            </p>
            <h4 id="multi-column-indexes">
                Multi-column indexes <a class="direct-link" href="#multi-column-indexes" aria-hidden="true">#</a>
            </h4>
            <p>We've only dealt with indexes which map a single key to a value so far. We need more than that to query multiple columns of a table, or multiple documents simultaneously.</p>
            <p>
                The most common type of multi-column index is a <strong>concatenated index.</strong>
                This type of index combines several fields into one key by appending the columns. This kind is useless if you only want to search for the values for one of the columns. The columns should be in the order of the common search queries/pattern, because the index will sort by the first column.
            </p>
            <p>
                Another approach is a <strong>multi-dimensional index.</strong>
                These kind are a more general way of querying several columns at once, which is useful for geospatial data, for example. Say you want to perform a search for records between both a longitude range <strong>and</strong>
                a latitude range, an LSM tree or B-tree cannot answer that efficiently. You can get all the records within a range of latitudes (but at any longitude), and within a range of longitudes, but not both simultaneously.
            </p>
            <p>An option is to translate a two-dimensional location into a single number using a space-filling curve(?) and then use a regular B-tree index.</p>
            <h4 id="full-text-search-and-fuzzy-indexes">
                Full-text search and fuzzy indexes <a class="direct-link" href="#full-text-search-and-fuzzy-indexes" aria-hidden="true">#</a>
            </h4>
            <p>
                The indexes we've discussed so far assume that we have exact data, and we know the exact values of a key, or a range of values of a key with a sort order. For dealing with things like searching <em>similar</em>
                keys, such as misspelled words, we look at <em>fuzzy</em>
                querying techniques.
            </p>
            <p>
                <em>Levenshtein automaton</em>
                : Supports efficient search for words within a given edit distance.
            </p>
            <h4 id="keep-everything-in-memory">
                Keep everything in memory <a class="direct-link" href="#keep-everything-in-memory" aria-hidden="true">#</a>
            </h4>
            <p>The data structures discussed so far provide answers to the limitations of disks. Disks are awkward to deal with compared to main memory. With both magnetic disks and SSDs, data on disk must be laid out carefully to get good read and write performance. Disks have 2 significant advantages over main memory though:</p>
            <ul>
                <li>They are durable. Content is not lost if power is turned off.</li>
                <li>They have a lower cost per gigabyte than RAM.</li>
            </ul>
            <p>
                There have been developments of <em>in-memory databases</em>
                lately, especially since RAM has become cheaper and many datasets are not that big so keeping them in memory is feasible.
            </p>
            <p>In-memory databases aim for durability by:</p>
            <ul>
                <li>Using special hardware (battery-powered RAM)</li>
                <li>Writing a log of changes to disk</li>
                <li>Writing periodic snapshots to disk</li>
                <li>Replicating the in-memory state to other machines.</li>
            </ul>
            <p>VoltDB, MemSQL and Oracle TimesTen are in-memory databases with a relational model.</p>
            <p>Interestingly, the performance advantage of in-memory databases is not due to the fact that they don't need to read from disk. A disk-based storage engine may never need to read from disk if there's enough memory, because the OS caches recently used data blocks in memory anyway. Rather, they can be faster because there's no overhead of encoding in-memory data structures in a form that can be written to disk.</p>
            <p>Besides performance, an interesting area for in-memory databases is that they allow for the use of data models that are difficult to implement with disk-based indexes. E.g. Redis offers a db-like interface to data structures such as priority queues and sets.</p>
            <p>Recent research indicates that in-memory database architecture can be extended to support datasets larger than the available memory, without bringing back the overheads of a disk-centric architecture. This approach works by evicting the least recently used data from memory to disk when there's not enough memory, and loading it back into memory when it's accessed again in the future.</p>
            <h3 id="transaction-processing-vs-analytics">
                Transaction Processing vs Analytics <a class="direct-link" href="#transaction-processing-vs-analytics" aria-hidden="true">#</a>
            </h3>
            <p>Transaction: A group of reads and writes that form a logical unit.</p>
            <p>
                <strong>OLAP</strong>
                : Online Analytics Processing. Refers to queries generally performed by business analytics that scan over a huge number of record and calculating aggregate statistics.
            </p>
            <p>
                <strong>OLTP:</strong>
                Online Transaction Processing. Interactive queries which typically return a small number of records.
            </p>
            <p>
                In the past, OLTP-type queries and OLAP-type queries were performed on the same databases. However, there's been a push for OLAP-type queries to be run on <strong>data warehouses.</strong>
            </p>
            <h4 id="data-warehousing">
                Data Warehousing <a class="direct-link" href="#data-warehousing" aria-hidden="true">#</a>
            </h4>
            <p>
                A data warehouse is a separate DB that analysts can query without affecting OLTP operations. The data warehouse contains a read-only copy of the data in all the various OLTP systems in the company. Data is extracted from OLTP databases and loaded into the warehouse using an ETL (<em>Extract-Transform-Load)</em>
                process.
            </p>
            <p>It turns out that the indexing algorithms discussed so far work well for OLTP, but not so much for answering analytics queries.</p>
            <p>Transaction processing and data warehousing databases look similar, but the latter is optimized for analytics queries. They are both often accessible through a common SQL interface though.</p>
            <p>A number of SQL-on-Hadoop data warehouses have emerged such as Apache Hive, Spark SQL, Cloudera Impala etc.</p>
            <h4 id="stars-and-snowflakes%3A-schemas-for-analytics">
                Stars and Snowflakes: Schemas for Analytics <a class="direct-link" href="#stars-and-snowflakes%3A-schemas-for-analytics" aria-hidden="true">#</a>
            </h4>
            <p>
                Many data warehouses are used in a fairly formulaic style - <em>star schema</em>
                (or dimensional modelling). The name &quot;star schema &quot;comes from the fact that when the table relationships are visualized, the fact tables is in the middle, surrounded by its dimension tables (these represent the who, what, where, when, how, and why); The connections to the these tables are like the rays of a star.
            </p>
            <p>
                We also have the <em>snowflake schema,</em>
                where dimensions are further broken down into subdimensions.
            </p>
            <h3 id="column-oriented-storage">
                Column Oriented Storage <a class="direct-link" href="#column-oriented-storage" aria-hidden="true">#</a>
            </h3>
            <p>
                In most OLTP databases, storage is laid out in a <em>row-oriented</em>
                fashion: all the values from one row of a table are stored next to each other. Document databases are similar: an entire document is typically stored as one contiguous sequence of bytes.
            </p>
            <p>Analytics queries often access millions of rows, but few columns.</p>
            <p>
                The idea behind <em>column-oriented storage</em>
                is straight forward: don't store all the values from one row together, but store all the values from each <em>column</em>
                together instead. If each column is stored in a separate file, a query only needs to read and parse the columns that it is interested in, which can save work.
            </p>
            <p>The column-oriented storage layout relies on each column file containing the rows in the same order.</p>
            <h4 id="column-compression">
                Column Compression <a class="direct-link" href="#column-compression" aria-hidden="true">#</a>
            </h4>
            <p>
                In addition to loading only the columns from disk that are required for a query, we can reduce the demands on disk throughput by compressing data. Column-oriented storage lends itself well to compression. Different compression techniques can be used such as <em>Bitmap encoding.</em>
                The number of distinct values in a column is often small compared to the number of rows. Therefore, we can take a column with <em>n</em>
                distinct values and turn it into <em>n</em>
                separate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if the row has that value, and 0 if not.
            </p>
            <p>So I believe that in column oriented storage, each 'file' for a column has one row, where the number of columns for that row will be the number of rows in a standard row-wise table, and columns contain the rows in the same order.</p>
            <h4 id="column-oriented-storage-and-column-families">
                Column-oriented storage and column families <a class="direct-link" href="#column-oriented-storage-and-column-families" aria-hidden="true">#</a>
            </h4>
            <p>
                Cassandra and Hbase have a concept of <em>column families</em>
                , which differs from being column oriented. Within each column family, they store all the columns from a row together, along with a row key, and do not use column compression.
            </p>
            <h4 id="sort-order-in-column-storage">
                Sort Order in Column Storage <a class="direct-link" href="#sort-order-in-column-storage" aria-hidden="true">#</a>
            </h4>
            <p>It doesn't really matter in which order the rows are stored in a column store. It's easiest to store them in the order of insertion, but we can choose to impose an order.</p>
            <p>
                It won't make sense to sort each column individually though because we'll lose track of which columns belong to the same row. Rather, we sort the entire row at a time, even though it is stored by column. We can choose the <em>columns</em>
                by which the table should be sorted.
            </p>
            <p>
                <strong>Several different sort orders</strong>
            </p>
            <p>
                <a href="http://vldb.org/pvldb/vol5/p1790_andrewlamb_vldb2012.pdf">
                    <strong>C-stores</strong>
                </a>
                provide an extension to sorting on column stores. Different queries benefit from different sort orders, so why not store the same data sorted in several different ways?
            </p>
            <p>
                <strong>Writing to Column-Oriented Storage</strong>
            </p>
            <p>Writes are more difficult with column-oriented storage. An update-in-place approach, like B-trees use, is not possible with compressed columns. To insert a row in the middle of a sorted table, you would likely have to rewrite all the column files.</p>
            <p>Fortunately, a good approach for writing has been discussed earlier: LSM-trees. All writes go to an in-memory store first, where they are added to a sorted structure and prepared for writing to disk. It doesn't matter whether the in-memory store is row-oriented or column-oriented.</p>
            <h1 id="further-reading%3A">
                Further reading: <a class="direct-link" href="#further-reading%3A" aria-hidden="true">#</a>
            </h1>
            <p>
                Oracle internals: <a href="https://stackoverflow.com/a/40740893" title="https://stackoverflow.com/a/40740893">https://stackoverflow.com/a/40740893</a>
            </p>
            <p>
                How indexes work: <a href="https://stackoverflow.com/questions/1108/how-does-database-indexing-work" title="https://stackoverflow.com/questions/1108/how-does-database-indexing-work">https://stackoverflow.com/questions/1108/how-does-database-indexing-work</a>
            </p>
            <p>
                SQL Server indexes: <a href="https://sqlity.net/en/2445/b-plus-tree/" title="https://sqlity.net/en/2445/b-plus-tree/">https://sqlity.net/en/2445/b-plus-tree/</a>
            </p>
            <p>
                How data is stored on disk: <a href="https://www.red-gate.com/simple-talk/sql/database-administration/sql-server-storage-internals-101/" title="https://www.red-gate.com/simple-talk/sql/database-administration/sql-server-storage-internals-101/">https://www.red-gate.com/simple-talk/sql/database-administration/sql-server-storage-internals-101/</a>
            </p>
            <p>
                <em>Last updated on 03-03-2021</em>
            </p>
            <p>
                <a href="/tags/learning-diary/" class="tag">learning-diary</a>
                <a href="/tags/ddia/" class="tag">ddia</a>
                <a href="/tags/distributed-systems/" class="tag">distributed-systems</a>
            </p>
        </main>
        <footer></footer>
        <!-- Current page: /posts/ddia/part-one/chapter-3/ -->
    </body>
</html>
