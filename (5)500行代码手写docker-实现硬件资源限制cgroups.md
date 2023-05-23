# (5)500行代码手写docker-实现硬件资源限制cgroups

> 本系列教程主要是为了弄清楚容器化的原理，纸上得来终觉浅，绝知此事要躬行，理论始终不及动手实践来的深刻，所以这个系列会用go语言实现一个类似docker的容器化功能，最终能够容器化的运行一个进程。

本章的源码已经上传到github，地址如下:
```shell
https://github.com/HobbyBear/tinydocker/tree/chapter5
```

之前我们对容器的网络命名空间，文件系统命名空间都进行了配置，说到底这些都是为了资源更好的隔离，但是他们无法办到对硬件资源使用的隔离，比如，cpu，内存，带宽，而今天要介绍的cgroups技术便能够对硬件资源的使用产生隔离。

## cgroups技术简介
cgroups技术是内核提供的功能，可以通过虚拟文件系统接口对其进行访问和更改。mount 命令可以查看cgroups在虚拟文件系统下的挂载目录。
```shell
root@ecs-295280:~# mount | grep  cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
root@ecs-295280:~#
```
一般默认的挂载目录是在/sys/fs/cgroup 目录下，系统内核在开机时，会默认挂载cgroup目录。这样便能通过访问文件的方式对cgroup功能进行使用。

在/sys/fs/cgroup/ 目录下，我们看到的每个目录例如cpu，blkio被称作subsystem子系统，每个子系统下可以设置各自要管理的进程id。
```shell
root@ecs-295280:~# ls /sys/fs/cgroup/
blkio    cpu,cpuacct  freezer  net_cls           perf_event  systemd
cpu      cpuset       hugetlb  net_cls,net_prio  pids        unified
cpuacct  devices      memory   net_prio          rdma
```
拿cpu这个目录下的文件举例
```shell
root@ecs-295280:/sys/fs/cgroup/cpu# ls
cgroup.clone_children  cpuacct.usage_percpu_sys   cpu.stat
cgroup.procs           cpuacct.usage_percpu_user  ebpf-agent
cgroup.sane_behavior   cpuacct.usage_sys          hostguard
cpuacct.stat           cpuacct.usage_user         notify_on_release
cpuacct.usage          cpu.cfs_period_us          release_agent
cpuacct.usage_all      cpu.cfs_quota_us           tasks
cpuacct.usage_percpu   cpu.shares
root@ecs-295280:/sys/fs/cgroup/cpu# ll -l
```
在cpu子系统这个目录下，有两个文件cgroup.procs，tasks文件，它们都是用来管理cgroup中的进程。但是，它们的使用方式略有不同：

cgroup.procs文件用于向cgroup中添加或删除进程，只需要将进程的task id写入该文件即可。

tasks文件则是用于将整个进程组添加到cgroup中。如果将一个进程组的pid写入tasks文件，则该进程组中的所有进程都会被添加到cgroup中。

进程被加入到这个cgroup组以后，其使用的cpu带宽将会受到cpu.cfs_quota_us和cpu.cfs_period_us的影响。通过shell命令查看他们的内容。
```shell
root@ecs-295280:/sys/fs/cgroup/cpu/test# cat cpu.cfs_period_us
100000
root@ecs-295280:/sys/fs/cgroup/cpu/test# cat cpu.cfs_quota_us
-1
```
默认情况下，cpu.cfs_period_us是100000，单位是微秒，cpu.cfs_period_us代表了cpu运行一个周期的时长，100000代表了100ms，cpu.cfs_quota_us代表进程所占用的周期时长，-1代表不限制进程使用cpu周期时长，如果cpu.cfs_quota_us是50000(50ms)则代表在cpu一个调度周期内，该cgroup下的进程最多只能运行半个周期，如果达到了运行周期的限制，那么它必须等待下一个时间片才能继续运行了。

## 命名行实践下cgroups隔离特性
我们来实验下:
### 对cpu使用率进行限制
在cpu的一级目录下，是包含了当前系统所有进程，为了不影响它们，我们在cpu的一级目录下创建一个test目录，然后单独的在test目录中的tasks文件加入进程id。

> 📢📢 ❗️cgroup的每个子系统是分级的，这个级别体现在目录层级上，默认子目录会继承父目录的属性，子目录也可以通过修改子目录下的文件，来覆盖掉父目录的属性。

```shell
root@ecs-295280:/sys/fs/cgroup/cpu/test# pwd
/sys/fs/cgroup/cpu/test
```
设置cpu.cfs_quota_us为一个时间片的一半，设置tasks，把当前进程加入到cgroup中
```shell
root@ecs-295280:/sys/fs/cgroup/cpu/test# cat cpu.cfs_quota_us
50000
root@ecs-295280:/sys/fs/cgroup/cpu/test# sh -c "echo $$ > tasks"
root@ecs-295280:/sys/fs/cgroup/cpu/test# cat tasks
65961
66314
```
在当前shell 界面，通过stress对cpu进行压力测试。我的虚拟机是一个核，我这里直接通过stress对这一个cpu核进行压测。
```shell
root@ecs-295280:/sys/fs/cgroup/cpu/test# stress --cpu 1 --timeout 60
```

