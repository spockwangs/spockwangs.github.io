---
layout: post
title: 使用Booch方法进行面向对象分析与设计
categories:
- programming practice
---

## Introduction

Brian Kernighan在_[Software Tools][software_tools]_一书中表达了这样的观点：“编程的本质就是控制
复杂性”。正是软件的复杂性导致了大量软件项目的失败、延期或者超出预算，甚至带
来巨大灾难。纵观整个编程方法学和软件工程的发展历程，无不是在为驯服软件的复
杂性而努力。如果把眼光不仅仅是局限在软件领域，我们发现其它领域也一样表现出巨
大的复杂性，例如，跟软件密切联系的硬件领域，生物，人体构造，物质结构和社会
组织等。自然界到处充斥着极其复杂的对象，但是软件的复杂性与自然界的复杂性不
一样，正如Brooks所说：

> Einstein argued that there must be simplified explanations of nature,
> because God is not capricious or arbitrary. No such faith comforts the
> software engineer. Much of the complexity that he must master is arbitrary
> complexity.

面向对象的编程方法正是为应对软件复杂性而提出的目前最流行的方法。面向对象编程
语言也因此变得极为流行，但是会流利地使用面向对象编程语言并不代表你会使用面向
对象的编程方法。是否采用了面向对象的编程方法跟是否使用面向对象的编程语言是毫
无关系的，只不过面向对象编程语言提供的特性使得面向对象软件构造变得更为容易。
即使使用非面向对象的语言如C语言也一样可以进行面向对象编程。
在_[Object-oriented Analysis and Design with Applications][ooad]_一书中，
Grady Booch将面向对象编程定义为：

> Object-oriented programming is a method of implementation in which
> programs are organized as cooperative collections of objects, each of
> which represents an instance of some class, and whose classes are all
> members of a hierarchy of classes united via inheritance relationships.

