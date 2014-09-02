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
mathjax: true
---

最近由于实验室需要设计和实现一个支持OLAP的列式内存数据库，所以对一些典型的同类型数据库产品进行了一番调研。这里我把SAP HANA里基于内存的列式数据库索引引擎p*Time中所使用的索引技术整理和记录了一下，希望对大家能有所帮助。

####wikipedia上对HANA的定义如下：
> SAP HANA, short for "High-Performance Analytic Appliance" is an in-memory, column-oriented, relational database management system developed and marketed by SAP AG. It runs massively parallel, thus exploiting the maximum out of multicore processors and subsequently enabling very fast query execution.

####概述

HANA采用读写分离策略，数据更新作用在一个专门为写优化的区域（delta partition），这个区域通常都很小并且采用树的结构（比如B类树）建立相关索引。而只读区域（main partition）一般都很大，并且同样需要建立索引以提高事务和分析业务的性能。p*Time提出了一套独特的main partition的索引方案Group-key Index，该方案在查询执行时能急剧降低内存消耗，因此对内可以快速进行索引查询，对外可以快速执行联合查询，这对OLTP和OLAP来说都非常有利。

####核心数据结构

为了更好的节省内存开销，Group-key Index索引方案中的大部分数据结构都采用了位压缩，比如在32位机器上，一个整型占用4个字节，如果我们要表示的数不是很大的情况下，这往往都是浪费内存的，因此更“经济”的解决方案是如果我们所需要的最大的数为n，那么就把表所要表示的数的位数限定为$  logn ^2$。尽管OLAP型业务往往需要操作整个表或者域，甚至对表进行多维度联合查询，但是对于列式数据库中拥有字典索引的列，p*Time采用疏松索引来取代密集索引以获得更高的查找效率。

p*Time所用到的核心数据结构有以下4个，其中前两项是main partition需要的，而后两项是Group-key Index需要的

* 排好序的字典 ( $ U_M^j $ )
* 位压缩过的属性向量表 ( $ V_M^j $ )：存储了标识原始数据在字典中的偏移，也叫value-ids。
* 偏移向量表 ($ I^j $)：辅助性数据结构，负责把指定的value-id映射到其在属性向量索引表的偏移。
* 属性向量索引表 ($ P^j $)：存储value-id在属性向量中的位置。

####下文这四种数据结构均以符号标示。

它们之间的关系见下图所示：

![relation](/images/fast1.png)

####如何查找

我来举个简单的例子说明这4个数据结构是如何协作的，比如需要查询charlie的位置（HANA叫RowID），首先在$ U_M^j $中用二分查找找到charlie，然后读出$ I^j $中相同偏移的值（1）以及下一偏移存储的值（3），注意字典项和偏移向量表项一一对应。拿到1和3说明charlie这个单词在$ P^j $中偏移为1和2，因为$ I^j $中相邻两项记录的是左闭右开区间。然后在$ P^j $中偏移为1和2处拿出值5和6，这步一般可以一次性取出值，因为$ U_M^j $中的指定值如果出现不止一次，它们在$ P^j $中的相关信息都是连续存放的。5和6就是该字符串的RowID，也就是其位置，而且可以发现$ V_M^j $中偏移为5和6位置的value-id都为1，恰好是charlie在$ U_M^j $中的偏移。

####如何重建Group-key Index

在拥有$ U_M^j $和$ V_M^j $的情况下，重建Group-key Index的步骤：