启动另一个终端，查看cpu占用情况
```shell
Tasks:  94 total,   2 running,  92 sleeping,   0 stopped,   0 zombie
%Cpu(s): 51.9 us,  0.0 sy,  0.0 ni, 48.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   1982.9 total,    451.3 free,    193.4 used,   1338.2 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   1597.9 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  66333 root      20   0    3856    100      0 R  50.3   0.0   0:06.00 stress
      1 root      20   0  102780  12420   8236 S   0.0   0.6   0:05.93 systemd
```
可以看到的是cpu占用率在达到百分之50时就不上去了，这正是由于stress进程是bash进程的子进程，继承了bash进程的cgroup，所以cpu使用率受到了限制。

### 对内存使用率进行限制

再来看看如何通过cgroup对内存进行限制，这次我们就应该进入到memory这个子系统的目录了，同样我们在其下面创建一个test目录。

```shell
root@ecs-295280:/sys/fs/cgroup/memory# mkdir test
root@ecs-295280:/sys/fs/cgroup/memory# cd test/
root@ecs-295280:/sys/fs/cgroup/memory/test# pwd
/sys/fs/cgroup/memory/test
```
然后把当前进程加进去
```shell
root@ecs-295280:/sys/fs/cgroup/memory/test# sh -c "echo $$ > tasks"
root@ecs-295280:/sys/fs/cgroup/memory/test# cat tasks
65961
66476
```
设置最大使用内存，memory目录下限制最大使用内存需要设置memory.limit_in_bytes 这个文件，默认情况下，它是一个大的离谱的值，我们将它改为100M
```shell
root@ecs-295280:/sys/fs/cgroup/memory/test# cat memory.limit_in_bytes
9223372036854771712
root@ecs-295280:/sys/fs/cgroup/memory/test# vim memory.limit_in_bytes
root@ecs-295280:/sys/fs/cgroup/memory/test# cat memory.limit_in_bytes
104857600
```

这个时候通过stress 对内存进行压力测试，我们限制了100M，但是如果stress要求分配200M内存，看看能正常分配吗？
```shell
root@ecs-295280:/sys/fs/cgroup/memory/test# stress --vm-bytes 200m --vm-keep  -m 1
stress: info: [66533] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: FAIL: [66533] (415) <-- worker 66534 got signal 9
stress: WARN: [66533] (417) now reaping child worker processes
stress: FAIL: [66533] (451) failed run completed in 0s
```

可以看到的是，程序崩溃了，原因则是由于发生了oom，因为内存已经被我们限制到了100M，通过test目录下的memory.oom_control文件可以看到发生oom的次数。
```shell
oom_kill_disable 0
under_oom 0
oom_kill 1
```
oom_kill 为1代表发生oom后，进程被kill掉的次数。

在简单看完cgroup如何对cpu和内存进行限制以后，看看golang代码如何实现。
## golang代码实现cgroups配置

在用代码对cgroup的操作本质上就是对cgroup的文件进行操作。

```golang
cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
		err = cmd.Start()
		if err != nil {
			fmt.Println(err)
		}

		containerName := os.Args[2]
		if err := cgroups.ConfigDefaultCgroups(cmd.Process.Pid, containerName); err != nil {
			log.Error("config cgroups fail %s", err)
		}

		if err := network.ConfigDefaultNetworkInNewNet(cmd.Process.Pid); err != nil {
			log.Error("config network fail %s", err)
		}
		cmd.Wait()
		cgroups.CleanCgroupsPath(containerName)
```

在前面代码的基础上，启动子进程后，父进程把子进程pid添加到一个新的cgroup种，cgroups.ConfigDefaultCgroups方法用于实现对cgroup的控制，以容器名作为cgroup子系统的目录，然后当子进程容器执行完毕后，通过cgroups.CleanCgroupsPath去对cgroup相关目录进行清理。


```shell
func CleanCgroupsPath(containerName string) error {
	output, err := exec.Command("cgdelete", "-r", fmt.Sprintf("memory:%s/%s", dockerName, containerName)).Output()
	if err != nil {
		log.Error("cgdelete fail err=%s output=%s", err, string(output))
	}
	output, err = exec.Command("cgdelete", "-r", fmt.Sprintf("cpu:%s/%s", dockerName, containerName)).Output()
	if err != nil {
		log.Error("cgdelete fail err=%s output=%s", err, string(output))
	}
	return nil
}
```

清理cgroupd的方式我用了cgdelete 命令 删除掉容器cgroup的配置，直接remove删除会出现删除失败情况。

## 总结
这也是我对于手写容器系列的终章，算是对容器原理的一个入门级讲解，其实后续还可以针对它做很多优化，比如实现不同主机上的容器互联，实现容器日志的功能，实现端口映射，实现卷映射功能，这些功能其实都是建立在我们讲的容器原理之上的，懂了原理便能一通百通，希望能给你带来启发。
