---
layout: post
title: "Benchmarking hosted MongoDB services"
date: 2013-02-26 08:00
comments: false
author: Paul Querna
published: false
categories: 
- Cloud Databases
- ObjectRocket
- MongoDB
---


When we started investigating the hosted MongoDB space, we quickly found that most of the companies involved were just hosting MongoDB on top of AWS instances. We were intrigued by the different approach taken by ObjectRocket.  Instead of using AWS primitives, they built their service on their own hardware in neighboring data centers, and utilized AWS DirectConnect to provide low latency connectivity.

In order to validate that their architectural choices made a difference, Rackspace conducted tests comparing ObjectRocket, MongoHQ and MongoLab.  As with any benchmark, sticks can be thrown, but we believe this represents a good baseline of the performance differences between the vendors.
<!--More-->

## Setup

For each providers we used their $150 per month price point configurations, and performed benchmarks against that configuration. Below are the setups used for each provider:

* [ObjectRocket](http://www.objectrocket.com/): $149/month, Standard, 5gb, in the us-east zone.
* [MongoHQ](http://www.mongohq.com/): $149/month, "Replica Set: Small", 5gb, in AWS us-east-1
* [MongoLab](https://mongolab.com/): $160/month, AWS Dedicated "Mini", 1.7gb RAM, 20gb Storage, in AWS us-east-1.

For the benchmark we leveraged the [YCSB tool](https://github.com/brianfrankcooper/YCSB/) open sourced by Yahoo Research. In order to use the latest MongoDB Java SDK we did make [small modifications to YCSB itself](https://github.com/brianfrankcooper/YCSB/pull/112).  While YCSB has received criticism when used to benchmark different backends, since we were only using the MongoDB backend, we were less concerned about this criticism.

We built a dataset of 2.5 million records, with a 1-kilobyte average size. This represents a good user-database type application, and puts us at a 2.5-gigabyte total size, which fits well with the $150 price point.

To test performance we selected two workloads:

* “Session Store" 50% reads, 50% updates.
* "Heavy Reads" 100% reads.

YCSB works by putting a target throughput on the service, and then observing actual performance of operations per second and latency. All workloads used 150 threads in YCSB, and 500,000 operations. We ran YCSB from an m3.xl EC2 instance in the AWS us-east-1 region.


### Session Store Workload

This workload exercises the ability for a data store to handle high in-place updates of data. MongoDB has well known limitations in this space, because of its [locking design](http://docs.mongodb.org/manual/faq/concurrency/) that will cause contention and performance degradation at high loads.


![](/a/2013-02-17-benchmarking-hosted-mongodb-services/session-store-throughput.png "Session Store, Throughput (Higher is better)")


ObjectRocket’s system met the desired throughput to over 3,000 ops/s, and showed no signs of breaking down while MongoHQ was unable to break 1,500 ops/s. MongoLab showed the worst performance, without being able to consistently break 1,000 ops/s.

Because this is a 50% write workload, the MongoDB Lock contention became a problem on all of the platforms.

![](/a/2013-02-17-benchmarking-hosted-mongodb-services/session-read-latency.png "Session Store, Read Latency (lower is better)")


MongoLab's performance was so poor we actually needed a second, zoomed-in graph to get a fair comparison.

![](/a/2013-02-17-benchmarking-hosted-mongodb-services/session-read-latency-zoomed.png "Session Store, Read Latency (lower is better)")

ObjectRocket produced a consistent latency of 2ms regardless of target throughput. MongoHQ sustained some consistency around 20ms but MongoLab perished to over 200ms of latency under load.

![](/a/2013-02-17-benchmarking-hosted-mongodb-services/session-update-latency.png "Session Store, Update Latency (lower is better)")

ObjectRocket repeated it's 2ms latency for all update operations, with both MongoLab and HQ growing to nearly 300ms.


## Heavy Reads Workload

Heavy reads workloads, such as web applications like CMS's which commonly have many viewers and few updaters, are MongoDB's bread and butter. MongoDB generally provides super low latency access to your data and little CPU overhead.

![](/a/2013-02-17-benchmarking-hosted-mongodb-services/heavy-reads-throughput.png "Heavy Reads, Throughput (higher is better)")

ObjectRocket met all target throughputs up to 10,500 ops/s and showed little signs of degradation. MongoHQ trailed off before 3,000 ops/s and MongoLab never got past 1,200 ops/s.

![](/a/2013-02-17-benchmarking-hosted-mongodb-services/heavy-reads-latency.png "Heavy Reads, Latency (lower is better)")

ObjectRocket delivered consistent 2ms results until above 6,500 ops/s, past which we saw latencies increase up to 20ms. MongoHQ kept up sub-10ms performances until load grew beyond 1500 ops/s, but then quickly degraded. We observed high variability in MongoLab's performance, and under peak load MongoLab delivered read results around 430ms.

## Conclusion

Don't just take out word for it, ObjectRocket is currently [offering 30 day free trials](http://objectrocket.com/pricing) so you can test out your own application and workloads.

P.S.: Rackspace is always hiring outstanding developers. For more information on software developer jobs at Rackspace, visit our [careers page](http://jobs.rackspace.com/go/software-developer-jobs/247407/)


