### Spark master node affinity

RayDP will create a ray actor called `RayDPSparkMaster`, which will then launch the java process, acting like a Master in a tradtional Spark cluster. By default, this actor could be scheduled to any node in the ray cluster. If you want it to be on a particular node, you can assign some custom resources to that node, and request those resources when starting `RayDPSparkMaster` by setting `spark.ray.raydp_spark_master.resource.*` in `init_spark`.

As an example:

```python
raydp.init_spark(...,
configs = {
    # ... other configs
    'spark.ray.raydp_spark_master.resource.CPU': 0,
    'spark.ray.raydp_spark_master.resource.spark_master': 1,  # Force Spark driver related actor run on headnode
})
```

In cluster config yaml:
```yaml
available_node_types:
  ray.head.default:
    resources:
      CPU: 0    # We intentionally set this to 0 so no executor is on headnode
      spark_master: 100  # Just gave it a large enough number so all drivers are there
```

### External Shuffle Service & Dynamic Resource Allocation

RayDP supports External Shuffle Serivce. To enable it, you can either set `spark.shuffle.service.enabled` to `true` in `spark-defaults.conf`, or you can provide a config to `raydp.init_spark`, as shown below:

```python
raydp.init_spark(..., configs={"spark.shuffle.service.enabled": "true"})
```

The user-provided config will overwrite those specified in `spark-defaults.conf`. By default Spark will load `spark-defaults.conf` from `$SPARK_HOME/conf`, you can also modify this location by setting `SPARK_CONF_DIR`.

Similarly, you can also enable Dynamic Executor Allocation this way. However, currently you must use Dynamic Executor Allocation with data persistence. You can write the data frame in spark to HDFS as a parquet as shown below:

```python
ds = RayMLDataset.from_spark(..., fs_directory="hdfs://host:port/your/directory")
```

### RayDP Executor Extra ClassPath 

