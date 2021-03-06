# 第1章 可靠性，可伸缩性，可维护性

- 数据密集型和计算密集型
  - 现在很多应用程序都是计算密集型，而非计算密集型
  - 因此CPU很少成为瓶颈，更大的问题来自数据量、数据复杂性、数据变更速度
- 数据密集型应用由标准组件构建而成，标准组件提供了通用的功能，例如
  - 数据库
  - 缓存
  - 搜索索引
  - 流处理，向其它进程发送消息，进行异步处理
  - 批处理，定期处理累积的大批量数据
- 上述这些功能，被抽象为数据系统
  - 平时大家都在用一些成熟的组件，例如数据库
  - 数据库、缓存、搜索索引都是多样化的，需要根据具体的业务场景进行选择
- 本书探讨的内容
  - 数据系统原理、实践与应用
  - 设计数据密集型应用的方法
  - 不同工具之间的异同，以及各自的实现原理
- 本章探讨的内容
  - 可靠、可伸缩、可维护的数据系统
  - 搞清楚这些词语的含义，并且概述如何尽量量化这些目标
  - 回顾一些后续章节需要的基础知识
- 后面的章节
  - 探讨设计数据密集型应用时，可能遇到的设计决策

## 1.1 关于数据系统的思考

- 数据库、消息队列、缓存等，差异明显，却总称为数据系统
  - 现在，很多数据存储和处理工具，使得这些类别之间的界限模糊
    - 数据存储可以当成消息队列用（Redis）
    - 消息队列有类似数据库的持久保证（Apache Kafka）
  - 单个工具不足以满足所有的数据处理和存储需求，总体工作被拆分成若干个工具，并通过应用代码缝合起来
    - 例如，如果将缓存和全文搜索功能从主数据库剥离出来，那么使缓存与主数据库保持同步通常是应用代码的责任，图1-1给出了这种架构可能的样子，细节会在后续章节讨论
- 应用程序编程接口（API）
  - 将多个工具组合在一起提供服务时，服务的接口就是API
  - 向用户隐藏这些工具以及组合的实现细节
  - 就这样通过把较小的通用组件组合起来，创建了一个数据系统
  - 因此，我们不仅要开发应用程序，还要设计数据系统
- 设计数据系统或应用时可能遇到的问题
  - 当系统出问题时，如何确保数据的正确性和完整性
  - 部分系统退化降级时，如何不降低服务质量（延时等指标）
  - 负载增加时，如何扩容
  - 怎样的API才是好的API
- 本书着重讨论三个在软件系统中很重要的问题
  - 可靠性
    - 遇到困境（故障、人为错误等）中仍可正常工作
  - 可伸缩性
    - 有合理的办法应对系统的增长（数据量、流量、复杂性等）
  - 可维护性
    - 许多不同的人（工程师，运维等）在不同的生命周期，都能高效在系统上工作

## 1.2 可靠性

- 可靠性可以粗略理解为，即使出现问题，也能继续正确工作
- 造成错误的原因叫故障
  - 故障和失效
    - 故障不等于失效
    - 故障通常是系统的某一部分状态偏离其标准
    - 失效是系统作为一个整体，停止向用户提供服务
    - 故障没法完全避免，而容错机制就是为了防止因为故障而导致失效
  - 通过故意触发来提高故障率是有意义的
    - 许多高危漏洞实际上是由糟糕的错误处理导致的，因此可以通过故意引发故障，确保容错机制不断运行并接受考验
    - 例如，在没有警告的情况下杀死一个进程
- 能预料并应对故障的系统特性称为容错
  - 容错是对于某些特定类型的错误而言的，不可能容忍所有错误

- 阻止错误和容忍错误
  - 比起阻止错误，更倾向于容忍错误
  - 也有预防胜于治疗的情况，尤其是当不存在治疗方法时，例如安全问题

### 1.2.1 硬件故障

- 磁盘的平均无故障事件为10~50年
- 为了减少故障率，增加单个硬件的冗余度
  - 磁盘可以组件RAID
  - 数据中心可能有电池和柴油发电机作为后备电源
  - ...
- 现在，一般应用的硬件冗余度已经足够，但是应用越来越大量使用机器，会增加机器故障率，为了在容忍机器故障的路上更进一步，在硬件冗余的基础上进一步引入软件容错机制
  - 为什么引入软件容错机制可以做到这些？？？

