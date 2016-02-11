HDHT
======

Some applications need to compute values based not only on current event flow but also on historical data. HDHT provides a simple interface to store
and access historical data in an operator. HDHT is a key value store designed to integrate well with Apache Apex. HDHT allows persistent storage while providing exactly once guarantees required by many streaming applications.

The programming model of a key-value store or hash table can be applied to a wide range of common use cases. Within most streaming applications, ingested events or computed data already carry the key that can be used for storage and retrieval. Many operations performed during computation require key based access. HDHT provides an embedded key value store for the application. The advantage of HDHT over other key value stores in streaming applications are

- Embedded, Hadoop native solution. Does not require install/manage of other services.
- Integrates with Apex check-pointing mechanism to provide exactly once guarantee.

This document provides overview of HDHT and instructions for using HDHT in an operator for storing
and retrieving state of the operator.

## Concepts
#### Write Ahead Log
Each tuple written to HDHT is written to Write Ahead Log (WAL) first. The WAL is used
to recover in-memory state and provide exactly once processing after failure of an operator. HDHT
stores WAL on HDFS to prevent data loss during node failure.

#### Uncommitted Cache
Uncommitted cache is in-memory key value store. Initially updates are written to this memory store, to avoid disk I/Os on every update. When data in Uncommitted cached reaches a configured limit, it is written on the disk. It avoids frequent data flushes and small file creation by writing data in bulk from the cache to disk thereby also improving throughput.

#### Data Files
HDHT flushes memory to persisted storage periodically. The data is kept indexed for efficient retrieval of given key. HDHT supports multiple data file backends. Default backend used is DTFile, which is a modified version of Hadoop TFile with block cache for speedy lookups.
Available backends are
- `TFile`: Files are written in hadoop [Tfile](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/io/file/tfile/TFile.html) format
- `DTFile`: Files are written in TFile format; during lookup HDHT maintains a block cache to reduce disk I/Os.
- `HFile` : Files are written in HBase format.

#### Metadata
Metadata file keeps information about data files. Each data file record contains start key and name of the file. Metadata file also contains WAL recovery information, which is used during recovery after failure.

#### Partition (HDHT Bucket)
By default, when the operator is partitioned, the partitioning is reflected by HDHT in the filesystem by using a separate directory for each operator partition. Each directory is accessed only by the associated operator partition. Each partition has its own WAL and metadata file. Each
HDHT partition is identified by bucketKey, which is also the name of the subdirectory used for
storing data for the partition.

## Interface
HDHT supports two basic operations **get** and **put**, they are wrapped by interfaces HDHT.Writer, HDHT.Reader and an abstract implementation is provided by the HDHTReader and HDHTWriter classes.

Operations supported by HDHT are.
```java
byte[] get(long bucketKey, Slice key) throws IOException;
void put(long bucketKey, Slice key, byte[] value) throws IOException;
byte[] getUncommitted(long bucketKey, Slice key);
```

All methods takes bucketKey as the first argument. The bucketKey is used as a partition key within HDHT.

* **put** store data in HDHT. The data written is written to the WAL first and then stored in uncommitted cache.
After enough dirty data is accumulated in cache or enough time has elapsed from last flush, this cache is flushed to the data files.
* **getUncommitted** does a lookup in uncommitted cache. Uncommitted cache is in-memory key value store.
* **get** does a lookup in persisted storage file and return the data. **Note** *get* does not
 return data from uncommitted cache.