As raydp starts the java executors, the classpath will contain the absolute path of ray, raydp and spark by default. When you run ray cluster on yarn([Deploying on Yarn](https://docs.ray.io/en/latest/cluster/yarn.html)), the jar files are stored on HDFS, which have different absolute path in each node. In such cases,  jvm cannot find the main class and ray workers will fail to start. 

To solve such problems, users can specify extra classpath when` init _spark` by configuring `raydp.executor.extraClassPath`. Make sure your jar files are distributed to the same path(s) on all nodes of the ray cluster.

```python
raydp.init_spark(..., configs={"raydp.executor.extraClassPath": "/your/extra/jar/path:/another/path"})
```

### Spark Submit

RayDP provides a substitute for spark-submit in Apache Spark. You can run your java or scala application on RayDP cluster by using `bin/raydp-submit`. You can add it to `PATH` for convenience. When using `raydp-submit`, you should specify number of executors, number of cores and memory each executor by Spark properties, such as `--conf spark.executor.cores=1`, `--conf spark.executor.instances=1` and `--conf spark.executor.memory=500m`. `raydp-submit` only supports Ray cluster. Spark standalone, Apache Mesos, Apache Yarn are not supported, please use traditional `spark-submit` in that case. Besides, RayDP does not support cluster as deploy-mode.

Here is an example:

1. To use `raydp-submit`, you need to start your ray cluster in advance. Let's say your ray address is `1.2.3.4:6379`
2. You should use a ray config file to provide your ray cluster configuration to `raydp-submit`. You can create it using this script: 
```python
ray.init(address="auto")
node = ray.worker.global_worker.node
options = {}
options["ray"] = {}
options["ray"]["run-mode"] = "CLUSTER"
options["ray"]["node-ip"] = node.node_ip_address
options["ray"]["address"] = node.address
options["ray"]["session-dir"] = node.get_session_dir_path()

ray.shutdown()
conf_path = "ray.conf"
with open(conf_path, "w") as f:
    json.dump(options, f)
 ```
 The file should look like this:
 ```json
 {
    "ray": {
        "run-mode": "CLUSTER",
        "node-ip": "1.2.3.4",
        "address": "1.2.3.4:6379",
        "session-dir": "/tmp/ray/session_xxxxxx"
    }
  }
 ```
 3. Run your application, such as `raydp-submit --ray-conf /path/to/ray.conf --class org.apache.spark.examples.SparkPi --conf spark.executor.cores=1 --conf spark.executor.instances=1 --conf spark.executor.memory=500m $SPARK_HOME/examples/jars/spark-examples.jar`. Note that `--ray-conf` must be specified right after raydp-submit, and before any spark arguments.

### Placement Group
RayDP can leverage Ray's placement group feature and schedule executors onto spcecified placement group. It provides better control over the allocation of Spark executors on a Ray cluster, for example spreading the spark executors onto seperate nodes or starting all executors on a single node. You can specify a created placement group when init spark, as shown below:

```python
raydp.init_spark(..., placement_group=pg)
```

Or you can just specify the placement group strategy. RayDP will create a coreesponding placement group and manage its lifecycle, which means the placement group will be created together with SparkSession and removed when calling `raydp.stop_spark()`. Strategy can be "PACK", "SPREAD", "STRICT_PACK" or "STRICT_SPREAD". Please refer to [Placement Groups document](https://docs.ray.io/en/latest/placement-group.html#pgroup-strategy) for details.

```python
raydp.init_spark(..., placement_group_strategy="SPREAD")
```

### Ray Client

RayDP works the same way when using ray client. However, spark driver would be on the local machine. This is convenient if you want to do some experiment in an interactive environment. If this is not desired, e.g. due to performance, you can define an ray actor, which calls `init_spark` and performs all the calculation in its method. This way, spark driver will be in the ray cluster, and is rather similar to spark cluster deploy mode.

### RayDP Hive Support
RayDP can read or write Hive, which might be useful if the data is stored in HDFS.If you want to enable this feature, please configure your environment as following:
+ Install spark to ray cluster's each node and set ENV SPARK_HOME
+ COPY your hdfs-site.xml and hive-site.xml to $SPARK_HOME/conf. If using hostname in your xml file, make sure /etc/hosts is set properly
+ Test: You can test if Hive configuration is successful like this
```python
from pyspark.sql.session import SparkSession
spark = SparkSession.builder().enableHiveSupport()
spark.sql("select * from db.xxx").show()  # db is database, xxx is exists table
```
RayDP using Hive example
```python
ray.init("auto")
spark = raydp.init_spark(...,enable_hive=True)
spark.sql("select * from db.xxx").show()
```


### Autoscaling
You can use RayDP with Ray autoscaling. When you call `raydp.init_spark`, the autoscaler will try to increase the number of worker nodes if the current capacity of the cluster can't meet the resource demands. However currently there is a known issue [#20476](https://github.com/ray-project/ray/issues/20476) in Ray autoscaling. The autoscaler's default strategy is to avoid launching GPU nodes if there aren't any GPU tasks at all. So if you configure a single worker node type with GPU, by default the autoscaler will not launch nodes to start Spark executors on them. To resolve the issue, you can either set the environment variable "AUTOSCALER_CONSERVE_GPU_NODES" to 0 or configure multiple node types that at least one is CPU only node. 


### Logging
+ Driver Log: By default, the spark driver log level is WARN. After getting a Spark session by running `spark = raydp.init_spark`, you can change the log level for example `spark.sparkContext.setLogLevel("INFO")`. You will also see some AppMaster INFO logs on the driver. This is because Ray redirects the actor logs to driver by default. To disable logging to driver, you can set it in Ray init `ray.init(log_to_driver=False)`
+ Executor Log: The spark executor logs are stored in Ray's logging directory. By default they are available at /tmp/ray/session_\*/logs/java-worker-\*.log
