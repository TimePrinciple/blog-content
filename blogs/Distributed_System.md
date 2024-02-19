---
author: Ruoqing He
title: Distributed System
date: 2024-02-19
summary: An overview of the progress of learning MIT 6.5840.
---

The course is located at [MIT 6.5840](http://nil.csail.mit.edu/6.5840/2023/)

The learning process is dictated by [MIT 6.5840 Schedule](http://nil.csail.mit.edu/6.5840/2023/schedule.html)

## LEC 1

### Distributed System

A group of computers cooperating to provide a service. *Infrastructure services* particularly in this class, e.g. storage for big web sites, MapReduce , peer-to-peer sharing.

#### Pros

- Increased capacity via parallel processing
- Tolerates faults via replication
- Increased security via isolation
- Matches distribution of physical devices, e.g. sensors

#### Cons

- Concurrency
- Complex interactions
- Partial failure
- Hard to get high performance

#### Infrastructure for Applications

- Storage
- Communication
- Computation

The purpose is to hide the complexity of distribution from applications.

#### Fault Tolerance

Achieved by Replication (of servers or instances). Which provides *high availability*: service continues despite failures.

#### Consistency

*Replica* servers are hard to keep identical, might cause different results, so well-defined behavior is required.

#### Performance

Primarily measured by throughput. The objective is to make the performance scalable as the resources scales, but it gets harder as resources (number of servers) grows, problems one might encounter:

- Load imbalance.
- Slowest-of-N latency.
- Things don't speed up with N: initialization, interaction.

#### Tradeoffs

Communication might be the bottleneck of fault-tolerance and consistency, because communication is often slow and non-scalable. Thus, many designs provide only *weak consistency* (i.e. `get()` might **not** yield the latest `put()`) to gain speed.

#### Tech Stack

- RPC
- Threading
- Concurrency control
- Configuration

#### MapReduce

Split computations across servers.

##### Scalability

MapReduce scales well. N "worker" computer (might) get N times throughput.

##### Encapsulation

MapReduce hides many details from user:
- Communication between applications and servers (MapReduce "master").
- Inspecting the status of tasks.
- "Shuffling" intermediate data from Maps to Reduces.
- Balancing load over "workers".
- Recovering from failures.

##### Limitation

MapReduce has limits:
- No interaction or state (other than via intermediate output).
- No iteration.
- No real-time ro streaming processing (the Reduces should not take place while there are Maps still running or yet to be running).

##### Input & Output

The Input and output are provided and stored on the *GFS* cluster file system. The GFS by default splits files over many servers, in 64 MiB chunks. GFS also replicates each file on 2 or 3 servers. Therefore, when applications require MapReduce to process a file from GFS, the file is read by Maps in parallel.

##### Coordinator

The "coordinator" (usually "master") manages all the steps in a job: basically, it acts as a barrier, or sort of synchronization mechanism, which only issues Reduces after all Maps are completed.

##### Performance Bottleneck

- Network: In 2004, the bottleneck is network (another proof of communication being bottleneck), since half of traffic goes through root switch (because of MapReduce's all-to-all shuffle).
- Unbalanced Load: Since Reduces require Maps to complete in advance, it would be wasteful if N-1 servers have to wait for 1 slow server (probably because the larger workload assigned) to finish.
- Partial Failure: It is possible that some of the workers crashes during a MapReduce.

###### Network

The first thing comes to mind might be to minimize the impact caused by network, i.e. to reduce the usage of network. This could be achieved by locality, which means try to manage the workers to work with data best available.

###### Unbalanced Load

Decrease the granularity of tasks, i.e. split many more tasks than worker machines, this might be helpful to reduce the likelihood of long-running tasks. And coordinator hands out new tasks to workers whi finish previous tasks.

###### Partial Failure

This is what fault-tolerance tries to address. Once a server crashed, coordinator knows it by a not-responding `dial()` to that server or the like. Therefore coordinator need to issue the failed Map/Reduce tasks.