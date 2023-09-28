---
title: "DAGScheduler"
date: 2019-04-03T10:57:56+08:00
draft: true
---

`DAGScheduler` 是Apache Spark实现面向阶段调度（stage-oriented scheduling）的调度层。负责将一个逻辑的执行计划（比如，通过RDD转换操作构建的RDD运算图）转换为一个物理的执行计划（由不同的阶段组成）。
![Transforming RDD Lineage Into Stage DAG](https://ws4.sinaimg.cn/large/006tKfTcgy1g1p9k2y9w5j30bt0ezmyz.jpg)

当一个动作（action）被调用时，`SparkContext`将一个逻辑计划移交给`DAGScheduler`，由`DAGScheduler`来负责将它转换成一系列的阶段，并封装为一个`TaskSets`进行提交执行（see `Execution Model`）。
![Executing action leads to new ResultStage and ActiveJob in DAGScheduler](https://ws3.sinaimg.cn/large/006tKfTcgy1g1p9p2wj5xj30ig0boq3m.jpg)

![DAGScheduler as created by SparkContext with other services](https://ws4.sinaimg.cn/large/006tKfTcgy1g1p9v89wffj30na0ec0ud.jpg)

`DAGScheduler`主要负责三件事：
1. 计算一个Job的执行DAG图
2. 确定每个task首选的执行位置
3. 处理失败