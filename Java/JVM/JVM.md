# JVM

[CMS GC 默认新生代是多大？](https://www.jianshu.com/p/832fc4d4cb53)

> 在Docker环境下，为什么cpu核心数会是1？
> 
> [https://stackoverflow.com/questions/55596774/runtime-getruntime-availableprocessors-returning-1-even-though-many-cores-av](https://stackoverflow.com/questions/55596774/runtime-getruntime-availableprocessors-returning-1-even-though-many-cores-av)
> 
> [https://github.com/eclipse/openj9/issues/1166](https://github.com/eclipse/openj9/issues/1166)
> 
> [https://juejin.im/post/6844903942262882311](https://juejin.im/post/6844903942262882311)
> 
> [https://developer.aliyun.com/article/562440](https://developer.aliyun.com/article/562440)

> [https://docs.docker.com/engine/reference/run/#cpu-share-constraint](https://docs.docker.com/engine/reference/run/#cpu-share-constraint)
> 
> By default, all containers get the same proportion of CPU cycles. This proportion can be modified by changing the container’s CPU share weighting relative to the weighting of all other running containers.
> 
> To modify the proportion from the default of 1024, use the `-c` or `--cpu-shares` flag to set the weighting to 2 or higher. If 0 is set, the system will ignore the value and use the default of 1024.
> 
> The proportion will only apply when CPU-intensive processes are running. When tasks in one container are idle, other containers can use the left-over CPU time. The actual amount of CPU time will vary depending on the number of containers running on the system.
> 
> For example, consider three containers, one has a cpu-share of 1024 and two others have a cpu-share setting of 512. When processes in all three containers attempt to use 100% of CPU, the first container would receive 50% of the total CPU time. If you add a fourth container with a cpu-share of 1024, the first container only gets 33% of the CPU. The remaining containers receive 16.5%, 16.5% and 33% of the CPU.
> 
> On a multi-core system, the shares of CPU time are distributed over all CPU cores. Even if a container is limited to less than 100% of CPU time, it can use 100% of each individual CPU core.

> [https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
> 
> When the kubelet starts a Container of a Pod, it passes the CPU and memory limits to the container runtime.
> 
> When using Docker:
> 
> - The `spec.containers[].resources.requests.cpu` is converted to its core value, which is potentially fractional, and multiplied by 1024. The greater of this number or 2 is used as the value of the [`--cpu-shares`](https://docs.docker.com/engine/reference/run/#cpu-share-constraint) flag in the `docker run` command.
> 
> - The `spec.containers[].resources.limits.cpu` is converted to its millicore value and multiplied by 100. The resulting value is the total amount of CPU time that a container can use every 100ms. A container cannot use more than its share of CPU time during this interval.
> 
> - The `spec.containers[].resources.limits.memory` is converted to an integer, and used as the value of the [`--memory`](https://docs.docker.com/engine/reference/run/#/user-memory-constraints) flag in the `docker run` command.
