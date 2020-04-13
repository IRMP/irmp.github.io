# MapReduce
## MapTask的个数取决于切分的文件个数，多个小文件如何优化？
FileInputFormat切片规则
Math.max(minSize,Math.min(maxSize,blockSize))
mapreduce.input.fileinputformat.split.minsize=1
mapreduce.input.fileinputformat.split.maxsize=Long.MAXValue
maxsize:如果调整的比blocksize小，则会让切片变小
minsize：参数调的比blocksize大，则会让切片变大
默认情况切片大小是blocksize，集群模式默认128M，本地模式32M
每次切片都要判断切完剩下的部分是否大于blocksize的1.1倍，不大于就直接划分成一块
针对每一个文件单独切片

使用combineTextInputformat代替默认的TextInputformat对小文件进行合并
## hadoop的序列化
java的序列化（Serializble)过于重量，hadoop自己实现了一套序列化（Writeble）

```java
package com.vic.mapreduce.flowsum;

import org.apache.hadoop.io.Writable;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

public class FlowBean implements Writable {

    private long upFlow;//上行流量
    private long downFlow;//下行流量
    private long sumFlow;//总流量

    public FlowBean() {
        super();
    }

    public FlowBean(long upFlow, long downFlow) {
        this.upFlow = upFlow;
        this.downFlow = downFlow;
        this.sumFlow = upFlow + downFlow;
    }

    /**
     * Serialize the fields of this object to <code>out</code>.
     *
     * @param out <code>DataOuput</code> to serialize this object into.
     * @throws IOException
     */
    @Override
    public void write(DataOutput out) throws IOException {
        out.writeLong(upFlow);
        out.writeLong(downFlow);
        out.writeLong(sumFlow);
    }

    /**
     * Deserialize the fields of this object from <code>in</code>.
     *
     * <p>For efficiency, implementations should attempt to re-use storage in the
     * existing object where possible.</p>
     *
     * @param in <code>DataInput</code> to deseriablize this object from.
     * @throws IOException
     */
    @Override
    public void readFields(DataInput in) throws IOException {
        //注意这里需要和序列号顺序一致
        upFlow = in.readLong();
        downFlow = in.readLong();
        sumFlow = in.readLong();
    }

    public long getUpFlow() {
        return upFlow;
    }

    public void setUpFlow(long upFlow) {
        this.upFlow = upFlow;
    }

    public long getDownFlow() {
        return downFlow;
    }

    public void setDownFlow(long downFlow) {
        this.downFlow = downFlow;
    }

    public long getSumFlow() {
        return sumFlow;
    }

    public void setSumFlow(long sumFlow) {
        this.sumFlow = sumFlow;
    }

    @Override
    public String toString() {
        return "FlowBean{" +
                "upFlow=" + upFlow +
                ", downFlow=" + downFlow +
                ", sumFlow=" + sumFlow +
                '}';
    }
}

```

## KeyValueTextInputFormat

```java
job.setInputFormatClass(KeyValueTextInputFormat.class);
conf.set(KeyValueLineRecordReader.KEY_VALUE_SEPERATOR, " ");//默认是\t
```

## NLineInputFormat

```java
//mapper keyIn LongWritable
NLineInputFormat.setNumLinesPerSplit(job,3);//3行一切分
job.setInputFormatClass(NLineInputFormat.class);
```

## 自定义InputFormat
1. 自定义一个类继承FileInputFormat

```java

package com.vic.mapreduce.inputformat;

import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.BytesWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.InputSplit;
import org.apache.hadoop.mapreduce.JobContext;
import org.apache.hadoop.mapreduce.RecordReader;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import java.io.IOException;

public class WholeFileInputFormat extends FileInputFormat<Text, BytesWritable> {
    /**
     * Is the given filename splitable? Usually, true, but if the file is
     * stream compressed, it will not be.
     *
     * <code>FileInputFormat</code> implementations can override this and return
     * <code>false</code> to ensure that individual input files are never split-up
     * so that {@link Mapper}s process entire files.
     *
     * @param context  the job context
     * @param filename the file name to check
     * @return is this file splitable?
     */
    @Override
    protected boolean isSplitable(JobContext context, Path filename) {
        return false;
    }

    /**
     * Create a record reader for a given split. The framework will call
     * {@link RecordReader#initialize(InputSplit, TaskAttemptContext)} before
     * the split is used.
     *
     * @param split   the split to be read
     * @param context the information about the task
     * @return a new record reader
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    public RecordReader<Text, BytesWritable> createRecordReader(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {
        WholeRecordReader recordReader = new WholeRecordReader();
        recordReader.initialize(split, context);
        return recordReader;
    }
}
```
2. 改写RecordReader

