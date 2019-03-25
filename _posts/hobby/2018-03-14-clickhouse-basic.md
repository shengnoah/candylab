---
layout: page-fullwidth
title: "ClickHouse与威胁日志分析"
subheadline: "服务监控防护"
teaser: "为从上层部署更好说明问题，我们用卡通一点的方式来描述系统结构，不涉及到更多的负载均衡和线路保障这种细节点。我们先回忆一下基一地类ELK的解决方案。日志的数据的被封装抽象成Stream流的概念，引用Pipelin管道，把日志从逻辑上进行更高一级的抽象，这样我们不对直接面对文件和索引这些概念，有了Stream、Input、Output、这种概念的模式设计，可以更好的把原生的日志数据更好的归类和业务靠近。"
categories:
  - design
tags:
  - APT
  
# Styling
#
header: 
   image_fullwidth: "header_roadmap_2.jpg"
image:          
    thumb: apt-thumb.png
    homepage: mediaplayer_js-home.jpg
    caption: 糖果
    caption_url: http://candylab.net
mediaplayer: true
---


<div class="row">
<div class="medium-4 medium-push-8 columns" markdown="1">
<div class="panel radius" markdown="1">
**目录**
{: #toc }
*  TOC
{:toc}
</div>
</div><!-- /.medium-4.columns -->


<div class="medium-8 medium-pull-4 columns" markdown="1">

{% include _improve_content.html %}



作者：糖果

ClickHouse与威胁日志分析

## 0x01 概要

威胁分析现在已经成为安全运维的标配工具。基于ELK这种大数据工具已经成为日志分析的一个很流行的选择方案，开源免费部署方便，对日志的检索及汇聚提供很好的用户体验。之前糖果实验室就介绍过基于Graylog这种类ELK工具的整体日志处理方案。在经过生产实践后的体会总结，发现了这种日志处理方案的好处，也发现了不足。随着后续系统不断接入新的安全日志数据，更多校的日志分析需求来讲，系统变会越加复杂。针对复杂的策略查询，有时我们基于ES和REST API的日志数据提供方式，在处理复杂查询开发时，开发效率会随着规模变大而变慢，我们需求一种更高抽象级别，和业务数据的直接关联的业务性语言，类似于SQL或是DSL一样的操作指令，让安全策略实现的落地成本降低。


## 0x02 ELK模式回顾

为从上层部署更好说明问题，我们用卡通一点的方式来描述系统结构，不涉及到更多的负载均衡和线路保障这种细节点。我们先回忆一下基一地类ELK的解决方案。日志的数据的被封装抽象成Stream流的概念，引用Pipelin管道，把日志从逻辑上进行更高一级的抽象，这样我们不对直接面对文件和索引这些概念，有了Stream、Input、Output、这种概念的模式设计，可以更好的把原生的日志数据更好的归类和业务靠近。


![1](http://image.3001.net/images/20180309/15205739622139.png)


从日志的收集、到数据的格式化、到ES存储、到REST API数据对外提供查询、到自动化查询、到数据的可视化是我们一般的使用套路，我们在这条思路上耕耘了有一段时间，更多的文章可以参考糖果实验室之前的文章，这里就是高度的概括一下这种系统的结构。


## 0x03 战斗民族的武器ClickHouse

我们再介绍一下基于ClickHouse的数据采集分析方案。其实从数据的收集、处理、查询、展示，对用户来说体验上大多数也有几分类似，不同的一点是ClickHouse提供SQL方式的查询。本身Graylog这种也内含了mongo和kafka等部件，而ClickHouse不是一种集成的解决处理方案。 相对简化的介绍一下。从系统部署构成来看，很相似。

![2](http://image.3001.net/images/20180309/15205739768864.png)



## 0x04 方案间的差异性

ClickHouse和ES是两种不同的数据检索引擎。ClickHouse提供了基于SQL的查询功能， ClickHouse对SQL支持和性能如何，在后期我们会给出相关的数据。像Graylog这种整体解决方案，提供了自己的数据查询DSL，但这种DSL是独立于Graylog本身的系统，而SQL具有更强的通用性。ES也支持ES SQL，但这点上就是谁用谁知道了。这两种方案核心的区别在于ES和ClickHouse不同的数据检索方案，安全业务会针对不同的数据产生不同的安全审计需求。对于数据收集和数据的展示的都是类似，当然ES也有ES SQL,但这不是SQL之间区别，而是两种生态和设计的不同。


![3](http://image.3001.net/images/20180309/15205739911869.png)

我们可以类似使用Mysql的方式来使用ClickHouse的表， 被监控服务器将自身的数据通过特定的工具推送kafka上，ClickHouse端去取得推送的数据，然后将数据存到二维数据结构的表中，之后我们就可以使用SQL语名去实现日志安全自动审计。Graylog这种类ELK的服务我们已经在生产中使用了，基于REST API为核心的设计很方便前端和移动端的审计应用扩展。基于威胁数据分析，我们基于ClickHouse实验出新的解决方案，
重要针对的是复杂的数据检索和业务数据碰撞。有了SQL这种高抽象实现，减少纯代码对DSL操作依赖，代码写的少了，安全策略都被翻译成SQL语句，但同时底层的引擎又不一样。

## 0x05 方案间的共性

对于使用者来说，这两个方案总体思路上还把日志和“流”和“管道”联系在一起，逻辑上的日志数据流向，无论采用什么样的工具和存储，日志数据聚合模式都类似，只是协议上，是采用syslog协议，还是JSON协议，还是两者都支持，基于数据汇聚的角度来说，两种方案都可以达到目标。但对安全策略实现，那种方案更快，更方便，后续我们还会有新实验内存和数据实现。最大的共性，就是数据收集到外放数据的模式类似。


![4](http://image.3001.net/images/20180309/1520574006453.png)


上面的图大大的简化了实际生产中的服务物理部署，用单点代替集群。简化到最后，就可以相对清晰的看到日志数据的流向。从访问者在请求服务者时产生的数据，到数据推送到kafka队列，再由kafka消费者消费数据给ClickHouse存储，然后提供Openresty为基础的API网关，再提供给API使用者作用。


![5](http://image.3001.net/images/20180309/15205740271422.png)


基于Graylog、ELK的API网关是基于ES的数据检索，网关会把安全策略转换成查询， 而基于ClickHouse的API网关，采用的就是基于ClickHouse的SQL查询为基础的安全策略落地执行。我们在设计系统时，让安全策略和系统不依赖，或者说通用的安全策略不考虑实现的方案到底是ELK还是ClickHouse， 只要是安全分析策略，用一种脚本或是类似DSL的语言可能解析和执行即可。

## 0x06 总结

ClickHouse是战斗民族的产品，CloudFlare公司已经用于生产分析中，糖果实验室也将继续探索些产品的新动向和实践。将流量分析和日志分析统计结合起来分析威胁，发现威胁。一些系统形式都是手段，系统可以实现安全人员的策略并行之有效的解决安全问题，是实践要达成的目标。我们可以基于ClickHouse开发更高级抽象的DSL描述安全的人员的安全策略与其它系统联动，完成威胁的分析与防护。后续会介绍一些相关的设计和工具及代码。




 [FreeBuf原文地址](http://www.freebuf.com/column/164671.html)
 



</div><!-- /.medium-8.columns -->





