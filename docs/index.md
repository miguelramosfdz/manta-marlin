---
title: Marlin
markdown2extras: tables, code-friendly
apisections: Task Control API
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2016, Joyent, Inc.
-->

# Marlin

Marlin is a facility for running compute jobs on Manta storage nodes.  This
document describes the desired end state for version 1.

**For an overview of what Marlin is and how users interact with it, see the
public Manta user documentation, then the Manta operator documentation in
manta.git.  Really, go read that first.  This document assumes familiarity with
that, including how the rest of the Manta object store works.**


# First principles

Marlin provides two main functions:

* a non-interactive Unix shell environment for doing "work" on Manta objects as
  local files
* a framework for distributing such work to the right physical servers, tracking
  which pieces are complete, capturing the output, and repeating the whole
  process to facilitate multi-phase computation on objects at rest

Example use cases include:

* log processing: grepping through logs, reformatting log entries, or generating
  summary reports
* image processing: converting formats, generating thumbnails
* video processing: transcoding, extracting segments, resizing
* "hardcore" data analysis, using NumPy, SciPy, R, or the like
* SQL-like queries over structured data (similar to what Hive provides for
  Hadoop)

Since the primitive abstraction is the Unix shell, lots of these operations can
be done with built-in tools like `awk`, `grep`, `sort`, `json`, `bunyan`, and so
on.  We also provide a rich suite of software like ImageMagic, ffmpeg, and so
on, as well as runtime environments like Perl, Python, Java, R, and so on so
that users can run their own software, too.

Besides that, using Unix as the basic abstraction makes it possible to develop
software locally and then run it in Marlin with little to no changes required.

