# Walking data: can a service's data move from machine to machine without the service stopping?

**Cross-tier research note · Substrate · filed under Tier 2 because walking is coordination across geography**
**Filed:** 2026-11-10 · **Author:** Jodson Graves
**Code examined:** [sohocloud-protocol](https://github.com/NTARI-RAND/sohocloud-protocol) `main` at `c3fac19`, [Cloudy](https://github.com/NTARI-RAND/Cloudy) `main` at `8bdbf17`, [SoHoLINK](https://github.com/NTARI-RAND/SoHoLINK) `main` at `0d497eb` (all 2026-11-09)
**Standard cited:** [Janus Facing Architecture](https://github.com/NTARI-RAND/Janus/blob/9aa2332/janus-facing-architecture.md) (JFA), `main` at `9aa2332`. "Line 7" and the like below mean its twelve numbered Lines
**Companion paper cited:** [P1-004 Substrate Constraints](https://github.com/NTARI-RAND/Janus/blob/9aa2332/P1-004_Substrate-Constraints_v0.1.md), v0.1 draft, which analyses and does not govern

## Summary

The proposal is a "mobile RAID": the data behind a service, for example a governance platform, should *walk* from one prosumer machine to another so that the service keeps running when a machine is switched off, seized, or loses power. The aim is security through continuity. Shutting the service down should take something as large as a power grid or a carrier network, never one machine, one home or one operator.

This note answers five questions.

1. **Can walking be done?** Yes. Stored data walks reliably today in several deployed networks. Live data (a database that is being written to) walks too, but needs at least three machines that fail independently, and a sudden crash costs a pause of seconds to minutes. It never costs data that was confirmed as saved. A running program does not walk through a crash; a fresh copy restarts from the walked data.
2. **How?** With six parts that are all known engineering: seal, split, spread, check, repair, and vote. Cloudy already has tested library code for sealing, spreading and checking, though no command uses it yet. Splitting with redundancy, repairing, voting, and the path that carries the bytes between machines do not exist yet.
3. **Can we predict service quality?** The formulas exist and are simple. The numbers to feed them do not exist yet for this substrate. Published work shows that predictions which assume machines fail independently are wrong by a factor of 100 or more, so a real prediction needs measured data on machines that fail together.
4. **What does it earn a prosumer compared with the electricity it uses?** A few dollars a month at most. Holding others' data costs about $0.53 a month in power for an extra disk and $2 to $4 for a machine kept on for the purpose, at the US average price. NTARI's adopted price rule pays the stated electricity and no more. The outside market pays $1.35 per TB a month, which covers a dedicated machine only once it holds 1.4 to 3.2 TB.
5. **Where should it be built?** Not in one place. The findings point to a split by function: the protocol carries a few small facts, the frontend (Cloudy) owns the keys and the decisions, orchestrators (SoHoLINK) supply reach across geography, and the Record commits to fingerprints of what walked. Building it only in the orchestrator, or inside the Record, would each break a rule the architecture already holds.

The largest single finding is a caution. Walking moves the weak point; it does not remove it. Today one orchestrator, one frontend process and one member device each sit on the path. Unless those walk too, an attacker shuts down the walker instead of the data.

## Terms used here

- **Walking.** Keeping a service's data alive across several machines so that any one of them can leave, on purpose or by crashing, without the service losing data or stopping for long.
- **Shard.** One piece of a file after it has been encrypted and cut up.
- **Replication.** Keeping whole copies on several machines.
- **Erasure coding.** Cutting data into *n* shards so that any *k* of them rebuild the whole. Written *k*-of-*n*. "3-of-10" survives the loss of any seven shards and takes 3.3 times the space of the original. RAID is erasure coding across the disks of one machine; this is the same idea across machines.
- **Repair.** Making new shards to replace ones that left, before too many are gone.
- **Churn.** Machines joining and leaving. Home machines churn far more than data-centre machines.
- **Consensus.** A way for several machines to agree on the order of changes, so that no two of them ever hold different versions of the truth. It works only while more than half of them can reach each other. That majority is called a **quorum**.
- **Failover.** A standby taking over when the active machine stops.
- **Failure domain.** A set of machines that tend to fail together: the same house, the same power line, the same internet carrier, the same owner, the same country.
- **Relay.** A machine with a public address that passes traffic between two machines that cannot reach each other directly.
- **Pinned / scavenged.** Cloudy's two kinds of contributed machine. Pinned is dedicated and stays on. Scavenged is lent while idle and taken back the moment its owner returns.

## 1. What has to walk

Three different things are called "data" in a running service, and they walk in different ways.

| What walks | Example | How it walks | Survives a sudden crash? |
|---|---|---|---|
| **Stored data** | Documents, backups, an archive of past votes | Erasure-coded shards on many machines, repaired as machines leave | Yes, with no pause for readers as long as *k* shards are reachable |
| **Live data** | The database a platform writes to every second | Each change is confirmed by a majority of three or more machines before it counts | Yes. Confirmed changes are never lost. Writes pause while a new leader is chosen |
| **The running program** | The platform's server process and its memory | Not walked. A new copy is started on another machine and reads the live data | The program is restarted, not moved |

Two kinds of walk also need separating.

- **A planned walk** is a machine leaving on notice. Its data is copied off first. This is the easy case and can be made lossless and nearly pause-free.
- **A sudden walk** is a crash, a power cut or a seizure. Nothing can be copied off the lost machine, so everything it held must already exist elsewhere.

The proposal asks for the sudden case. That sets the bar: at every moment, every byte the service has confirmed must already be on enough *other* machines, in enough *other* failure domains.

## 2. Can walking be done?

### Stored data: yes, and it is proven on machines like ours

Several networks already keep data alive on machines their operators do not control.

- Tahoe-LAFS splits each file 3-of-10 by default [[19]](#references).
- Sia splits each chunk 10-of-30 [[21]](#references).
- Storj's documentation gives 29-of-80 as its example [[18]](#references).
- Glacier, a research system, was built to survive 60% of its machines failing at once, at eleven times the storage [[14]](#references).
- Backblaze, in data centres, uses 17-of-20 [[9]](#references).
- Filecoin has each host prove, on a fixed schedule, that it still holds what it was given [[20]](#references).
- IPFS alone promises nothing: a node keeps only what it chooses to pin [[22]](#references).

Erasure coding is far more efficient than whole copies. For the same durability it uses roughly a tenth of the storage and bandwidth [[6]](#references), though with real home-machine behaviour the advantage narrows and its drawbacks can outweigh it [[7]](#references).

The limit on home machines is not disk space. It is the **upload traffic that repair needs**. Every time a machine leaves for good, its shards must be rebuilt elsewhere, and that data has to travel. Blake and Rodrigues argued that a network of home machines can offer high availability, large storage, or tolerance of fast churn, but not all three [[8]](#references). Section 4 puts numbers on this.

### Live data: yes, with three conditions

A database can keep running through a crash if each change is confirmed by a majority of its copies. This is consensus, and it is mature [[4]](#references) [[5]](#references): etcd, CockroachDB and rqlite all work this way [[26]](#references), and PostgreSQL can do it with synchronous replication plus a separate coordinator [[25]](#references). The conditions are firm.

1. **At least three machines.** Three tolerate one loss. Five tolerate two [[4]](#references). Two machines cannot do it: when they lose contact, neither can tell a dead partner from a cut cable.
2. **The machines must fail independently.** A majority that shares a house shares its power cut.
3. **A short pause on a crash is unavoidable.** The survivors need time to notice and to choose a new leader. In a data centre this is under a second [[4]](#references). Across home connections it will be longer.

Three results in computer science set the outer limits, and no design escapes them.

- With no assumption about timing, no method of agreement is guaranteed to finish if even one machine may crash. The original paper's words: "No consensus protocol is totally correct in spite of one fault" [[1]](#references). Real systems get around this with timeouts, which is why a pause exists.
- When the network splits, a service must choose: stay consistent and refuse writes on the smaller side, or stay available and let the two sides diverge [[2]](#references) [[3]](#references). For votes and ledgers the first choice is the right one. The cost is that the minority side stops accepting writes until it rejoins.
- Data that can be merged after the fact (shared notes, tallies that only grow) can avoid consensus altogether using conflict-free replicated data types, at the price of a narrower set of things the data can express [[27]](#references) [[28]](#references).

### The running program: restart, not walk

Moving a running program with its memory from one machine to another is called live migration. It works with pauses as short as 60 milliseconds, but only between machines on the same local network, with shared storage, and with both machines alive throughout [[23]](#references). It is a tool for planned maintenance in a data centre. It cannot help when a machine has already crashed. Checkpointing a container to disk and restoring it elsewhere has the same limit: the checkpoint has to be taken before the failure [[24]](#references).

So for a sudden crash the program does not walk. Another machine starts a fresh copy, and the copy picks up from the live data. This is why the data is the thing that has to walk.

### What "without losses in overall service" can honestly mean

| Promise | Achievable? |
|---|---|
| No confirmed data is ever lost in a single crash | Yes |
| Readers of stored data notice nothing | Yes, while *k* shards are reachable |
| Writers notice nothing | No. They see a pause of seconds to minutes |
| The service survives the loss of one home, one carrier or one operator | Yes, if copies are spread across those domains **and** the coordinating parts also walk |
| The service survives a grid-wide or network-wide outage | Only as far as its copies reach beyond that grid or network |

## 3. How

### The six parts

| Part | What it does | State in NTARI code today |
|---|---|---|
| **Seal** | Encrypt on the member's device, so a host stores bytes it cannot read | Built as a tested library. Cloudy seals each shard with AES-256-GCM under a random key per object ([`internal/storage/seal.go`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/internal/storage/seal.go)). The design keeps that key in the member's own "locker", which is not written yet, and no Cloudy command calls the sealing code yet |
| **Split** | Erasure-code into *k*-of-*n* shards | Interface only. The one implementation is a stand-in that "tolerates ZERO shard loss" ([`internal/storage/erasure.go:33`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/internal/storage/erasure.go#L33)). Reed-Solomon is named as the follow-up |
| **Spread** | Put shards on machines that fail independently | Partly built. Cloudy refuses to give one host two shards of the same object and prefers hosts with different owners while any remain ([`internal/storage/placement.go:28-33`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/internal/storage/placement.go#L28-L33)). Nothing yet knows about houses, carriers or power |
| **Check** | Challenge each host to prove it still holds its shard | Built as a stand-in in Cloudy ([`internal/storage/audit.go`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/internal/storage/audit.go#L24-L29)); today the agent sets and marks its own challenge. The protocol has the messages: `ProofChallenge` and `ProofResponse` ([SPEC.md:270-287](https://github.com/NTARI-RAND/sohocloud-protocol/blob/c3fac19/SPEC.md?plain=1#L270-L287)) |
| **Repair** | Rebuild lost shards on new machines before too many are gone | Not built. The protocol assigns it: "re-placing the shard is the frontend's concern" ([SPEC.md:268](https://github.com/NTARI-RAND/sohocloud-protocol/blob/c3fac19/SPEC.md?plain=1#L268)) |
| **Vote** | Consensus among three or more machines, for live data | Not built anywhere in the stack |

Two more things are needed that are not parts of the mechanism but carry it.

- **A path for the bytes.** Substrate nodes only call out; nothing calls in ([SPEC.md:364](https://github.com/NTARI-RAND/sohocloud-protocol/blob/c3fac19/SPEC.md?plain=1#L364), registered as `SUB-no-inbound-requirement`). Two such machines cannot reach each other without help. Cloudy's design sends all shard traffic through a relay run by the frontend ([`internal/storage/relay.go:5-13`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/internal/storage/relay.go#L5-L13)). That relay has not been written.
- **A place to keep shards.** The Cloudy agent holds shards in memory, and its own comment says durability "needs on-disk persistence" ([`cmd/cloudy-agent/storageexec.go:24-26`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/cmd/cloudy-agent/storageexec.go#L24-L26)).

### A sudden walk, step by step

For **stored data**, split 4-of-8 across eight homes:

1. One home loses power. Its shard is unreachable. Seven remain, and any four are enough, so readers notice nothing.
2. The home's machine misses its heartbeats and its next storage check.
3. Nothing is repaired yet. Most absences are short. In one year-long trace of 632 machines on a research network there were 21,255 temporary failures and 219 disk failures, about 97 to 1 [[11]](#references). Repairing on every absence wastes bandwidth. Google's file system waits 15 minutes [[12]](#references). A home network needs a longer wait, but not too long: a long wait leaves time for a second loss before the first is repaired. Taking a returning machine's shards back, instead of discarding them, is the cheaper defence [[11]](#references).
4. If the machine stays away past the waiting period, or the count of reachable shards falls to a set threshold, a repair job fetches any four shards, rebuilds the missing one, and places it in a ninth home. The repair works on sealed bytes, so the repairer never needs the key.
5. The lease on the lost machine ends, and so does its pay.

For **live data**, on three pinned machines in three homes:

1. The machine currently leading crashes.
2. The other two stop hearing from it. After a timeout they hold a vote, and one becomes leader. Every change that had been confirmed is already on both, because confirmation required a majority.
3. The orchestrator starts a fresh copy of the service on the new leader's machine, or the service was already running there on standby.
4. Users reconnect. Changes that had been sent but not yet confirmed are retried.
5. The group is now two machines and cannot survive another loss. A fourth machine is recruited, copies the data, and joins. That copy is a planned walk.

### Rules the Substrate already has, and what they do to the design

**Hosts can read what they compute.** JFA says substrate work is "integrity-assured, not confidentiality-assured: a host can read what its node computes" ([JFA, Substrate section](https://github.com/NTARI-RAND/Janus/blob/9aa2332/janus-facing-architecture.md?plain=1#L43)). Sealed shards are safe from this: the host holds only encrypted bytes. A running database is not. Every machine in a voting group sees the live data in the clear. Walking a governance platform's live data across prosumer machines therefore shows each of those prosumers the platform's contents. This is a property of the hardware, not a defect to be fixed in code, and it bounds which services should walk live.

**No inbound connections.** Nodes that only call out need a relay to exchange bytes. Direct connection between two machines behind home-style routers by "hole punching" succeeded about 70% of the time in a 2025 study of 4.4 million attempts reaching peers in 167 countries, and the authors say their sample probably flatters the figure [[29]](#references). So at least three pairs in ten need a relay for all their traffic, and every pair needs one to set up the direct link. A relay is something with a public address that can be switched off. Storj avoids the problem by requiring every node to open a port [[18]](#references). JFA rules that out (`SUB-no-inbound-requirement`), and P1-004 gives the reason: it would put a chokepoint at every carrier.

**The pull rhythm is too slow for voting.** A Cloudy agent asks for work every 10 seconds and the frontend asks the orchestrator every 30 ([`cmd/cloudy-agent/main.go:57`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/cmd/cloudy-agent/main.go#L57), [`internal/noderelay/noderelay.go:62`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/internal/noderelay/noderelay.go#L62)). Consensus needs messages to cross in a small fraction of the time it waits before declaring a leader dead [[4]](#references). Live data cannot vote over the polling path. It needs a held-open connection through the relay, such as the WebSocket on port 443 that P1-004 already names as the right carrier.

**Standard library only, in the protocol.** The `P-lean-code` invariant holds protocol software to its language's standard library, and only sohocloud-protocol is bound to it by test. Cloudy is not bound, though today it imports nothing beyond the standard library and the protocol module. SoHoLINK imports third-party libraries freely. Reed-Solomon coding and a consensus algorithm would have to be written in-house only if they went into the protocol module, or if Cloudy chose to keep its current practice. Both are well documented. Neither is small.

**Services must be pinned.** Cloudy already refuses to run a long-lived service on a scavenged machine ([`internal/contribute/contract.go:13`](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/internal/contribute/contract.go#L13)). Its stated reason is that a service cannot yield to the owner part-way through. Section 4 gives a second reason: a voting group made of machines that are on four-fifths of the time is down for weeks each year.

**Two independent hosts is already the floor for work that counts.** JFA requires that "work whose result settles a spend or enters the record runs on at least two independent hosts" (registered as `SUB-redundant-execution`). What "independent" means for hosts is not defined in JFA, P1-004 or the registry. The only definition in the corpus is for witnesses, who must be independent of the operator they observe, and it does not say when two hosts are independent of each other. Walking needs that definition more than anything else it needs.

## 4. Can we predict service quality?

Yes in method. Not yet in numbers.

### Four numbers describe the quality of a walk

| Number | Meaning | What sets it |
|---|---|---|
| **Durability** | The chance the data still exists after a year | How fast machines leave for good, against how fast repair replaces them |
| **Read availability** | The share of time at least *k* shards are reachable | How often each host is on, and how many extra shards there are |
| **Write availability** | The share of time a majority of the voting group is reachable | How often each host is on, and how alike their failures are |
| **Pause on a crash** | How long writers wait | Detection timeout, plus the vote, plus restarting the program |

Durability and availability are different, and the difference matters. Durability asks whether the data exists. Availability asks whether it can be reached right now. Durability is much cheaper to get. The rule for it is simple: new copies must be made faster than permanent failures destroy them, or no amount of redundancy helps [[11]](#references).

### The formulas

**Read availability.** If each host is on a fraction μ of the time and hosts fail independently, the chance that at least *k* of *n* shards are reachable is the sum, for *i* from *k* to *n*, of C(*n*,*i*) · μ^*i* · (1−μ)^(*n*−*i*). Total Recall used this to choose redundancy automatically from measured host availability [[10]](#references).

**Write availability.** The same formula with *k* set to a majority of the group.

**Repair traffic.** A host that stores *D* bytes and stays in the network for a lifetime *T* costs about 2·*D*/*T* in continuous bandwidth, counting both the upload when its data is first placed and the upload that rebuilds it when it leaves [[8]](#references).

**Durability.** Storj's published model treats the number of shards lost in a repair period as a Poisson count and adds up the cases where no more than *n*−*k* are lost [[17]](#references). Appendix A reproduces their figure for 20-of-40 at 10% monthly churn (99.9999998%) as a check on the arithmetic here.

### What the formulas say for machines like ours

The best published figure for volunteer machines comes from about 330,000 SETI@home computers in 2006, a little over half of them at home. On average the client was running 81% of the time, with a network connection for 83% of that. Machines that had left the project had stayed 91 days, which the authors call an underestimate because machines still active were left out [[32]](#references). Storj demands 99.3% of its node operators [[18]](#references). The tables below use both ends. They are computed by the script in Appendix A, and they all assume hosts fail independently. That makes them a best case, for reasons given just after.

**Table 1. Read availability of stored data (best case)**

| Layout | Space used | Hosts on 81% | Hosts on 95% | Hosts on 99% |
|---|---|---|---|---|
| One copy | 1× | 81% | 95% | 99% |
| Two copies | 2× | 96.4% | 99.75% | 99.99% |
| Three copies | 3× | 99.3% | 99.99% | 99.9999% |
| 2-of-4 | 2× | 97.6% | 99.95% | 99.9996% |
| 4-of-8 | 2× | 99.2% | 99.998% | over 99.99999% |
| 3-of-10 | 3.3× | 99.995% | over 99.99999% | over 99.99999% |
| 10-of-30 | 3× | over 99.99999% | over 99.99999% | over 99.99999% |

At three times the space, thirty small shards on ordinary machines (10-of-30 at 81%) beat three whole copies on good ones (99%). At twice the space they do not: 4-of-8 at 81% gives 99.2%, below two copies at 95%.

**Table 2. Write availability of live data (best case)**

| Host is on | One machine | Group of 3 | Group of 5 | Group of 7 |
|---|---|---|---|---|
| 81% | down 1,666 h/yr | down 829 h/yr | down 443 h/yr | down 245 h/yr |
| 95% | down 438 h/yr | down 64 h/yr | down 10 h/yr | down 2 h/yr |
| 99% | down 88 h/yr | down 2.6 h/yr | under 1 h/yr | under 1 h/yr |
| 99.3% | down 61 h/yr | down 1.3 h/yr | under 1 h/yr | under 1 h/yr |

A group of three machines that are each on 81% of the time is still down five weeks a year. Live data needs pinned machines at 99% or better. Cloudy already requires pinned placement for a service because a service cannot yield to its owner; this arithmetic is a second reason for the same rule.

**Table 3. Repair traffic per host**

| Stored on the host | Host stays 30 days | Host stays 91 days | Host stays a year |
|---|---|---|---|
| 100 GB | 0.6 Mbps (200 GB a month) | 0.2 Mbps (66 GB) | 0.05 Mbps (16 GB) |
| 500 GB | 3.1 Mbps (1,000 GB) | 1.0 Mbps (330 GB) | 0.25 Mbps (82 GB) |
| 1 TB | 6.2 Mbps (2,000 GB) | 2.0 Mbps (659 GB) | 0.5 Mbps (164 GB) |

These figures count each lost byte once. With erasure coding, rebuilding one shard means first fetching *k* shards [[11]](#references), so repair traffic can be up to *k* times higher unless several shards are rebuilt from one fetch.

For scale: the median US fixed upload was 72.6 Mbps in September 2026 [[30]](#references), but the average advertised cable upload in the FCC's last measured report was 16.6 Mbps [[31]](#references), and one large US carrier still caps every plan at 1.25 TB a month [[33]](#references). A host that keeps 1 TB for others and stays three months uses half that cap on repair alone. How long hosts stay matters more than how fast their connection is.

### Why the best case is too good

The tables assume each machine fails on its own. Real machines fail together.

- In Google's storage fleet, 37% of machine failures came in bursts, and models that ignored this typically overstated how long data stays available by at least a factor of 100 [[12]](#references).
- In wide-area systems, the largest 1% of failure events dominate the outcome. For layouts such as half-of-*n* and for majority voting, adding shards or copies gives sharply diminishing returns against them; layouts with a small *k* did not show this [[13]](#references).
- Home machines are not coin flips. In a study of about 230,000 volunteer hosts, only 21% of the 169,000 with enough data passed three tests for random availability. The rest showed trends, cycles or drift, though only about 6% showed a clear daily cycle [[32]](#references).
- Not every study finds hosts failing together. In a 2003 file-sharing network, availability followed the time of day but was otherwise roughly independent from host to host [[16]](#references). Which of these a prosumer network resembles is what has to be measured.
- Power is the plainest case. US customers averaged 662.6 minutes without power in 2024 and 269.2 in 2025. Leaving out major events, both years were about 130 minutes [[34]](#references). Most lost hours come from large events that take a whole region at once.

One worked example shows the size of the effect. Take three voting machines that are each on 99% of the time. Table 2 says the group is down 2.6 hours a year. Now put two of the three in the same house, and say that house is dark or offline 1% of the time. The group is down 90 hours a year, 35 times worse than Table 2. Part of that is the extra house downtime: three independent machines with the same individual uptimes would be down 7 hours. Sharing the house accounts for the remaining factor of 13.

Published "mean time to data loss" figures deserve the same caution. A result of thousands of years says little about the next twelve months [[15]](#references).

### What it takes to predict

Prediction worked for Google once they measured correlated failures and fed them to a model: predicted and measured values agreed to within a factor of ten [[12]](#references). The same route is open here.

1. **Record, per host:** when it was reachable, and whether each absence ended. SoHoLINK already computes an uptime figure from heartbeats ([`internal/store/uptime.go`](https://github.com/NTARI-RAND/SoHoLINK/blob/0d497eb/internal/store/uptime.go#L50-L59)). It is used only in the portal, to filter the marketplace listing and to show providers their uptime class. The scheduler does not use it. Its divisor also assumes a heartbeat every 30 seconds, so a node relayed through Cloudy at one a minute would read about half its true uptime.
2. **Record which hosts were absent at the same moments.** This is the missing input. It needs no personal data: only the fact that hosts A and B were away together.
3. **Estimate two rates:** how often a host is away, and how often a host never returns.
4. **Simulate** the chosen layout against the recorded history, not against a formula.
5. **Publish a range, not a figure,** and compare it with what then happens.

Until steps 1 to 3 have run on real hosts for some months, any figure quoted for this substrate is a guess. That includes the tables above.

**The pause can be bounded today.** Under SoHoLINK's current settings a silent node is dropped five minutes after its last heartbeat ([`cmd/orchestrator/main.go:149`](https://github.com/NTARI-RAND/SoHoLINK/blob/0d497eb/cmd/orchestrator/main.go#L149)), checked every 30 seconds ([`internal/orchestrator/registry.go:253-258`](https://github.com/NTARI-RAND/SoHoLINK/blob/0d497eb/internal/orchestrator/registry.go#L253-L258)). A crash would go unnoticed for roughly four to five and a half minutes. Then nothing would happen, because a job that is already running on a vanished node is never recovered. Only jobs that have not started are moved.

## 5. What it earns a prosumer, against the electricity it uses

Short answer: a few dollars a month at most. Under NTARI's own adopted price rule it earns the electricity bill and nothing more, by design.

### The electricity

**Table 4. Electricity used, at the US average residential price of 18.16 ¢/kWh (August 2026) [[35]](#references)**

| Setup | Watts | kWh a month | Cost a month |
|---|---|---|---|
| One extra disk in a machine that is on anyway | 4.0 | 2.9 | $0.53 |
| A mini PC kept on for this, plus one disk | 14.5 | 10.6 | $1.92 |
| A desktop kept on for this, plus one disk | 32.2 | 23.5 | $4.27 |

The watts are a Seagate IronWolf 8 TB disk, the 5,400-rpm model, at idle (4.0 W; the 7,200-rpm model idles at 7.8 W) [[37]](#references); an Intel N100 fanless mini PC at idle (10.5 to 12 W measured; the low end is used) [[38]](#references); and an office desktop near the top of the Energy Star range, on and unused with its screen lit (28.2 W) [[39]](#references). That same desktop draws 11.2 W once its screen sleeps, and the median certified desktop draws 10.5 W, so the desktop row is a high figure for a new machine. Older towers were not measured here. A month is taken as 730 hours.

Place matters more than hardware. A 15-watt node costs per month: Kentucky $1.51, US average $1.99, California $3.26, Hawaii $5.17 [[35]](#references); EU average €3.10, Germany €3.96, Hungary €1.25 [[36]](#references); Brazil $1.99, India $0.80 [[36]](#references).

### The income, by three yardsticks

**1. NTARI's adopted rule.** Cloudy Decision 0002 sets the price of work as "the producer's attested electricity rate × the computer's attested power draw while running × run time" ([Decision 0002](https://github.com/NTARI-RAND/Cloudy/blob/8bdbf17/docs/decisions/0002-pricing-fees-and-member-privacy.md)). Under it, income equals stated electricity. Walking adds to a prosumer's income exactly what it adds to their bill, paid in escrowed money or community credit. The decision does not mention storage. Read as a rule for jobs that run and finish, it leaves three things open for data that is held:

- *Which draw counts?* For a machine that is on anyway, the disk adds $0.53 a month but the whole machine draws $4.27. If the whole machine may be attested, the difference is real income.
- *How is one disk's cost shared among the many members whose shards it holds?*
- *What does it do to price?* A 4-watt disk costs the same full or empty. Filled with 8 TB, the cost-basis price is about 7 cents per TB a month. That is a hundredth of commercial prices. It pays nothing toward the disk itself, the connection, or staying reliable.

**2. SoHoLINK's earlier price table.** It lists storage at $0.010 per GB a month with 65% to the contributor, which is $6.50 per TB a month ([migration 004](https://github.com/NTARI-RAND/SoHoLINK/blob/0d497eb/internal/store/migrations/004_resource_pricing.up.sql#L27)). Decision 0002 replaces the payout path behind it, though that code is still present.

**3. The outside market.** Storj pays node operators $1.35 per TB stored a month and $1.00 per TB sent, including data sent for repair. Both were cut in September 2026, from $1.50 and $2.00 [[40]](#references). Its customers pay $7 per TB a month [[41]](#references); Backblaze charges $6.95, Wasabi $7.99, and Amazon S3 $23 [[42]](#references). A host is paid about a fifth as much per TB it holds as a customer pays per TB stored. Part of the gap is redundancy: at the 29-of-80 layout in Storj's documentation, the network holds about 2.8 TB for each TB sold, so hosts together would receive about $3.70 of each $7. Storj reports its network about half empty [[40]](#references). Storj Labs filed for Chapter 11 reorganization in July 2026, and a company notice in November 2026 says its storage business is to be sold through the court [[43]](#references). The going rate is not evidence of a sustainable one.

**Table 5. Monthly result at the outside market's storage rate ($1.35 per TB), after electricity**

| Filled | Paid | Extra disk | Mini PC | Desktop |
|---|---|---|---|---|
| 1 TB | $1.35 | +$0.82 | −$0.57 | −$2.92 |
| 4 TB | $5.40 | +$4.87 | +$3.48 | +$1.13 |
| 8 TB | $10.80 | +$10.27 | +$8.88 | +$6.53 |

A machine kept on only for this loses money until it holds 1.4 TB (mini PC) or 3.2 TB (desktop). An extra disk in a machine that is on anyway covers its power at 0.4 TB.

### What walking itself adds

- **Redundant shards** are more TB held per TB of a member's data. Hosts hold more; the member pays for all of it. A 4-of-8 layout doubles the member's cost.
- **Repair** pays little. A community estimator that Storj's documentation points to assumes repair traffic of 1.5% of stored data a month [[44]](#references). On 8 TB that is 0.12 TB, or 12 cents.
- **A pinned replica for live data** has no market price. Under Decision 0002 it earns the machine's electricity: $1.92 to $4.27 a month in Table 4.

### What the comparison leaves out

- **The connection may cost more than the power.** On older capped plans, one US carrier charges $10 for each 50 GB over the limit [[45]](#references). One overage block costs about what 7 TB earns in a month.
- **The disk.** Hardware is not priced here.
- **Time to fill.** The same estimator assumes at most about 0.8 TB of new data a month, so the 8 TB row is a second-year figure [[44]](#references).
- **Community credit is not dollars.** Where a network settles in mutual credit, the figures above are a measure of value, not cash.

The plain reading: walking is cost recovery for a prosumer, not a wage. It pays where the machine is already on and the power is cheap. Whether held storage should be priced on electricity alone is a decision the federation has not yet made.

## 6. Where to build it: four options and their consequences

### Option A. In the SohoCloud Protocol (Tier 1)

**What it would mean.** New signed messages for groups of leases, hand-off between nodes, and repair.

**What is already there.** More than expected. The protocol has a `StorageLease` (one sealed shard, one node, a bounded term), `LeaseRelease` (a node may leave early), and proof-of-possession messages. It states the division of labour outright: how data is "encrypted, sharded, and audited is a frontend/agent concern outside this protocol" ([SPEC.md:173-175](https://github.com/NTARI-RAND/sohocloud-protocol/blob/c3fac19/SPEC.md?plain=1#L173-L175)).

**Consequences.**

- *It conflicts with a rule the protocol holds.* "The protocol MUST NOT grow a field that dictates matching" ([SPEC.md:338-339](https://github.com/NTARI-RAND/sohocloud-protocol/blob/c3fac19/SPEC.md?plain=1#L338-L339)). A message saying "these eight leases must sit in eight different homes" dictates matching.
- *It would tell every coordinator which shards belong together.* Cloudy pads shards to uniform sizes, gives no host two shards of an object, and fetches in random order, so that no single host can regroup an object. Its design also schedules checks as cover traffic to hide when a member reads, though nothing sends that traffic yet. A group identifier on the wire undoes this and gives each coordinator a map of where a service's data lives. That map is what someone wanting to shut the service down would ask for.
- *Independence needs facts about a machine that the protocol does not carry.* A node's listing has no location field. The no-inbound test fails only fields naming something that can be dialled, such as an address, host, port or URL ([`no_inbound_test.go:157-164`](https://github.com/NTARI-RAND/sohocloud-protocol/blob/c3fac19/no_inbound_test.go#L157-L164)); a country or region field would pass it and be left to review. Cloudy Decision 0002 goes the other way: operators are to register machines with no country attached, though SoHoLINK has not built that yet.
- *It is the slowest and widest change.* A wire change means the specification first, a version bump, new test vectors, and updates to every consumer. Both consumers still pin protocol v0.1.0, which predates the lease messages, and the lease messages have no reference transport yet.
- *It is the only place that makes walking work across coordinators.* If a shard can only walk among the nodes of one coordinator, that coordinator is the thing to shut down.

**Assessment.** Keep the protocol to facts, as it is now. Two small additions are worth study: an optional notice period on `LeaseRelease`, so a planned walk can finish before the node leaves, and a way for a lease to be proven to a second coordinator.

### Option B. As a Record Layer function

**What it would mean.** The Record keeps the application's data, using the six-holder and public-chain machinery it already defines.

**Consequences.**

- *The Record does not hold content.* It holds "hashes, types, timestamps and references only" (Line 7), and what it carries is "the fact of commitment, not the content." Application data is content.
- *The Record never deletes.* "The record is append-only — forgive harm by annotating, never by erasing" (Line 6). Application data changes, and some of it must be deletable. A member's draft, a withdrawn comment or a personal detail cannot go into a store that forbids erasure.
- *The cost of leaving runs the wrong way.* JFA makes leaving the Substrate cheap and leaving the Record "catastrophic." Putting a service's data in the Record moves it from the layer built for cheap exit to the layer built against it.
- *The dependency runs the wrong way.* The Record is defined as "a compensated function of the substrate." It consumes storage from the Substrate. It also needs walking itself: each copy of the record "lasts only as long as it is kept," and JFA's only rule for a keeper leaving is that the record "survives in whichever copies remain." Nothing provides for replacing the lost copy.
- *There is little to build on yet.* Cloudy has a per-operator witnessed log ([`internal/record`](https://github.com/NTARI-RAND/Cloudy/tree/8bdbf17/internal/record), a single-witness stand-in). The public chain, the record orchestrator and the storage market are not built: Tell, Beacon and Route are each a licence and a two-line README. How the public chain is constructed is still open (JFA open question 12).

**Assessment.** The Record is not the home for walking. It is walking's first customer, and it has one job in the design: to hold the fingerprint of what walked. If the hash of a data set's layout is committed to the Record at each walk, anyone can check that what arrived is what left, and two machines that both claim to hold the current version can be told apart. That job needs only hashes, which is what the Record is for.

### Option C. As a Substrate orchestration service (SoHoLINK, Tier 2)

**What it would mean.** The orchestrator watches nodes, decides when a shard is lost, and schedules repair.

**What is already there.** A scheduler that knows each node's region and country, a liveness registry, a loop that moves unstarted jobs off silent nodes, an uptime score, and replica tiers named Reliable (two nodes) and Premium (three) that no code path uses yet ([`internal/orchestrator/orchestrator.go:29-31`](https://github.com/NTARI-RAND/SoHoLINK/blob/0d497eb/internal/orchestrator/orchestrator.go#L29-L31)).

**Consequences.**

- *It is the fastest route to a working demonstration.* Most of the watching and placing exists.
- *It strains what JFA says of an orchestrator.* JFA: "Transport is delivery, not execution." An orchestrator "is never relied upon to determine whether an exchange occurred, and may be lossy and retry-based." JFA says this of carrying exchanges, not of scheduling work, but the direction is clear. A repair service makes the orchestrator the thing durability depends on. Storj's own paper states the result for its equivalent: if the coordinating server stops, repair stops and the data decays [[17]](#references).
- *Today it would create the single point the proposal exists to remove.* SoHoLINK's orchestrator runs as a single process with its node registry in memory, and its conformance file says "this is a single-coordinator deployment" ([CONFORMANCE.md:34](https://github.com/NTARI-RAND/SoHoLINK/blob/0d497eb/CONFORMANCE.md?plain=1#L34)). P1-004 already notes that the reference deployment sits behind a single commercial content-delivery network and is "not conformant with line 11 in spirit."
- *Its scheduler is weighted to pull the wrong way.* Placement is designed locality-first: a node in the requester's own region gets a bonus larger than the class, freshness and capacity terms combined ([`internal/scheduler/scheduler.go:52-56`](https://github.com/NTARI-RAND/SoHoLINK/blob/0d497eb/internal/scheduler/scheduler.go#L52-L56)). The requester's location is not collected yet, so the bonus does not fire today. Walking wants the opposite, shards as far apart as possible.
- *It would hold the map of every service's data.* See Option A.
- *Repair traffic is transport, and transport is what orchestrators charge for.* The size and basis of the transport fee are undecided (Decision 0002). Walking would be its heaviest user.

**Assessment.** Orchestrators should supply what only they can: reach. That means candidate nodes in other regions and under other carriers, the facts about which nodes are answering, and transport between networks. They should not be the only party able to repair, and the design should work when any one orchestrator is gone. One gap needs fixing regardless of this proposal: a job running on a node that disappears is never recovered.

### Option D. As a prosumer option in Cloudy (Tier 3)

**What it would mean.** Two choices in the interface. A member who stores data picks how hard it should be to lose. A member who contributes a machine picks whether to hold walking shards, do repair work, or host a pinned replica.

**What is already there.** The most. Sealing, size padding, the one-shard-per-host rule, storage checks and the split-and-relay interfaces are in `internal/storage`, as a tested library that no command calls yet. The protocol already assigns re-placement to the frontend.

**Consequences.**

- *The keys stay with the member.* No host, frontend or orchestrator can read walked data. This is the property Tahoe-LAFS calls provider-independent security [[19]](#references).
- *The member's device becomes a single point.* The key lives only in the member's locker. If the device is lost, the data is unreadable however many shards survive. The key has to walk too, for example by splitting it among people the member trusts.
- *The frontend becomes a single point.* All shard traffic is designed to cross the frontend's relay, and `cloudyd` keeps everything in memory today. Decision 0001 accepts that "a frontend outage stops its members' work." For walking, that means a frontend outage stops repair for its members.
- *The frontend's connection bounds everything.* If every repaired byte crosses one relay, that relay's upload is the ceiling for the whole network's repair.
- *Only Cloudy's members get it.* Another frontend would write its own, and shards would not walk between them unless the protocol carries enough to make that possible. This is the one argument for Option A.
- *It fits how prosumers already choose.* The contributor's choice maps onto pinned and scavenged. A scavenged machine can hold shards of stored data, since shards tolerate absence. Only a pinned machine can join a voting group.
- *The pricing rule does not cover it.* See section 5.

**Assessment.** This is where the decisions belong, because the member owns the data and the keys. It is also the tier with the most to build.

### The four options side by side

| | Protocol | Record | Orchestrator | Cloudy |
|---|---|---|---|---|
| Exists today | Lease and proof messages | A per-operator log in Cloudy; no chain | Scheduler, liveness, unused replica tiers | Seal, spread, check, interfaces |
| Who can read the data | No one (content never on the wire) | Would be public and permanent | No one, if sealed | Only the member |
| Who holds the map | Every coordinator, if groups go on the wire | Everyone | The orchestrator | The member's frontend |
| New single point | None | The chain | The orchestrator | The frontend and the member's device |
| Rule it strains | "No field that dictates matching" | Lines 6 and 7 | "Never relied upon"; Line 11 | None, but Decision 0001's outage rule now covers data |
| Reaches across networks | Yes | Yes | Yes | No |
| Speed to a first test | Slow | Not possible yet | Fast | Medium |

### What the findings point to

No one tier can hold walking without breaking something. A split by function fits the rules each tier already has.

| Tier | Its part in walking |
|---|---|
| **Protocol** | Stay a carrier of facts: lease, release, proof. Study two additions: notice on release, and a lease provable to more than one coordinator |
| **Cloudy** | Own the keys, the split, the layout, the checks and the decision to repair. Offer the two prosumer choices |
| **SoHoLINK** | Supply nodes across regions and carriers, report which are answering, carry bytes between networks. Never be the only one who can repair |
| **Record** | Commit the fingerprint of each layout and each walk. Become walking's first customer for its own copies |

Repair itself can be ordinary substrate work, paid like any job: fetch *k* sealed shards, rebuild the missing ones, deliver them. JFA's open-questions record already treats witnessing this way, "assigned as compute work that prosumers perform and are paid for" ([entry 6](https://github.com/NTARI-RAND/Janus/blob/9aa2332/OPEN-QUESTIONS.md?plain=1#L51), registered as `REC-witness-work`). Making repair a job means no fixed party has to do it, which is what keeps it from becoming a single point.

## 7. What walking does not solve

- **The walker is the new target.** A service whose data survives any machine can still be stopped by stopping whatever decides on repairs, relays the bytes, or holds the key. Each of those must exist in more than one place before the proposal's security claim holds. Line 11 already demands it: "No single host, account, or vendor whose removal could stop the network."
- **Independence can be faked.** One person running eight machines in one room under eight accounts looks like eight hosts. Cloudy's placement prefers different owners, which stops the single-account case only. Walking's whole safety margin rests on hosts being independent, and the architecture forbids identity data in the shared record (Line 7), which rules out the cheap ways to check. JFA open question 8 already records this tension for voting. It applies here in full.
- **Repair storms.** A regional outage removes many hosts at once. If every frontend starts repairing at the same moment, the survivors' upload links saturate and the repair fails when it is most needed. Repairing only below a threshold, delaying repair, and taking back shards from hosts that return are the known ways to avoid needless repair [[10]](#references) [[11]](#references). Google also limits the rate of recovery after a burst [[12]](#references).
- **Live data is visible to its hosts.** See section 3.
- **Carrier terms.** P1-004 section 5 leaves open whether residential acceptable-use terms allow paid hosting at all. Holding other people's data continuously is closer to "running a server" than occasional compute is. This is a question for counsel.
- **Unknown contents.** A prosumer holding sealed shards cannot know what is in them. What duties that creates, and how a lawful request to remove something would be honoured when the host cannot identify it, are also questions for counsel and are not answered here.

## 8. Tests that would settle the open claims

Each is small, uses no member data, and can fail.

| # | Test | It passes if |
|---|---|---|
| T1 | Write Reed-Solomon in Cloudy's `Coder` interface using the standard library only. Split a file 4-of-8, destroy any four shards, rebuild | The rebuilt file matches for every choice of four, Cloudy's `go.mod` gains no new requirement, and the existing tests stay green |
| T2 | Three machines in three different buildings on three carriers, all outbound-only. Store through a relay. Cut power to one without warning | A reader on a fourth network sees no error, and the lost shard is rebuilt on a new machine with no person involved |
| T3 | The same three machines run a voting group for a small database over a held-open relay connection. Cut power to the leader during writes | No confirmed write is lost. Record the pause; report it, whatever it is |
| T4 | Log reachability for every willing host for 90 days, including which hosts were away together | The log yields the two rates and the together-failure pattern needed for section 4, with nothing in it that identifies a person |
| T5 | Measure how much one relay can carry on a residential uplink while repair runs | A figure in megabits per second that sets the ceiling for Option D |
| T6 | Stop the orchestrator during T2 | Repair still completes through a second coordinator, or the note's central caution is confirmed with evidence |

## 9. Questions for #federation-substrate

1. What makes two hosts "independent"? Different owner, different building, different carrier, different power utility, different country? `SUB-redundant-execution` and walking both depend on the answer. The corpus defines independence for witnesses only.
2. How can a frontend learn that two hosts are independent without learning who or where they are? If it cannot, which gives way: privacy or the independence guarantee?
3. Should repair be a paid substrate job open to any node, or a duty of the frontend that placed the data?
4. Should live data walk at all on prosumer machines, given that every host in a voting group can read it? Or should the Substrate offer walking for stored data only, and leave live services to platforms that accept the exposure in writing?
5. Does a planned walk deserve a protocol message (notice on release), or is it enough that a node may leave at any time?
6. Who may hold the map of where a service's data lives, and what must be true so that no single holder can be compelled to reveal or destroy it?
7. Should data that is held, not run, be priced on electricity alone under Decision 0002? If so, which draw counts, and how is one disk's cost shared among the members it serves?
8. For counsel, not this federation: do residential carrier terms permit continuous paid storage for others, and what duties attach to a prosumer holding sealed data they cannot read?

## Method and ethics

The research was carried out by Claude (Anthropic) at the direction of the author. Five independent readers each took one body of evidence: the protocol and P1-004; Cloudy and SoHoLINK; the Record Layer texts; the published literature; and power, hardware and payout figures. Each was told to report what the sources say, with file and line or a working link, and to say plainly where something was absent or could not be verified. Two further reviewers then tried to refute every claim: one against the code and JFA texts at the commits named at the top, one against the published sources. Of 120 claims checked, 82 were confirmed, 35 held only in part and 3 were refuted. The partial and refuted ones are corrected or reworded above. Every table was recomputed from the script in Appendix A.

Limits a reader should know:

- Reference [[7]](#references) was confirmed from its abstract only. Every other source was opened and read. All links in the reference list resolved on 2026-11-11. Several lead to live pages (prices, payout rates, speed indexes) whose figures will change.
- No test in section 8 has been run. Every statement about what "would" happen under NTARI's code is a reading of the code, not an observation.
- The tables in section 4 assume independent failures and are marked as a best case for that reason.
- The host-uptime figures come from volunteer-computing projects between 2006 and 2009, and a little over half of those machines were at home. Home machines today may differ.

No live deployment, database or member account was contacted. No personal data was used. Test T4 proposes collecting reachability data from hosts; it should be run only with each host owner's consent and should record nothing that identifies a person or a place.

## References

**Limits**

1. Fischer, Lynch, Paterson. "Impossibility of Distributed Consensus with One Faulty Process." *Journal of the ACM* 32(2), 1985. <https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf>
2. Gilbert, Lynch. "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services." *ACM SIGACT News* 33(2), 2002. <https://www.comp.nus.edu.sg/~gilbert/pubs/BrewersConjecture-SigAct.pdf>
3. Brewer. "CAP Twelve Years Later: How the 'Rules' Have Changed." *IEEE Computer* 45(2), 2012. <https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/>
4. Ongaro, Ousterhout. "In Search of an Understandable Consensus Algorithm." USENIX ATC 2014 (extended version). <https://raft.github.io/raft.pdf>
5. Lamport. "Paxos Made Simple." *ACM SIGACT News* 32(4), 2001. <https://lamport.azurewebsites.net/pubs/paxos-simple.pdf>

**Redundancy and repair**

6. Weatherspoon, Kubiatowicz. "Erasure Coding vs. Replication: A Quantitative Comparison." IPTPS 2002. <https://oceanstore.cs.berkeley.edu/publications/papers/pdf/erasure_iptps.pdf>
7. Rodrigues, Liskov. "High Availability in DHTs: Erasure Coding vs. Replication." IPTPS 2005. <https://link.springer.com/chapter/10.1007/11558989_21> (abstract only)
8. Blake, Rodrigues. "High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two." HotOS IX, 2003. <https://www.usenix.org/legacy/events/hotos03/tech/full_papers/blake/blake.pdf>
9. Wilson. "Backblaze Durability Calculates at 99.999999999% — And Why It Doesn't Matter." Backblaze, 2018. <https://www.backblaze.com/blog/cloud-storage-durability/>
10. Bhagwan, Tati, Cheng, Savage, Voelker. "Total Recall: System Support for Automated Availability Management." NSDI 2004. <https://www.usenix.org/legacy/events/nsdi04/tech/full_papers/bhagwan/bhagwan.pdf>
11. Chun et al. "Efficient Replica Maintenance for Distributed Storage Systems." NSDI 2006. <https://www.usenix.org/legacy/event/nsdi06/tech/full_papers/chun/chun.pdf>

**Failures that happen together**

12. Ford et al. "Availability in Globally Distributed Storage Systems." OSDI 2010. <https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Ford.pdf>
13. Nath, Yu, Gibbons, Seshan. "Subtleties in Tolerating Correlated Failures in Wide-area Storage Systems." NSDI 2006. <https://www.usenix.org/legacy/event/nsdi06/tech/full_papers/nath/nath.pdf>
14. Haeberlen, Mislove, Druschel. "Glacier: Highly Durable, Decentralized Storage Despite Massive Correlated Failures." NSDI 2005. <https://www.usenix.org/legacy/event/nsdi05/tech/full_papers/haeberlen/haeberlen.pdf>
15. Greenan, Plank, Wylie. "Mean Time to Meaningless: MTTDL, Markov Models, and Storage System Reliability." HotStorage 2010. <https://www.usenix.org/legacy/event/hotstorage10/tech/full_papers/Greenan.pdf>
16. Bhagwan, Savage, Voelker. "Understanding Availability." IPTPS 2003. <https://www.sysnet.ucsd.edu/projects/recall/papers/iptps.pdf>

**Deployed networks**

17. Storj Labs. "Storj: A Decentralized Cloud Storage Network Framework," v3.1, 2024. <https://www.storj.io/storjv3.pdf>
18. Storj documentation: file repair, node prerequisites, port forwarding. <https://storj.dev/learn/concepts/file-repair>, <https://storj.dev/node/get-started/prerequisites>, <https://storj.dev/node/get-started/port-forwarding>
19. Tahoe-LAFS documentation. <https://tahoe-lafs.readthedocs.io/en/latest/about-tahoe.html>
20. Filecoin documentation, "Proofs." <https://docs.filecoin.io/basics/the-blockchain/proofs>
21. Sia documentation, "Storing Your Data." <https://docs.sia.tech/store-your-data/setting-up-renterd/renting-storage>
22. IPFS documentation, "Persistence." <https://docs.ipfs.tech/concepts/persistence/>

**Moving a running service**

23. Clark et al. "Live Migration of Virtual Machines." NSDI 2005. <https://www.usenix.org/legacy/event/nsdi05/tech/full_papers/clark/clark.pdf>
24. CRIU, <https://criu.org/Main_Page>; Kubernetes Kubelet Checkpoint API, <https://kubernetes.io/docs/reference/node/kubelet-checkpoint-api/>
25. PostgreSQL documentation, "Log-Shipping Standby Servers," <https://www.postgresql.org/docs/current/warm-standby.html>; Patroni documentation, <https://patroni.readthedocs.io/en/latest/replication_modes.html>
26. CockroachDB replication layer, <https://www.cockroachlabs.com/docs/stable/architecture/replication-layer>; etcd FAQ, <https://etcd.io/docs/v3.5/faq/>; rqlite clustering, <https://rqlite.io/docs/clustering/general-guidelines/>
27. Shapiro, Preguiça, Baquero, Zawirski. "Conflict-free Replicated Data Types." INRIA Research Report RR-7687, 2011; a shorter version appeared at SSS 2011. <https://inria.hal.science/inria-00609399>
28. Kleppmann, Wiggins, van Hardenberg, McGranaghan. "Local-First Software: You Own Your Data, in spite of the Cloud." Onward! 2019. <https://martin.kleppmann.com/papers/local-first.pdf>
29. Trautwein, Ihle, Schubotz, Gipp. "Challenging Tribal Knowledge: Large Scale Measurement Campaign on Decentralized NAT Traversal." arXiv 2510.27500, 2025. <https://arxiv.org/abs/2510.27500>

**Connections, machines and power**

30. Ookla Speedtest Global Index, United States, September 2026. <https://www.speedtest.net/global-index/united-states>
31. FCC. Thirteenth Measuring Broadband America Fixed Broadband Report (data from 2022). <https://data.fcc.gov/download/measuring-broadband-america/2023/2023-Fixed-Measuring-Broadband-America-Report.pdf>
32. Anderson, Fedak. "The Computational and Storage Potential of Volunteer Computing." CCGrid 2006. <https://arxiv.org/abs/cs/0602061>. Javadi, Kondo, Vincent, Anderson. "Discovering Statistical Models of Availability in Large Distributed Systems: An Empirical Study of SETI@home." *IEEE TPDS* 22(11), 2011. <https://mescal.imag.fr/membres/derrick.kondo/pubs/javadi_tpds10.pdf>
33. Cox residential data usage page. <https://www.cox.com/residential/internet/learn/data-usage.html>
34. US Energy Information Administration. Electric Power Annual, Table 11.1. <https://www.eia.gov/electricity/annual/html/epa_11_01.html>

**Electricity, hardware and payouts**

35. US Energy Information Administration. Electric Power Monthly, Table 5.6.A, August 2026. <https://www.eia.gov/electricity/monthly/epm_table_grapher.php?t=epmt_5_6_a>
36. Eurostat, "Electricity price statistics," first half 2026, <https://ec.europa.eu/eurostat/statistics-explained/index.php?title=Electricity_price_statistics>; GlobalPetrolPrices, Brazil and India residential prices, March 2026, <https://www.globalpetrolprices.com/Brazil/electricity_prices/>, <https://www.globalpetrolprices.com/India/electricity_prices/> (an aggregator, not an official tariff)
37. Seagate IronWolf datasheet DS1904-23-2509US. <https://www.seagate.com/content/dam/seagate/en/content-fragments/products/datasheets/ironwolf-12tb/ironwolf-16tb-DS1904-23-2509US-en_US.pdf>
38. ServeTheHome, fanless Intel N100 appliance review, 2023. <https://www.servethehome.com/fanless-intel-n100-firewall-and-virtualization-appliance-review/4/>
39. Energy Star certified computers dataset, version 9.0. <https://data.energystar.gov/resource/rxdj-2c88.json>
40. Storj node payouts, <https://storj.dev/node/payouts>; "Rightsizing Storj network payout rates," August 2026, <https://forum.storj.io/t/rightsizing-storj-network-payout-rates/32391>
41. Storj pricing. <https://storj.dev/dcs/pricing/simplified>
42. Backblaze B2 pricing, <https://www.backblaze.com/cloud-storage/pricing>; Wasabi pricing, <https://docs.wasabi.com/docs/may-2026-wasabi-pricing>; Amazon S3 pricing, <https://aws.amazon.com/s3/pricing/>
43. Storj. "Storj announces voluntary financial restructuring," July 2026, <https://storj.io/blog/storj-announces-voluntary-financial-restructuring-to-resolve-legacy-liabilities-and-position-the-business-for-growth>; Storj forum notice of asset sale, November 2026, <https://forum.storj.io/t/storj-asset-sale-valdi-ai-object-mount-and-storj-object-storage/33047>
44. Storj community earnings estimator, linked from <https://storj.dev/node/faq/how-do-i-estimate-my-potential-earnings-for-given-amount-of-space-and-bandwidth> (assumptions, not measurements)
45. Xfinity data usage plan. <https://www.xfinity.com/support/articles/data-usage-plan>

## Appendix A. The script behind the tables

Python, standard library only. Run it to reproduce Tables 1 to 5, the shared-house example, the costs by place, and the check against Storj's published figure.

```python
from math import comb, exp, factorial

def at_least(k, n, mu):
    """Chance that at least k of n independent hosts are on, each on a fraction mu of the time."""
    return sum(comb(n, i) * mu**i * (1 - mu)**(n - i) for i in range(k, n + 1))

HOURS_PER_YEAR = 8766

print("Table 1: read availability")
for name, k, n in [("1 copy", 1, 1), ("2 copies", 1, 2), ("3 copies", 1, 3),
                   ("2-of-4", 2, 4), ("4-of-8", 4, 8), ("3-of-10", 3, 10), ("10-of-30", 10, 30)]:
    print(name, [round(at_least(k, n, mu) * 100, 7) for mu in (0.81, 0.95, 0.99)])

print("Table 2: hours a year without a majority")
for mu in (0.81, 0.95, 0.99, 0.993):
    print(mu, [round((1 - at_least(n // 2 + 1, n, mu)) * HOURS_PER_YEAR, 1) for n in (1, 3, 5, 7)])

print("Table 3: repair traffic, 2 * D / T")
for gb in (100, 500, 1000):
    for days in (30, 91, 365):
        mbps = 2 * gb * 1e9 * 8 / (days * 86400) / 1e6
        print(gb, "GB", days, "days:", round(mbps, 2), "Mbps,", round(2 * gb * 30 / days), "GB a month")

print("Shared house: three hosts at 99%, two behind one house that is out 1% of the time")
print(round((1 - 0.99 * at_least(2, 3, 0.99)) * HOURS_PER_YEAR, 1), "hours a year")

a = b = 0.99 * 0.99   # the two housemates, counted as independent machines
c = 0.99
together = a * b * c + a * b * (1 - c) + a * (1 - b) * c + (1 - a) * b * c
print("Same uptimes, no shared house:", round((1 - together) * HOURS_PER_YEAR, 1), "hours a year")

print("Tables 4 and 5: electricity, break-even and net income")
HOURS_PER_MONTH = 730
US_PRICE = 0.1816     # dollars per kWh, US residential, August 2026
PAYOUT = 1.35         # dollars per TB stored per month
for name, watts in [("extra disk", 4.0), ("mini PC + disk", 14.5), ("desktop + disk", 32.2)]:
    kwh = watts * HOURS_PER_MONTH / 1000
    cost = kwh * US_PRICE
    print(name, round(kwh, 1), "kWh,", round(cost, 2), "dollars, break-even", round(cost / PAYOUT, 2), "TB, net",
          [round(tb * PAYOUT - cost, 2) for tb in (1, 4, 8)])

print("A 15-watt node by place, in local currency per month")
for place, price in [("Kentucky", 0.1380), ("US", 0.1816), ("California", 0.2975), ("Hawaii", 0.4718),
                     ("EU", 0.2832), ("Germany", 0.3618), ("Hungary", 0.1139), ("Brazil", 0.182), ("India", 0.073)]:
    print(place, round(15 * HOURS_PER_MONTH / 1000 * price, 2))

print("Check against Storj v3.1: 20-of-40 at 10% monthly churn")
lam = 0.10 * 40
print(sum(exp(-lam) * lam**i / factorial(i) for i in range(0, 40 - 20 + 1)))
```

---

*Licensed under [Creative Commons Attribution-ShareAlike 4.0](../LICENSE).*
