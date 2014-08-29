---
layout: post
title:  列式内存数据库的一种快速索引方案浅析
category: articles
tags: Fast-Loopups In-Memory Column-Store
image:
    feature: head11.jpg
author:
    name:   WuYu
    avatar: bio-photo-alt.jpg
comments: true
share: true
---

最近由于实验室需要设计和实现一个支持OLAP的列式内存数据库，所以对一些典型的同类型数据库产品进行了一番调研。这里我把SAP HANA里基于内存的列式数据库索引引擎p*Time中所使用的索引技术整理和记录了一下，希望对大家能有所帮助。

####wikipedia上对HANA的定义如下：
SAP HANA, short for "High-Performance Analytic Appliance" is an in-memory, column-oriented, relational database management system developed and marketed by SAP AG. It runs massively parallel, thus exploiting the maximum out of multicore processors and subsequently enabling very fast query execution.具体可以参加HANA的白皮书。

基于内存的列式数据库一般都采用读写分离策略，数据更新作用在一个专门为写优化的区域，这个区域通常都很小并且采用树结构（比如B类树）的索引来加速更新操作。而只读区域一般都很大，并且同样需要建立索引以提高事务和分析的性能。p*Time提出了一套只读区域的索引方案，该方案在查询执行时能急剧降低内存消耗，因此可以快速查询和进行合并，非常有利于事务处理和分析处理。
