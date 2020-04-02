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

![截屏2020-04-02下午7.41.26](https://irmp.github.io/images/mr2.png)



### reduce要等待所有maptask结束之后开始，不会出现map没到100%，而reduce不是0%的情况