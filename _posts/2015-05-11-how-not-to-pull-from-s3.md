---
title: How NOT to pull from S3 using Apache Spark
subtitle: And how to actually do it
tags: [apache spark, scala, S3, performance]
layout: post
---

### S3 is NOT a file system

Apache Spark comes with the built-in functionality to pull data from S3 as it would with HDFS using the SparContext's `textFiles` method. `textFiles` allows for
glob syntax, which allows you to pull hierarchal data as in `textFiles(s3n://bucket/2015/*/*)`.
Though this seems great at first, there is an underlying issue with treating S3 as a HDFS; that is that S3 is not a file system.

Though it is common to organize your S3 keys with slashes (`/`), and AWS S3 Console will present you said keys is a nice interface if you do, this is actually
misleading. S3 isn't a file system, it is a key-value store. The keys `2015/05/01` and `2015/05/02` do not live in the "same place". They just happen to have
a similar prefix: `2015/05`.

This causes issues when using Apache Spark's `textFiles` since it assumes that anything being put through it behaves like an HDFS.


### The problem

The Kinja Analytics team runs an Apache Spark cluster on AWS EMR continuously. As soon as a EMR step finishes, a job adds the next step. The jobs on the cluster
pull data from S3 (placed there using our event stream), runs multiple computations on that data set and persist the data into a MySQL table. The data in S3 is stored
in chronological order as `bucket/events/$year/$month/$day/$hour/$minute/data.txt`. Each `data.txt` file is about 10MB large.

Originally we were pulling the data using SparkContext's `textFiles` method as such `sc.textFiles(s3n://bucket/events/*/*/*/*/*/*)`. This worked fine at first but
as the dataset grew we noticed that there would always be a large period of inactivity between jobs.

![Large periods of inactivity]({{ site.url }}/assets/kala/cpu_pre_perf1.png)

When the periods were as long as 3 hours, we figured something was wrong. At first we thought that simply adding more machines would solve this
(since that's how you are supposed to speed up Spark/Hadoop), but when that failed, we dug deeper.

### Hints

Using Ganglia graphs, we noticed that during that time only one of the boxes was actually doing any work (which explains why adding more boxes did nothing). This
box was the driver for that given application. We went ahead and looked at the logs for the driver and noticed something peculiar
(NOTE: The logs that EMR places in S3 are behind, so you would need to wait for your application to finish before seeing the complete logs. If you want live logs you need to log into the machine).

~~~
...
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/20 with recursive false
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/21 with recursive false
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/22 with recursive false
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/23 with recursive false
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/00/01 with recursive false
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/00/02 with recursive false
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/00/04 with recursive false
15/05/05 21:43:26 INFO s3n.S3NativeFileSystem: listStatus s3n://kinesis-click-stream-us-east-1-bucket-4myg4if5x7au/events/2015/03/21/00/05 with recursive false
...

~~~

This command, listStatus, seems to be recursively calling each "folder" (each * in the glob syntax we passed into `textFiles`). In a normal file system, that glob syntax is fast enough to not care
but S3 is over the network and each * call does a "get all with prefix" call, which isn't negligible (remember, S3 is a key-value store. `2015/01/01` and `2015/01/02` do not live in the same place).
To make matters worse, all of those calls are done sequentially in the driver before being passed to the workers. This means that as more and more data is inserted into S3, the longer
it will take each job to actually start working.

### The Solution

The solution is quite simple: do not use `textFiles`. Instead use the [AmazonS3Client](https://github.com/aws/aws-sdk-java/blob/master/aws-java-sdk-s3/src/main/java/com/amazonaws/services/s3/AmazonS3Client.java) to manually get every key (maybe with a prefix), then parallelize the data pulling using SparkContext's `parrallelize` method and said AmazonS3Client.

{% highlight scala %}

import com.amazonaws.services.s3._, model._
import com.amazonaws.auth.BasicAWSCredentials

val request = new ListObjectsRequest()
request.setBucketName(bucket)
request.setPrefix(prefix)
request.setMaxKeys(pageLength)
def s3 = new AmazonS3Client(new BasicAWSCredentials(key, secret))

val objs = s3.listObjects(request) // Note that this method returns truncated data if longer than the "pageLength" above. You might need to deal with that.
sc.parallelize(objs.getObjectSummaries.map(_.getKey).toList)
    .flatMap { key => Source.fromInputStream(s3.getObject(bucket, key).getObjectContent).getLines }

{% endhighlight %}

Above, we get all of the keys for a bucket and a prefix (events) and parallelize all of the keys (give them to the workers/partitions) and make each worker pull the data for that one key.

After this change, our "time of inactivity" went down to a couple of minutes!

![After the upgrade]({{ site.url }}/assets/kala/cpu_post_perf.png)

And elapsed times went from 4 hours to 1 hour:

![Step times]({{ site.url }}/assets/kala/steps_time_post_perf.png)
