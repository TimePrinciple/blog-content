---
# Must present
author: Ruoqing He
title: Raft Consensus Algorithm
# Optional
date: 2024-03-03
summary: An exploration of paper "In Search of an Understandable Consensus Algorithm (Extended Version)" by Diego Ongaro and John Ousterhout.
---

## Consensus Algorithm

Consensus algorithms allow **a collection of machines to work as a coherent group that can survive the failures of some of its members**. Because of this, they play a key role in building reliable large-scale software systems.

Consensus algorithms for practical systems typically have the following properties:
- They ensure *safety* (never returning an incorrect result) under all non-Byzantine conditions, including:
  - network delays
  - network partitions
  - packet loss
  - packet duplication
  - reordering
- They are fully functional (available) as long as any majority of the servers are operational and can communicate with each other and with clients. Thus, a typical cluster of five servers can tolerate the failure of any two servers (three of them still functioning). Servers are assumed to fail by stopping, they may later recover from state on stable storage and rejoin the cluster.
- They do not depend on timing to ensure the consistency of the logs: faulty clocks and extreme message delays can, at worst, cause availability problems.
- In the common case, a command can complete as soon as a majority of the cluster has responded to a single round of remote procedure calls; a minority of slow servers need not impact overall system performance.

## Paxos

Paxos first defines a protocol capable of reaching agreement on a single decision, such as a single replicated log entry. Paxos ensures both safety and liveness, and it supports changes in cluster membership. Its correctness has been proven, and ti is efficient in the normal case.

Unfortunately, Paxos has two significant drawbacks:
1. Paxos is exceptionally difficult to understand. The full explanation is notoriously opaque; few people succeed in understanding it, and only with great effort.
2. Paxos does not provide a good foundation for building practical implementations. one reason is that there is no widely agreed-upon algorithm for multi-Paxos.
3. Paxos architecture is a poor one for building practical systems; this is another consequence of the single-decree decomposition. For example, there is little benefit to choosing a collection of log entries independently and then melding them into a sequential log; this just adds complexity. It is simpler and more efficient to design a system around a log, where new entries are appended sequentially in a constrained order.
4. Paxos uses a symmetric peer-to-peer approach at its core (though it eventually suggests a weak form of leader ship as a performance optimization). This makes sense in a simplified world where only one decision will be made, but few practical systems use this approach. If a series of decisions must be made, it is simpler and faster to first elect a leader, then have the leader coordinate the decisions. 

As a result, practical systems bear little resemblance to Paxos. Each implementation begins with Paxos, discovers the difficulties in implementing it, and then develops a significantly different architecture. Paxos formulation may be a good one for proving theorems about its correctness, but real implementations are so different from Paxos that the proofs have little value.

## Raft

A consensus algorithm for managing a replicated log. Specific techniques to improve understandability, including decomposition (Raft separates leader election, log replication, and safety) and state space reduction (relative to Paxos, Raft reduces the degree of nondeterminism and the ways servers can bou inconsistent with each other).

Raft implements consensus by first electing a distinguished *leader*, then giving the leader complete responsibility for managing the replicated log. The leader accepts log entries from clients, replicates them on other server, and tells server when it is safe to apply log entries to their state machines. 

Having a leader simplifies the management of the replicated log. For example:
- The leader can decide where to place new entries in the log without consulting other servers.
- Data flows in a simple fashion from the leader to other servers.

The absence of leader (disconnection or failure) will trigger an election.

### Basics

A Raft cluster contains several servers; five is a typical number, which allows the system to tolerate two failures. At any given time each server is in one of three states:

- **Leader**: handles all client requests (if a client contacts a follower, the follower redirects it to the leader).
- **Follower**: passive, they issue no quests on their own but simply respond to requests from leaders and candidates.
- **Candidate**: used to elect a new leader.

In normal operation there is exactly one leader and all of the other servers are followers.

#### Term