```java
package com.vic.mapreduce.inputformat;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.BytesWritable;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.InputSplit;
import org.apache.hadoop.mapreduce.RecordReader;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import java.io.IOException;

public class WholeRecordReader extends RecordReader<Text, BytesWritable> {
    Configuration conf;
    FileSplit fileSplit;
    Text currentKey = new Text();
    BytesWritable currentValue = new BytesWritable();
    boolean isProgress = true;
    /**
     * Called once at initialization.
     *
     * @param split   the split that defines the range of records to read
     * @param context the information about the task
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    public void initialize(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {
        conf = context.getConfiguration();
        fileSplit = (FileSplit) split;
    }

    /**
     * Read the next key, value pair.
     *
     * @return true if a key/value pair was read
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    public boolean nextKeyValue() throws IOException, InterruptedException {
        if (isProgress) {
            //1 定义缓存区
            byte[] buf = new byte[(int) fileSplit.getLength()];
            //2 获取文件系统
            Path path = fileSplit.getPath();
            FileSystem fs = path.getFileSystem(conf);
            //3 读取数据
            FSDataInputStream fis = fs.open(path);
            IOUtils.readFully(fis,buf,0,buf.length);
            //4 输出文件内容
            currentValue.set(buf, 0, buf.length);
            //5 获取文件路径和名称
            currentKey.set(fileSplit.getPath().toString());
            IOUtils.closeStream(fis);
            isProgress = false;
            return true;
        }
        return false;
    }

    /**
     * Get the current key
     *
     * @return the current key or null if there is no current key
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    public Text getCurrentKey() throws IOException, InterruptedException {
        return currentKey;
    }

    /**
     * Get the current value.
     *
     * @return the object that was read
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    public BytesWritable getCurrentValue() throws IOException, InterruptedException {
        return currentValue;
    }

    /**
     * The current progress of the record reader through its data.
     *
     * @return a number between 0.0 and 1.0 that is the fraction of the data read
     * @throws IOException
     * @throws InterruptedException
     */
    @Override
    public float getProgress() throws IOException, InterruptedException {
        return 0;
    }

    /**
     * Close the record reader.
     */
    @Override
    public void close() throws IOException {

    }
}
```
## MapReduce工作流程