1. 生成$ I^j $：先创建一个空的$ I^j $（每一项都填0），长度和$ U_M^j $长度相同，遍历$ V_M^j $查找value-ids的出现次数，比如上图，假如需要根据左面两个数据结构生成Group-key Index，apple的value-id是0，在$ V_M^j $中查找0出现的次数发现只有1个，就在$ I^j $第1项填1（0+1），注意填$ I^j $时从第一项开始（所有数据结构的偏移都是从0开始的，程序猿应该很好理解）。同理，charlie的value-id是1，在$ V_M^j $查找发现出现2次，在$ I^j $第2项填上一次求得结果加上出现次数，即1+2=3，以此类推直至扫到字典末尾。
2. 生成$ P^j $：扫描$ I^j $，获取每一项的键值对（key,value），key是$ I^j $每一项的位置：第0项，第1项等等，value就是该位置存储的值，也是该value-id映射在$ P^j $中的位置信息，比如上图取出（0,0），（1,1），（2,3）......。拿着key去$ V_M^j $中查询，比如key 0在$ V_M^j $中偏移是4，这是我们要填入$ P^j $的值，key 0对应的value即是要填入的位置，这里是0，所以把4填入$ P^j $中偏移为0的位置。然后用$ I^j $的下一项键值对的值减去本键值对的值来获取该key在$ V_M^j $中出现的次数，这里发现1-0=1，说明只出现1次，所以已经填入完毕。但是对于键值对（1,1）发现该key出现的次数是3-1=2次，所以在$ V_M^j $中搜索key 1要搜两次并把其在$ V_M^j $中的位移依次填入$ P^j $的相关位置，以此类推，直至扫描完$ I^j $。

####如何生成main partition中的字典和属性向量

如下图所示：

![rebuild main](/images/fast2.png)

不要被图吓到，其实操作步骤很简单。首先必须要说明一下，我们刚才讲的索引数据结构只适用于main partition，那么对于delta partition是怎样进行索引的呢？其实前面也提到了就是B+树。上图已经给出了步骤我这里解释一下，首先标示为Inputs Step 1是第一步我们需要的输入，Outputs Step 1就是我们第一步需要输出的东西，包括新字典$ U_M^j $，辅助性数据结构增量向量表$ X_M^j $和delta partition的属性向量表$ V_D^j $。$ U_M^j $由$ U_M^j $和delta partition中的B+树合并而成，我们首先同时扫描$ U_M^j $和B+树的叶子节点，按照字符串顺序（字典顺序）做归并排序即可生成$ U_M^j $，在生成$ U_M^j $的同时需要生成$ X_M^j $，比如新插入个bodo，这是$ U_M^j $没有的，就在bodo所在的$ U_M^j $偏移1开始，依次把$ X_M^j $中的每一项加1，如上图中的$ X_M^j $所示。delta partition中的$ V_D^j $就根据新字典来填即可。

Inputs Step2的输入部分是$ X_M^j $ ，$ V_M^j $和刚才第一步新生成的$ V_D^j $ ，Outputs Step 2输出的就是新的属性向量$ V_M^j $ ，那么它是如何生成的呢？先把$ V_M^j $根据$ X_M^j $做一个更新然后和$ V_D^j $做合并，比如$ V_M^j $中第一项是4，找到$ X_M^j $中偏移为4的项值是1，所以将5（4+1）填入$ V_M^j $第一项，以此类推。全部更新完成后直接把$ V_D^j $放到$ V_M^j $末尾，也就是简单地合并即可。

这样新的字典和属性向量就生成了，之前还有Group-key Index的偏移向量和属性索引向量的生成，有没有觉得很麻烦，其实还有一种更省内存的方式来同时生成这4个向量数据结构。如下图所示：

![rebuild all](/images/fast3.png)

这里给出伪代码，我就不详细解释了，其实就是我刚才讲的重建Group-key Index和构建mian partition中字典和属性向量步骤的融合，当然中间HANA又用了一些小技巧和辅助性数据结构，试着跟着代码走一下相信大家就可以理解如何操作了，其实这个算法本身并不是难点，而是如何实现以尽量节省内存开销才是p*Time重点关注的事情，也是它的一个核心技术。

![code](/images/fast4.png)

如果大家对此还觉得有些绕，可能是因为之前有一些是HANA自身架构设计导致的，建议看一下HANA的白皮书 [^1]，相信就会有比较深刻的认识。

####本文源自论文 [^2]

[^1]: <http://www.saphana.com/docs/DOC-1381>

[^2]: <http://adms-conf.org/faust_adms12.pdf>
