# 节点问题检测器（Node Problem Detector）

是一个守护程序，用于监视和报告节点的健康状况（包括内核死锁、OOM、系统线程数压力、系统文件描述符压力等指标）。你可以将节点问题探测器以 DaemonSet 或独立守护程序运行。节点问题检测器从各种守护进程收集节点问题，并以 NodeCondition 和 Event 的形式报告给 API Server。您可以通过检测相应的指标，提前预知节点的资源压力，可以在节点开始驱逐 Pod 之前手动释放或扩容节点资源压力，防止 Kubenetes 进行资源回收或节点不可用可能带来的损失。

## 处理的问题

eg.

- 基础架构守护程序问题：ntp服务关闭；
- 硬件问题：CPU，内存或磁盘损坏；
- 内核问题：内核死锁，文件系统损坏；
- 容器运行时问题：运行时守护程序无响应

## 问题 API

- NodeCondition：使节点对pod 不可用的永久问题报告为NodeCondition
- Event：对pod影响但提供信息的临时问题应报告为Event

## 参数

- hostname-override：供 NPD 使用的自定义的节点名称，NPD 会优先获取该参数设置的节点名称，其次是从 NODE_NAME 环境变量中获取，最后从 os.Hostname() 方法获取。
- system-log-monitor：使用不同的 log monitors 来监控不同的系统日志
- config.system-log-monitor：会取代上面这个参数，如果和上面参数同时使用，则会引发恐慌
- system-states-monitor：使用不同的 status monitors 来监控系统的不同状态
- custom-plugin-monitor：使用不同的自定义插件监视器来监控不同的系统问题（k8s内置、prometheus、stackdriver）
- config.custom-plugin-monitor：会取代上面这个参数，如果和上面参数同时使用，则会引发恐慌

---

k8s exporter

- enable-k8s-exporter: 是否开启上报信息到 API Server，默认为 true
- apiserver-override: 一个URI参数，用于自定义node-problem-detector连接apiserver的地址（上面参数为false时，则此参数忽略）
- address: 绑定 NPD 服务器的地址
- port: NPD 服务端口，如果为0，表示禁用 NPD 服务

---

prometheus exporter

- prometheus-address: 绑定Prometheus抓取端点的地址，默认为127.0.0.1
- prometheus-port: 绑定Prometheus抓取端点的端口，默认为20257。使用0禁用

---

stackdriver exporter

- exporter.stackdriver: Stackdriver exporter程序配置文件的路径，例如 config/exporter/stackdriver-exporter.json，默认为空字符串。设置为空字符串以禁用。

## node-strick-detector（节点探测器）

DaemonSet类型

如果节点问题检测器作为集群插件运行，则不支持覆盖配置。插件管理器不支持 ConfigMap。

![image-20210908180222457](/Users/liyang/Documents/k8s/NPD.assets/image-20210908180222457.png)

NPD启动流程：

1. 打印 NPD 版本号
2. 设置节点名称，优先使用命令行中设置的节点名称，其次是环境变量 NODE_NAME 中的节点名称，最次是 os.Hostname()
3. 校验命令行参数的合法性
4. 初始化 problem daemons
5. 初始化默认 Exporters（包含 K8s Exporter、Prometheus Exporter）和可插拔 Exporters（如 Stackdriver Exporter）
6. 使用 problem daemons 和 Exporters 构建 Problem Detector，并启动

## 核心组件

### Problem Daemon

Problem Daemon 是监控任务子守护进程，NPD 会为每一个 Problem Daemon 配置文件创建一个守护进程，这些配置文件通过 --config.custom-plugin-monitor、--config.system-log-monitor、--config.system-stats-monitor 参数指定。每个 Problem Daemon监控一个特定类型的节点故障，并报告给NPD。目前 Problem Daemon 以 Goroutine 的形式运行在NPD中，未来会支持在独立进程（容器）中运行并编排为一个Pod。在编译期间，可以通过相应的标记禁用每一类 Problem Daemon。

### Exporter

Exporter 用于上报节点健康信息到某种控制面。在 NPD 启动时，会根据需求初始化并启动各种 Exporter。Exporter 分为三类：

- K8s Exporter：会将节点健康信息上报到 API Server。
- Prometheus Exporter：负责上报节点指标信息到 Prometheus。
- Plugable Exporters：可插拔的 Exporter（如 Stackdriver Exporter），我们也可以自定义 Exporter，并在 init() 方法中注册，这样在 NPD 启动时就会自动初始化并启动。

### Condition Manager

K8s Exporter 获取到的异常 Condition 信息会上报给 Condition Manager， Condition Manager 每秒检查 Condition 的变化，并同步到 API Server 的 Node 对象中。

### Problem Client

Problem Client 负责与 API Server 交互，并将巡检过程中生成的 Events 和 Conditions 上报给 API Server。

### Problem Detector

Problem Detector 是 NPD 的核心对象，它负责启动所有的 Problem Daemon（也可以叫做 Monitor），并利用 channel 收集 Problem Daemon 中发现的异常信息，然后将异常信息提交给 Exporter，Exporter 负责将这些异常信息上报到指定的控制面（如 API Server、Prometheus、Stackdriver等）。

### States

Status 是 Problem Daemon 向 Exporter 上报的异常信息对象。

### Tomb

用于从外部控制协程的生命周期， 准备结束生命周期时：

1. 外部协作者发起一个通知
2. 协作线程接收到通知，进行清理
3. 清理完成后，协程反向通知外部协作者
4. 外部协作者退出阻塞

## 节点自愈

采集节点的健康状态是为了能够在业务Pod不可用之前提前发现节点异常，提供了根据采集到的节点状态从而进行不同自愈动作的能力。根据节点不同的状态配置相应的自愈能力，如重启Docker、重启Kubelet或重启CVM节点等。同时为了防止集群中的节点雪崩，在执行自愈动作之前做了严格的限流，防止节点大规模重启。同时为了防止集群中的节点雪崩，在执行自愈动作之前做了严格的限流。具体策略为：

1. 在同一时刻只允许集群中的一个节点进行自愈行为，并且两个自愈行为之间至少间隔1分钟
2. 当有新节点添加到集群中时，会给节点2分钟的容忍时间，防止由于节点刚刚添加到集群的不稳定性导致错误自愈