### 1.2.2 软件错误

- 硬件故障通常是随机的、相互独立的
  - 例如，一台机器的磁盘失效通常不意味着另一台机器的磁盘也会失效
  - 除非它们存在比较弱的相关性，例如服务器机架的温度
- 除了硬件错误，另一类错误是内部的系统性错误
  - 这些错误难以预料，而且通常是跨节点相关的，会造成更多的系统失效
  - 一些例子
    - 接受特定的错误输入，导致所有应用服务器示例崩溃的bug
    - 失控进程会用尽一些共享资源
    - 系统以来的服务变慢，或者返回错误的响应
    - 级联故障，一个组件的故障触发另一个组件的故障，从而触发更多故障
- 导致软件故障的bug会潜伏很长时间，直到被异常情况触发
  - 软件对其环境做出了某种假设，这种假设当时可能是正确的，但由于某些原因，之后不再成立
- 虽然软件中的系统性故障没有速效药，但还是有很多小办法
  - 仔细考虑系统的假设和交互
  - 彻底的测试
  - 进程隔离
  - 允许进程崩溃并重启
  - ...
  - 如果系统能够提供一些保证（例如在消息队列中，进入和发出的消息数量相等），那么系统就可以在运行时不断自检，并在出现差异时报警

### 1.2.3 人为错误

- 人的行为是不可靠的，例如运维配置错误导致服务终端
- 在这个前提下，需要让系统更可靠，有以下几种办法
  - 以最小化犯错机会的方式设计系统
    - 例如精心设计的抽象、API和管理后台
    - 但是如果接口限制太多，使用起来不方便
  - 将容易犯错误的地方与可能失效的地方解耦
    - 沙箱？？？
  - 在各个层次进行彻底的测试
    - 自动化测试适用于覆盖正常情况中少见的边缘场景
  - 允许从人为错误中简单快速地恢复，以减少失效带来的影响
    - 例如，快速回滚配置的变更
    - 分批量发布新代码
  - ...

### 1.2.4 可靠性有多重要

- ###

## 1.3 可伸缩性

- 系统当下能可靠运行，不见得以后也能可靠运行
  - 服务降级的一个常见原因是负载增加，如并发用户数量的增长
- 可伸缩性是用来描述系统应对负载增长能力的属于
  - 如果系统以特定方式增长，有什么选项可以应对增长
  - 如何增加计算资源来处理额外的负载

### 1.3.1 描述负载

- 用一些称为负载参数的数字来描述负载，参数的选择取决于系统架构，例如

  - 每秒向Web服务器发出的请求
  - 数据库中的读写比率
  - 聊天室中同时活跃的用户数量
  - 缓存命中率
  - ...

- 一个描述负载的例子，以推特在2012年11月发布的数据为例

  - 推特的两个主要业务

    - 发布推文
      - 用户可以向粉丝发布新消息（平均4.6k请求/秒，峰值12k请求/秒）
      - 这里的请求，指的是发布推文
    - 主页时间线
      - 用户可以查询它们关注的人发布的推文（300k请求/秒）

  - 处理每秒12000次写入（发推文）是很容易的， 真正的伸缩性挑战来自于扇出

    - 扇出描述了为了服务一个传入请求，要执行的其他服务的请求数量

  - 一种实现方式

    - 发布推文时，将新推文插入全局推文集合

    - 当一个用户请求自己的主页时间线时，首先查找他关注的所有人，查询这些被关注用户发布的推文，并按事件顺序合并

      - 查询语句？？？

        ```sql
        SELECT tweets.*, users.*
          FROM tweets
          JOIN users ON tweets.sender_id = users.id # 在users表中找到发送推文的用户
          JOIN follows ON follows.followee_id = users.id # 在followers表中找到当前用户的所有关注用户
          WHERE follows.follower_id = current_user # 在
        ```

    - 图1-2展示了关系型模式的简单实现

  - 另一种实现方式

    - 为每个用户的主页时间线维护一个缓存
    - 当一个用户发布推文时，查找所有关注该用户的人，并将新的推文插入到每个主页时间线缓存中（相当于收件箱）
    - 读取主页时间线的开销很小，因为结果已经提前计算好了

  - 两个实现方式的比较
    - 推特的第一个版本使用了方法一，但是主页时间线查询的负载太高，因此后来转向方法二
    - 方法二的缺点是，发推要做大量额外工作，有些用户粉丝千万级，那么每发一次推文，就会导致主页时间线缓存千万次的写入
    - 现在逐步转向两种方法的混合，大部分用户发的推文采取方法二，但是少量有海量粉丝的用户采取方法一
  - 第12章会对这个例子做更详细的讨论