面向对象编程的先驱Alan Kay这样[定义面向对象编程](http://userpage.fu-berlin.de/~ram/pub/pub_jf47ht81Ht/doc_kay_oop_en)：

> OOP to me means only messaging, local retention and protection and hiding
> of state-process, and extreme late-binding of all things.

Alan Kay关于面向对象编程的定义最核心的是对象间的消息通信，就如同人与人之间
的交流一样。一个团队或组织的成员之间相互交流沟通一起合作完成一定的任务，其
实就是一个面向对象的系统，而这正是我们每天要参与其中的事情，我们每个人就在
扮演一个对象的角色，这也难怪面向对象编程的思想即使对于不懂计算机的人来讲仍
然是非常直观的。

## 如何进行面向对象分析与设计

有许多人曾经将编程与写作进行类比，因为编程并不太像其它工程学科，编写优秀的
程序需要一定的创造性，并没有什么套路可供遵循。因此，面向对象的分析与设计过
程也无法像菜谱一样精确描述，但是我们仍然可以提供一套经验指南。

Grady Booch将面向对象的分析与设计定义为一系列迭代过程，每个迭代包括如下过程：

1. 需求分析：分析用户的需求，定义问题的范畴。
2. 分析与设计：设计系统的架构和实现机制，即定义类和对象及其之间的关系。
3. 实现：实现、单元测试并与已有设计集成，形成一个可供执行的系统。
4. 测试：集成测试以保证满足了系统需求。
5. 部署：部署已实现的系统以供客户使用。

每次迭代的侧重点不一样，根据其侧重点的不同每个迭代处于不同的阶段：

1. Inception：对客户的需求进行分析，确定问题的边界和范畴，哪些需要解决，哪
   些不需要解决。对需求排出优先级并确定核心需求。确定开发环境和理解风险。
2. Elaboration: 这个阶段的主要目标是设计出稳定的系统架构。架构的设计需要考
   虑到满足相互冲突的需求之间的权衡，优先考虑核心需求。
3. Construction: 这个阶段的目标是构造一个可供测试的系统。
4. Transition: 这个阶段的目标是确保软件可被客户接受，保证易用性。

这些就是Booch所说的宏观过程，可以看出与一些流行的敏捷开发过程很相似。

宏观过程中的“分析与设计”又进一步分解为下面的微观过程：

1. Identify the elements: 找出需要哪些对象。一些对象可以从问题领域所用的术
   语中寻找，一些对象需要在设计过程中发明出来，这正是需要创造性的地方。
2. Define the collaborations between the elements: 定义对象的行为及其之间的
   合作关系。分析需求场景，为每个对象分配责任，定义它们之间的合作关系，并注
   意寻找行为的共同点和相似性，将其抽象出来以便复用。
3. Define the relationships between the elements: 识别对象间的语义关系，并细化这种关系（语义依赖、聚合关系、继承关系）。
4. Define the semantics of the elements: 细化每个对象的责任，明确其接口。

注意上述过程不一定要按上面描述的顺序进行，也不一定要严格分开。

如果上述过程有点抽象难懂，我们可以将其与团队管理相类比进行理解。比如，为了完成某个任务，
仅凭你一个人是不够的。你需要自己筹建一个团队通过成员的合作来完成这个任务。
你首先要考虑需要什么样的人（identify the elements)，每个人需要什么技能，招
聘广告怎么写（define the collaborations between the elements）。招到你需要
的人后要给讲清楚他们之间的关系（哪些人可以与哪些人沟通，哪些人向哪些
人汇报工作，哪些人之间是平级关系，哪些人是上下级关系等等）（define the
relationships between the elements)，并详细规定他们各自的责任和义务，哪些可以做，哪些
不可以做（define the semantics of the elements)。这个场景也折射出一些设计原
则。比如分配责任时要避免过于集中，每个人都应该只负责某一个方面，以发挥出其
优势，这不就是著名的“Single Responsiblity Principle”吗。成员之间需要的沟通
路径要尽可能少，如果任何两个成员都需要沟通才能完成任务，则会浪费大量的时间
和精力，导致效率低下。如果任何两个对象都有关联，这个架构也就够乱的，完全没
有结构可言，理解起来也很困难。如果这个团队的任务越来越多，人手不够就需要继
续招人，人一多沟通路径就必然会增加，管理将变的更困难，效率必然下降，这时就
需要分离出一些独立的小组，行成一个层次关系，以方便管理。

## 面向对象与并发和分布式计算

随着多核处理器的普及，并发编程变得日益重要。但是并发编程往往涉及到管理线程
和保护共享状态，容易出错。面向对象的消息通信机制有效解决这个问题。我们可以
用对象将控制流抽象出来，这样的对象称为active object，其它对象叫做passive
object. 不同的active object可以并发执行并通过消息传递来同步，他们之间没有共
享状态，大大减轻了程序员的负担。

为了减轻并发编程的痛苦，也出现了一些其它的编程模型，比较著名的有Tony Hoare的
[Communicating Sequential Process][csp]和Carl Hewitt等人提出的[Actor
Model][actor_model]，不过它们都大同小异，都是通过某个实体（CSP中的Process，
Actor Model中的Actor）是将控制流抽象出来并通过消息通信机制来同步以避免共享状
态。

面向对象编程还可以用于分布式计算，只不过这时active object分布在不同的计算机
上，通过网络相互通信。

## References

* Brian W. Kernighan, P. J. Plauger, _[Software Tools][software_tools]_.
* Grady Booch, _[Object-Oriented Analysis and Design with Applications][ooad]_, Second Edition.
* [Actor Model][actor_model]
* [Communicating Sequential Process][csp]

[software_tools]: http://www.amazon.com/Software-Tools-Brian-W-Kernighan/dp/020103669X
[ooad]: http://www.amazon.com/Object-Oriented-Analysis-Design-Applications-3rd/dp/020189551X
[actor_model]: http://en.wikipedia.org/wiki/Actor_model
[csp]: http://en.wikipedia.org/wiki/Communicating_sequential_processes
