---
title: "Java 常用工具"
date: 2019-02-06T18:18:39+08:00
draft: true
---



### jps

```bash
> jps -help
usage: jps [-help]
       jps [-q] [-mlvV] [<hostid>]

Definitions:
    <hostid>:      <hostname>[:<port>]
```

| 选项 | 作用                                                   |
| ---- | ------------------------------------------------------ |
| -q   | 只输出 LVMID，忽略主类的名称                           |
| -m   | 输出虚拟机进程启动时传递给主类 main() 函数的参数       |
| -l   | 输出主类的全名，如果进程执行的是 jar 包，输出 jar 路径 |
| -v   | 输出虚拟机进程启动时 JVM 参数                          |

### jstat

```bash
> jstat -help
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:
                     <lvmid>[@<hostname>[:<port>]]
                Where <lvmid> is the local vm identifier for the target
                Java virtual machine, typically a process id; <hostname> is
                the name of the host running the target Java virtual machine;
                and <port> is the port number for the rmiregistry on the
                target host. See the jvmstat documentation for a more complete
                description of the Virtual Machine Identifier.
  <lines>       Number of samples between header lines.
  <interval>    Sampling interval. The following forms are allowed:
                    <n>["ms"|"s"]
                Where <n> is an integer and the suffix specifies the units as
                milliseconds("ms") or seconds("s"). The default units are "ms".
  <count>       Number of samples to take before terminating.
  -J<flag>      Pass <flag> directly to the runtime system.
```

| 选项              | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| -class            | 监视类装载、写在卸载数量、总空间以及类装载所耗费的时间       |
| -gc               | 监视 Java 堆状况，包括 Eden 区、两个 Survivor 区、老年代、永久代的容量、已用空间、GC时间合计等信息 |
| -gccapacity       | 监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间 |
| -gcutil           | 监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比 |
| -gccause          | 与 -gcutil 功能一样，但是会二外输出导致上一次GC产生的原因    |
| -gcnew            | 监视新生代GC状况                                             |
| -gcnewcapacity    | 监视内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间 |
| -gcold            | 监视老年代GC状况                                             |
| -coldcapacity     |                                                              |
| -compiler         | 输出JIT编译器编译过的方法、耗时等信息                        |
| -printcompilation | 输出已经被JIT编译的方法                                      |

### jinfo

