# mesos-maintenance
tool for mesos to automatically maintain nodes.

Table of Contents
=================

  * [Mesos 维护原语](#mesos- 维护原语)
  * [维护模式简介](#维护模式简介)
    * [Drain mode](#drain-mode)
    * [Down mode](#down-mode)
    * [Up mode](#up-mode)
    * [维护状态](#维护状态)
  * [运维](#运维)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

# Mesos 维护原语

维护原语 (Maintenance Primitives) 是在 Mesos 0.25.0 中引进的新功能。主要的作用就是保证在集群中的某些机器处于维护状态的时候，运行其上的 framework 任务不能受影响。

我们可以利用它对 mesos agents 完成如下几种情形的操作：

 - 硬件维护

 - 内核升级

 - agent 升级

维护原语中提出了一些新的概念，比如：

- **Maintenance** : 是一种操作，它可以让某台 agent 的资源不可用。

- **Maintenance window** : 由两部分组成，一部分是 `machine_ids` ，它指定哪些 agent 需要进入维护状态，一部分是 `Unavailability` ，它指定这些机器从什么时候开始维护及需要多少时间维护。

- **Maintenance schedule** : 一系列的 `Maintenance window`，即可以一批接着一批地按照给定的 `Maintenance window` 依次进行维护

- **Drain** : 一种状态，处于这种状态的 agents ，发出的 offer 将会包含不可用的信息。跑在其上面的 frameworks 将会接收 inverse offer ，即会被收缴之前发放出去的资源；同时也可以让 frameworks 知道这些 agents 将不可用，后面的任务将不会发送给这些 agents。

- **Down** : 一种状态，它可以让某台 agent 的瞬间脱离 mesos 集群并自行下线 mesos agent 服务。

- **Up** : 一种状态，它可以让某台 agent 重新加入到 mesos 集群中。

- **Inverse offer** : 是一种让 master 从 framework 中拿回资源的通信机制。它用于告之 framework 资源的不可用性并且如果 framework 准备好了，即会 响应这种事件表示它们有能力来遵守这种游戏规则。inverse offer 和 offer 相似，都可被接受、拒绝、重新发放和取消等。

# 维护模式简介
![这里写图片描述](http://img.blog.csdn.net/20160406234943612)

所有的维护调度都围绕 `Drain`, `Down`, `Up` 三种模式展开。

## Drain mode
---

通过调用 `/maintenance/schedule` 机器就从 `Up` 模式进入了 `Drain` 模式。为了进入这个模式，operator 需要构建维护调度并 POST 到 Mesos master 。

维护调度用 json 格式表示，注意，时间是以纳秒为单位进行的，示例如下：

```
{
  "windows" : [
    {
      "machine_ids" : [
        { "hostname" : "machine1", "ip" : "10.0.0.1" },
        { "hostname" : "machine2", "ip" : "10.0.0.2" }
      ],
      "unavailability" : {
        "start" : { "nanoseconds" : 1443830400000000000 },
        "duration" : { "nanoseconds" : 3600000000000 }
      }
    }, {
      "machine_ids" : [
        { "hostname" : "machine3", "ip" : "10.0.0.3" }
      ],
      "unavailability" : {
        "start" : { "nanoseconds" : 1443834000000000000 },
        "duration" : { "nanoseconds" : 3600000000000 }
      }
    }
  ]
}
```

```
curl http://localhost:5050/master/maintenance/schedule
  -H "Content-type: application/json"
  -X POST
  -d @schedule.json
```

**注意**：如果需要取消该操作，只需要 POST `空值`即可。

## Down mode
---

通过调用 `/machine/down` , 机器就从 `Drain` 模式进入了 `Down` 模式。为了进入这个模式，operator 需要构建维护调度并 POST 到 Mesos master 。

维护调度用 json 格式表示，示例如下：

```
[
  { "hostname" : "machine1", "ip" : "10.0.0.1" },
  { "hostname" : "machine2", "ip" : "10.0.0.2" }
]
```

```
curl http://localhost:5050/machine/down
  -H "Content-type: application/json"
  -X POST
  -d @machines.json
```

## Up mode
---

通过调用 `/machine/up` 机器就从 `Down` 模式进入了 `Up` 模式。为了进入这个模式，operator 需要构建维护调度并 POST 到 Mesos master 。

维护调度用 json 格式表示，示例如下：

```
[
  { "hostname" : "machine1", "ip" : "10.0.0.1" },
  { "hostname" : "machine2", "ip" : "10.0.0.2" }
]
```

```
curl http://localhost:5050/machine/up
  -H "Content-type: application/json"
  -X POST
  -d @machines.json
```

## 维护状态
---

通过调用 `/maintenance/status` 可以看到 agent 状态信息。这个对于 framework 想要知道哪些机器处于维护模式很有用，这也是 framework 与 operator 之间的一种协调方式。通过它可以查看到哪些 agent 处于 `Drain` 模式，哪些处于 `Down` 模式。这样 Mesos 可以方便协调 framework 和 operator 之间的关系以保证 task 的正常运行。这里的 operator 可以是 person,  tool 或者 script。

# 运维

在 [github maintenance](https://github.com/chenqiangzhishen/Shell/blob/master/mesos-maintenance/maintenance.sh) 项目中的 `maintenance.sh` 中，我实现了 maintenance 的各种操作，这样方便运维。

使用格式如下：

```bash
[root@10.23.80.34 mesos-deploy]# ./maintenance.sh
Usage: maintenance.sh <command> <cluster> <host-pattern> [duration]

Supported command:
    drain                               put the specified nodes to DRAIN mode
    down                                put the specified nodes to DOWN mode
    up                                  put the specified nodes to UP mode
    cancel                              cancel maintenance
    status                              get maintenance status
    help                                display help info

Required argument:
    cluster                             the cluster operated on, ansible inventory file
    host-pattern                        host-pattern that contains part slaves in the cluster

Optional argument:
    duration                            maintenance duration time, unit is hour, default is 2 hours

Examples:
    # use default maintenance duration time (2 hours)
    ./maintenance.sh status hosts/cqdx-dev-chenqiang part-slaves

    # set 4 hours for the maintenance duration time
    ./maintenance.sh status hosts/cqdx-dev-chenqiang part-slaves 4
```

**注意**

1. 由于 maintenance primitives 设计的问题，机器只有在进入了 `Drain` 模式才可以进入 `Down` 模式，所以在进行维护的时候，遵寻 `Drain` -> `Down` -> `Up` 的次序进行。这几个状态之间的时间间隔原则上没有讲究，只要按照这个次序即可随时进入下一个状态。

2. 因需要保证 mesos 集群中 frameworks 的任务正常运行，所以在维护的时候注意批量的机器不宜过多。

3. 机器的维护时间程序中默认设置了 2 小时，可以修改成更大，超过设定的 2 小时后，agents 会自动上线加入到 mesos 集群。

4. 该脚本后配合了 Ansible 部分进行自动化运维部署，以实现 mesos 集群的全自动升级等工作。