With rich access controls in Manta, it will be possible to run compute on other
users' data that's been made available to you.  This may be especially useful
for government organizations with lots of public data (e.g.,
http://community.topcoder.com/pds-challenge/).  In version 1, the only access
controls available in Manta are "private to my account" and "public to the
internet".

Marlin must be easy to use, monitor, and debug.  As a benchmark, "Word frequency
count", the MR equivalent of "hello world", should be trivial to run from the
command line.


# Examples

There are lots more examples in the public (muskie) documentation.

## Build system

Marlin could be an effective distributed build system, at least for Joyent
internally.  Using Marlin this way guarantees that each repo is built in a clean
zone with whichever dataset it requires, the builds themselves are automatically
distributed for parallelism, and the artifacts end up in Manta where we probably
want them anyway.

The main challenge is that git repos themselves would be hard to store in Manta,
since they're typically a whole tree of files presumably updated with partial
writes.  But git repos can be bundled into a single file using "git bundle", and
the resulting bundle stored in Manta.  If we add a post-commit hook to our git
repos that save a "git bundle" into Manta, then the distributed build system is
a pretty simple map job:

    "phases": [ {
        "image": "smartos-1.8.4",
        "exec": "git clone $mr_input_file repo && cd repo && make &&
            mpipe < build/dist/latest.tar.bz2"
    } ]

The input objects would be the git bundle for each repo in Manta.  If different
repos needed to be built with different images, those would have to be separate
jobs.


# Marlin implementation

The following sections are somewhat out-of-date and subject to change.  When in
doubt, check the source.


## APIs

End users manage compute jobs through the "jobs" entry points under the main
Manta API.  See the Manta API documentation for details.

Internally, Marlin supervisor processes (which are responsible for managing the
distributed execution of jobs) communicate with agents (which actually execute
user commands) through small JSON records stored in Moray, the Manta metadata
tier.  This mechanism is documented in the supervisor source code.

On each physical server where jobs are executed, the Marlin agent communicates
with a lackey running inside each compute zone using a private HTTP API which is
documented in the agent source code.  This API is a superset of the public Manta
API, allowing allowing users to securely perform the usual Manta operations from
within jobs.  The non-public entry points in this API are private implementation
details, though future versions may stabilize these for consumption by
higher-level frameworks.

All DateTimes in all APIs use W3C DTF in UTC ("YYYY-MM-DDThh:mm:ss.sZ").


## Network architecture

Marlin makes use of two generic Manta components:

* muskie, the Manta web tier, which handles the Jobs API (requests to POST new
  jobs and GET job state).
* moray, where all Marlin state is stored.

On top of these, Marlin adds two components:

* Job supervisor tier, which locates new and abandoned jobs in Moray, assigns
  task groups to individual storage and compute nodes, and monitors job
  progress.
* Compute node agents, which execute and monitor tasks and report progress and
  completion.

There's a cluster of web tier servers and job supervisors in each datacenter,
but there's no real DC-awareness among these components.

**Job state**

To keep the service resilient to partitions and individual node failures, **all
job state is stored in Moray** (i.e., all components are essentially
stateless), and **all communication happens via Moray**.  Rather than talking
to each other directly, each component drops state changes into Moray and polls
for whatever other state changes it needs to know about.  This model makes it
relatively easy to deal with individual component failure and to scale
horizontally.  See "Invariants and failure modes" below for details.

Recall that Moray spans datacenters, is eventually consistent, and remains
available in the case of minority DC partitions.  Using Moray in this way also
implies that Moray becoming unavailable makes Marlin unavailable, and that
Moray must be able to keep up with the Marlin workload.  See "Scalability"
below.

Job state is stored across several Moray buckets with indexed fields that allow
the web tier, job supervisors, and agents to poll on the subset of jobs and job
state that they care about.  For details on these buckets and the records
contained therein, see the documentation in lib/schema.js.  A design constraint
is that the size of every JSON record in the system must be bounded (and should
be fairly small).  This applies to records received from and sent back to users
as well as records used internally.


**Web tier**

muskie retrieves current job state by querying "marlinJobs" for the job with the
given jobId.  When a new job is submitted, muskie drops the job definition into
the jobs bucket.


**Job supervisors**

To synchronize changes to job state, each job is assigned to one supervisor at a
time.  This assignment is stored in the job object in Moray.  An assignment is
valid as long as the job's modification time (mtime) is within the last
WORKER\_TIMEOUT seconds.  Workers save their jobs periodically to indicate
liveness.  They also poll periodically for unassigned jobs (to pick them up) and
jobs that have not been updated within the timeout period (to handle supervisor
failure).

Managing the streaming, distributed execution of jobs is a complex process.  The
basic idea is that supervisors query Moray for state changes needing attention
(new jobs, new job inputs, and completed tasks) and process them by issuing new
tasks, updating job state, and so on.  Each input is processed by locating the
corresponding object and writing out either a task assignment (for map tasks) or
a taskinput record (for reduce tasks).  For details, see the supervisor source.

**Compute node agents**

Like supervisors, the per-storage-node agents periodically poll Moray for state
changes needing attention (newly assigned tasks, cancelled tasks, and so on) and
process them by scheduling execution.  Agents are responsible for assigning
tasks to zones, managing the lifecycle of those zones, keeping track of output,
and proxying front-door requests between zones and Manta.  As tasks complete
(successfully or otherwise), the agent writes new updates to Moray that will be
processed by the supervisor.


**Analysis of distributed system failure modes**

Invariants:

* All state is stored in Moray, so all components are stateless.
* At most one supervisor can ever "own" a job.  Only the owner of a job can write
  task assignments or update the job record in "marlinJobs".
* To a first approximation, a component is partitioned iff it cannot reach
  Moray.  This is an oversimplification because Moray itself is sharded, so it's
  theoretically possible to be partitioned from one shard but not another.
  This case is very unlikely, since each shard has a presence in each DC, but
  the system will handle this.  Most importantly, the only partitioning that
  matters is that between a component and a given Moray shard.

If a user's code fails within a task (e.g., dumps core), and the on-node retry
policy has been exhausted, the lackey reports the task as failed, and that will
become visible in the final job state.  It's the user's problem at that point.

If a compute node agent fails transiently (e.g., dumps core and restarts), fails
for an extended period of time, or becomes partitioned from its Moray shard, any
running tasks are aborted by the supervisor.  The supervisor will retry aborted
tasks up to once.  This retry should be invisible to the user's code and the
rest of the system, modulo increased task completion time.  Agents ignore tasks
dispatched before they began running, which covers the case where the agent
actually crashes.  To cover the case where the agent simply becomes partitioned,
the supervisor explicitly cancels tasks that it has retried elsewhere.

If a job supervisor fails transiently, it can pick up exactly where it left off,
since all state is stored in Moray.  This should be invisible to the rest of the
system, modulo slightly increased job completion time.  If a job supervisor
fails for an extended period, another job supervisor will pick up its jobs.
This becomes exactly like the case where a supervisor fails transiently: the
state in Moray is sufficient for the new supervisor to pick up where the old one
left off.

If a job supervisor becomes partitioned from one or more Moray shards, it will
stop processing updates (since it won't see them).  After the job timeout
elapses, the supervisor will stop trying to save the record and another
supervisor will pick up the job.  This looks just like a supervisor failure.

Web tier nodes are completely stateless.  Any failure results in failing any
in-flight requests.  Persistent failures have no additional impact as long as
enough nodes are in service to handle load.  Partitions between the web node and
the Moray shard storing job records would result in failed requests.

If an entire datacenter becomes partitioned from the rest, then its web tier
nodes certainly won't be able to service any requests.  All of its job
supervisors become partitioned (see above -- the work should eventually be
picked up by supervisors in other datacenters).  All of its compute node agents
effectively also become partitioned, and their work *may* be duplicated in other
datacenters.  Importantly, jobs submitted to the majority partition should
always be able to complete successfully in this state, since there's always
enough copies of the data to process the job.

**Scalability analysis**

Being stateless, web tier nodes can be scaled arbitrarily horizontally.  If load
exceeds capacity, requests will queue up and potentially eventually fail, but
alarms should ring long before that happens.

Similarly, job supervisors can be scaled arbitrarily horizontally, and capacity
monitoring should allow administrators to ensure that sufficient capacity is
always available.  The failure mode for excess load is increased job completion
time.  Task updates may accumulate in Moray faster than the supervisors can
process them, but the total space used in Moray is the same regardless of how
fast the job supervisors process them, so there's no cascading failure.  If job
supervisors cannot update their locks in Moray, the system may thrash as
supervisors try to take each others' jobs, but this would be very extreme.

Compute nodes rely on the OS to properly balance the workload within that box.
They may queue tasks after some very large number of concurrent tasks, but it's
not anticipated that we would ever reach that point in practice.  If this
becomes a problem, job supervisors could conceivably redispatch the task to
another node with the same data available.  If this truly became a serious
problem, we could support compute agents that download data, rather than
operating locally (just as reducers already do).

Given that there's enough capacity in each of the Marlin tiers, the remaining
question is Moray.  Moray will be sharded, and we will say that each compute
node reports its tasks to exactly one (uniformly distributed) shard.  All job
state will be stored on a single shard, since the number of updates is
relatively minimal.  We should be able to reshard to scale this out.

***Future ideas***

Each compute node can batch its Moray updates from all tasks for up to N
seconds, sharply reducing the number of requests required.  Similarly, job
supervisors can batch job state updates for many jobs up to N seconds.  Batching of
"location" GET requests is discussed above, under "job supervisors".  Thus, the
total number of Moray-wide requests should be boundable to something like (# of
supervisors \* # of compute zone lackeys) / (N seconds), where N is how frequently
each agent updates Moray.  These requests should mostly be distributed across
all shards.

Batching requests for the same record over multiple seconds reduces both network
throughput and database operations, as multiple state changes can be combined at
the source.

Batching requests over multiple records helps network throughput, but not the
total number of database operations, since each record must be operated on
separately.  The total number of database operations could still be around (# of
active tasks \* # of agents \* # of active jobs \* # of job supervisors).  Again,
these are distributed across many Moray shards, which can be scaled
horizontally.


## Cancellation

To cancel a job, the web tier writes an update to the job record.
Asynchronously, the job supervisor will need to be polling on the job state
occasionally to notice the cancelled flag.  It propagates this to individual
task assignments, where the compute node agents will notice that the job has
been cancelled and abort them.  Eventually, all work on the job will complete,
though this may take some time to propagate through the system.


# Compute metering

## Principles and design goals

We want to meter each task run based on several factors, including type (map or
reduce), memory usage, filesystem operations, CPU time used, and elapsed wall
clock time.  Since tasks can take a very long time, we want to meter them as
they run, not wait for them to finish.  In the future, we may want to meter jobs
themselves to disincentivize people from leaving them "open" and idle, but it
would probably be better to just reduce the cost of such jobs.

The most important use case is generating per-user (and possibly per-job)
summaries of resources used, matching that to a pricing schedule, and generating
a bill.  The data should also support reports on resources used by-user, by-job,
and so on.  We may ultimately even expose such reports to users, but they will
certainly be valuable for engineering to understand how the system is being
used.

It may also prove valuable to be able to graph tasks executed per unit time,
number of tasks in flight when an agent crashes, and so on.

The metering data collected by Marlin should be rich enough to support whatever
billing policy we choose to implement, but the details of that policy should be
kept out of Marlin.  The data should support billing on any of the resources
mentioned above, and it should also support a pricing schedule where the cost of
resources increases as the ratio of resources used to input size decreases
(i.e., less efficient tasks are more expensive).

An important assumption of this design is that there is no easy way to avoid
billing customers for execution time that was wasted due to system failures
outside of their control.  The assumption is that such failures will be rare,
and we can deal with significant cases on an ad-hoc basis by issuing credits
after the fact.


## Data collection

The Marlin agent running on each compute node is the only trusted component that
can collect the desired execution metrics.  The agent will record this data into
its log, and the metering component will scan this log (just as it does for
other Manta components) to generate summary reports and eventually bills.

An important design goal is that the metering component be able to scan the log
with constant memory usage.  Aggregated summaries obviously require additional
memory, but it should not be necessary in the common case to keep parser state
while processing the log.  This implies that each record contains *deltas* in
key metrics since the previous record, rather than cumulative totals.  This way,
records can be processed without any extra context.   The meterer doesn't need
to worry about keeping state across rotated logs or anything like that.

The main records used for metering are **task-checkpoint** and **task-done**
events, which both indicate resources used during task execution over a given
period.  Every *completed* task will have a **task-done** record.  Tasks *may or
may not* have any **task-checkpoint** records.  The agent will periodically
(say, every 5 minutes) write these out for any tasks which are currently
running.  If a task straddles two of these 5-minute windows, even if it runs for
a short period, it may get a **task-checkpoint** record.  Otherwise, it will
not.  Thus, **task-checkpoint** and **task-done** records may cover any amount
of elapsed time.

The agent also emits **agent-start** and **task-start** records that are not
used for metering directly.  They're present so that we can audit why certain
tasks may not have any **task-done** record (which should only be because the
agent crashed while processing them).


## Metering record format

The specific metering record format is described in lib/meter.js.  They're
bunyan log records with high resolution timestamps, jobids, taskids, and
resource utilization statistics.

**task-checkpoint** and **task-done** records logically both contain both deltas
and cumulative statistics.  As an optimization, either the delta or cumulative
stats may be missing, in which case it's assumed that they're the same.  (For
example, the first checkpoint record has no deltas, nor does a "done" record for
a task that had no checkpoints.)

**Timestamps vs. high resolution timestamps**: each record includes both a
wall timestamp and a system high-resolution timestamp.  Some of them also
include a high-resolution delta.  There are several things to be aware of:

* The only timestamps that should be used for metering are the high-resolution
  *deltas*, which are normal integers denoting nanoseconds elapsed.
* **The wall timestamps are subject to clock drift and should not be used to
  measure elapsed time.**
* The high resolution timestamps are arbitrary nanosecond values.  They're only
  valid for comparison between "agent-start" records, and they're 64-bit values
  represented as a two-array tuple (see Node.js's `process.hrtime()`).  As such,
  these are only used for reference and auditing.

## Log consumers (meterer)

The main use case is the meterer scans these logs to generate a report by jobid.
Jobids can later be joined with moray to figure out which user owned them.

If costs are uniform over the lifetime of the task, then the meterer only needs
to keep in-memory counters for the resources we want to bill on, indexed by
jobid and billing period.  It would maintain a hash with keys of the form
`($jobid, "2013-02-28T01")`, denoting resource usage for `$jobid` during a
specific hour.  Within each bucket are the counters we bill on, including wall
time spent (which may exceed an hour, if tasks run in parallel), CPU time used,
and so on.  The counters all start at zero, and they're bumped by the values in
each **task-checkpoint** and **task-done** record.

Since each record is context-free, this process can be almost arbitrarily
parallelized: such hashes can be computed for arbitrarily small chunks of log
entries, then aggregated up.  The final hash can be joined with Moray to
associate jobs with users and present per-user, per-job, per-time-period reports
of resource usage.

If costs vary over the lifetime of the task, then the aggregation needs
an additional key based on the buckets of the pricing schedule. For example, if
we say that cost per second goes up as the number of seconds goes up, then the
hash keys would look like `($jobid, "2013-02-28T01", <1s)`,
`($jobid, "2013-02-28T01", 1-2s)`, and so on.  With this scheme, when processing
each record, it's a little harder to figure out which buckets' counters to bump.
Either the record itself needs to include the cumulative time elapsed (in
addition to the delta), or else the meterer would need to keep per-task state
in-memory and roll it up to the per-job state only when it has found all
**task-checkpoint** and **task-done** events for a given task.  This latter
approach would be expensive, since the required records could span a large
number of different log files, which constrains how different log files can be
processed in parallel.

It's possible that the buckets wouldn't be raw seconds, but instead would be a
function of the size of the input object.  This would work basically the same
way, but the keys would be symbolic (e.g., "first schedule tier", "second
schedule tier", and so on), and the appropriate key would be computed on a
per-task basis based on the size of the object.  This works fine for map tasks,
where the input size (and therefore appropriate pricing tier) is known when the
task starts running, but it doesn't work for reduce, where the input size is not
known until possibly much later than that.  If we want to bill this way for
reduce tasks, we'll have no choice but to take the more expensive, less
parallelizable approach of not metering until we've seen all of the records for
a task.


## Other consumers

As an audit of the metering system, it would be good to periodically scan all
tasks (scoped to a jobid or an interval) that ever started running and make sure
we found audit records for them, followed either by a **task-done** event or an
**agent-start** event.  This report would be more expensive than routine
auditing, since it has to track more in-memory state, but it wouldn't need to be
run frequently.

Using the metering data, we could also generate reports of things like:

* distribution of task execution time
* distribution of CPU utilization during task execution
* distribution of filesystem operations executed during task execution

Such reports may use the cumulative metrics reported only in **task-done**
events.  Resource utilization may show up in the wrong time period, but there's
no need to deal with combining partial results for a task stored in multiple
different log files.


## Metric format

The current plan is to collect the following information for the metrics fields
above.  As described above, these may be cumulative or deltas, depending on
context.

* elapsed high-resolution time, using `process.hrtime()` (`gethrtime(3C)`)
* memory usage: all data in the `memory_cap` kstat
* cpu usage: all data in the `zone_misc` class kstat
* vfs usage: all data in the `zone_vfs` kstat
* zfs usage: all data in the `zone_zfs` kstat

The "metrics" object will have top-level fields for "hrtime", "cpu", "vfs", and
"zfs".  Each next-level object will contain fields from the kstats.


## Design choices

**Space usage**: We've erred on the side of including too much data (i.e., both
delta and cumulative data) in order to make the metering and reporting processes
efficient and highly parallelizable.  We've heard too many horror stories about
hourly reports that take almost an hour to run, so we want to start with as
efficient an architecture as we can.  We've also biased towards including more
metrics to allow us to run more historical queries in the future.

**Metrics**: Are there other metrics we want to include here?  What about input
size (that is, object size for map, and total input size for reduce)?
Compression ratio (for map only)?

**Large numeric values**: JavaScript is only capable of storing numbers with up
to 53 bits of precision.  For nanosecond timestamps, this corresponds to a
little over three months.  For byte counters, this is 8PB.  I'm leaning towards
saying that both node-kstat and the log records contain two-tuples for numbers
over a certain configurable size, and for testing we can set that threshold to 0
so that we ensure that all code paths handle both representations of numbers.

This comes up in a few places: in the implementation, kstats are reported with
numeric types, so we'll be in trouble if any counter that we're recording
exceeds these values.  On a 32-CPU system, a user could conceivably overflow
this in CPU time in about 3.5 days, though they'd have to own the whole box.

Assuming we can process these in the implementation, we also have the issue of
dealing with them on the wire.  In cases where we report deltas, this should
never be an issue: assuming a 5-minute (or even hourly) reporting period, we
shouldn't be able to overflow these values.  However, there are places where we
report summary data, and we'll have the same issue described with kstats above.

For both of these cases, the obvious options are replace numeric values with
two-tuples when they get large, or to cap tasks at some reasonable upper bound
of execution time that will not allow them to overflow anything.  3 days would
probably do it.  This would fix the on-the-wire problem, but not the kstat
problem, since kstats aren't always reset between tasks.  We'd still need to do
the two-tuple approach internally, or reboot a zone if the counter could
overflow during the next task.

On the plus side, once we know these values can be encoded in JSON, they should
be readable by any JavaScript implementation.


## Alternative approaches

As alluded to above, this entire proposal assumes that users are billed for
tasks whose results are thrown away due to internal failures (i.e., box panic,
agent crash, and so on).  There's no objectively correct answer to this.  It's
expected that such failures would be rare, and in the current proposal, the cost
is explicit and amortized across customers in proportion to usage.  Reversing
this assumption would amount to increasing our own costs, which would also
presumably be amortized across all users in proportion to usage (but
implicitly).  In both cases, the expectation is that such costs are in the
noise.  If that's not the case (due to a significant outage), we can issue
credits to affected users (as competitors have done after significant outages).
(Note that there are other uncommon cases where users pay for our failures, as
in when muskie (or any JPC service) pitches errors and the user is paying for
the compute required to retry requests.)

While one can argue that this assumption doesn't have significant impact on user
costs, it has a very significant impact on the architecture of metering.  Since
the result of the task determines whether it's billable, we cannot bill a task
until the result is known.  To avoid letting people "borrow" compute (e.g.,
running a single task for days, weeks, or months), we'd need to bound the
maximum task execution time to something like 1 day.  This isn't a huge deal,
but imposes an arbitrary limit on the system that will surely be too large for
some cases and too small for others.

Additionally, relaxing this constraint actually imposes significant constraints
on the metering component (the component that scans Marlin activity logs): in
particular, it has to resolve multiple successful retries for the same task (as
in the case where an agent becomes partitioned), which may span multiple hours'
log records from multiple systems.  It has to keep in-memory state while it
scans the logs so that it can "dedup" unique executions of the same logical
task.  We'd have to require that retries happen within a window of N hours from
the original attempt (and somehow ensure that), and the metering component would
have to look back at window of N hours' worth of logs across all storage nodes
at once, instead of being arbitrarily parallelizabe.  This is all doable, but
*significantly* less scalable, more complex, and more time-consuming (which
means more expensive, since that's time we're not selling to customers).

Finally, if someone had a way to trigger an internal failure (e.g., crashing an
agent), they would be able to get arbitrary compute for free by doing the
compute, saving the results somewhere, and then inducing the failure.

Many of these issues would be resolved by "crediting" users for retries (rather
than avoiding billing them in the first place), but team experience suggests
that this approach is prohibitively painful.


# Future work

* flesh out maggr
* apache2json?
* Crazy idea #1: eliminate the supervisor by writing a library to lookup input
  objects and dispatch them to the next phase.  Have muskie do this for phase 0,
  and agents do this for subsequent phases.  (Should this be a service instead
  of a library?)
* Crazy idea #2: make things pushed based for the common case, resorting to
  polling only as a catch-all.  Once polling is cheap, eliminate "state" for
  jobs entirely -- there's just (have all inputs submitted so far been
  processed?)

## Node, Python, Ruby, etc. framework

The Jobs API is built in terms of Unix shell commands for generality.  This
makes it easy to run simple jobs like "grep" and "wc".  For slightly more
complex cases, custom utilities can be built in the Unix style and composed with
existing system utilities without writing significant new code.  For more
complex cases you want entirely custom code, you can simply run your program via
the "exec" string.

We may also provide a higher-level interface for Node.js, so that people that
want to build jobs with custom code can use Node easily.  We could have
consumers implement an interface like:

    function processObject(file, callback) { ... }

and we'd provide convenience routines for API operations (like "taskParams()",
"commit()", "abort()", and so on).

Without fleshing this out completely, we know such an interface can be
implemented without changes to the Jobs API.  When people want to use the Node
API, we tell them to write phases like this:

    "phases": [ {
        "assets": [ "my/npm/package" ],
        "exec": "mnode /assets/my/npm/package"
    } ]

"mnode" is an executable Node.js program included with the zone that installs
the given npm package (and therefore its dependencies, too) and then invokes our
Marlin interface in that package.

If we care to, we can do something similar for Python, Ruby, R, or any other
environment.  It would be just as easy for third parties to build their own.
Users would only need to reference a third party asset:

    "phases": [ {
        "assets": [
            "/third-party/public/framework/start.sh"
            "/dap/stor/script"
        ],
        "exec": "/assets/third-party/public/framework/start.sh /assets/dap/stor/script"
    } ]
