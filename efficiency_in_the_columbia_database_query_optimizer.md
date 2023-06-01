# Efficiency in the columbia database query optimizer

## Chapter 1. Introduction
### 1.1 Motivation for This Research
尽管查询优化器作为课题已经被研究超过15年了，但查询优化器仍然是数据库系统最大和最复杂的模块。一些新的应用领域比如 DSS（决策支持系统）、OLAP、数据湖等反过来需要新的数据库技术，比如和传统事务处理应用程序中不同的新的查询语言、新的查询处理技术。过去这些年，人们研究出了几代用于商用和学术的查询优化器，持续提升优化器的可扩展性和执行效率。
第一代查询优化器的出现始于10年以前，包括 Exodus 和 Starburst 项目。他们的目标是为了方便查询优化器更容易摸坏化和实现。它们使用的技术包括 layering of components、rule-based transformations 等，主要的缺点在于难于进行扩展、搜索性能欠佳和偏向于面向记录的数据模型。
第二代可扩展性优化器工具，比如 Volcano 优化器
第三代优化器框架比如Cascades，使用面向对象的设计简化了实现、扩展和修改优化器的复杂度，同时保持查询效率让搜索策略更加灵活。最新一代的优化器满足了现在商业数据库系统的要求，诸多的基于Cascades框架的优化器陆续面世。
这三代优化器可以被划分为联众类型，以Starburst为代表的自底向上的动态规划的优化器和以Cascades为代表的自顶向下的规则驱动的基于成本的优化器。自底向上的优化在传统商业数据库系统中被广泛应用，因为他被认为是高效的。但自底向上的优化本质上是不如自顶向下的优化可扩展，因为它需要将原问题分解为子问题。同时为了提高查询性能，自底向上的优化器需要启发式的优化策略。
尽管以往基于自顶向下策略实现的优化器在性能上比不上自底向上的优化器，但我们相信自顶向下的优化器在效率和扩展性上都更具有优势。本文余下的内容会介绍我们基于自顶向下优化器框架实现的Columbia框架。
### 1.2 Overview of This Thesis

## Chapter 2. Terminology
本章节回顾查询优化的基础概念和术语，这些概念同样被应用到了Columbia优化器的设计和实现中。
### 2.1 Query Optimization

<img width="800" alt="image" src="https://github.com/yucorn/db_readings/assets/54345716/2188e833-5d52-4891-9dcc-55b04794e881">
