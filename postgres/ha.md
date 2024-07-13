# Postgresql(openGauss)的高可用

注：由于Postgresql本身不支持多主集群的高可用，因为本文使用openGauss进行讲解（主要是因为笔者在给openGauss开发），以下简称openGauss为OG。

OG的高可用是实现了一个称为CM（cluster manager）的集群监控管理工具，基于这个工具加上OM脚本就可以实现OG的高可用。

## CM怎么实现的

CM工具是一个使用etcd（一个基于Raft分布式一致性协议实现的分布式kv存储）实现的集群管理工具，在一个节点上，一个CM进程实现了以上的服务：
- om_monitor(OMM)：OM监控工具
- cm_server(CMS)：主服务，负责仲裁，触发故障转移，接受agent的消息。
- cm_agent(CMA)：节点上的openGauss的代理，并定期使用心跳机制检查openGauss存活，也承担执行命令，日志采集，性能监控等的功能
- cm_ctl：集群管理工具，只能作用于主节点

### CMS

### CMA

### OMM
