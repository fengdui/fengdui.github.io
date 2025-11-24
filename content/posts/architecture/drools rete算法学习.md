---
title: "drools rete算法学习"
date: "2017-04-03"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

fact: 对象之间及对象属性之间的关系，例如Product类及其类目下的points、gift属性等Root。  
Node：根节点，Rete网络入口。  
Type Node：对应不同的fact对象，也就是规则用到的POJO，每个fact就是一个TypeNode节点  
Alpha Node ：对应规则里的每个条件，例如规则条件Product(points < 100)中，points<100就是一个AlphaNode  
Beta Node：用于组合两个fact的alpha条件或BetaNode与fact条件的组合。  
LeftInputAdapter Node：用来对2个规则队形进行比较，将一个single Object 转化为一个单对象数组（因为BetaNode左边入口往往是一个list规则队形），传播到 JoinNode 节点。  
Join Node ：用于聚合BetaNode节点的结果。  
Drools 中的 Rete 算法被称为 ReteOO，表示 Drools 为面向对象系统（Object Oriented systems）增强并优化了 Rete 算法。在 Drools 中，规则被存 放在 Production Memory（规则库）中，推理机要匹配的 facts（事实）被存在 Working Memory（工作内存）中。当时事实被插入到工作内存中后，规则引擎会把事实和规则库里的模式进行匹配，对于匹配成功的规则再由 Agenda 负责具体执行推理算法中被激发规则的结论部分，同时 Agenda 通过冲突决策策略管理这些冲突规则的执行顺序，Drools 中规则冲突决策策略有：(1) 优先级策略 (2) 复杂度优先策略 (3) 简单性优先策略 (4) 广度策略 (5) 深度策略 (6) 装载序号策略 (7) 随机策略 。  

Rete算法是目前使用最广泛的规则匹配算法，由Charles L. Forgy博士在1979年发明。Rete算法是一种快速的Forward-Chaining推理算法，其匹配速度与规则的数量无关。 Rete的高效率主要来自两个重要的假设：  

时间冗余性。 facts在推理过程中的变化是缓慢的, 即在每个执行周期中,只有少数的facts发生变化，因此影响到的规则也只占很小的比例。所以可以只考虑每个执行周期中已经匹配的facts.  
结构相似性。许多规则常常包含类似的模式和模式组。  
Rete算法的基本思想是保存过去匹配过程中留下的全部信息,以空间代价来换取执行效率 。对每一个模式 ,附加一个匹配元素表来记录WorkingMemory中所有能与之匹配的元素。当一个新元素加入到WorkingMemory时, 找出所有能与之匹配的模式, 并将该元素加入到匹配元素表中; 当一个无素从WorkingMemory中删除时,同样找出所有与该元素匹配的模式,并将元素从匹配元素表中删除。 Rete算法接受对工作存储器的修改操作描述 ,产生一个修改冲突集的动作 。  

Rete算法的步骤如下：  

将初始数据（fact）输入Working Memory。  
使用Pattern Matcher比较规则（rule）和数据（fact）。  
如果执行规则存在冲突（conflict），即同时激活了多个规则，将冲突的规则放入冲突集合。  
解决冲突，将激活的规则按顺序放入Agenda。  
使用规则引擎执行Agenda中的规则。重复步骤2至5，直到执行完毕所有Agenda中的规则。  


https://www.jianshu.com/p/5b0668ebc440?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation  
https://www.cnblogs.com/shangxiaofei/p/6245626.html