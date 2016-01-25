---
layout: post
title:  "Apache Kudu as a More Flexible And Reliable Kafka-style Queue"
date:   2016-01-24 13:16:00
---

Howdy friends!

One of the more exciting recent developments in data processing is the rise of high-throughput message queues. Systems like Apache Kafka allow you to share data between different parts of your infrastructure: for example, changes in your production database can be replicated easily and reliably to your full-text search index and your analytics data warehouse, keeping everything in sync. If you're interested in more detail of how this all works, [this tome](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) is your jam.

The key advantage is that these systems have enough throughput that you can just replicate everything and let each consumer filter down the data its interested in. But in achieving this goal, Kafka has made some tradeoffs that optimize for throughput over flexibility and availability. This means that many use cases, such as maintaining a queue per-user or modifying queue elements after enqueue, are hard or impossible to implement using Kafka.

In this blog post, I show how Kudu, a new key-value store, can be made to function as a more flexible queueing system with nearly as high throughput as Kafka.

**Contents:**

* TOC
{:toc}

### What's Apache Kafka?
![Kafka logo](/img/kafka_logo.png)

[Apache Kafka](http://kafka.apache.org/) is a message queue implemented as a distributed commit log. There's a good [introduction](http://kafka.apache.org/documentation.html#introduction) on their site, but the takeaway is that from the user's point of view, you log a bunch of messages (Kafka can handle millions/second/queue), and Kafka holds on to them while one or more consumers process them. Unlike traditional message queues, Kafka pushes the responsibility of keeping track of which messages have been read to the reader, which allows for extremely high throughput and low overhead on the message broker. Kafka has quickly become indispensable for most internet-scale companies' analytics platform.

### What's Apache Kudu?
![Kudu logo](/img/kudu_logo_small.png)

[Apache Kudu (incubating)](http://getkudu.io/) is a new sorted key-value store. Its interface is similar to [Google Bigtable](https://cloud.google.com/bigtable/docs/), [Apache HBase](https://hbase.apache.org/), or [Apache Cassandra](https://cassandra.apache.org/). Like those systems, Kudu allows you to distribute the data over many machines and disks to improve availability and performance. Unlike Bigtable and HBase, Kudu layers directly on top of the local filesystem rather than GFS/HDFS. Unlike Cassandra, Kudu implements the [Raft consensus algorithm](https://raft.github.io/) to ensure full consistency between replicas. And unlike all those systems, Kudu uses a new compaction algorithm that's aimed at bounding compaction time rather than minimizing the numbers of files on disk.

### Kudu as a Kafka replacement

So the proposal is to make Kudu (a sorted key-value store) act like Kafka (a messaging queue). Why would any arguably sane person want to do this? Those of you who aren't familiar with the space but have gamely made it this far are likely wondering what the payoff might be, while those are more familiar are probably remembering posts like [this one](http://www.datastax.com/dev/blog/cassandra-anti-patterns-queues-and-queue-like-datasets) expressly suggesting that doing this is a bad idea.

Firstly, there are two advantages Kudu can provide as a queue compared to Kafka:

#### Advantage 1: Availability

![a quorum of Kudus (source http://bit.ly/1nghnwf )](/img/kudu_quorum.jpg)

<sup>A quorum of Kudus (source http://bit.ly/1nghnwf )</sup>

Kafka has replication between brokers, but by default it's asynchronous. The system that it uses to ensure consistency is homegrown and suffers from the problems endemic to achieving consistency on top of an asynchronous replication system. If you're familiar with the different iterations of traditional RDBMS replication systems, the challenges will sound familiar. Comparatively, Kudu's replication is based on the Raft consensus algorithm, which guarantees that as long as you have enough nodes to form a quorum, you'll be able to satisfy reads and writes within a bounded amount of time and a guarantee of a consistent view of the data.

#### Advantage 2: Flexibility
![an inflexible Kudu (source http://bit.ly/1WIxNtL )](/img/kudu_statue.jpg)

<sup>An inflexible Kudu (source http://bit.ly/1WIxNtL )</sup>

There are many use cases that initially seem like a good fit for Kafka, but that need flexibility that Kafka doesn't provide. A key-value store like Kudu gives more options.

For instance, some applications require processing of infrequent events that require queue modification, which Kafka doesn't provide:

 * A Twitter firehose-style app needs to be able to process Tweet deletions/edits.

 * A system for storing RDBMS replication logs needs to be able to handle replica promotions, which sometimes requires rewriting history due to lost events.

Many other applications require many independent queues (e.g. one per domain or even one per user).

In practice, Kafka suffers significant bottlenecks as the number of queues increase: leader reassignment latency increases dramatically, memory requirements increase significantly, and many of the metadata APIs become noticeably slower. As the queues become smaller and more numerous, the workload begins to resemble a plain key-value store.

If Kudu can be made to work well for the queue workload, it can bridge these use cases. And indeed, [Instagram](http://www.slideshare.net/planetcassandra/cassandra-summit-2014-cassandra-at-instagram-2014), [Box](http://www.slideshare.net/HBaseCon/dev-session-5a), and others have used HBase or Cassandra for this workload, despite having serious performance penalties compared to Kafka (e.g. Box [reports](http://www.slideshare.net/HBaseCon/dev-session-5a) a peak of only ~1.5K writes/second/node in their presentation and Instagram has given multiple talks about the Cassandra-tunning heroics required).

#### Reasons disadvantages don't render this insane

Happily, Kudu doesn't have many of the disadvantages of other sorted key-value stores when it comes to queue-based workloads:

* Unlike Cassandra, Kudu doesn't require a lengthy tombstone period holding onto deleted queue entries.

* Unlike Bigtable and HBase, Kudu doesn't have to optimize for large files suitable for storing in GFS-derived filesystem. Its compaction strategy (read section 4.10 of [of the Kudu paper](http://getkudu.io/kudu.pdf) for the gory details), decreases compaction-related overhead for this use case from scaling with the superlinearly with the amount of data put in the queue.

### The mechanics of making Kudu act like a Kafka-style message queue

At a high level, we can think of Kafka as a system that has two endpoints:

 * `enqueueMessages(String queue, List<String> message)`
 * `(List<String> messages, int nextPosition) <- retrieveNextMessages(String queue, long position)`

for which `enqueueMessage` is high throughput and `retrieveNextMessage` is high throughput as long as its called with sequential positions.

Behind the scenes, `enqueueMessages` needs to assign each message passed in a position and store it in a format where `retrieveNextMessages` can iterate over the messages in a given queue efficiently.

This API suggests that we want our Kudu primary key to be something like `(queue STRING, position INT)`. But how do we find the next position for `enqueueMessage`? If we require the producers submitting the messages to find their own position, it would both be an annoying API to use and likely a performance bottleneck. Luckily, the outcome of the Raft consensus algorithm is a fully ordered log, which gives each message a unique position in its queue.

Specifically, in Kudu's implementation of Raft, the position is encoded as an `OpId`, which is a tuple of `(term, index)`, where `term` is incremented each time a new consensus leader is elected and `index` is incremented each time. There's an additional wrinkle in that each Raft entry can hold multiple row inserts, so we need to add a third int called `offset` to add an arbitrary but consistent ordering within each Raft entry. This means that `(term, index, offset)` monotonically increases with each element and we can use it as our position.

There's an implementation challenge here: the position isn't computed until after we've done the Raft round, but we need to give each message a primary key before we submit the entry to Raft. Fortunately, since the Raft log produced is the same on each replica, we can get around this by submitting the entry with a placeholder value and then substituting in the position whenever we read the entry from the consensus log. The full change is [here](https://github.com/frew/kudu/commit/c3009733d3c977f2f26888079459446836098efa).

### Results

I first tested the change by instrumenting Kudu and the client and ensuring that the position generated was correctly generated.

Then I wanted to check to see if the expected theoretical performance was in practice.

I tested the change on a cluster of 5x Google Compute Engine [n1-standard-4](https://cloud.google.com/compute/docs/machine-types#standard_machine_types) instances. Each had a [1 TB standard persistent HDD](https://cloud.google.com/compute/docs/disks/#comparison_of_disk_types) shared between both the logs and the tablets.  I built Kudu from [this revision](https://github.com/frew/kudu/tree/aad51ba69f1b3250155e8977b24fbdce7cc03aea). I ran the master on one machine, the client on another, and tablet servers on the remaining three nodes.

It's important to note that this is not a particularly optimized setup nor is it a particularly close comparison to [publicly available Kafka benchmarks](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines). However, it should give us a ballpark estimate of whether this is workable.

I used a 100 byte string value with this schema:
{% highlight cpp %}
KuduSchema schema;
KuduSchemaBuilder b;
b.AddColumn("queue")->Type(KuduColumnSchema::INT32)->NotNull();
b.AddColumn("op_id_term")->Type(KuduColumnSchema::INT64)->NotNull();
b.AddColumn("op_id_index")->Type(KuduColumnSchema::INT64)->NotNull();
b.AddColumn("op_id_offset")->Type(KuduColumnSchema::INT32)->NotNull();
b.AddColumn("val")->Type(KuduColumnSchema::STRING)->NotNull();
vector<string> keys;
keys.push_back("queue");
keys.push_back("op_id_term");
keys.push_back("op_id_index");
keys.push_back("op_id_offset");
b.SetPrimaryKey(keys);
{% endhighlight %}

The whole client harness is available [here](https://github.com/frew/kudu/blob/aad51ba69f1b3250155e8977b24fbdce7cc03aea/src/kudu/client/samples/sample.cc).

The results are encouraging. Inserting 16M rows in 1024 row batches sharded across 6 queues took 104 seconds for a total of 161k rows/second.

Given that this is unoptimized and with 1/6th the hard drive bandwidth of the Kafka benchmark, it's definitely in the same realm as Kafka's 2 million row/second result.

### Reasons you might not want to do this in production just yet/Further work

First off, the change above isn't committed to Apache Kudu. It breaks the abstraction between the consensus layer and the key-value storage layer. That's a tradeoff that's probably only worth making if this is something that people actively want. If you have a matching use case and you'd be interested in trying out Kudu for it, I'd love to hear from you: [email me](mailto:frew@cs.stanford.edu).

Secondly, Apache Kudu is in beta. Most relevantly, it doesn't have multi-master support yet, so there is a single point of failure in the system. Currently, the team's best guess is that this support is coming [late summer](https://getkudu.slack.com/archives/kudu-general/p1453255802003306), but as in all software development, that's subject to change.

Beyond that, there's some support work that needs to be done before this is something I'd be looking to deploy in production:

* Currently the bottleneck appears to be network bandwidth between two nodes in the cluster: ![chart](/img/bytes_sent.png) vs. an iperf result between the two nodes of 800MB/s with no other resources maxed out. This is likely to be an optimization opportunity for the replication protocol.

* More generally, the performance testing so far suggests that Kudu is in the same neighborhood as Kafka, but much more rigorous testing would be needed to show the full picture. Since reads and writes are competing for many of the same resources, a full performance test would need to compare read/write workloads. Also, for the smaller queue use case, the test would need to parameterize on Kudu partition sizes as well as number of queues.

* It would probably make sense to add a long polling scan call to Kudu to decrease latency while tailing a queue.

I'm interested in working on this, but don't have a compelling production use case for it at present. Interested in this for your use case? Convinced it won't work? [Email](mailto:frew@cs.stanford.edu), comment, or [tweet](https://twitter.com/fredwulff), as is your wont.
