---
layout: post
title: "SDN和OpenFlow介绍"
date: 2016-12-31 14:14:24 +0800
comments: true
categories: OpenStack
---
传统网络架构下，数据转发和转发决策逻辑都集中在网络设备内部，而各厂商的网络设备都有自己的配置方法, 运维和管理的复杂性很高。云计算等业务发展对网络调整的迅速响应需求越来越高，传统方式网络架构越来越不能满足灵活性要求。因而业界提出了SDN方案，将数据平面和控制平面分离，数据平面由网络设备完成，而决策逻辑交由独立的控制器来完成，从而控制器能够对整个网络进行全局管控。除此之外，控制器对上层应用程序提供API(称为北向API)，上层的业务逻辑可以使用北向API处理流量和部署服务，网络控制逻辑可以直接通过编写程序来完成，网络调整的响应速度大为加快，灵活性大为提高。控制器与负责转发数据的网络设备之间通过标准协议交互转发策略，这之间协议称为南向协议或南向API。

整体架构如图:

{% img /images/2016-12-31/1.png %}

OpenFlow是一种南向API协议，并且定义了交换机规范。实现OpenFlow规范的交换机称为OpenFlow交换机。

<!--more-->

OpenFlow相关规范可以从官网下载:
https://www.opennetworking.org/sdn-resources/technical-library

OpenFlow规范还在不停改进，本文基于OpenFlow-1.3.5进行说明，规范下载地址:
https://www.opennetworking.org/images/stories/downloads/sdn-resources/onf-specifications/openflow/openflow-switch-v1.3.5.pdf

OpenFlow交换机基于多个流表(FlowTable)和一个组表(GroupTable)转发数据包，通过OpenFlow Channel与控制器通信。控制器可以向交换机下发配置来添加、删除、更新流表中的流表项(flow entry)，交换机也可以将数据包转发至控制器，由控制器来判断如何处理数据包。

整体架构如图:

{% img /images/2016-12-31/2.png %}

本文简要介绍OpenFlow交换机如何基于流表处理数据包。

交换机中至少需要有一个流表，流表包含多个流表项。流表以数字标识。流表项中含有匹配规则和指令集(instruction set)。当数据包进入交换机，交换机固定从表0开始查找流表项，并给数据包关联一个action set，此时为空。当匹配到流表项后，则执行表项所关联的指令集。指令集中的goto-table指令跳到更高的表处理来数据包，也可以使用其他指令修改数据包所关联的action set。若没有goto-table指令, 则执行数据包的行为集。如果没有匹配到任何表项，则触发table-miss所关联的指令集。若没有配置table-miss表项，则直接丢弃数据包。

上述流程如图:

{% img /images/2016-12-31/3.png %}

流表项结构如下:

{% img /images/2016-12-31/4.png %}

* match fields: 匹配项，交换机使用这些项来匹配数据包，主要包括收包端口，各协议包头(如Ethernet包头，IP包头，TCP/UDP包头，VLAN ID等），以及用于流表之间传输数据的metadata
* preority: 匹配优先级，0表示优先级最低，交换机依据优先级从大到小匹配流表项
* counters: 用于统计匹配该表项的数据包和字节数
* instructions: 匹配表项后，需要执行的指令集。指令集可以修改数据包的action set，可以直接对数据包进行修改，也可以跳至更高的流表进行处理
* timeout: 表项的过期时间，每个流表项有两个timeout:
    * hard_timeout: 表项从添加开始持续了该时间后，表项将被移除
    * idle_timeout: 表项从最后一次匹配持续了该时间后，表项将被移除
* cookie: 用于标识表项，由controller来设定，用于区分流表项，交换机不使用该值
* flags: 用于修改管理表项的行为

一般table-miss表项所有match fields都为空，并且优先级为0。

除了直接由流表项处理数据包，还可以由流表项指定通过组表(Group Table)来转发数据包。

组表包含多个组，结构如下:

{% img /images/2016-12-31/5.png %}

* Group Indentifier: 组标识符
* Group Type: 组类型，决定如何处理数据包
* Counters: 用于统计该组处理的数据包
* Action Buckets: 组关联的行为集，他们组织为有序列表，每个行为集包含多个Action及其参数

示意图如下:

{% img /images/2016-12-31/6.png %}

OpenFlow定义了4种组类型:
* indirect: 这种Group只能包含一个ActionBucket，主要用于多个流表项处理相同时，可以指向同一个Group。这样修改处理逻辑时只需要修改一个Group
* all: 处理时执行Group包含的所有Action Bucket，主要用于广播和多播。交换机会为每个Action Bucket复制一个数据包单独进行处理
* select: 交换机根据算法选择一个Action Bucket来执行，主要用于负载均衡功能实现
* fast failover: 每个Action Bucket和一个Port关联。按Action Bucket的顺序，执行第一个处于Live状态的Action Bucket，用于提高可靠性和高可用

主要的Instructions包括:

* apply-actions actions: 直接执行actions, 而不是放到action set中最后再执行
* clear-actions: 清空action set
* write-actions actions: 将action添加到action set中
* write-metadata metadata/mask: 将使用掩码处理的metadata写入metadata field中
* goto-table next-table-id: 跳至指定的表来处理数据包

指令集中只能有一种同类型的指令，这些指令依照上述列出的顺序来执行。

action set中的主要的action包括:

* output port_no: 将数据包转发到指定的PORT
* group group_id: 使用指定的group来处理数据包
* drop: 丢弃数据包
* push-tag/pop-tag ethertype: 添加和去除协议标签
* set-field field_type value: 修改数据包头

本文简要介绍了OpenFlow-1.3.5版本的交换机规范，说明了OpenFlow交换机如何处理数据包，后续其他文章再来介绍交换机与控制器之间的交互协议。
