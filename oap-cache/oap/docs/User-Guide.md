# User Guide

* [Prerequisites](#Prerequisites)
* [Getting Started](#Getting-Started)
* [Configuration for YARN Cluster Mode](#Configuration-for-YARN-Cluster-Mode)
* [Configuration for Spark Standalone Mode](#Configuration-for-Spark-Standalone-Mode)
* [Working with SQL Index](#Working-with-SQL-Index)
* [Working with SQL Data Source Cache](#Working-with-SQL-Data-Source-Cache)
* [Run TPC-DS Benchmark](#Run-TPC-DS-Benchmark)
* [Advanced Configuration](#Advanced-Configuration)


## Prerequisites

SQL Index and Data Source Cache on Spark requires a working Hadoop cluster with YARN and Spark. Running Spark on YARN requires a binary distribution of Spark, which is built with YARN support. We provide pre-built [Spark-2.4.4](https://github.com/Intel-bigdata/spark/releases/download/v2.4.4-intel-oap-0.8.2/spark-2.4.4-bin-hadoop2.7-intel-oap-0.8.2.tgz) based on hadoop-2.7 whith numa patch applied to accelerate performance. If you use a different hadoop version, you should build it from source [here](https://github.com/Intel-bigdata/spark/releases/tag/v2.4.4-intel-oap-0.8.2).

## Getting Started

### Building

We have provided a Conda package which will automatically install dependencies and build OAP jars, please follow [OAP-Installation-Guide](../../../docs/OAP-Installation-Guide.md) and you can find compiled OAP jars under
 `$HOME/miniconda2/envs/oapenv/oap_jars` once finished the installation.

If you’d like to build from source code, please refer to [Developer Guide](Developer-Guide.md) for the detailed steps.

### Spark Configurations

Users usually test and run Spark SQL or Scala scripts in Spark Shell,  which launches Spark applications on YRAN with ***client*** mode. In this section, we will start with Spark Shell then introduce other use scenarios. 

Before you run `$SPARK_HOME/bin/spark-shell `, you need to configure Spark for integration. You need to add or update the following configurations in the Spark configuration file `$SPARK_HOME/conf/spark-defaults.conf` on your working node.

```
spark.sql.extensions              org.apache.spark.sql.OapExtensions
# absolute path of the jar on your working node
spark.files                       $HOME/miniconda2/envs/oapenv/oap_jars/oap-cache-<version>-with-spark-<version>.jar,$HOME/miniconda2/envs/oapenv/oap_jars/oap-common-<version>-with-spark-<version>.jar
# relative path of the jar
spark.executor.extraClassPath     ./oap-cache-<version>-with-spark-<version>.jar:./oap-common-<version>-with-spark-<version>.jar
# absolute path of the jar on your working node
spark.driver.extraClassPath       $HOME/miniconda2/envs/oapenv/oap_jars/oap-cache-<version>-with-spark-<version>.jar:$HOME/miniconda2/envs/oapenv/oap_jars/oap-common-<version>-with-spark-<version>.jar
```
### Verify Integration 

After configuration, you can follow these steps to verify the OAP integration is working using Spark Shell.

1. Create a test data path on your HDFS. `hdfs:///user/oap/` for example.
   ```
   hadoop fs -mkdir /user/oap/
   
   ```
2. Launch Spark Shell using the following command on your working node.
   ``` 
   . $SPARK_HOME/bin/spark-shell
   ```

3. Execute the following commands in Spark Shell to test OAP integration. 
   ```
   > spark.sql(s"""CREATE TABLE oap_test (a INT, b STRING)
          USING parquet
          OPTIONS (path 'hdfs:///user/oap/')""".stripMargin)
   > val data = (1 to 30000).map { i => (i, s"this is test $i") }.toDF().createOrReplaceTempView("t")
   > spark.sql("insert overwrite table oap_test select * from t")
   > spark.sql("create oindex index1 on oap_test (a)")
   > spark.sql("show oindex from oap_test").show()
   ```

This test creates an index for a table and then shows it. If there are no errors, the OAP `.jar` is working with the configuration. The picture below is an example of a successfully run.

![Spark_shell_running_results](./image/spark_shell_oap.png)

## Configuration for YARN Cluster Mode

Spark Shell, Spark SQL CLI and Thrift Sever run Spark application in ***client*** mode. While Spark Submit tool can run Spark application in ***client*** or ***cluster*** mode, which is decided by `--deploy-mode` parameter. [Getting Started](#Getting-Started) session has shown the configurations needed for ***client*** mode. If you are running Spark Submit tool in ***cluster*** mode, you need to follow the below configuration steps instead.

Add the following OAP configuration settings to `$SPARK_HOME/conf/spark-defaults.conf` on your working node before running `spark-submit` in ***cluster*** mode.
```
spark.sql.extensions              org.apache.spark.sql.OapExtensions
# absolute path on your working node
spark.files                       $HOME/miniconda2/envs/oapenv/oap_jars/oap-cache-<version>-with-spark-<version>.jar,$HOME/miniconda2/envs/oapenv/oap_jars/oap-common-<version>-with-spark-<version>.jar
# relative path    
spark.executor.extraClassPath     ./oap-cache-<version>-with-spark-<version>.jar:./oap-common-<version>-with-spark-<version>.jar
# relative path 
spark.driver.extraClassPath       ./oap-cache-<version>-with-spark-<version>.jar:./oap-common-<version>-with-spark-<version>.jar
```

## Configuration for Spark Standalone Mode

In addition to running on the YARN cluster manager, Spark also provides a simple standalone deploy mode. If you are using Spark in Spark Standalone mode:

1. Make sure the OAP `.jar` at the same path of **all** the worker nodes. 
2. Add the following configuration settings to `$SPARK_HOME/conf/spark-defaults.conf` to the working node.
```
spark.sql.extensions               org.apache.spark.sql.OapExtensions
# absolute path on worker nodes
spark.executor.extraClassPath      /home/oap/jars/oap-cache-<version>-with-spark-<version>.jar:/home/oap/jars/oap-common-<version>-with-spark-<version>.jar
# absolute path on worker nodes
spark.driver.extraClassPath        /home/oap/jars/oap-cache-<version>-with-spark-<version>.jar:/home/oap/jars/oap-common-<version>-with-spark-<version>.jar
```

## Working with SQL Index

After a successful OAP integration, you can use OAP SQL DDL to manage table indexes. The DDL operations include `index create`, `drop`, `refresh`, and `show`. Test these functions using the following examples in Spark Shell.

```
> spark.sql(s"""CREATE TABLE oap_test (a INT, b STRING)
       USING parquet
       OPTIONS (path 'hdfs:///user/oap/')""".stripMargin)
> val data = (1 to 30000).map { i => (i, s"this is test $i") }.toDF().createOrReplaceTempView("t")
> spark.sql("insert overwrite table oap_test select * from t")       
```

### Index Creation

Use the CREATE OINDEX DDL command to create a B+ Tree index or bitmap index. 
``` 
CREATE OINDEX index_name ON table_name (column_name) USING [BTREE, BITMAP]
```
The following example creates a B+ Tree index on column "a" of the `oap_test` table.
``` 
> spark.sql("create oindex index1 on oap_test (a)")
```
Use SHOW OINDEX command to show all the created indexes on a specified table.
```
> spark.sql("show oindex from oap_test").show()
```
### Use Index

Using index in a query is transparent. When SQL queries have filter conditions on the column(s) which can take advantage of the index to filter the data scan, the index will automatically be applied to the execution of Spark SQL. The following example will automatically use the underlayer index created on column "a".
```
> spark.sql("SELECT * FROM oap_test WHERE a = 1").show()
```

### Drop index

Use DROP OINDEX command to drop a named index.
```
> spark.sql("drop oindex index1 on oap_test")
```
## Working with SQL Data Source Cache

Data Source Cache can provide input data cache functionality to the executor. When using the cache data among different SQL queries, configure cache to allow different SQL queries to use the same executor process. Do this by running your queries through the Spark ThriftServer as shown below. For cache media, we support both DRAM and Intel PMem which means you can choose to cache data in DRAM or Intel PMem if you have PMem configured in hardware.

### Use DRAM Cache 

1. Make the following configuration changes in Spark configuration file `$SPARK_HOME/conf/spark-defaults.conf`. 

   ```
   spark.memory.offHeap.enabled                      false
   spark.oap.cache.strategy                          guava
   spark.sql.oap.cache.memory.manager                offheap
   # according to the resource of cluster
   spark.executor.memoryOverhead                     50g
   # equal to the size of executor.memoryOverhead
   spark.executor.sql.oap.cache.offheap.memory.size  50g
   # for parquet fileformat, enable binary cache
   spark.sql.oap.parquet.binary.cache.enabled        true
   # for orc fileformat, enable binary cache
   spark.sql.oap.orc.binary.cache.enabled            true
   ```

   ***NOTE***: Change `spark.executor.sql.oap.cache.offheap.memory.size` based on the availability of DRAM capacity to cache data, and its size is equal to `spark.executor.memoryOverhead`

2. Launch Spark ***ThriftServer***

   Launch Spark Thrift Server, and use the Beeline command line tool to connect to the Thrift Server to execute DDL or DML operations. The data cache will automatically take effect for Parquet or ORC file sources. 
   
   The rest of this section will show you how to do a quick verification of cache functionality. It will reuse the database metastore created in the [Working with Data Source Cache Index](#Working-with-SQL-Index) section, which creates the `oap_test` table definition. In production, Spark Thrift Server will have its own metastore database directory or metastore service and use DDL's through Beeline for creating your tables.

   When you run ```spark-shell``` to create the `oap_test` table, `metastore_db` will be created in the directory where you ran '$SPARK_HOME/bin/spark-shell'. ***Go to that directory*** and execute the following command to launch Thrift JDBC server and run queries.

   ```
   . $SPARK_HOME/sbin/start-thriftserver.sh
   ```

3. Use Beeline and connect to the Thrift JDBC server, replacing the hostname (mythriftserver) with your own Thrift Server hostname.

   ```
   . $SPARK_HOME/bin/beeline -u jdbc:hive2://<mythriftserver>:10000       
   ```

   After the connection is established, execute the following commands to check the metastore is initialized correctly.

   ```
   > SHOW databases;
   > USE default;
   > SHOW tables;
   ```
 
4. Run queries on the table that will use the cache automatically. For example,

   ```
   > SELECT * FROM oap_test WHERE a = 1;
   > SELECT * FROM oap_test WHERE a = 2;
   > SELECT * FROM oap_test WHERE a = 3;
   ...
   ```

5. Open the Spark History Web UI and go to the OAP tab page to see verify the cache metrics. The following picture is an example.

   ![webUI](./image/webUI.png)


### Use PMem Cache 

#### Prerequisites

The following are required to configure OAP to use PMem cache.

- PMem hardware is successfully deployed on each node in cluster.

- Directories exposing PMem hardware on each socket. For example, on a two socket system the mounted PMem directories should appear as `/mnt/pmem0` and `/mnt/pmem1`. Correctly installed PMem must be formatted and mounted on every cluster worker node.

   ```
   // use ipmctl command to show topology and dimm info of PMem
   ipmctl show -topology
   ipmctl show -dimm
   // provision PMem in app direct mode
   ipmctl create -goal PersistentMemoryType=AppDirect
   // reboot system to make configuration take affect
   reboot
   // check capacity provisioned for app direct mode(AppDirectCapacity)
   ipmctl show -memoryresources
   // show the PMem region information
   ipmctl show -region
   // create namespace based on the region, multi namespaces can be created on a single region
   ndctl create-namespace -m fsdax -r region0
   ndctl create-namespace -m fsdax -r region1
   // show the created namespaces
   fdisk -l
   // create and mount file system
   echo y | mkfs.ext4 /dev/pmem0
   echo y | mkfs.ext4 /dev/pmem1
   mkdir -p /mnt/pmem0
   mkdir -p /mnt/pmem1 
   mount -o dax /dev/pmem0 /mnt/pmem0
   mount -o dax /dev/pmem1 /mnt/pmem1
   ```

   In this case file systems are generated for 2 numa nodes, which can be checked by "numactl --hardware". For a different number of numa nodes, a corresponding number of namespaces should be created to assure correct file system paths mapping to numa nodes.
   
   For more information you can refer to [Quick Start Guide: Provision Intel® Optane™ DC Persistent Memory](https://software.intel.com/content/www/us/en/develop/articles/quick-start-guide-configure-intel-optane-dc-persistent-memory-on-linux.html)

- Make sure [Vmemcache](https://github.com/pmem/vmemcache) library has been installed on every cluster worker node if vmemcache strategy is chosen for PMem cache. If you have finished [OAP-Installation-Guide](../../docs/OAP-Installation-Guide.md), vmemcache library will be automatically installed by Conda.
  
  Or you can follow the [build/install](./Developer-Guide.md#build-and-install-vmemcache) steps and make sure `libvmemcache.so` exist in `/lib64` directory in each worker node.

- Currently, using Community Spark occasionally has the problem of two executors being bound to the same PMem path, so we recommend you use our pre-built numa-patched [Spark-2.4.4](https://github.com/Intel-bigdata/spark/releases/download/v2.4.4-intel-oap-0.8.2/spark-2.4.4-bin-hadoop2.7-intel-oap-0.8.2.tgz), which can not only improve performance, but also solve this problem.

#### Configure for NUMA

1. Install `numactl` to bind the executor to the PMem device on the same NUMA node. 

   ```yum install numactl -y ```

2. We strongly recommend you use numa-patched Spark to achieve better performance gain.
   
   Build Spark from source to enable numa-binding support, refer to [enable-numa-binding-for-PMem-in-spark](./Developer-Guide.md#Enabling-NUMA-binding-for-PMem-in-Spark). Or you can just download our pre-built [Spark-2.4.4](https://github.com/Intel-bigdata/spark/releases/download/v2.4.4-intel-oap-0.8.2/spark-2.4.4-bin-hadoop2.7-intel-oap-0.8.2.tgz).

#### Configure for PMem 

Create `persistent-memory.xml` in `$SPARK_HOME/conf/` if it doesn't exist. Use the following template and change the `initialPath` to your mounted paths for PMem devices. 

```
<persistentMemoryPool>
  <!--The numa id-->
  <numanode id="0">
    <!--The initial path for Intel Optane DC persistent memory-->
    <initialPath>/mnt/pmem0</initialPath>
  </numanode>
  <numanode id="1">
    <initialPath>/mnt/pmem1</initialPath>
  </numanode>
</persistentMemoryPool>
```

#### Configure to enable PMem cache

Make the following configuration changes in `$SPARK_HOME/conf/spark-defaults.conf`.

```
# 2x number of your worker nodes
spark.executor.instances          6
# enable numa
spark.yarn.numa.enabled           true
# Enable OAP jar in Spark
spark.sql.extensions              org.apache.spark.sql.OapExtensions

# absolute path of the jar on your working node, when in Yarn client mode
spark.files                       $HOME/miniconda2/envs/oapenv/oap_jars/oap-cache-<version>-with-spark-<version>.jar,$HOME/miniconda2/envs/oapenv/oap_jars/oap-common-<version>-with-spark-<version>.jar
# relative path of the jar, when in Yarn client mode
spark.executor.extraClassPath     ./oap-cache-<version>-with-spark-<version>.jar:./oap-common-<version>-with-spark-<version>.jar
# absolute path of the jar on your working node,when in Yarn client mode
spark.driver.extraClassPath       $HOME/miniconda2/envs/oapenv/oap_jars/oap-cache-<version>-with-spark-<version>.jar:$HOME/miniconda2/envs/oapenv/oap_jars/oap-common-<version>-with-spark-<version>.jar

# for parquet file format, enable binary cache
spark.sql.oap.parquet.binary.cache.enabled                   true
# for ORC file format, enable binary cache
spark.sql.oap.orc.binary.cache.enabled                       true
spark.oap.cache.strategy                                     vmem 
spark.executor.sql.oap.cache.persistent.memory.initial.size  256g 
# according to your cluster
spark.executor.sql.oap.cache.guardian.memory.size            10g
```
The `vmem` cache strategy is based on libvmemcache (buffer based LRU cache), which provides a key-value store API. Follow these steps to enable vmemcache support in Data Source Cache.

- `spark.executor.instances`: We suggest setting the value to 2X the number of worker nodes when NUMA binding is enabled. Each worker node runs two executors, each executor is bound to one of the two sockets, and accesses the corresponding PMem device on that socket.
- `spark.executor.sql.oap.cache.persistent.memory.initial.size`: It is configured to the available PMem capacity to be used as data cache per exectutor.
 
**NOTE**: If "PendingFiber Size" (on spark web-UI OAP page) is large, or some tasks fail with "cache guardian use too much memory" error, set `spark.executor.sql.oap.cache.guardian.memory.size ` to a larger number as the default size is 10GB. The user could also increase `spark.sql.oap.cache.guardian.free.thread.nums` or decrease `spark.sql.oap.cache.dispose.timeout.ms` to free memory more quickly.

### Verify PMem cache functionality

- After finishing configuration, restart Spark Thrift Server for the configuration changes to take effect. Start at step 2 of the [Use DRAM Cache](#use-dram-cache) guide to verify that cache is working correctly.

- Verify NUMA binding status by confirming keywords like `numactl --cpubind=1 --membind=1` contained in executor launch command.

- Check PMem cache size by checking disk space with `df -h`.For `vmemcache` strategy, disk usage will reach the initial cache size once the PMem cache is initialized and will not change during workload execution. For `Guava/Noevict` strategies, the command will show disk space usage increases along with workload execution. 


## Run TPC-DS Benchmark

This section provides instructions and tools for running TPC-DS queries to evaluate the cache performance of various configurations. The TPC-DS suite has many queries and we select 9 I/O intensive queries to simplify performance evaluation.

We created some tool scripts [oap-benchmark-tool.zip](https://github.com/Intel-bigdata/OAP/releases/download/v0.8.4-spark-2.4.4/oap-benchmark-tool.zip) to simplify running the workload. If you are already familiar with TPC-DS data generation and running a TPC-DS tool suite, skip our tool and use the TPC-DS tool suite directly.

### Prerequisites

- Python 2.7+ is required on the working node. 

### Prepare the Tool

1. Download [oap-benchmark-tool.zip](https://github.com/Intel-bigdata/OAP/releases/download/v0.8.4-spark-2.4.4/oap-benchmark-tool.zip) and unzip to a folder (for example, `oap-benchmark-tool` folder) on your working node. 
2. Copy `oap-benchmark-tool/tools/tpcds-kits` to ***ALL*** worker nodes under the same folder (for example, `/home/oap/tpcds-kits`).

### Generate TPC-DS Data

1. Update the values for the following variables in `oap-benchmark-tool/scripts/tool.conf` based on your environment and needs.

   - SPARK_HOME: Point to the Spark home directory of your Spark setup.
   - TPCDS_KITS_DIR: The tpcds-kits directory you coped to the worker nodes in the above prepare process. For example, /home/oap/tpcds-kits
   - NAMENODE_ADDRESS: Your HDFS Namenode address in the format of host:port.
   - THRIFT_SERVER_ADDRESS: Your working node address on which you will run Thrift Server.
   - DATA_SCALE: The data scale to be generated in GB
   - DATA_FORMAT: The data file format. You can specify parquet or orc

   For example:

```
export SPARK_HOME=/home/oap/spark-2.4.4
export TPCDS_KITS_DIR=/home/oap/tpcds-kits
export NAMENODE_ADDRESS=mynamenode:9000
export THRIFT_SERVER_ADDRESS=mythriftserver
export DATA_SCALE=1024
export DATA_FORMAT=parquet
```

2. Start data generation.

   In the root directory of this tool (`oap-benchmark-tool`), run `scripts/run_gen_data.sh` to start the data generation process. 

```
cd oap-benchmark-tool
sh ./scripts/run_gen_data.sh
```

   Once finished, the `$scale` data will be generated in the HDFS folder `$format/genData$scale`. And a database called `tpcds_$format$scale` will contain the TPC-DS tables.

### Start Spark Thrift Server

Start the Thrift Server in the tool root folder, which is the same folder you run data generation scripts. Use either the PMem or DRAM script to start the Thrift Server.

#### Use PMem as Cache Media

Update the configuration values in `scripts/spark_thrift_server_yarn_with_PMem.sh` to reflect your environment. 
Normally, you need to update the following configuration values to cache to PMem.

- --driver-memory
- --executor-memory
- --executor-cores
- --conf spark.oap.cache.strategy
- --conf spark.executor.sql.oap.cache.guardian.memory.size
- --conf spark.executor.sql.oap.cache.persistent.memory.initial.size

These settings will override the values specified in Spark configuration file ( `spark-defaults.conf`). After the configuration is done, you can execute the following command to start Thrift Server.

```
cd oap-benchmark-tool
sh ./scripts/spark_thrift_server_yarn_with_PMem.sh start
```
In this script, we use `vmem` as cache strategy for Parquet Binary data cache. 

#### Use DRAM as Cache Media 

Update the configuration values in `scripts/spark_thrift_server_yarn_with_DRAM.sh` to reflect your environment. Normally, you need to update the following configuration values to cache to DRAM.

- --driver-memory
- --executor-memory
- --executor-cores
- --conf spark.executor.sql.oap.cache.offheap.memory.size
- --conf spark.executor.memoryOverhead

These settings will override the values specified in Spark configuration file (`spark-defaults.conf`). After the configuration is done, you can execute the following command to start Thrift Server.

```
cd oap-benchmark-tool
sh ./scripts/spark_thrift_server_yarn_with_DRAM.sh  start
```

### Run Queries

Execute the following command to start to run queries.

```
cd oap-benchmark-tool
sh ./scripts/run_tpcds.sh
```

When all the queries are done, you will see the `result.json` file in the current directory.

## Advanced Configuration

In addition to usage information provided above, we provide more strategies for SQL index and data source cache in this section.

Their needed dependencies like ***Memkind*** ,***Vmemcache*** and ***Plasma*** can be automatically installed when following [OAP Installation Guide](../../../docs/OAP-Installation-Guide.md), corresponding feature jars can be found under `$HOME/miniconda2/envs/oapenv/oap_jars`.

- [Additional Cache Strategies](#Additional-Cache-Strategies)  In addition to **vmem** cache strategy, Data Source Cache also supports 3 other cache strategies: **guava**, **noevict**  and **external cache**.
- [Index and Data Cache Separation](#Index-and-Data-Cache-Separation)  To optimize the cache media utilization, Data Source Cache supports cache separation of data and index, by using same or different cache media with DRAM and PMem.
- [Cache Hot Tables](#Cache-Hot-Tables)  Data Source Cache also supports caching specific tables according to actual situations, these tables are usually hot tables.
- [Column Vector Cache](#Column-Vector-Cache)  This document above use **binary** cache as example for Parquet file format, if your cluster memory resources is abundant enough, you can choose ColumnVector data cache instead of binary cache for Parquet to spare computation time.
- [Large Scale and Heterogeneous Cluster Support](#Large-Scale-and-Heterogeneous-Cluster-Support) Introduce an external database to store cache locality info to support large-scale and heterogeneous clusters.
## Additional Cache Strategies
Following table shows features of 4 cache strategies on PMem.

| guava | noevict | vmemcache | external cache |
| :----- | :----- | :----- | :-----|
| Use memkind lib to operate on PMem and guava cache strategy when data eviction happens. | Use memkind lib to operate on PMem and doesn't allow data eviction. | Use vmemcache lib to operate on PMem and LRU cache strategy when data eviction happens. | Use Plasma/dlmalloc to operate on PMem and LRU cache strategy when data eviction happens. |
| Need numa patch in Spark for better performance. | Need numa patch in Spark for better performance. | Need numa patch in Spark for better performance. | Doesn't need numa patch. |
| Suggest using 2 executors one node to keep aligned with PMem paths and numa nodes number. | Suggest using 2 executors one node to keep aligned with PMem paths and numa nodes number. | Suggest using 2 executors one node to keep aligned with PMem paths and numa nodes number. | Node-level cache so there are no limitation for executor number. |
| Cache data cleaned once executors exited. | Cache data cleaned once executors exited. | Cache data cleaned once executors exited. | No data loss when executors exit thus is friendly to dynamic allocation. But currently it has performance overhead than other cache solutions. |

- For cache solution `guava/noevict`, make sure [Memkind](https://github.com/memkind/memkind/tree/v1.10.1-rc2) library installed on every cluster worker node. If you have finished [OAP Installation Guide](../../../docs/OAP-Installation-Guide.md), libmemkind will be installed. Or manually build and install it following [build and install memkind](./Developer-Guide.md#build-and-install-memkind), then place `libmemkind.so.0` under `/lib64/` on each worker node.

- For cache solution `vmemcahe/external` cache, make sure [Vmemcache](https://github.com/pmem/vmemcache) library has been installed on every cluster worker node. If you have finished [OAP Installation Guide](../../../docs/OAP-Installation-Guide.md), libvmemcache will be installed. Or you can follow the [build/install](./Developer-Guide.md#build-and-install-vmemcache) steps and make sure `libvmemcache.so.0` exist under `/lib64/` directory on each worker node.

- Data Source Cache use Plasma as a node-level external cache service, the benefit of using external cache is data could be shared across process boundaries.  [Plasma](http://arrow.apache.org/blog/2017/08/08/plasma-in-memory-object-store/) is a high-performance shared-memory object store, it's a component of [Apache Arrow](https://github.com/apache/arrow). We have modified Plasma to support PMem, and open source on [Intel-bigdata Arrow](https://github.com/Intel-bigdata/arrow/tree/branch-0.17.0-oap-0.9) repo. Build and install step can refer to [build and install plasma](./Developer-Guide.md#build-and-install-plasma). If you have finished [OAP Installation Guide](../../../docs/OAP-Installation-Guide.md), Plasma will be automatically installed.

If you have followed [OAP Installation Guide](../../../docs/OAP-Installation-Guide.md), ***Memkind*** ,***Vmemcache*** and ***Plasma*** will be automatically installed. 
Or you can refer to [Developer-Guide](../../../docs/Developer-Guide.md), there is a shell script to help you install these dependencies automatically.

### Guava cache

Guava cache is based on memkind library, built on top of jemalloc and provides memory characteristics. To use it in your workload, follow [prerequisites](#prerequisites-1) to set up PMem hardware correctly, also make sure memkind library installed. Then follow configurations below.

**NOTE**: `spark.executor.sql.oap.cache.persistent.memory.reserved.size`: When we use PMem as memory through memkind library, some portion of the space needs to be reserved for memory management overhead, such as memory segmentation. We suggest reserving 20% - 25% of the available PMem capacity to avoid memory allocation failure. But even with an allocation failure, OAP will continue the operation to read data from original input data and will not cache the data block.

For Parquet file format, add these conf options:
```
# enable numa
spark.yarn.numa.enabled                           true
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND      1
spark.sql.oap.parquet.binary.cache.enabled        true
spark.sql.oap.cache.memory.manager                pm 
spark.oap.cache.strategy                          guava
# PMem capacity per executor, according to your cluster
spark.executor.sql.oap.cache.persistent.memory.initial.size    256g
# Reserved space per executor
spark.executor.sql.oap.cache.persistent.memory.reserved.size   50g
spark.sql.extensions                              org.apache.spark.sql.OapExtensions
```
For Orc file format, add these conf options:
```
# enable numa
spark.yarn.numa.enabled                           true
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND      1
spark.sql.oap.orc.binary.cache.enabled            true
spark.sql.oap.orc.enabled                         true
spark.sql.oap.cache.memory.manager                pm 
spark.oap.cache.strategy                          guava
# PMem capacity per executor, according to your cluster
spark.executor.sql.oap.cache.persistent.memory.initial.size    256g
# Reserved space per executor
spark.executor.sql.oap.cache.persistent.memory.reserved.size   50g
spark.sql.extensions                             org.apache.spark.sql.OapExtensions
```

Memkind library also support DAX KMEM mode. Refer [Kernel](https://github.com/memkind/memkind#kernel), this chapter will guide how to configure persistent memory as system ram. Or [Memkind support for KMEM DAX option](https://pmem.io/2020/01/20/memkind-dax-kmem.html) for more details.

Please note that DAX KMEM mode need kernel version 5.x and memkind version 1.10 or above. If you choose KMEM mode, change memory manager from `pm` to `kmem` as below.
```
spark.sql.oap.cache.memory.manager           kmem
```

### Noevict cache

The noevict cache strategy is also supported in OAP based on the memkind library for PMem.

To apply noevict cache strategy in your workload, please follow [prerequisites](#prerequisites-1) to set up PMem hardware correctly, also make sure memkind library installed. Then follow configurations below.

For Parquet file format, add these conf options:
```
# enable numa
spark.yarn.numa.enabled                                  true
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND             1
spark.sql.oap.parquet.binary.cache.enabled               true 
spark.oap.cache.strategy                                 noevict 
spark.executor.sql.oap.cache.persistent.memory.initial.size  256g 
```
For Orc file format, add these conf options:
```
# enable numa
spark.yarn.numa.enabled                                  true
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND             1
spark.sql.oap.orc.binary.cache.enabled                   true 
spark.oap.cache.strategy                                 noevict 
spark.executor.sql.oap.cache.persistent.memory.initial.size  256g 
```

### External cache using plasma


External cache strategy is implemented based on arrow/plasma library. For performance reason, we recommend using numa-patched Spark-2.4.4. 

To use this strategy, follow [prerequisites](#prerequisites-1) to set up PMem hardware. 

Besides, with BIOS configuration settings below, PMem could get noticeable performance gain, especially on cross socket write path.

```
Socket Configuration -> Memory Configuration -> NGN Configuration -> Snoopy mode for AD : enabled
Socket configuration -> Intel UPI General configuration -> Stale Atos :  Disabled
``` 


It's strongly advised to use [Linux device mapper](https://pmem.io/2018/05/15/using_persistent_memory_devices_with_the_linux_device_mapper.html) to interleave PMem across sockets and get maximum size for Plasma.

You can follow these commands to create or destroy interleaved PMem device:

```
# create interleaved PMem device
umount /mnt/pmem0
umount /mnt/pmem1
echo -e "0 $(( `sudo blockdev --getsz /dev/pmem0` + `sudo blockdev --getsz /dev/pmem0` )) striped 2 4096 /dev/pmem0 0 /dev/pmem1 0" | sudo dmsetup create striped-pmem
mkfs.ext4 -b 4096 -E stride=512 -F /dev/mapper/striped-pmem
mkdir -p /mnt/pmem
mount -o dax /dev/mapper/striped-pmem /mnt/pmem

# destroy interleaved PMem device
umount /mnt/pmem
dmsetup remove striped-pmem
mkfs.ext4 /dev/pmem0
mkfs.ext4 /dev/pmem1
mount -o dax /dev/pmem0 /mnt/pmem0
mount -o dax /dev/pmem1 /mnt/pmem1
```
Then copy `arrow-plasma-0.17.0.jar` to your ***$SPARK_HOME/jars*** directory. Refer to configurations below to apply external cache strategy and start plasma service on each node and start your workload. (Currently web UI cannot display accurately, this is a known [issue](https://github.com/Intel-bigdata/OAP/issues/1579))

For Parquet data format, add these conf options:


```
spark.sql.oap.parquet.binary.cache.enabled                 true 
spark.oap.cache.strategy                                   external
spark.sql.oap.dcpmm.free.wait.threshold                    50000000000
# according to your executor core number
spark.executor.sql.oap.cache.external.client.pool.size     10
# absolute path of the jar on your working node
spark.files                       $HOME/miniconda2/envs/oapenv/oap_jars/oap-cache-<version>-with-spark-<version>.jar,$HOME/miniconda2/envs/oapenv/oap_jars/arrow-plasma-0.17.0.jar,$HOME/miniconda2/envs/oapenv/oap_jars/oap-common-<version>-with-spark-<version>.jar
# relative path of the jar
spark.executor.extraClassPath     ./oap-cache-<version>-with-spark-<version>.jar:./oap-common-<version>-with-spark-<version>.jar:./arrow-plasma-0.17.0.jar
# absolute path of the jar on your working node
spark.driver.extraClassPath       $HOME/miniconda2/envs/oapenv/oap_jars/oap-cache-<version>-with-spark-<version>.jar:$HOME/miniconda2/envs/oapenv/oap_jars/oap-common-<version>-with-spark-<version>.jar:$HOME/miniconda2/envs/oapenv/oap_jars/arrow-plasma-0.17.0.jar

```

For Orc file format, add these conf options:

```
spark.sql.oap.orc.binary.cache.enabled                     true 
spark.oap.cache.strategy                                   external
spark.sql.oap.dcpmm.free.wait.threshold                    50000000000
# according to your executor core number
spark.executor.sql.oap.cache.external.client.pool.size     10
```

- Start plasma service manually

plasma config parameters:  
 ```
 -m  how much Bytes share memory plasma will use
 -s  Unix Domain sockcet path
 -d  Pmem directory
 ```

You can start plasma service on each node as following command, and then you can run your workload. If you install OAP by Conda, you can find plasma-store-server in the path **$HOME/miniconda2/envs/oapenv/bin/**.

```
./plasma-store-server -m 15000000000 -s /tmp/plasmaStore -d /mnt/pmem  
```

 Remember to kill `plasma-store-server` process if you no longer need cache, and you should delete `/tmp/plasmaStore` which is a Unix domain socket.  
  
- Use yarn to start Plamsa service  
We can use yarn(hadoop version >= 3.1) to start Plasma service, you should provide a json file like following.
```
{
  "name": "plasma-store-service",
  "version": 1,
  "components" :
  [
   {
     "name": "plasma-store-service",
     "number_of_containers": 3,
     "launch_command": "plasma-store-server -m 15000000000 -s /tmp/plasmaStore -d /mnt/pmem",
     "resource": {
       "cpus": 1,
       "memory": 512
     }
   }
  ]
}
```

Run command  ```yarn app -launch plasma-store-service /tmp/plasmaLaunch.json``` to start plasma server.  
Run ```yarn app -stop plasma-store-service``` to stop it.  
Run ```yarn app -destroy plasma-store-service```to destroy it.

## Index and Data Cache Separation

Data Source Cache now supports different cache strategies for DRAM and PMem. To optimize the cache media utilization, you can enable cache separation of data and index with same or different cache media. When Sharing same media, data cache and index cache will use different fiber cache ratio.


Here we list 4 different kinds of configs for index/cache separation, if you choose one of them, please add corresponding configs to `spark-defaults.conf`.
1. DRAM as cache media, `guava` strategy as index & data cache backend. 

```
spark.sql.oap.index.data.cache.separation.enabled       true
spark.oap.cache.strategy                                mix
spark.sql.oap.cache.memory.manager                      offheap
```
The rest configurations can refer to the configurations of  [Use DRAM Cache](#use-dram-cache) 

2. PMem as cache media, `vmem` strategy as index & data cache backend. 

```
spark.sql.oap.index.data.cache.separation.enabled       true
spark.oap.cache.strategy                                mix
spark.sql.oap.cache.memory.manager                      tmp
spark.sql.oap.mix.data.cache.backend                    vmem
spark.sql.oap.mix.index.cache.backend                   vmem

```
The rest configurations can refer to the configurations of [PMem Cache](#use-pmem-cache) and  [Vmemcache cache](#configure-to-enable-pmem-cache)

3. DRAM(`offheap`)/`guava` as `index` cache media and backend, PMem(`tmp`)/`vmem` as `data` cache media and backend. 

```
spark.sql.oap.index.data.cache.separation.enabled            true
spark.oap.cache.strategy                                     mix
spark.sql.oap.cache.memory.manager                           mix 
spark.sql.oap.mix.data.cache.backend                         vmem

# 2x number of your worker nodes
spark.executor.instances                                     6
# enable numa
spark.yarn.numa.enabled                                      true
spark.memory.offHeap.enabled                                 false
# PMem capacity per executor
spark.executor.sql.oap.cache.persistent.memory.initial.size  256g
# according to your cluster
spark.executor.sql.oap.cache.guardian.memory.size            10g

# equal to the size of executor.memoryOverhead
spark.executor.sql.oap.cache.offheap.memory.size             50g
# according to the resource of cluster
spark.executor.memoryOverhead                                50g

# for orc file format
spark.sql.oap.orc.binary.cache.enabled                       true
# for Parquet file format
spark.sql.oap.parquet.binary.cache.enabled                   true
```

4. DRAM(`offheap`)/`guava` as `index` cache media and backend, PMem(`pm`)/`guava` as `data` cache media and backend. 

```
spark.sql.oap.index.data.cache.separation.enabled            true
spark.oap.cache.strategy                                     mix
spark.sql.oap.cache.memory.manager                           mix 

# 2x number of your worker nodes
spark.executor.instances                                     6
# enable numa
spark.yarn.numa.enabled                                      true
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND                 1
spark.memory.offHeap.enabled                                 false
# PMem capacity per executor
spark.executor.sql.oap.cache.persistent.memory.initial.size  256g
# Reserved space per executor
spark.executor.sql.oap.cache.persistent.memory.reserved.size 50g

# equal to the size of executor.memoryOverhead
spark.executor.sql.oap.cache.offheap.memory.size             50g
# according to the resource of cluster
spark.executor.memoryOverhead                                50g
# for ORC file format
spark.sql.oap.orc.binary.cache.enabled                       true
# for Parquet file format
spark.sql.oap.parquet.binary.cache.enabled                   true
```

## Cache Hot Tables

Data Source Cache also supports caching specific tables by configuring items according to actual situations, these tables are usually hot tables.

To enable caching specific hot tables, you can add below configurations to `spark-defaults.conf`.
```
# enable table lists fiberCache
spark.sql.oap.cache.table.list.enabled          true
# Table lists using fiberCache actively
spark.sql.oap.cache.table.list                  <databasename>.<tablename1>;<databasename>.<tablename2>
```

## Column Vector Cache

This document above use **binary** cache for Parquet as example, cause binary cache can improve cache space utilization compared to ColumnVector cache. When your cluster memory resources are abundant enough, you can choose ColumnVector cache to spare computation time. 

To enable ColumnVector data cache for Parquet file format, you should add below configurations to `spark-defaults.conf`.

```
# for parquet file format, disable binary cache
spark.sql.oap.parquet.binary.cache.enabled             false
# for parquet file format, enable ColumnVector cache
spark.sql.oap.parquet.data.cache.enabled               true
```

## Large Scale and Heterogeneous Cluster Support

***NOTE:*** Only works with `external cache`

OAP influences Spark to schedule tasks according to cache locality info. This info could be of large amount in a ***large scale cluster***, and how to schedule tasks in a ***heterogeneous cluster*** (some nodes with PMem, some without) could also be challenging.

We introduce an external DB to store cache locality info. If there's no cache available, Spark will fall back to schedule respecting HDFS locality.
Currently we support [Redis](https://redis.io/) as external DB service. Please [download and launch a redis-server](https://redis.io/download) before running Spark with OAP.

Please add the following configurations to `spark-defaults.conf`.
```
spark.sql.oap.external.cache.metaDB.enabled            true
# Redis-server address
spark.sql.oap.external.cache.metaDB.address            10.1.2.12
spark.sql.oap.external.cache.metaDB.impl               org.apache.spark.sql.execution.datasources.RedisClient
```