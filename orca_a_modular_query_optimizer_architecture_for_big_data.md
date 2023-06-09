# Orca: A Modular Query Optimizer Architecture for Big Data

## Abstract
数据库系统的分析查询的性能主要取决于系统的查询优化器的能力。数据量的增加和对处理复杂的分析查询系统兴趣的提高促使 Pivotal 建立了一个新的查询优化器。

在本文中我们将会介绍 Orca 架构。它是 Pivotal 所有数据管理产品使用的新优化器，目前已经被应用在 Pivotal Greenplum 数据库和 Pivotal HAWQ 产品中。Orca 是一个全面发展，将最先进的查询优化技术和自身的原创研究相结合形成的一个模块化的和可移植的优化器架构。除了描述 Orca 的整体架构，我们还重点评价它的几个独特的功能和提供和其他系统的性能比较。

## 1. Introduction 
大数据让查询优化这个研究方向重新变得火热。由于新一代的数据管理系统拥有较好的扩展能力，数百兆甚至千兆字节的数据集都可以通过 SQL 或类 SQL 工具进行分析。好的和平庸的查询优化器之间的差距一直是很大的，而数据量的提升会进一步放大优化的错误和彰显查询优化的重要性。

尽管在查询优化领域已经有大量的研究，但在商业和开源项目中大多数现有的查询优化器仍然主要基于商业数据库发展早期的技术，并且经常容易产生次优的结果。认识到理论研究和实际实现的差距后，我们着手设计了一套架构满足当前的查询优化需求，并且能够为未来的发展提供足够的扩展性。

在本文中，我们描述了 Orca 架构，这是我们最近在 Greenplum/Pivotal 的研究和开发成果。Orca 是一个先进的查询优化器，专门为苛刻的分析工作负载设计。它和其他优化器的区别主要体现在以下几个方面：
* 模块化，使用高度可扩展的元数据和系统描述抽象。它不再像传统的查询优化器一样局限于一个特定的系统，能够通过 Metadata Provider SDK 提供的插件快速移植到其他的数据管理系统；
* 可扩展性，将查询的所有元素和它的优化作为同等重要的一等公民，避免了多阶段优化的陷阱；
* 多线程优化好，Orca 部署了多核感知调度器，能够将各个细粒度的优化子任务分布在多个核心上以加快优化的过程；
* 可验证性，Orca 提供了专门的机制来验证查询优化的正确性和性能，改善了工程实践的体验，并且这些工具能够提高开发效率，缩短开发新功能和 bug 修复的时间；
* 性能，Orca 比以前的系统在性能上提升很多，大部分情况下查询速度提高了10倍甚至能够达到1000倍。

我们将会描述 Orca 的架构和介绍它的设计带来的一些先进功能。

<img width="500" alt="image" src="https://github.com/yucorn/db_readings/assets/54345716/3cb8f448-4b24-4d49-90ef-0cf51229f479">


## 2. Preliminaries
### 2.1 Massively Parallel Processing

### 2.2 SQL on Hadoop

## 3. Orca Architecture
Orca 是基于 Cascade 框架的自顶向下的查询优化器。虽然许多 Cascade 优化器和主机系统关联紧密，但 Orca 的独特功能使得它能够作为独立的查询优化器运行在数据库系统以外。这种能力对于使用优化器能力支持不同架构的产品至关重要。它还允许运行将优化器作为一个独立的产品来运行而不需要和主机系统绑定。

<img width="500" alt="image" src="https://github.com/yucorn/db_readings/assets/54345716/f1bccbd4-fbd3-4cd3-b760-720d7c971919">
<img width="500" alt="image" src="https://github.com/yucorn/db_readings/assets/54345716/33021064-8c9c-40fe-9b69-e36ccb3a9792">

**DXL** 

## 4. Query Optimization

## 5. Metadata Exchange

## 6. Verify Ability

## 7. Experiments

## 8. Related Work

## 9. Summary