### 1.3.2 描述性能

- 一旦负载被描述好，就可以研究负载增加时会发生什么，可以从两种角度来看
  - 增加负载参数，并保持系统资源不变时，系统性能将受到什么影响
  - 增加负载参数，并保持性能不变时，需要增加多少系统资源
- 不同的系统，对上述两种角度的关注会有所侧重
  - Hadoop这样的批处理系统，更关心吞吐量，即每秒可以处理的记录数量
  - 在线系统，通常更重要的是响应时间
- 延迟和响应时间
  - 响应时间是用户看到的，除了实际处理请求的时间（服务时间）外，还包括网络延迟和排队延迟等
  - 延迟是某个请求处于休眠状态，等待处理的持续时长

- 响应时间的均值和百分位点
  - 图1-4展示了一个服务的100次请求响应时间的均值和百分位数
  - 均值一般被理解为算术平均值，但是这并不是一个很好的指标，因为无法得知有多少个用户经历了多大的延迟
  - 百分位点
    - 中位数是第50百分位点，缩写为p50
    - 如果p95是1.5秒，意味着100个请求中的95个响应时间快于1.5秒
    - 高百分位点（尾部延迟）很重要，可以看到这部分延迟最高的用户到底忍受了多大的延迟，而这部分客户往往是数据量最大的（数据量大更容易导致延迟）
  - 排队延迟通常占了高百分位点处响应时间的很大一部分
    - 处理器只能并行处理少量事务，后续请求被阻碍（头部阻塞），客户端最终看到的是缓慢的总体响应时间
    - 为了测试系统的可伸缩性，可以人为产生负载，产生负载的客户端要不断发送请求，而不是等待先前的请求完成再发送下一个请求，这才符合实际场景

### 1.3.3 应对负载的方法

- 前两节分别讨论了用于描述负载的参数和用于衡量性能的指标，在此基础上可以讨论伸缩性了：当负载参数增加时，如何保持良好的性能
- 纵向伸缩和横向伸缩
  - 纵向伸缩，转向更强大的机器
  - 横向伸缩，将负载分布到多台小机器（分布式）
  - 优秀架构会将这两种方法结合，因为可以在单台机器上运行的系统通常更简单，但高端机器非常贵
    - 跨多台机器部署无状态服务非常简单，但将带状态的数据系统从单节点变为分布式配置会引入额外复杂度
    - 直觉上，应该将数据库放在单个节点上（纵向伸缩），直到伸缩成本或可用性需求迫使其改变为分布式
    - 随着分布式系统的工具和抽象越来越好，上述常识可能改变，本书的其余部分会介绍多种分布式系统，讨论它们的可伸缩性，易用性和可维护性
- 弹性和手动伸缩
  - 有些系统是弹性的，可以在检测到负载增加时，自动增加计算资源
  - 其他系统是手动伸缩，即人工分析容量，并决定向系统添加更多机器
  - 负载难以预测的情况下，弹性非常有用，但手动伸缩系统更简单，且意外操作更少（见“重新平衡分区”）
- 系统架构通常是应用特定的，没有万金油
  - 一个良好适配应用的可伸缩架构，是围绕假设建立的：哪些操作常见，哪些操作罕见（负载参数）
  - 如果假设错误，那么为了伸缩而做的工程投入就白费了
  - 虽然架构是应用特定的，但是是由通用的积木块搭建成的，并以常见的模式排列，在本书中会讨论这些构建和模式

## 1.4 可维护性

- 软件的大部分开销不在开发阶段，而在持续的维护阶段，因此应当以这样一种方式设计软件：设计之初就尽量考虑如何减少维护的痛苦，为此需要关注软件系统的三个设计原则
  - 可操作性
  - 简单性
  - 可演化性