![截屏2020-04-02下午7.41.07](https://irmp.github.io/images/mr1.png)
```text
（1）Read阶段：MapTask通过用户编写的RecordReader，从输入InputSplit中解析出一个个key/value。
（2）Map阶段：该节点主要是将解析出的key/value交给用户编写map()函数处理，并产生一系列新的key/value。
（3）Collect收集阶段：在用户编写map()函数中，当数据处理完成后，一般会调用OutputCollector.collect()输出结果。在该函数内部，它会将生成的key/value分区（调用Partitioner），并写入一个环形内存缓冲区中。
（4）Spill阶段：即“溢写”，当环形缓冲区满后，MapReduce会将数据写到本地磁盘上，生成一个临时文件。需要注意的是，将数据写入本地磁盘之前，先要对数据进行一次本地排序，并在必要时对数据进行合并、压缩等操作。
    溢写阶段详情：
    步骤1：利用快速排序算法对缓存区内的数据进行排序，排序方式是，先按照分区编号Partition进行排序，然后按照key进行排序。这样，经过排序后，数据以分区为单位聚集在一起，且同一分区内所有数据按照key有序。
    步骤2：按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out（N表示当前溢写次数）中。如果用户设置了Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作。
    步骤3：将分区数据的元信息写到内存索引数据结构SpillRecord中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过1MB，则将内存索引写到文件output/spillN.out.index中。
（5）Combine阶段：当所有数据处理完成后，MapTask对所有临时文件进行一次合并，以确保最终只会生成一个数据文件。
当所有数据处理完后，MapTask会将所有临时文件合并成一个大文件，并保存到文件output/file.out中，同时生成相应的索引文件output/file.out.index。
在进行文件合并过程中，MapTask以分区为单位进行合并。对于某个分区，它将采用多轮递归合并的方式。每轮合并io.sort.factor（默认10）个文件，并将产生的文件重新加入待合并列表中，对文件排序后，重复以上过程，直到最终得到一个大文件。
让每个MapTask最终只生成一个数据文件，可避免同时打开大量文件和同时读取大量小文件产生的随机读取带来的开销。
```
![截屏2020-04-02下午7.41.26](https://irmp.github.io/images/mr2.png)
```text
（1）Copy阶段：ReduceTask从各个MapTask上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘上，否则直接放到内存中。
（2）Merge阶段：在远程拷贝数据的同时，ReduceTask启动了两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多。
（3）Sort阶段：按照MapReduce语义，用户编写reduce()函数输入数据是按key进行聚集的一组数据。为了将key相同的数据聚在一起，Hadoop采用了基于排序的策略。由于各个MapTask已经实现对自己的处理结果进行了局部排序，因此，ReduceTask只需对所有数据进行一次归并排序即可。
（4）Reduce阶段：reduce()函数将计算结果写到HDFS上。
ReduceTask的并行度同样影响整个Job的执行并发度和执行效率，但与MapTask的并发数由切片数决定不同，ReduceTask数量的决定是可以直接手动设置：
// 默认值是1，手动设置为4
job.setNumReduceTasks(4);
```

### reduce要等待所有maptask结束之后开始，不会出现map没到100%，而reduce不是0%的情况

## partition 分区
### 默认的分区机制 hash
根据key的hashcode对reducetask个数取模得到，用户只能控制分几个区，无法控制哪个key分到哪个区

```java
/** Partition keys by their {@link Object#hashCode()}. */
@InterfaceAudience.Public
@InterfaceStability.Stable
public class HashPartitioner<K, V> extends Partitioner<K, V> {

  /** Use {@link Object#hashCode()} to partition. */
  public int getPartition(K key, V value,
                          int numReduceTasks) {
    return (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
  }

}
```

### 自定义分区

1. 继承Partitioner类

```java
package com.vic.mapreduce.flowsum;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Partitioner;

public class MyPhonePartitioner extends Partitioner<Text, FlowBean> {

    /**
     * Get the partition number for a given key (hence record) given the total
     * number of partitions i.e. number of reduce-tasks for the job.
     *
     * <p>Typically a hash function on a all or a subset of the key.</p>
     *
     * @param k          the key to be partioned.
     * @param v         the entry value.
     * @param numPartitions the total number of partitions.
     * @return the partition number for the <code>key</code>.
     */
    @Override
    public int getPartition(Text k, FlowBean v, int numPartitions) {
        if (k.toString().length() < 3) {
            return 4;
        }
        String partStr = k.toString().substring(0, 3);
        switch (partStr) {
            case "136":
                return 0;
            case "137":
                return 1;
            case "138":
                return 2;
            case "139":
                return 3;
            default:
                return 4;
        }
    }
}
```

2. driver类中增加配置：

```java
//设置自定义分区
job.setPartitionerClass(MyPhonePartitioner.class);
//如果不设置reduceTaskNum，默认为1
job.setNumReduceTasks(5);
```

这里要注意numreducetask的设置
- 如果reducetask设置为1，那结果就是1个分区
- 如果1 < numreducetask < getpartition的分区数，则一部分数据没处放，抛出IO异常（Illeagal partition for XXX)
- 如果NumReduceTask > getpartition的分区数，则会生成一些空文件
- 分区号必须从0开始

## combiner 合并
combiner的父类是reducer
combiner运行在maptask节点
reducer是接收全局所有maptask的结果
combiner的意义是对每一个maptask的结果进行局部汇总，以减小网络传输量
combiner能够应用的前提是不会对最终业务逻辑产生影响（累加可以，求平均值不行）

## GroupingComparator 分组
```text
对Reduce阶段的数据根据某一个或几个字段进行分组。
分组排序步骤：
（1）自定义类继承WritableComparator
（2）重写compare()方法
@Override
public int compare(WritableComparable a, WritableComparable b) {
		// 比较的业务逻辑
		return result;
}
（3）创建一个构造将比较对象的类传给父类
protected OrderGroupingComparator() {
		super(OrderBean.class, true);
}
```
```java
package com.atguigu.mapreduce.order;
import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;

public class OrderGroupingComparator extends WritableComparator {

	protected OrderGroupingComparator() {
		super(OrderBean.class, true);
	}

	@Override
	public int compare(WritableComparable a, WritableComparable b) {

		OrderBean aBean = (OrderBean) a;
		OrderBean bBean = (OrderBean) b;

		int result;
		if (aBean.getOrder_id() > bBean.getOrder_id()) {
			result = 1;
		} else if (aBean.getOrder_id() < bBean.getOrder_id()) {
			result = -1;
		} else {
			result = 0;
		}

		return result;
	}
}
```
```java
// 8 设置reduce端的分组
	job.setGroupingComparatorClass(OrderGroupingComparator.class);
```

## 自定义OutputFormat
- 使用场景
为了实现控制最终文件的输出路径和输出格式

- 自定义步骤
自定义一个类继承FileOutputFormat
改写RecordWriter

## Join
### Reduce端Join
```text
map端的主要工作：对来自不同表的kv对打标签以区别来源，用连接字段作为key，其余部分和新加的标志作为value，最后输出。
reduce端的主要工作：在reduce端的每个分组中区分不通来源的记录，关联更新合并输出
reduce端join会导致reduce压力过大，reducetask一般很少，采用map端join能提高资源利用率
```
### Map端Join
```text
适用于大表关联小表，缓存小表，读大表。
采用DistributedCache，在driver中缓存，在map的setup中读取缓存
// 缓存普通文件到Task运行节点。
job.addCacheFile(new URI("file://e:/cache/pd.txt"));
```
```java
@Override
protected void setup(Mapper<LongWritable, Text, Text, NullWritable>.Context context) throws IOException, InterruptedException {

    // 1 获取缓存的文件
    URI[] cacheFiles = context.getCacheFiles();
    String path = cacheFiles[0].getPath().toString();
    
    BufferedReader reader = new BufferedReader(new InputStreamReader(new FileInputStream(path), "UTF-8"));
}
```