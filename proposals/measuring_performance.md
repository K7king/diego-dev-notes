# Performance Experimentation Plans On Diego

**Table of Contents**

- [Events](#events)
  - [Tasks](#tasks)
  - [LRPs](#lrps)
- [Metrics](#metrics)
- [Experiments](#experiments)
  - [Scaling the Deployment](#scaling-the-deployment)
  - [Tuning the Deployment](#tuning-the-deployment)
  - [Experiment 1: Fezzik](#experiment-1-fezzik)
  - [Experiment 2: Launching and running many CF applications](#cf-app-expt)
  - [Experiment 3: Fault-recovery](#experiment-3-fault-recovery)
  - [Experiment 4: Tolerating catastrophic cell and etcd failure](#experiment-4-tolerating-catastrophic-cell-and-etcd-failure)


## Events

Given the centralization of the logic on Diego, we've decided to focus our
efforts in analyzing logs straight from the BBS. Since every state transition
operation goes through it, we should have easy access to logging those and then
later processing those logs effectively. These should be accessible through the
BBS logs.

### Tasks

Running:
- Requested
- Started
- Completed

### LRPs

Running:
- Desired
- Claimed
- Running

Stopping:
- Stopping
- Stopped

On stable system, if no catastrophes happen, no events should be triggered.

These can be used to analyze both cold start smoke tests and full scale stress
tests.

## Metrics

We want to observe the running system using the following metrics. It is also
interesting to observe how they change as we're filling up the deployment.


| Quantity | Source | Metric | Pathway to Influx |
|---|---|---|---|
| cedar: Time to push CF app (no-start) | cedar report  |  | perfchug |
| cedar: Time to start CF app | cedar report  |  | perfchug |
| BBS: latency by endpoint | BBS logs |  | perfchug |
| auctioneer: latency by endpoint | auctioneer logs |  | perfchug |
| Time from desiring task to completing task (successful) | BBS logs |  | perfchug |
| Time from desiring task to completing task (failed) | BBS logs |  | perfchug |
| Time from desiring LRP to starting instance at index N | BBS logs |  | perfchug |
| auctioneer: time to place LRP instance | auctioneer logs |  | perfchug |
| auctioneer: time to place Task | auctioneer logs |  | perfchug |
| BBS: LRP convergence duration | BBS metrics | bbs.ConvergenceLRPDuration | influxdb-firehose-nozzle |
| BBS: Task convergence duration | BBS metrics | bbs.ConvergenceTaskDuration | influxdb-firehose-nozzle |
| BBS: Total Claimed DesiredLRP instances | BBS metrics | bbs.LRPsClaimed | influxdb-firehose-nozzle |
| BBS: Total Crashing DesiredLRP instances | BBS metrics | bbs.CrashedActualLRPs | influxdb-firehose-nozzle |
| BBS: Total DesiredLRP instances | BBS metrics | bbs.LRPsDesired | influxdb-firehose-nozzle |
| BBS: Total Extra DesiredLRP instances | BBS metrics | bbs.LRPsExtra | influxdb-firehose-nozzle |
| BBS: Total Missing DesiredLRP instances | BBS metrics | bbs.LRPsMissing | influxdb-firehose-nozzle |
| BBS: Total Running DesiredLRP instances | BBS metrics | bbs.LRPsRunning | influxdb-firehose-nozzle |
| BBS: Total Unclaimed DesiredLRP instances | BBS metrics | bbs.LRPsUnclaimed | influxdb-firehose-nozzle |
| Cell: Available container capacity | rep metrics | rep.CapacityRemainingContainers | influxdb-firehose-nozzle |
| Cell: Available disk capacity | rep metrics | rep.CapacityRemainingDisk | influxdb-firehose-nozzle |
| Cell: Available memory capacity | rep metrics | rep.CapacityRemainingMemory | influxdb-firehose-nozzle |
| Cell: Garden Container Creation duration | rep metrics | rep.GardenContainerCreationDuration | influxdb-firehose-nozzle |
| Cell: Rep bulk duration | rep metrics | rep.RepBulkSyncDuration | influxdb-firehose-nozzle |
| Golang metrics: GC pause time | component metrics | \*.memoryStats.lastGCPauseTimeNS | influxdb-firehose-nozzle |
| Golang metrics: goroutine count | component metrics | \*.numGoRoutines | influxdb-firehose-nozzle |
| Golang metrics: heap | component metrics | \*.memoryStats.numBytesAllocatedHeap | influxdb-firehose-nozzle |
| Golang metrics: stack | component metrics | \*.memoryStats.numBytesAllocatedStack | influxdb-firehose-nozzle |
| nsync-bulker: sync duration | nsync-bulker metrics | nsync_bulker.DesiredLRPSyncDuration | influxdb-firehose-nozzle |
| route-emitter: sync duration | route-emitter metrics | route_emitter.RouteEmitterSyncDuration | influxdb-firehose-nozzle |
| route-emitter: Total routes | route-emitter metrics | route_emitter.RoutesTotal | influxdb-firehose-nozzle |
| BOSH VM metrics: CPU system | BOSH HM | bosh.healthmonitor.system.cpu.sys | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: CPU user | BOSH HM | bosh.healthmonitor.system.cpu.user | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: CPU wait | BOSH HM | bosh.healthmonitor.system.cpu.wait | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: Load average, 1m | BOSH HM | bosh.healthmonitor.system.load.1m | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: Memory usage | BOSH HM | bosh.healthmonitor.system.mem.kb | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: Swap usage | BOSH HM | bosh.healthmonitor.system.swap.kb | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: Ephemeral disk usage | BOSH HM | bosh.healthmonitor.system.disk.ephemeral.percent | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: Persistent disk usage | BOSH HM | bosh.healthmonitor.system.disk.persistent.percent | boshhmforwarder + influxdb-firehose-nozzle |
| BOSH VM metrics: System disk usage | BOSH HM | bosh.healthmonitor.system.disk.system.percent | boshhmforwarder + influxdb-firehose-nozzle |

Success:

- Diego can run 250,000 LRPs in 1000 cells.
  - There is little or no degradation when pushing an app on an empty env as a full env.
  - The bulk loops perform within their loop interval at scale.
  - Cells can be upgrades when the system is at load with no loss of running applications.
    - Cell evacuation time on systems with X% full.
  - Diego recovers from data loss in a "reasonable" amount of time and currently running apps remain running.

### Extracting Metrics

We have previously used Loggregator to send metrics to Datadog via the
datadog-firehose-nozzle. We experienced frustration trying to generate custom
graphs that are more presentable and easier to understand outside of observing
a live deployment. For those reasons we want to have access to raw metrics
somehow as part of running the experiments described in this document.

We decided to go with InfluxDB and Grafana to solve those issues:

- A version of the influxdb-firehose-nozzle that works with current loggregator
  can be found at: https://github.com/luan/influxdb-firehose-nozzle. That fork
  updates loggregator libraries and code with most recent
  datadog-firehose-nozzle
- Grafana (http://grafana.org) can be either deployed with BOSH
  (https://github.com/vito/grafana-boshrelease) or run locally and pointed to a
  remote InfluxDB.
- InfluxDB can be deployed via BOSH
  (https://github.com/vito/influxdb-boshrelease) its configuration is really
  minimal and should straightforward.


## Experiments

### Scaling the Deployment

Let `N` denote the number of cells in the deployment.

We intend to run a total of `250 * N` app instances in the deployment. The mix of apps
is defined in the [next section](#experiment-2-apps-matrix) of this document.


The following estimates  come from an estimate of total requests per second and logs per second in the system (where logs = log and metric messages going through the Loggregator
system).

We initially estimate the following VM types and resources for the CF, Diego, and database VMs in the deployment.
Some VMs are horizontally scalable, and so will scale in proportion to the `N` cell VMs:

| Name | Quantity | VM type (AWS) |
|------|----------|-----|
| CC API | N / 10 | m3.large: 7.5 GB, 2 CPU |
| CC Worker | N / 20 | c3.large: 3.75 GB, 2 CPU |
| Doppler | N / 10 | c3.large: 3.75 GB, 2 CPU |
| Traffic Controller | N / 40 | c3.large: 3.75 GB, 2 CPU |
| Router | N / 10 | c3.xlarge: 7.5 GB, 4 CPU |
| UAA | N / 250 | m3.large: 7.5 GB, 2 CPU |
| Diego Cell | 1.25 * N | m3.large: 7.5 GB, 2 CPU + 100GB gp2 ephemeral disk (see [128239799](https://www.pivotaltracker.com/story/show/128239799)) |
| CC Bridge | N / 100 | c3.large: 3.75 GB, 2 CPU |
| Access VM | N / 250 | m3.medium: 3.75 GB, 1 CPU |

Some VMs host services that are scaled out only for redundancy across availability zones, but may require vertical scaling as `N` increases.
We estimate the following capacities for these types of VMs at the `250 * N` app-instance scale:

| Name | Quantity | VM type (AWS) |
|------|----------|-----|
| etcd | 3 | c3.large: 3.75 GB, 2 CPU |
| consul | 3 | c3.large: 3.75 GB, 2 CPU |
| NATS | 2 | c3.large: 3.75 GB, 2 CPU |
| Postgres | 1 | c3.large: 3.75 GB, 2 CPU |
| Diego Brain | 2 | c3.xlarge: 7.5 GB, 4 CPU |
| Diego Database | 2 | c4.4xlarge: 30 GB, 4 CPU |
| Route-Emitter | 2 | c3.large: 3.75 GB, 2 CPU |

When using an HA CF-MySQL deployment as the Diego datastore, we expect to require the following sizes at the `250 * N` app-instance scale:

| Name | Quantity | VM type (AWS) |
|------|----------|-----|
| MySQL Node | 2 | c3.2xlarge: 15 GB, 8 CPU |
| MySQL Aribtrator | 1 | m3.medium: 3.75 GB, 1 CPU |
| MySQL Proxy | 2 | c3.xlarge: 7.5 GB, 4 CPU |

When using a single Postgres deployment as the Diego datastore, we expect to require the following sizes at the `250 * N` app-instance scale:

| Name | Quantity | VM type (AWS) |
|------|----------|-----|
| Postgres (Diego) | 1 | c3.2xlarge: 15 GB, 8 CPU |

### <a name='tuning-the-deployment'></a>Tuning the Deployment

In order to perform correctly at scale or to generate the necessary logs for analysis, certain services in the CF and Diego deployments require additional tuning.

- Set the `diego.executor.memory_capacity_mb` BOSH property to `32768` so that the Diego cells are consistently configured with 32 GiB of memory to allocate.
- Set `garden.max_containers` to `384` and `garden.network_pool` to `10.254.0.0/20` to give Garden enough container headroom when running at high container densities.
- Set `diego.bbs.enable_access_log` to `true` to enable the BBS access log for later analysis by perfchug.


### Experiment 1: Fezzik

Fezzik exercises the BBS API and launches very many Tasks and LRPs in tandem.
These Tasks and LRPs are very lightweight so Fezzik is isolated to benchmarking
scheduling Tasks/LRPs and spinning up/tearing down containers.

This is supposed to be a simple "start as many instances as fast as possible"
kind of test.  These instances should consist of very minimal workloads.
The workload in question should have little to no ongoing activity and should
be started up all at the same time to measure scheduling performance and find
general bottlenecks with basic functionality of the system.

For this experiment we plan to start (for `N` = number of cells in the deployment).

- `N * 10` Tasks
- `N * 10` LRPs
- `N * 50` Tasks
- `N * 50` LRPs
- `N * 100` Tasks
- `N * 100` LRPs
- `N * 200` Tasks
- `N * 200` LRPs


### <a name='cf-app-expt'></a>Experiment 2: Launching and running many CF applications

Apps are defined as different modes of [this app](https://github.com/cloudfoundry/diego-stress-tests/tree/master/assets/stress-app).

The mix of apps can be found in the [manifest
template](https://github.com/cloudfoundry/diego-stress-tests/tree/master/assets/stress-app/manifest.yml.template)
in the app's repository.

<span id="experiment-2-apps-matrix"></span>Initial mix of apps:

| Name           | Req/s     | Crash?   | # of Apps  | Instances/App  | # of Instances   | Memory/Instance (MB) | Total Memory (MB) |
| -----------    | ------    | -------- | ---------- | -------------- | ---------------- | -------------------- | ----------------- |
| *light-group*  | 0.3       | no       | 1          | 4              | 4                | 32                   | 128               |
| *light*        | 0.3       | no       | 9          | 1              | 9                | 32                   | 288               |
| *medium-group* | 0.6       | no       | 1          | 3              | 3                | 128                  | 384               |
| *medium*       | 0.6       | no       | 6          | 1              | 6                | 128                  | 768               |
| *heavy*        | 0.75      | no       | 1          | 1              | 1                | 1024                 | 1024              |
| *crashing*     | 0         | 30s-360s | 2          | 1              | 2                | 128                  | 256               |
| **Total**      | **10.45** |          | **20**     |                | **25**           |                      | **2848M**         |

For an N-cell deployment, push 10 * N batches of the above mix, N at a time, for a total of 200 * N LRPs with a total of 250 * N instances.
These instances will allocate a total of 28.48 * N GB of memory, with non-crashing app instances accounting for 25.92 * N GB of this allocation.

For a 1000-cell deployment, this results in 250,000 LRP instances from 200,000 LRPs (10,000 pushes of the above mix).
These instances will allocate a total of 28,480 GB of memory.
Non-crashing app instances account for 25,920 GB of this allocation.

Crashing apps will crash at a random period from 30s to 6 minutes to make their
crash-count reset once in a while, so that the deployment contains app instances that continue to crash indefinitely.

We will push the apps N batches at a time, to make it easier to collect environmental data at each 10% increase in baseline load, and to fail faster if the environment is overloaded. After each push of 25 * N instances, we will collect logs from the BBS and auctioneer from that push timeframe, convert them and the push logs into metrics, and import them into the influxdb instance for analysis.

Once the system is filled to this initial baseline, we will push an additional N / 25 batches of non-crashing apps in parallel, measuring the time it takes these apps to stage and then start running. For a 1000-cell deployment, this is an additional 1000 instances. We will then monitor these apps and make sure that they continue to run as expected and to remain routable.

<span id="experiment-2-apps-matrix-extra"></span>Mix of additional apps:

| Name           | Req/s  | Crash?   | # of Apps  | Instances/App  | # of Instances   | Memory/Instance (MB) | Total Memory (MB) |
| -----------    | ------ | -------- | ---------- | -------------- | ---------------- | -------------------- | ----------------- |
| *light-group*  | 0      | no       | 1          | 4              | 4                | 32                   | 128               |
| *light*        | 0      | no       | 9          | 1              | 9                | 32                   | 288               |
| *medium-group* | 0      | no       | 2          | 2              | 4                | 128                  | 512               |
| *medium*       | 0      | no       | 7          | 1              | 7                | 128                  | 896               |
| *heavy*        | 0      | no       | 1          | 1              | 1                | 1024                 | 1024              |
| **Total**      | **0**  |          | **20**     |                | **25**           |                      | **2848M**         |



### Experiment 3: Fault-recovery

After a day, kill N/10 cells across the AZs and see how long it takes to recover the missing applications.

We'll want:
- Plots to see how long recovery takes
- Convergence logs to analyze how long convergence takes to handle this scenario.

Leave those cells dead for the next experiment.

### Experiment 4: Tolerating catastrophic cell and database failure

Kill another N/10 cells, again roughly balanced across AZs.
At this point, the workload will exceed capacity.
Instances that are running should continue to run, except for
instances of crashing apps that have not yet crashed when these cells are killed.
Instances that are not running should not be re-placed,
except as some capacity opens up from instances that crash as configured.
Verify that we observe this behavior.

Kill all the database nodes.  Make sure they (and the N/5 dead cells) are down
for a while, long enough for several convergence ticks.

`bosh cck` to recover the missing Cells and Database VMs.  How long does it
take for us to recover?  How does the routing table handle this?

## GCE Information

### Instance Sizing

| Deployment | Job | Scalability H/V | Number of Instance |Resource Pool | 5000 Test Instance Size  |
| ---------- | --- | --------------- | ------------------ | ------------ | ------------------------ |
|CF| api | H |2 |large | n1-standard-2|
|CF| api_worker | H | 2 | small | n1-standard-2|
|CF| blobstore | V | 1 | medium | n1-standard-2
|CF| consul|V | 3 | small | n1-standard-2|
|CF|doppler|H|2|medium|n1-standard-2|
|CF|etcd|V|3|medium|n1-standard-2|
|CF|ha_proxy|H|2|ha_proxy|n1-standard-4|
|CF|loggregator_trafficcontroller|H|1|small|n1-standard-2
|CF|nats|V|2|medium|n1-standard-2
|CF|postgres|V|1|postgres|n1-standard-8
|CF|router|H|2|router|n1-standard-4
|CF|uaa|H|2|medium|n1-standard-2
|Diego|access|H|2|access|n1-standard-1
|Diego|brain|V|2|brain|n1-standard-4
|Diego|cc_bridge|H|2|cc_bridge|n1-standard-2
|Diego|cell|H|25|cell|n1-standard-2
|Diego|database|V|2|database|n1-standard-8
|Diego|route-emitter|V|2|route-emitter|n1-standard-4
|Influx|grafana|V|1|standard|n1-standard-4
|Influx|influxdb|V|1|standard|n1-standard-4
|Influx|influxdb-firehose-nozzle|V|1|standard|n1-standard-4
|MySQL|arbitrator|V|0|
|MySQL|mysql|V|0|
|MySQL|proxy|V|0|
|Perf|cedar|V|1|perf|n1-standard-2|
|Diego-Postgres|postgres|V|0|

### Performance Tuning Comments


1. Set the `ha_proxy.log_to_file` property to true in the manifest. This makes the HA Proxy store logs on `/var/vcap/sys/log` instead of `/var/log`.
1. Validate size of Influxdb VM during larger runs--it seems CPU-bound.
1. Validate size of Perf VM during larger runs as disk can fill due to logs and results.
