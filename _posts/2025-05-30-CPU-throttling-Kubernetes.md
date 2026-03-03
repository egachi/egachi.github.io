

174472756200375

CPU utilization, observing spikes in Kubernetes workloads.
For certain high-traffic applications, we're seeing significant CPU consumption variance between pods, despite even traffic distribution.


Factors
- CPU Processors
- Load distrubutions
- 


CPU architecture can significantly impact application performance. The performance variations we're seeing with the AMD EPYC 9R14 Processor could be attributed to several factors, example:
- Thread scheduling and synchronization (race conditions, deadlocks)
- I/O operations (file system, network, database)
- CPU instruction set differences between architectures
- Memory management and garbage collection
- Background processes or system tasks

CPU throttling in containers occurs due to resource constraints set by control groups (cgroups). Kubernetes and other container orchestrators rely on cgroups to enforce resource limits. When a container attempts to use more CPU than its assigned quota, it gets throttled, delaying execution of tasks. (When containers have CPU limits defined, they will be converted to a cgroup CPU quota.)

How to Identify CPU Throttling

It’s often difficult to catch CPU throttling because it can happen even when the host CPU usage is low. It’s critical to have the right level of monitoring set up in order to see CPU throttling when it happens, or even better, before it becomes a problem.


Monitor Your cgroup Metrics
Linux cgroups provide detailed metrics about CPU usage and throttling. Look for the cpu.stat file within the container’s cgroup directory (usually under /sys/fs/cgroup):
Within the cpu.stat file there are three key metrics:

nr_throttled: Number of times the container was throttled.
throttled_time: Total time spent throttled – which I believe is in nanoseconds
nr_periods: Total CPU allocation periods.
Example:
cat /sys/fs/cgroup/cpu/cpu.stat

Output:
nr_periods 12345
nr_throttled 543
throttled_time 987654321

If nr_throttled or throttled_time is high relative to nr_periods, then you have CPU throttling on your container.


Understanding CPU Throttling Metrics: The cpu.stat (nr_periods, nr_throttled, throttled_time) metric [1] is measured at the cgroup level and indicates when containers exceed their allocated CPU limits. 


CPU Throttling Details:
    
    container id: 4d4e5219b4b986fdd575474ed13dc04cde5ef54ecd9f4b19cdaa730a46affe9c
    nr_periods: 151358
    nr_throttled: 4640
    throttled_time: 2637683010568 nano seconds
    Image: us-east-1-single-aws-registry.furycloud.io/dogstatsd:7.38.0
    Instance: i-0e118336ebe8a768e


(Throttled rate = nr_throttled / nr_periods)

Datadog Pod:
- Throttled rate: 3.06% (4640 / 151358 periods)
- Total Throttled Time: ~44 minutes

OpenTelemetry Pod:
- Throttled Rate: 3.23% (1,904/58,886 periods)
- Total Throttled Time: ~24 minutes


pprof


While throttling might seem to reduce CPU usage, it often results in longer processing times as the CPU needs more time in non-idle states to complete tasks. This can potentially lead to higher overall CPU utilization.

Next Steps: Have you implemented any APM (Application Performance Monitoring) tools? This would help identify specific operations causing increased CPU usage on dense nodes and pinpoint resource-intensive functions at the application level.


go tool pprof -http=: read-low_10-108-75-91_2-40-1_cpu_20250421143600.pprof

Operations that are consuming the most CPU on this instance

Flamegraphs


Specifically, the most resource-intensive functions:

1. Adjust the CPU limits to better match current needs [1]. 
2. Review and optimize the data transmission frequency. eg. (min_collection_interval parameter), review references [2][3]. 
3. Consider fine-tuning the sampling rates, review references [4][5]. 

Additionally, we suggest verifying that traffic distribution is balanced equally across pods to ensure uniform workload distribution.



If adjusting the current configuration limits doesn't fully address your CPU utilization concerns, one additional option you may want to consider is exploring AMD-based Node instance types https://aws.amazon.com/ec2/amd/. AMD instances have shown performance improvements across certain workloads.


References:
https://guide.aws.dev/articles/ARC4SzUcnVQgm2aJGTXIxrAg/effective-usage-of-vcpu-in-ecs-under-cpu-contention
https://guide.aws.dev/articles/AR-O0QfnZxQgufbQFarx_-Dw/to-know-how-the-cpu-resource-is-important-for-coredns-and-nodelocal-dnscache-in-the-eks-cluster
https://guide.aws.dev/articles/ARI2sfk-9ZSMeIL5yxFh4Q2w/how-to-control-linux-processes-on-eks-nodes