Raft divides time into *terms* of arbitrary length. Terms are numbered with consecutive integers. Each term begins with an *election*, in which one or more candidates attempt to become leader. If a candidate wins the election, then is serves as leader for the rest of the term. In some situations an election will result in a split vote. In this case, the term will end with no leader; a new term (with a new election) will begin shortly. Raft ensures that there is at most one leader in a given term.

Different servers may observe the transitions between terms at different times, and in some situations a server may not observe an election or even entire terms. Terms act as a logical clock in Raft, and they allow servers to detect obsolete information such as stable leaders. Each server stores a *current term* number, which increases monotonically over time. Current terms are exchanged whenever servers communicate; if one server's current term is smaller than the others', then it updates its current term to the larger value. If a candidate or leader discovers that its term is out of date, it immediately reverts to follower state. If a server receives a request with a stale term number, it rejects the request.

#### RPC

Raft servers communicate using remote procedure calls (RPCs), and the basic consensus algorithm requires only two types of RPCs:

- `RequestVote`: initiated by candidates during elections.
- `AppendEntries`: initiated by leaders to replicate log entries and to provide a form of heartbeat.

Besides, there might be RPC for transferring snapshots between servers. Servers retry RPCs if they do not receive a response in a timely manner, and they issue RPCs in parallel for best performance.

### Leader Election

Raft uses a heartbeat mechanism to trigger leader election. When servers start up, they begin as followers. A server remains in follower state as long as it receives valid RPCs from a leader or candidate. Leaders send periodic heartbeats (`AppendEntries` RPC that carry no log entries) to all followers in order to maintain their authority. If a follower receives no communication over a period of time called the *election timeout*, then is assumes there is no viable leader and begins an election to choose a new leader.

To begin an election, a follower increments its current term and transitions to candidate state. It then votes for itself and issues `RequestVote` RPC in parallel to each of the other servers in the cluster. A candidate continues in this state until one of three things happens:
- It wins the election
- Another server establishes itself as leader.
- A period of time goes by with no winner.

#### Candidate Wins An Election

If the candidate receives votes from a majority of the servers in the full cluster for the same term. Each server will vote for at most one candidate in a given term, on a first-come-first-served basis. The majority rule ensures that **at most one candidate can win the election for a particular term** (the Election Safety Property). Once a candidate wins an election, it becomes leader. It then sends heartbeat messages to all of the other servers to establish its authority and prevent new elections.

#### Another Server Establish Itself as Leader

While waiting for votes, a candidate may receive an `AppendEntries` RPC from another server claiming to be leader. If the leader's term (included in RPC) is at least as large as the candidate's current term, then the candidate recognizes the leader as legitimate and returns to follower state. If the term in the RPC is smaller than the candidate's current term, then the candidate rejects theRPC and continues in candidate state.

#### Split Votes, No Leader

A candidate neither wins nor loses the election: if many followers become candidates at the same time, votes could be split so that no candidate obtains a majority. When this happens, each candidate will time out and start a new election by incrementing its term and initiating another round of `RequestVote` RPC. However, without extra measures split votes could repeat indefinitely.

## Replicated State Machines

Consensus algorithms typically arise in the context of *replicated state machines*. In this approach, state machines on a collection of servers compute identical copies of the same state and can continue operating even if some of the servers are down. Replicated state machines are used to solve a variety of **fault tolerance** problems in distributed systems.

When a request from client lands on a server, these commands are added to its log. That server then communicates with other servers to ensure that every log eventually contains the same requests in the same order, even if some servers fail. Once commands are properly replicated, each server's state machine processes them in log order, and hte outputs are returned to clients. As a result, the servers appear to form a single, highly reliable state machine.

### Replicated Log

Replicated state machines are typically implemented using a **replicated log**. Each server stores a log containing a series of commands (operations), which its state machine executes in order. Each *log* contains the same commands in the same order, so each state machine processes the same sequence of commands.

Keeping the replicated log consistent is the job of the consensus algorithm.

### State Machine

Processes logs sequentially.
