# Potential Pitfalls in Conducting RFC2544 Throughput Test within OpenShift

Authors: Jianzhu Zhang

RFC2544 is an established industry benchmarking methodology utilized to assess performance metrics of network devices, including throughput, latency, and packet loss.

This blog will concentrate on RFC2544 throughput tests and offer insights into optimizing the test results.

## Test Topology and Toolings

<img width="1089" alt="Test connection" src="https://raw.githubusercontent.com/redhat-eets/netgauge/main/diagrams/RFC2544-direct-connection.png">

In this testing setup, the host housing the TestPMD container serves as the device under test (DUT). This DUT operates on a single-node OpenShift (SNO) installation.

The Trafficgen container operates on a host equipped with RHEL 8.6.

For readers interested in replicating the process, they can refer to the instructions in the following repository to construct their own test images and execute the RFC2544 test: https://github.com/redhat-eets/netgauge

In the same repository, the SNO manifests necessary for configuring the system to run the TestPMD container can be located: https://github.com/redhat-eets/netgauge/tree/main/manifests/single-node-openshift

Plentiful information regarding these manifests can be sourced from the official OpenShift documentation. Rather than duplicating this information, our focus will be solely on exploring possible challenges encountered when conducting RFC2544 tests within the OpenShift environment.


## Noisy neighbor problem

The DPDK worker threads run in the poll mode, and each worker thread expects to take the full core cpu time. When hyperthreading is enabled,  a worker thread that is assigned to an isolated CPU is not necessarily guaranteed with the full core cpu time, as the sibling cpu thread of the isolated cpu can be allocated to other threads and share the same core with the DPDK worker thread. When this happens, the performance of the DPDK worker thread running on a CPU will be impacted. This is referred to as by the noisy neighbor problem.

To address this issue, the Kubelet's CPU manager should activate the "full-pcpus-only" policy. When this policy is enabled, the static policy ensures the allocation of complete physical cores. Consequently, the sibling threads are exclusively reserved, prohibiting any other containers from running on them.

The following performance profile setting will activate the "full-pcpus-only" policy,
```
spec:
  numa:
    topologyPolicy: "single-numa-node"
```

Alternatively, if there's a specific need to employ the "restricted" topology policy, it's possible to explicitly configure the "full-pcpus-only" setting,
```
metadata:
   annotations:
     kubeletconfig.experimental: "{\"cpuManagerPolicyOptions\": {\"full-pcpus-only\": \"true\"}}"
spec:
  numa:
    topologyPolicy: restricted
```
Once the performance profile has been applied and the system has completed the necessary reboots, users can log in to the node. They should then review the contents of /etc/kubernetes/kubelet.conf to ensure that the intended policy is correctly configured. If, for any reason, the performance profile is not functioning as expected, this file will not reflect the appropriate policy settings.
 
The "full-pcpus-only" policy guarantees that a container occupies complete CPU cores. However, to entirely mitigate the noisy neighbor issue, the application must retain control over CPU allocation. For instance, if a DPDK worker thread utilizes CPU 4, the DPDK application should take measures to prevent its other threads from executing on the sibling of CPU 4, such as CPU 36. This ensures the intended isolation of resources.

This noisy neighbor problem does not exist if the hyperthreading is disabled.
  
To calculate the CPU allocation and effectively bind threads to distinct cores—irrespective of hyperthreading status—we implemented a wrapper for TestPMD. This wrapper manages CPU pinning for TestPMD without altering the core functionality of TestPMD itself. Additionally, the wrapper incorporates a REST API, which allows for control and querying of TestPMD operations. While the details of utilizing this API fall outside the scope of this blog, readers keen on exploring it further are encouraged to refer to the documentation provided in the repository.

## CPU quota problem

In a DPDK application pod spec, users very often use whole number to indicate the required CpuSet, for example,
```
spec:
 containers:
   resources:
      limits:
        cpu: "6"
      requests:
        cpu: "6"
```

The “cpu limits” serves a purpose of CPU quota, in spite of the fact that the entire CPU is already allocated to the container. The CPU quota can lead to the throttling of the application.

Here is a sign that the DPDK worker threads are being throttled,

<img width="1089" alt="DPDK worker throttled" src="https://raw.githubusercontent.com/jianzzha/rfc2544-blog/main/diagrams/DPDK_worker_throttled.png">

Without throttling, the CPU utilization by the DPDK worker threads should have reached 100%.

Another way to observe that a container is being throttled is to examine the cpu.stat content under /sys/fs/cgroup/cpu,cpuacct/kubepods.slice/, for example,
```
# cat /sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-podd2709cbe_ad24_4cab_83ae_a3b395f00866.slice/cpu.stat
nr_periods 60582
nr_throttled 31857
throttled_time 4792267674277
```

To disable the CPU quota for the pod, the following pod spec is used,
```
metadata:
  annotations:
    cpu-quota.crio.io: "disable"
spec:
  runtimeClassName: performance-performance
```

To find the right runtimeClassName, one can use “oc get performanceprofile performance -o yaml | grep runtimeClass”.

While having the correct configuration is important, it doesn't automatically guarantee the deactivation of CPU quotas. An example of this situation occurred during the transition from cgroup v1 to cgroup v2 in CRI-O, when Initially the implementation of disabling CPU quotas per pod was not implemented, resulting in performance deterioration during RFC2544 testing. Instances of RFC2544 throughput problems due to CPU quotas are noticeable in specific releases, like 4.13.0. 

To verify whether CRI-O is genuinely attempting to disable the CPU quota, it's possible to inspect the CRI-O log. By logging into the node and reviewing the output using the command "journalctl -u crio," you can observe relevant information. When the DPDK container initiates, CRI-O should generate a message akin to: "Disable cpu cfs quota for container…" indicating the action taken.

## Conclusions

This blog aims to present the most significant challenges we faced during an RFC2544 test, rather than offering an exhaustive list of potential issues. While it doesn't encompass all possible problems, the blog sheds light on the intriguing obstacles we encountered. The information we share is intended to assist individuals in gaining insight into these challenges, as well as discovering and resolving them effectively.

Furthermore, the test tools and manifests discussed in this blog are tailored to meet our specific testing goals and yield positive outcomes, although they may not constitute a universally applicable standard.