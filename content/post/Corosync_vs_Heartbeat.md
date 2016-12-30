+++
Tags = ["HA", "Corosync", "Heartbeat"
]
date = "2016-12-30T16:13:50+08:00"
title = "Corosync vs Heartbeat"
Categories = ["架构"
]

+++

## Corosync
&emsp;&emsp;Corosync是一个带有高可用功能的组间通信系统，用C语言实现，有四个主要功能：

- 一个封闭的进程组间通信模型，可以保证分布式状态机间的同步
- 一个简单的可用性管理器，用于重启失败的应用程序
- 一个基于内存的配置、统计数据库，可以读写数据、感知信息的变化
- 一个资源配额管理系统，当应用使用的资源已达到其配额，或配额被削减，可以通知应用

&emsp;&emsp;Corosync常备用作Apache Qpid和Pacemaker的高可用框架。

## Heartbeat
&emsp;&emsp;Heartbeat是一个后台进程，可以为其客户端提供集群环境（通信和成员关系）。让处于不同机器的客户端知道彼此的存在（或是下线），以及交换数据。
&emsp;&emsp;Heartbeat应该与CRM（集群资源管理）系统结合在一起使用，后者可以起停集群中的各种资源，如IP地址、web服务等等，以保证系统的高可用。Pacemaker就是Heartbeat推荐的高可用框架。