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

**DXL** DXL 为了将数据库系统和优化器解耦需要建立一种通信机制来处理查询，这种机制在 Orca 中被抽象为 Data eXchange Language 即 DXL。Orca 框架利用一种基于 XML的语言来编码优化器和数据库系统通信的必要信息，比如输入的查询、输出的查询计划和元数据。DXL 上层是一个简单的通信协议，用于发送初始数据查询结构和检索查询计划。
Figure 2 展示了 Orca 依赖 XML 和外部数据库系统的交互方式。Orca 的输入是用 DXL 表示的查询，输出是用 DXL 表示的查询计划。Orca 通过允许数据库系统注册元数据的提供方来抽象元数据的访问细节，该提供者负责将元数据发送给 Orca 之前先序列化成 DXL 格式。Orca 也支持从包含 DXL 格式序列化的元数据对象的常规文件中消费元数据。
 数据库系统需要引入消费、输出 DXL 格式数据的翻译器。Query2DXL 翻译器将查询查询语法树解析成 DXL 查询，而 DXL2Plan 翻译器将 DXL 计划翻译成数据库可执行的查询计划。这些翻译器的实现是在 Orca 框架以外完成的，不同的数据库系统可以通过提供适当的翻译器来使用 Orca。
Orca 的架构具有高度的可扩展性，所有组件都可以被个性化的配置或替代。Figure 3 展示了 Orca 的不同组件，下面我们简要介绍下这些组件。
**Memo**  Orca 产生的备选执行计划被编码在紧凑的内存数据结构中，我们称之为 Memo。 Memo 结构由一组称为 groups 的容器组成，每个 group 上包含逻辑等价的表达式。Memo groups 又称为 group expressions 记录了查询的不同子目标（比如查询的条件过滤、两个表之间的连接）。Group member 以不同的逻辑方式实现组目标。每个 group expression 是一个包含其他 group 作为子节点的算子。这种递归的结构允许对巨大的可能计划空间进行紧凑的编码，我们在 4.1 节会继续讨论。 
Search and Job Scheduler Orca 使用搜索的机制来浏览所有可能得查询计划，并确定评估出成本最低的执行计划。搜索机制是由专门的 Job Scheduler 负责的，它通过创建相互依赖或并行的工作单元在三个主要的步骤中进行查询优化：
* exploration 生成等价的逻辑表达式
* implementation 生成物理执行计划
* optimization 强制执行所需要的物理属性，并计算计划的可替代方案的执行成本。
我们会在 4.2 节继续讨论优化工作调度的细节。
**Transformations** 备选查询计划是通过应用转换规则生成的，这些规则可以产生任何等价的逻辑表达式，比如 InnerJoin(A, B) -> InnerJoin(B, A)，或者是当前表达式的物理实现，比如 Join(A, B) -> HashJoin(A, B)。通过应用转换规则产生的结果被复制到 Memo 中，对应的操作是创建新的 group 或者向已有的 groups 中添加新的 group。每种转换规则都是一个独立的部分，可以在 Orca 中配置为激活或者停用。
**Property Enfircement** Orca 包括一个可扩展的框架，用来描述查询要求和基于正式属性规范的计划特征。Propertiy 有不同的类型，包括逻辑属性，比如 输出列，物理属性，比如 sort order 和 data distribution 和 标量属性，比如 用于连接条件的列。在查询优化过程中，每个算子可以从其子节点中请求特定的属性，一个优化的子计划可以自己满足所需的属性或者需要在计划中插入一个执行者。Orca 框架允许每个算子根据子计划的属性和算子的本地行为来控制执行者的位置。我们会在 4.1 节中讨论更多细节。
Metadata Cache 由于元数据变化的并不频繁，每次查询都重新传输元数据会产生交工的开销。Orca 在优化器上缓存元数据，只有当缓存中的数据不可用或者缓存数据发生了变化时才会从 catalog 中检索需要的部分元数据。元数据缓存还将数据库系统的细节从优化器中抽象出来，这在测试和调试时非常有用。
GPOS

## 4. Query Optimization
我们会在 4.1 节讨论 Orca 优化器的工作流程，然后在第 4.2 节展示优化过程是如何被并行的构建的。

### 4.1 Optimization Workflow
我们用下边的实例来说明查询优化的工作流程：
SELECT T1.a FROM T1, T2 WHERE T1.a = T2.b ORDER BY T1.a，其中 T1 表的分布是 Hashed(T1.a)，T2 表的分布是 Hashed(T2.a)。
List1 展示了这个查询的 DXL 表示方式，其中给出了输出列、排序列、数据分布和逻辑查询。元数据用 Mdid 表示，以便于在优化过程中请求进一步的信息。 Mdid 是一个唯一的标识符，由数据库系统标识符、对象标识符和版本号组成。
DXL 查询信息被传送到 Orca 后被解析并转化为呢村中的逻辑表达树，然后被复制到 Memo 中。
Figure4 展示了 Memo 的初始内容。该逻辑表达式为两个表和 InnerJoin 算子创建了三个 group。为了简洁，我们在图中省略了具体的 Join 条件。Group0 被称为 root group 因为它和逻辑表达式的 root 想对应。逻辑表达式中算子之间的依赖关系被描述为不同 group 之间的引用，比如 InnerJoin(1, 2) 代表的是引用 Group1 和 Group2 作为子节点。优化工作按以下步骤进行。
（1）Exploration 生成对数等价表达式的规则被触发，比如 Join Commutativity 规则被触发，从InnerJoin(1, 2) 中产生 InnerJoin(2, 1)。Exploration 的结果是向现有的 group 添加新的 group expression 或者创建新的 group。Memo 结构有一个内置的基于表达式拓扑结构的重复检测机制，用来检测和消除任何由不同转换规则生成的重复表达式。
（2）Statistics Derivation 当 Exploration 阶段完成后，Memo 中包含了给定查询的完整的逻辑空间。然后 Orca 的统计推到机制被触发，开始计算 Memo groups 的统计数据。Orca 的统计对象主要是一个列图的集合，用于推到
（3）Implementation 
（4）Optimization 

### 4.2 Parallel Query Optimization	

## 5. Metadata Exchange

## 6. Verify Ability

## 7. Experiments

## 8. Related Work

## 9. Summary
