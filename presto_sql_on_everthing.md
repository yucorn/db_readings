# Presto: SQL on everything

## 1. Introduction
## 2. Use Cases
## 3. Architecture Overview
Presto 集群由一个 coordinator 节点和多个 work 节点两部分组成，Figure 1 展示的是 Presto 集群的架构概览图。
* coordinator node，负责受理请求、解析请求、生成查询计划、进行查询优化以及协调查询的执行；
* worker node，负责查询的具体执行，包括访问数据源获取查询结果等步骤。
<img width="529" alt="image" src="https://user-images.githubusercontent.com/54345716/236815277-4982237f-3854-40e7-a601-f7e2f5360646.png">
一条查询语句是这样被 Presto 执行的：
* Client 向 coordinator 发送一条包含 SQL 语句的 HTTP 请求；
* Coordinator 通过评估队列策略、解析和分析 SQL 语句，创建和优化分布式的执行计划；
## 4. System Design
## 5. Query Processing Optimization
## 6. Performance
## 7. Related Work
## 8. Conclusion
