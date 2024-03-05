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

Raft uses randomized election timeouts to ensure that split votes are rare and that they are resolved quickly. To prevent split votes in the first place, election timeouts are chosen randomly from a fixed interval (e.g., 150-300ms). This spreads out the servers so that in most cases only a single server will time out; it wins the election and sends heartbeats before any other servers timeout.

The same mechanism is used to handle split votes. Each candidate restarts its randomized election timeout at the start of an election, and it waits for that timeout to elapse before starting the next election; this reduces the likelihood of another split vote in the new election.

### Log Replication

Once a leader has been elected, it begins servicing client requests. Each client request contains a command to be executed by the replicated state machines. The leader appends the command to its log as a new entry, then issues `AppendEntries` RPC in parallel to each of the other servers to replicate the entry. When the entry has been safely replicated, the leader applies the entry to its state machine and returns the result of that execution to the client. If followers crash or run slowly, or if network packets are lost, the leader retries `AppendEntries` RPC indefinitely (even after it has responded to the client) until all followers eventually store all log entries.

Each log entry stores a state machine command along with the term number when the entry was received by the leader. The term numbers in log entires are used to detect inconsistencies between logs and to ensure some of the properties. Each log entry also has an integer index identifying its position in the log.

The leader decides when it is safe to apply a log entry to the state machines; such an entry is called *committed*. Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines. A log entry is committed once the leader that created the entry has replicated it on a majority of the servers. This also commits all preceding entries in the leader's log, including entries created by previous leaders.

The leader keeps track of the highest index it knows to be committed, and it includes that index in future `AppendEntries` RPC (including heartbeats) so that the other servers eventually find out. Once a follower leans that a log entry is committed, it applies the entry to its local state machine (in log order).

The Raft log mechanism is designed to maintain a high level of coherency between the logs on different servers. Not only does this simplify the system's behavior and make it more predictable, but it is an important component of ensuring safety. Raft maintains the following properties, which together constitute the `Log Matching Property`:

- If two entries in different logs have the same index and term, then they store the same command.
- If two entries in different logs have the same index and term, then the logs are identical in all preceding entries.

The first property follows from the fact that a leader creates at most one entry with a given log index in a given term, and log entries never change their position in the log.

The second property is guaranteed by a simple *consistency check* performed by `AppendEntries`. When sending an `AppendEntries` RPC, the leader includes the **index** and **term** of the entry in its log that **immediately precedes** the new entries. If the follower does not find an entry in its log with the same index and term, then it refuses the new entries. The consistency check acts as an induction step: the initial empty state of hte logs satisfies the Log Matching Property, and the consistency check preserves the Log Matching Property whenever logs are extended. As a result, whenever `AppendEntries` returns successfully, the leader knows that the follower's log is identical to its own log up through the new entries.

#### Inconsistency Handling

In Raft, the leader handles inconsistencies by forcing the follower's logs to duplicate its own. This means that conflicting entries in follower logs will be overwritten with entries from the leader's log.

To bring a follower's log into consistency with its own, the leader must find the latest log entry where the two logs agree, delete any entries in the follower's log after that point ,and send the follower all of the leader's entries after that point. All of these actions happen in response to the consistency check performed by `AppendEntries` RPC. The leader maintains a `nextIndex` for each follower, which is the index of the next log entry the leader will send to that follower. When a leader first comes to power, it initializes all `nextIndex` values to the index just after the last one in its log. If a follower's log is inconsistent with the leader's, the `AppendEntries` consistency check will fail in the next `AppendEntries` RPC. After a rejection, the leader decrements `nextIndex` and retries the `AppendEntries` RPC. Eventually `nextIndex` will reach a point where the leader and follower logs match. When this happens, `AppendEntries` will succeed, which removes any conflicting entries in the follower's log and appends entries from the leader's log (if any). Once `AppendEntries` succeeds, the followers's log is consistent with the leader's, and it will remain that way for the rest of hte term.

If desired, the protocol can be optimized to reduce the number of rejected `AppendEntries` RPCs. For example, when rejecting an `AppendEntries` request, the follower can include the term of the conflicting entry and the first index it stores for that term. With this information, the leader can decrement `nextIndex` to bypass all of the conflicting entries in that term; one `AppendEntries` RPC will be required for each term with conflicting entries, rather than one RPC per entry. In practice, we doubt this optimization is necessary, since failures happen infrequently and it is unlikely that there will be many inconsistent entries.

With this mechanism, a leader does not need to take any special actions to restore log consistency when it comes to power. It just begins normal operation, and the logs automatically converge in response to failures of the `AppendEntries` consistency check. A leader never overwrites or deletes entries in its own log (the Leader Append-Only Property).

### Safety

The mechanisms described so far are not quite sufficient eo ensure that each state machine executes exactly the same commands in the same order. For example, a follower might be unavailable while the leader commits several log entries, then it could be elected leader and overwrite these entries with new ones; as a result, different state machines might execute different command sequences.

This section completes the Raft algorithm by adding a restriction on which servers may be elected leader. The restriction ensures that the leader for any given term contains all of the entries committed in previous terms (the Leader Completeness Property).

#### Election restriction

