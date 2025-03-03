---
title: "技术风口上的限流"
author: "张稀虹"
authorlink: "https://github.com/sofastack"
description: "技术风口上的限流"
categories: "SOFAStack"
tags: ["SOFAStack"]
date: 2021-09-14T15:00:00+08:00
cover: "https://gw.alipayobjects.com/mdn/sofastack/afts/img/A*PhN5Sp2T9NYAAAAAAAAAAAAAARQnAQ"
---

## 站在风口上

要问近两年最火的技术话题是什么？

Service Mesh 一定不会缺席。

如果用一句话来解释什么是 Service Mesh。

可以将它比作是应用程序或者说微服务间的 TCP/IP，负责服务之间的网络调用、限流、熔断和监控。*

对于编写应用程序来说一般无须关心 TCP/IP 这一层（比如通过 HTTP 协议的 RESTful 应用），同样使用 Service Mesh 也就无须关心服务之间的那些原本通过服务框架实现的事情，只要交给 Service Mesh 就可以了。

Service Mesh 作为 sidecar 运行，对应用程序来说是透明，所有应用程序间的流量都会通过它，所以对应用程序流量的控制都可以在 Serivce Mesh 中实现，这对于限流熔断而言就是一个天然的流量劫持点。

如今蚂蚁 80% 以上的应用都已经完成了 Mesh 化，Mesh 统一限流熔断的建设自然是水到渠成了。

**服务网格**（Service Mesh）是处理服务间通信的基础设施层。它负责构成现代云原生应用程序的复杂服务拓扑来可靠地交付请求。

在实践中，Service Mesh 通常**以轻量级网络代理阵列**的形式实现，这些代理与应用程序代码部署在一起，对应用程序来说无需感知代理的存在。