## Architecture
Please refer to [HDHT Blog](https://www.datatorrent.com/data-store-for-scalable-stream-processing/)
for the architecture of HDHT.

### Codec
HDHT provides an abstract implementation *AbstractSinglePortHDHTWriter*, which uses a user defined codec for serialization and de-serialization.
```java
public interface HDHTCodec<EVENT> extends StreamCodec<EVENT>
{
  byte[] getKeyBytes(EVENT event);
  byte[] getValueBytes(EVENT event);
  EVENT fromKeyValue(Slice key, byte[] value);
}
```

It has following methods
- **getKeyBytes** Return key as a byte array from the event.
- **getValueBytes** Return value as a byte array from event.
- **fromKeyValue** HDHT will use this function to deserialize key and value byte arrays to reconstruct the user event object.
- **getPartition** This method is inherited from StreamCodec, its return value is used to determine HDHT bucket where this event will be written. The same stream codec is used
 for partition of the input port which make sure that data for same event goes to a single partition
 of the operator.


## Configuration

### fileStore

This setting determines the format in which files are written. Default is DTFileImpl.

### basePath 

Location in HDFS where data files are stored. This is required configuration parameter.

Property File Syntax

```xml
   <property>
     <name>dt.operator.{name}.store.basePath</name>
     <value>/home/hdhtdatadir</value>
  </property>
```

Java API.

```java
/* select DTFile backend with basePath set to HDHTdata */
TFileImpl.DTFileImpl hdsFile = new TFileImpl.DTFileImpl();
hdsFile.setBasePath("HDHTdata");
store.setFileStore(hdsFile);
```

### maxFileSize

Size of each file. Default value is 134217728134217728 (i.e 128MB).

Property File Syntax
```xml
 <property>
   <name>dt.operator.{name}.maxFileSize</name>
   <value>{value in bytes}</value>
 </property>
```
Java API.
```java
store.setMaxFileSize(64 * 1024 * 1024);
```
### flushSize 

HDHT will flush data to files after number of unwritten tuples crosses this limit. Default value is 1000000.

Property File Syntax
```xml
<property>
  <name>dt.operator.{name}.flushSize</name>
  <value>{number}</value>
</property>
```
Java API.
```java
store.setFlushSize(1000000);
```

### flushIntervalCount

This setting will force data flush even if number of tuples are below *flushSize*. Default value is 120 windows.

  Property File Syntax
```xml
 <property>
   <name>dt.operator.{name}.flushIntervalCount</name>
   <value>{number of windows}</value>
 </property>
```
 Java API.
```java
store.setFlushIntervalCount(120);
```
### maxWalFileSize

Write Ahead Log segment size. Older segments are deleted once data is written to the data files. Default value is 67108864 (64MB)

 Property File Syntax
```xml
<property>
  <name>dt.operator.{name}.maxWalFileSize</name>
  <value>{value in bytes}</value>
</property>
```
Java API.
```java
store.setMaxWalFileSize(128 * 1024 * 1024);
```

## Example
 This is a sample reference implementation, which computes how many times a word was seen in an
 application. The partial count is stored in the HDHT. The application does a lookup for
the previous count and writes back the incremented count in HDHT.

### Store Operator
HDHT provides following abstract implementations
* `HDHTReader` - This class implements functionality required for *get*, It access HDHT
in read-only mode.
* `HDHTWriter` - This class extends functionality of `HDHTReader` by adding support for *put*,
  this class also maintains *uncommitted cache*, which can be accessed through *getUncommitted*
  method.
* `AbstractSinglePortHDHTWriter` - This class extends from `HDHTWriter` and provides common functionality
required for the operator. This class support code for operator partitioning. Also it provides an input port with a default implementation of
storing value received on the port to the HDHT using the coded provided.

For this example we will use `AbstractSinglePortHDHTWriter` for the store, we need to
implement codec which is used by `AbstractSinglePortHDHTWriter` for serialization and deserialization. Following is
a simple serializer which serializes key and ignores the value part, as the input to the operator is only keys.

### Implement a Codec
```java
  public static class StringCodec extends KryoSerializableStreamCodec<String> implements HDHTCodec<String> {
    @Override
    public byte[] getKeyBytes(String s)
    {
      return s.getBytes();
    }

    @Override
    public byte[] getValueBytes(String s)
    {
      return s.getBytes();
    }

    @Override
    public String fromKeyValue(Slice key, byte[] value)
    {
      return new String(key.buffer, key.offset, key.length);
    }
  }
```

The store operator is implemented as shown below, we will need to provide an implementation of
`getCodec`, and override `processEvent` to change default behavior of storing data in HDHT
directly.

```java
public class HDHTWordCounter extends AbstractSinglePortHDHTWriter<String>
{
  public transient DefaultOutputPort<Pair<String, Long>> out = new DefaultOutputPort<>();
  private transient HashMap<String, AtomicLong> cache;

  @Override
  protected HDHTCodec<String> getCodec()
  {
    return new StringCodec();
  }

  @Override
  protected void processEvent(String word) throws IOException
  {
    AtomicLong count = cache.get(word);
    if (count == null) {
      count = new AtomicLong(0L);
      cache.put(word, count);
    }
    count.incrementAndGet();
  }

  private void updateCount() throws IOException
  {
    for(Map.Entry<String, AtomicLong> entry : cache.entrySet()) {
      String word = entry.getKey();
      long prevCount = 0;
      byte[] key = codec.getKeyBytes(word);
      Slice keySlice = new Slice(key);
      long bucketKey = getBucketKey(word);
      /** First look for cached data */
      byte[] value = getUncommitted(bucketKey, keySlice);
      if (value == null) {
        /** look into persisted data files */
        value = get(bucketKey, keySlice);
        if (value == null) {
          value = ByteBuffer.allocate(8).putLong(0).array();
        }
      }

      prevCount = ByteBuffer.wrap(value).getLong();

      /** update count by taking new event into account */
      prevCount += entry.getValue().get();

      /** save computed result back to HDHT */
      put(bucketKey, keySlice, ByteBuffer.wrap(value).putLong(prevCount).array());

      /* emit updated counts on the output port */
      out.emit(new Pair<>(word, prevCount));
    }
  }

  @Override
  public void beginWindow(long windowId)
  {
    super.beginWindow(windowId);
    cache = new HashMap<>();
  }

  @Override
  public void endWindow()
  {
    try {
      updateCount();
      super.endWindow();
    } catch (IOException e) {

      throw new RuntimeException("Unable to flush to HDHT");
    }
  }
}
```

Sample Application.
```java
@ApplicationAnnotation(name="HDHTWordCount")
public class HDHTWordCountApp implements StreamingApplication
{
  @Override
  public void populateDAG(DAG dag, Configuration configuration)
  {
    AbstractFileInputOperator.FileLineInputOperator gen = dag.addOperator("Reader", new AbstractFileInputOperator.FileLineInputOperator());
    gen.setDirectory("/home/data");

    WordSplitter splitter = dag.addOperator("Splitter", new WordSplitter());

    HDHTWordCounter store = dag.addOperator("Store", new HDHTWordCounter());
    ConsoleOutputOperator console = dag.addOperator("Console", new ConsoleOutputOperator());

    TFileImpl.DTFileImpl hdsFile = new TFileImpl.DTFileImpl();
    hdsFile.setBasePath("WALBenchMarkDir");
    store.setFileStore(hdsFile);

    dag.addStream("lines", gen.output, splitter.input);
    dag.addStream("s1", splitter.output, store.input);
    dag.addStream("s2", store.out, console.input);
  }
}
```


### Performance tuning

#### Effect of frequent WAL flushes.
HDHT stores Write Ahead Log (WAL) on HDFS, WAL is flushed at end of every application window. Operator will be blocked till WAL is persisted on the disks. This flush will add additional delay
to the operator. To avoid frequent delay we can reduce the frequency of flush by increasing APPLICATION_WINDOW_SIZE.

#### Application Level Cache
Maintain a cache to avoid frequent serialization and de-serialization of events while accessing HDHT. For example in the provided example the operator keeps computed counts till the endWindow and flushes the data to HDHT at end of the application window. If duplicate keys are seen within an application window we will save on serialization and de-serialization time.

#### Key design
HDHT gives best performance if keys are monotonically increasing, In this case
HDHT does not have to overwrite existing files, which avoids expensive disk I/O thus yielding
optimal performance. Overwriting existing file is costly operation, as it involves reading file data
to memory and applying new changes from committed cached which falls within the key range of file, and
writing back changes to disk again. If you are storing time series data in HDHT, it is best to
use timestamp as the leading field in the key.

For keys which are not monotonically increasing, key design should be such that hot
keys falls in small number of files. For example, suppose each file size is `S` bytes and flush is
triggered every `T` seconds, and HDFS write bandwidth per container is `B` bytes per second, in this case we can sustain the write throughput, if keys processed within 30 seconds span at most `(T / (S/B))` files. 

If S, T and B are 128MB, 30 seconds, and 40MB respectively, this expression evaluates to 10, so if your keys span more than 10 files with 30 seconds, the write cannot be sustained.

#### File Backend
Prefer `DTFile` backend implementation over `TFile` backend implementation if you are going to issue frequent `get` operations. DTFile backend caches file blocks
in memory which reduces disk I/O during cache hit.

## Limitations
- Dynamic Partitioning is not supported yet.
- Write to same bucket from multiple operator is not supported. The default implementation use derives bucketKey based on number of operator partitions and hashcode of the event. If user chooses to use different bucketKey he needs to make sure that a single bucketKey is handled by only one operator partition.