In any leader-based consensus algorithm, the leader must eventually store all of hte committed log entries. Raft uses a simpler approach where it guarantees that all the committed entries from previous terms are present on each new leader from the moment of its election, without the need to transfer those entries to the leader. This means that log entries only flow in one direction, from leaders to followers, and leaders never overwrite existing entries in their logs. Raft uses the voting process to prevent a candidate from winning an election unless **its log contains all committed entires**. A candidate must contact a majority of the cluster in order to be elected, which means that every committed entry must be present in at least one of those servers. If the candidate's log is at least as up-to-date as any other log in that majority, then it will hold all the committed entries. The `RequestVote` RPC implements this restriction: the RPC includes information about the candidate's log, and the voter denies its its vote if its own log is more up-to-date than that of the candidate.

Raft determines which of two logs is more up-to-date by comparing the index and term of the last entries in the logs:

- If the logs have last entries with different term, then the log with the later term is more up-to-date
- If the logs end with the same term, then whichever log is longer is more up-to-date.

#### Committing Entries from Previous Terms

A leader knows that an entry from its current term is committed once that entry is stored on a majority of the servers. If a leader crashes before committing an entry, future leaders will attempt to finish replicating the entry. However, a leader cannot immediately conclude that an entry from a previous term is committed once it is stored on a majority of servers.

Raft never commits log entries from previous terms by counting replicas. Only log entires from the leader's current term are committed by counting replicas; once an entry from the current term has been committed in this way, then all prior entries are committed indirectly because of the Log Matching Property. There are some situations where a leader could safely conclude that an older log entry is committed (e.g. if that entry is stored on every server), but Raft takes a more conservative approach for simplicity.

Raft incurs this extra complexity in the commitment rules because log entries retain their original term numbers when a leader replicates entries from previous terms. In other consensus algorithms, if a new leader re-replicates entries from prior "terms", it must do so with its new "term number". Raft's approach makes it easier to reason about log entries, since they maintain the same number over time and across logs. In addition, new leaders in Raft send fewer log entries from previous terms than in other algorithms (other algorithms must send redundant log entries to renumber them before they can be committed).

#### Safety Argument

Given the complete Raft algorithm, we can now argue more precisely that the Leader Completeness Property holds (this argument is based on the safety proof). We assume that the Leader Completeness Property does not hold, then we prove a contradiction. Suppose the leader for term T (leader[T]) commits a log entry from its term, but that log entry is not stored by the leader of some future term. Consider the smallest term U > T whose leader (leader[U]) does not store the entry.

1. The committed entry must have been absent from leader[U]'s log at the time of its election (leaders never delete or overwrite entries).
2. Leader[T] replicated the entry on a majority of the cluster, and leader[U] received votes from a majority of the cluster. Thus, at least one server ("the voter") both accepted the entry from leader[T] and voted for leader[U]. The voter is key to reaching a contradiction.
3. The voter must have accepted the committed entry from leader[T] *before* voting for leader[U]; otherwise it would have rejected the `AppendEntries` request from leader[T] (its current term would have been higher than T).
4. The voter still stored the entry when it voted for leader[T], since every intervening leader contained the entry (by assumption), leaders never remove entries, and followers only remove entries if they conflict with the leader.
5. The voter granted its vote to leader[T], so leader[U]'s log must have been as up-to-date as the voter's. This leads to one of two contradictions.
6. First, if the voter and leader[U] shared the same last log term, then leader[U]'s log must have been at least as long as the voter's, so its log contained every entry in the voter's log. This is a contradiction, since the voter contained the committed entry and leader[U] was assumed not to.
7. Otherwise, leader[U]'s last log term must have been larger than the voter'. Moreover, it was larger than T, since the voter's last log term was at least T (it contains the committed entry from term T). The earlier leader that created leader [U]'s last log entry must have contained the committed entry in its log (by assumption). Then, by the Log Matching Property, leader[U]'s log must also contain the committed entry, which is a contradiction.
8. This completes the contradiction. Thus, the leaders of all terms greater than T must contain all entries from term T that are committed in term T.
9. The Log Matching Property guarantees that future leaders will also contain entries that are committed indirectly.

Given the Leader Completeness Property, the State Machine Safety Property can be proved, which states that if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index. At the time a server applies a log entry to its state machine, its log must be identical to the leader's log up through that entry and the entry must be committed. Now consider the lowest term in which any server applies a given log index; the Log Completeness Property guarantees that the leaders for all higher terms will store that same log entry, so servers that apply the index in later terms will apply the same value. Thus, the State Machine Safety Property holds.

Finally, Raft requires servers to apply entries in log index order. Combined with the State Machine Safety Property, this means that all severs will apply exactly the same set of log entries to their state machines, in the same order.

## Replicated State Machines

Consensus algorithms typically arise in the context of *replicated state machines*. In this approach, state machines on a collection of servers compute identical copies of the same state and can continue operating even if some of the servers are down. Replicated state machines are used to solve a variety of **fault tolerance** problems in distributed systems.

When a request from client lands on a server, these commands are added to its log. That server then communicates with other servers to ensure that every log eventually contains the same requests in the same order, even if some servers fail. Once commands are properly replicated, each server's state machine processes them in log order, and hte outputs are returned to clients. As a result, the servers appear to form a single, highly reliable state machine.

### Replicated Log

Replicated state machines are typically implemented using a **replicated log**. Each server stores a log containing a series of commands (operations), which its state machine executes in order. Each *log* contains the same commands in the same order, so each state machine processes the same sequence of commands.

Keeping the replicated log consistent is the job of the consensus algorithm.

### State Machine

Processes logs sequentially.
