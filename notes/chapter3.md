# Chapter 3 Storage and Retrieval

In Chapter 2 we discussed data models and query languages—i.e., the format in which you (the application developer) give the database your data, and the mechanism by which you can ask for it again later. In this chapter we discuss the same from the **database’s point of view**: how we can **store the data** that we’re given, and how we can **find it again** when we’re asked for it.

## 1. Data Structures That Power Your Database

In order to efficiently find the value for a particular key in the database, we need a different data structure: an **index**. The general idea behind them is to keep some additional metadata on the side, which acts as a signpost（路标） and helps you to locate the data you want.

An index is an additional structure that is derived from the primary data. Many databases allow you to add and remove indexes, and this doesn’t affect the contents of the database; it only affects the performance of queries. Maintaining additional structures incurs overhead, especially on writes. For writes, it’s hard to beat the performance of simply appending to a file, because that’s the simplest possible write operation. Any kind of index usually slows down writes, because the index also needs to be updated every time data is written. **This is an important trade-off in storage systems: well-chosen indexes speed up read queries, but every index slows down writes.**

### 1.1 Hash Indexes

Let’s say our data storage consists only of appending to a file. Then the simplest possible indexing strategy is this: keep an in-memory hash map where every key is mapped to a byte offset in the data file—the location at which the value can be found. Whenever you append a new key-value pair to the file, you also update the hash map to reflect the offset of the data you just wrote.

<img src="images/image-20220102192010276.png" alt="image-20220102192010276" style="zoom: 33%;" />

Hash indexes is well suited to situations where the value for each key is updated frequently—there are a lot of writes, but there are not too many distinct keys.

As described so far, we only ever append to a file—so how do we avoid eventually running out of disk space? A good solution is to **break the log into segments of a certain size**. We can then perform **compaction** on these segments. Compaction means throwing away duplicate keys in the log, and **keeping only the most recent update for each key**.

Moreover, since compaction often makes segments much smaller, we can also merge several segments together at the same time as performing the compaction. **The merging and compaction of old segments can be done in a background thread**, and while it is going on, we can still continue to serve read and write requests as normal, using the old segment files. After the merging process is complete, we switch read requests to using the new merged segment instead of the old segments—and then the old segment files can simply be deleted.

<img src="images/image-20220105222643021.png" alt="image-20220105222643021" style="zoom:67%;" />

**Each segment now has its own in-memory hash table**, mapping keys to file offsets. In order to find the value for a key, we first check the most recent segment’s hash map; if the key is not present we check the second-most-recent segment, and so on.

Some of the issues that are important in a real implementation are:

- Deleting records: If you want to delete a key and its associated value, you have to append a special
  deletion record to the data file.
- Crash recovery: If the database is restarted, the in-memory hash maps are lost. We can store a snapshot of each segment's hash map on disk.
- Partially written records: The database may crash at any time, including halfway through appending a
  record to the log. We can add checksums for records.
- Concurrency control: As writes are appended to the log in a strictly sequential order, a common implementation
  choice is to **have only one writer thread**.

An append-only design turns out to be good for several reasons:

- Appending and segment merging are sequential write operations, which are generally much faster than random writes, especially on magnetic spinning-disk hard drives.
- Concurrency and crash recovery are much simpler if segment files are appendonly or immutable. For example, you don’t have to worry about the case where a crash happened while a value was being overwritten, leaving you with a file containing part of the old and part of the new value spliced together.
- Merging old segments avoids the problem of data files getting fragmented（碎片化） over time.

However, the hash table index also has limitations:

- The hash table must fit in memory, so if you have a very large number of keys, you’re out of luck. In principle, you could maintain a hash map on disk, but unfortunately it is difficult to make an on-disk hash map perform well. It
  requires a lot of random access I/O, it is expensive to grow when it becomes full, and hash collisions require complex logic.
- Range queries are not efficient. For example, you cannot easily scan over all keys between kitty00000 and kitty99999—you’d have to look up each key individually in the hash maps.

### 1.2 SSTables and LSM-Trees





