>![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

相较于传统的限流组件，Mesh 限流具备很多优势，在研发效能和研发成本上都取得了明显的收益：

**-** MOSN 架构天然的流量劫持让应用无需逐个接入 SDK

**-** 也无需为特定语言开发不同版本的限流组件

**-** 限流能力的升级也无需业务同步升级

**「背景业务」**

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyZm8bFa83iaSunFlDElbaRUYPpzAuuO0dEUwA3fqT8xx8lfSBSN1ET4w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 Mesh 统一限流实现前，蚂蚁集团内部存在多个不同的限流产品，分别提供不同的流量控制策略：

**不同类型的流量**（SOFARPC、无线网关 RPC、HTTP、消息等）限流配置分散在不同的平台，由不同的团队维护，产品质量和文档质量参差不齐，学习成本高、使用体验差。

**不同的限流策略**需要接入不同的 SDK，引入很多间接依赖，安全漏洞引起的升级频繁，维护成本高。

不仅在开发建设上存在不必要的人力投入，也给业务方使用造成了困扰和不便。

另一方面，我们的业务规模越来越大，但大量服务仍在使用最简单的单机限流策略。没有通用的自适应限流、热点限流、精细化限流、集群限流等能力。

因为限流能力缺失、限流漏配、错误的限流配置等问题引起的故障频发。

Mesh 架构下，sidecar 对流量管理具备天然的优势，业务无需在应用中接入或升级限流组件，中间件也无需针对不同的技术栈开发或维护多个版本的限流组件。

在目前 Service Mesh 蚂蚁内部大规模接入完成的背景下，将多种不同的限流能力统一收口至 MOSN，将所有限流规则配置统一收口至“统一限流中心”，可以进一步提高 MOSN 的流量管理能力，同时大幅降低业务限流接入及配置成本。

基于这样的背景下，我们在 MOSN 中进行了统一限流能力建设。

## 站在巨人肩膀上

在建设统一限流能力的过程中，我们调研了许多成熟的产品，既包括我们自己的 Guardian、Shiva、都江堰等，也包括开源社区的 concurrency-limits 、Hystrix、Sentinel 等产品。

我们发现阿里巴巴集团开源的 **Sentinel** 是其中的集大成者。

之前我们在打造 Shiva 的过程中也与集团 Sentinel 的同学进行过交流学习，他们也正在积极建设 Golang 版本的 sentinel-golang。

MOSN 作为一款蚂蚁自研的基于 Golang 技术建设的 Mesh 开源框架，如果搭配上 Sentinel 的强大的流控能力和较为出色的社区影响力，简直是强强联合、如虎添翼、珠联璧合、相得益彰...啊。

不过 Sentinel 对于我们而言也并不是开箱即用的，我们并不是完全没有历史包袱的全新业务，必须要考虑到蚂蚁的基础设施和历史限流产品的兼容，经过我们调研发现主要存在几个需要投入建设的点：

1. *控制面规则下发需要走蚂蚁的基础设施*

2. *Sentinel-golang 的单机限流、熔断等逻辑，**和我们之前的产品有较大差异*

3. *集群限流也要用蚂蚁的基础设施实现*

4. *Sentinel 自适应限流粒度太粗，**蚂蚁有更加精细化的需求*

5. *日志采集方案需要调整*

综合考虑后，我们决定基于 Sentinel 做扩展，站在巨人的肩膀上打造蚂蚁自己的 Mesh 限流能力。

基于 Sentinel 良好的扩展能力，我们对单机限流、服务熔断、集群限流、自适应限流等都做了蚂蚁自己的实现，也将部分通用的改动反哺到了开源社区，同时配套建设了统一的日志监控报警、统一限流中心。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

最终我们在 MOSN 里将各种能力都完成了建设，下表展示了 MOSN 限流和其他限流组件的能力对比：

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyNMssZWAkvyfLzicQic3TLETVZbf1FxEPDHICrk32SCBaqDams5EEU7Cg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 奥卡姆剃刀

*Pluralitas non est ponenda sine necessitate.*

*如无必要，勿增实体* 

![图片](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyGAoKiavqbrqoJ9lt7L4voOkIeibNOG0mic9RKWvkyfaqooFVI6hk4MF3A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

一个限流策略就配套一个 SDK 和一个管理后台七零八落，交互体验参差不齐，文档和操作手册质量也良莠不齐，交由不同的团队维护和答疑，如果你全都体验过一遍一定会深恶痛绝。

而 Mesh 统一限流的核心目的之一就是砍掉这些东西，化繁为简，降低业务同学的学习成本和使用成本，降低我们自己的维护成本。

**-** *流量控制的能力全部集成到 MOSN 里，取众家之长，去其糟粕*

**-** *流量控制的管控台全部收口到统一限流中心*

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyGAoKiavqbrqoJ9lt7L4voOkIeibNOG0mic9RKWvkyfaqooFVI6hk4MF3A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这应该是我们造的最后一个限流轮子了吧

**青出于蓝而胜于蓝**

上文提到了我们是站在 Sentinel 的肩膀上实现的 Mesh 统一限流，那我们又做了什么 Sentinel 所不具备的能力呢？

实际上我们对几乎所有的 Sentinel 提供的限流能力都做了一套自己的实现，其中也有不少的亮点和增强。

下面分享几个我们的技术亮点。

 **自适应限流** 

**-** *对于业务同学而言逐个接口做容量评估和压测回归费时费心，有限的精力只能投入到重点的接口保障上，难免会漏配一些小流量接口的限流。*

**-** *而负责质量和稳定性保障的同学经常在故障复盘时看到各种漏配限流、错配限流、压测故障、线程阻塞等造成的各种故障。*

我们希望即使在系统漏配错配限流的情况下，在系统资源严重不足时 MOSN 能够精准的找到导致系统资源不足的罪魁祸首，并实时根据系统水位自动调节异常流量。

在此需求背景下我们实现了一套符合成熟云原生定义的自检测、自调节的限流策略。

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyYNM4e7WsibiajbicyHOBAqfgqzNmeHTBVMTkZF63g0iaEwGJreaoprem9A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

自适应限流的实现原理并不复杂，朴素的解释就是，***触发限流后实时检测系统整体水位，同时秒级按比例调节流量***。

核心逻辑如下：

**-** **系统资源检测**：秒级检测系统资源占用情况，如果连续超过阈值 N 秒（默认 5 秒）则触发基线计算，同时将压测流量阻断腾挪出资源给线上业务使用；

**-** **基线计算**：将当前所有的接口统计数据遍历一遍，通过一系列算法找出资源消耗大户，再把这些大户里明显上涨的异常流量找出来，把他们当前的资源占用做个快照存入基线数据中；

**- 基线调节器**：将上一步骤存入的基线数据根据实际情况进行调整，根据系统资源检测的结果秒级的调整基线值，仍然超过系统阈值则按比例下调基线值，否则按比例恢复基线值，如此反复；

**- 限流决策**：

系统流量不断经过自适应限流模块，会尝试获取该接口的基线数据，如果没有说明该接口未被限流直接放过；

如果有基线数据则对比当前并发是否超过基线数据，根据实际情况决策是否允许该请求通过。

>![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

这套自主实现的自适应限流有如下几点优势:

**- 省心配置**：无代码入侵，极简配置；

**- 秒级调控**：单机自检测自调节，无外部依赖，秒级调整水位；

**- 智能识别**：压测资源腾挪、异常流量识别等特性；

**- 精准识别**：相较于其他的自适应限流技术，例如 Netflix 的 concurrency-limits，Sentinel 基于 BBR 思想的系统维度自适应限流等，精准识别能做到接口维度，甚至参数或者应用来源维度的自适应限流。

 **集群限流** 

在介绍集群限流之前，我们先简单思考一下单机限流在什么场景下会存在不足。

单机限流的计数器是在单机内存中独立计数的，独立的机器之间的数据彼此不关心，并且每台机器通常情况下采用了相同的限流配置。

考虑一下以下场景：

**-**假设业务希望配置的总限流阈值小于机器总量，例如业务有 1000 台机器，但希望限制 QPS总量为 500，均摊到每台机器 QPS\<1，单机限流的值该怎么配置呢？

**-** 假设业务希望限制 QPS 总量为 1000，一共有 10 台机器，但分布到每台机器上的业务流量不是绝对均匀的，单机限流的值又该怎么配置呢？*

计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决，我们很容易想到通过一个统一的外部的计数器来存储限流统计数据，这就是集群限流的基本思想。

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLybnnccoia77wWPKDoe4R0uVEyUY68rBtEWVdIB71cOjlj19D03nkxrNg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

不过每个请求都去同步请求缓存存在一些问题：

**- **如果请求量很大，缓存的压力会很大，需要申请足够多的资源；

**- **同步请求缓存，尤其是在跨城访问缓存的情况下，耗时会明显增加，最坏情况下 30ms+ 的跨城调用耗时可不是每个业务都能接受的。

**- **我们在集群限流中提供了同步限流和异步限流两种模式。针对流量很大或耗时敏感的情况我们设计了一个二级缓存方案，不再每次都请求缓存，而是在本地做一个累加，达到一定的份额后或者达到一定时间间隔后再咨询缓存，如果远端份额已扣减完，则将阻止流量再进入，直到下一个时间窗口后恢复。异步限流模式在大流量场景下对集群限流的性能和精度实现了尽可能的平衡。

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyWcAlsNJ9n0YZb5lc1AibVFMAUqxMVS68EaoLSk9kjVqf9kLcNrSu2Mg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**精细化限流**

传统的接口粒度的限流可能无法满足某些复杂的业务限流需求，例如同一个接口业务希望根据不同的调用来源进行区别对待，或者根据某个业务参数的值（例如商户 ID、活动 ID 等）配置独立的限流配置。

**精细化限流就是为了解决这样的复杂限流配置而设计的。**

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLy6C5SNIoOWaFqG0H6wShGOmYflMjLxKZx9MA3rBZ3AcO5NINGDo8aXQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们先梳理一下业务同学可能希望支持的条件有哪些，基本上概括起来有几类：

1. **按业务来源**

例如 A 应用对外提供的服务被B、C、D 三个系统调用，希望只对来自 B 的流量做限制，C、D 不限制。

2. **按业务参数值**

例如按 UID、活动 ID、商户 ID、支付场景 ID 等。

3. **按全链路业务标**¹

例如“花呗代扣”、“余额宝申购支付”等。

**[注1]**：全链路业务标是根据业务配置的规则生成的标识，该标识会在 RPC 协议中透传，达到跨服务识别业务来源的目的。

更复杂的场景下，可能上述条件还有一些逻辑运算关系，例如业务来源是 A 并且活动 ID 是 xxx 的流量，业务标是 A 或者 B 并且参数值是 xxx 等。

上面这些条件，有的是可以直接从请求的 header 中获取到的，例如业务来源应用、来源 IP 等可以直接获取到，我们称之为基本信息，而业务参数和全链路标识则不是每个应用都有，我们称之为业务信息。

流量条件规则就是让基本信息、业务信息等支持基本的逻辑运算，根据运算结果生成独立的子资源点。

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyahgItB9rlkrRialDCXwJ6AyVkvENkhemcic0wQLtw3RXIFqdMF3wCjHw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

根据业务配置的条件规则将流量拆分成若干个子资源点，再针对“子资源点”配置独立的限流规则，从而实现了精细化限流的需求。

## Do More

实现了限流熔断的能力大一统之后，我们还可以做什么？下面跟大家聊一下我们的一些思考。

 **限流 X 自愈** 

在实现了自适应限流后，我们很快在集团内进行了大规模的推广覆盖，几乎每天都有自适应限流触发的 case，但我们发现很多时候自适应限流触发都是单机故障引起的。数十万容器在线上运行，难免偶尔会出现单机抖动。

限流解决的是总体的容量问题，对于强依赖的服务限流后业务仍然表现为失败，更好的办法是将流量快速转移到其他健康机器。

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLyY0fBnjTM8zr3eOrjoe9HCx2wWURC1ACTIicPKic3JMYf4oC2f2Ep2rgw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

传统的自愈平台都是通过监控发现机器故障，继而执行后续的自愈动作，监控通常会有 2～3 分钟的数据延迟，如果在自适应限流触发后立即上报数据给自愈平台，自愈平台再进行判断确认是否是单机问题，随后执行自愈处理，则可以提高自愈的实效性，进一步提高业务可以率。

同样的思路，在自愈平台收到自适应限流触发的消息后如果发现不是单机问题而是整体容量问题，则可以进行快速扩容实现容量问题自愈。

 **限流 X 降级中台** 

当业务强依赖的服务发生故障时，限流保障的是服务不会因为容量问题导致服务雪崩，并不能提高业务可用率。单机故障可以做流量转移，但整体的故障发生时该怎么办呢？

更好的办法是**将请求转发到提前准备好的降级服务中**。

>![](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw0ib4hT03JTkibpYDjcicib5fnLydicljSMaEC0XxCaXKSLdSOVWucibmXmykBzIGN2iabCJHIj83KpTF17uQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

基于 Serverless 平台实现的降级中台，可以将降级的一些通用逻辑下沉到基座中（例如：缓存记账、异步恢复等），业务可以根据实际需求实现自己的 Serverless 业务降级模块，这样即使在服务完全不可用的状态下，MOSN 仍然可以将请求转发到降级服务中，从而实现更高的业务可用率。

**「总 结」**

随着 MOSN 限流能力的逐步丰富与完善以及未来更多 Mesh 高可用能力建设，MOSN 逐渐成为了技术风险和高可用能力基础设施中重要的一环。

以上就是我们 Mesh 限流实践与落地的一些经验分享，希望大家能通过这些分享对 Service Mesh 能有更深入的认识和了解，也期待大家更多的关注 MOSN ，让我们能得到更多社区的反馈，帮助我们做得更好。

希望大家一起努力, 共同进步。
