
# Evnet-Sourcing模式

Event-Sourcing模式使用仅附加存储来记录或描述域中数据所采取的动作，从而记录完整的一系列系列事件，而不是仅存储实体的当前状态。因为存储包含全部的事件，可以用来具体化域对象。Event-Sourcing模式可以简化复杂的域中的任务，避免了数据模型和业务领域的同步和引发的争用问题；增强性能，扩展性，以及响应；为事物数据提供一致性；保留全部的事件执行历史，可以跟踪和实现回滚之类的补偿操作。

## 问题

大多数的应用都会涉及到数据的处理，而通常的方法是由应用来保证数据的状态，当用户请求数据的时候，就会更新数据的状态。举个例子，在传统的CRUD模型中，典型的数据处理过程是，从数据仓库读取数据，做出一些修改，使用新的数据更新（通常通过事务来保护数据）当前数据的状态。CRUD的方式会有如下的一些限制：

* 事实上，CRUD系统直接针对数据存储执行更新操作，可能会受到数据仓库的处理开销而影响系统的性能和响应时间，和限制的可扩展性。
* 在许多并发用户的协作域中，数据更新有很大的可能发生冲突，因为更新操作都是发生在单个数据项上。
* 除非有一个额外的检查跟踪机制，它记录在一个单独的日志中的每个操作的详细信息，否则，数据项的操作历史会丢失。

> 想要更深入的了解CRUD架构的一些限制，可以参考微软的[CRUD, Only When You Can Afford It](http://msdn.microsoft.com/en-us/library/ms978509.aspx)一文。

## 解决方案

Event-Sourcing模式定义了一种处理由一系列事件来驱动的数据操作的方法，每一事件都记录在一个仅能追加的存储中。应用将生成一系列事件来描述每一个动作，每个事件对应存储数据发生的变更，然后将这些事件持久化。每个事件都表示一组数据的变化（比如在订单中增加数据项）。

事件可以持久化到事件存储仓库中。事件存储仓库作为对当前数据的状态的数据源或记录系统（数据源或信息的权威数据源）。事件存储通常会发布这些事件，以便可以通知消费者，并在消费者有需要的时候处理它们。例如，消费者可以初始化一些任务将事件中的操作应用于其他系统来执行，或执行完成该操作所需的任何其他相关操作。注意，生成事件的应用程序代码与订阅事件的系统是解耦的。

通常来说，事件存储所发布的事件是用来在应用程序对实体进行修改的时候，更新实体的Materialized视图，并与外部系统集成。例如，系统可以维护所有客户订单的Materialized视图，这些订单信息用于填充UI的部分。当应用程序添加新的订单，添加或删除项目的顺序，并添加航运信息，消费者将处理描述这些变化的事件，并更Materialized视图。

> 参考[Materialized-View模式](../Materialized-View/mvp.md)来了解更多的详细信息。

In addition, at any point in time it is possible for applications to read the history of events, and use it to materialize the current state of an entity by effectively “playing back” and consuming all the events related to that entity. This may occur on demand in order to materialize a domain object when handling a request, or through a scheduled task so that the state of the entity can be stored as a materialized view to support the presentation layer.

此外，在任何时候都可以让应用读取事件的历史，并使用它来实现实体的当前状态，通过有效地“回放”和消耗有关该实体的所有事件。这可能会发生在需求，以实现一个域对象处理请求时，或通过一个预定的任务，使实体的状态可以被存储为物化视图，以支持表示层。

Figure 1 shows a logical overview of the pattern, including some of the options for using the event stream such as creating a materialized view, integrating events with external applications and systems, and replaying events to create projections of the current state of specific entities.

图1展示了Event-Sourcing模式的逻辑概览，同时也包括了使用事件流来创建一个Materialized视图，与其他外部应用和系统集成，以及通过重复事件流来展示某些实体的当前状态。

![](leanote://file/getImage?fileId=58b911d5c9b47e3298000000)

图1. 关于Event-Sourcing模式的例子和概览

The Event Sourcing pattern provides many advantages, including the following:

Event-Sourcing模式提供了如下的优点，包括：


• Events are immutable and so can be stored using an append-only operation. The user interface, workflow, or process that initiated the action that produced the events can continue, and the tasks that handle the events can run in the background. This, combined with the fact that there is no contention during the execution of transactions, can vastly improve performance and scalability for applications, especially for the presentation level or user interface.

* 事件都是不可变对象，可以存储在一个仅支持追加的队列中。任何用户界面，workflow，或者程序触发了产生事件的行动，都可以以非阻塞继续，并且这些任务可以都在后台持续执行。这一点，结合事务执行过程中不会产生任何争用，可以极大地提高应用程序的性能和可扩展性，特别是对于用户界面的展示。

• Events are simple objects that describe some action that occurred, together with any associated data required to describe the action represented by the event. Events do not directly update a data store; they are simply recorded for handling at the appropriate time. These factors can simplify implementation and management.

* 事件是描述事件发生的简单对象，以及描述事件所代表的动作所需的相关数据。事件不会直接更新数据存储区，它们只是在适当的时候进行记录处理。这些因素可以简化实施和管理。
* 
• Events typically have meaning for a domain expert, whereas the complexity of the object-relational impedance mismatch might mean that a database table may not be clearly understood by the domain expert. Tables are artificial constructs that represent the current state of the system, not the events that occurred.

* 事件通常在某些领域都有各自的特殊含义，而对应的对象可能只是和数据库的表对应的，无法表示其在领域中的意义。表只能表示系统的当前状态，而表现不了发生的事件。
* 
• Event sourcing can help to prevent concurrent updates from causing conflicts because it avoids the requirement to directly update objects in the data store. However, the domain model must still be designed to protect itself from requests that might result in an inconsistent state.

* Event-Sourcing可以帮助防止并发更新引起冲突，因为它避免了直接更新数据存储区中对象的要求。当然，域模型仍然必须设计来保护其自身的一致性。
• The append-only storage of events provides an audit trail that can be used to monitor actions taken against a data store, regenerate the current state as materialized views or projections by replaying the events at any time, and assist in testing and debugging the system. In addition, the requirement to use compensating events to cancel changes provides a history of changes that were reversed, which would not be the case if the model simply stored the current state. The list of events can also be used to analyze application performance and detect user behavior trends, or to obtain other useful business information.

* 仅追加存储事件提供Trace的线索，可以用来监测对数据存储区所采取的行动，使当前状态的物化视图或预测事件的回放，在任何时间，并协助测试和系统调试。此外，使用补偿事件取消更改的要求提供了一个历史的变化被逆转，如果模型只存储当前状态，这将是不可能实现的。事件列表也可用于分析应用程序性能和检测用户行为趋势，或获得其他有用的业务信息。

• The decoupling of the events from any tasks that perform operations in response to each event raised by the event store provides flexibility and extensibility. For example, the tasks that handle events raised by the event store are aware only of the nature of the event and the data it contains. The way that the task is executed is decoupled from the operation that triggered the event. In addition, multiple tasks can handle each event. This may enable easy integration with other services and systems that need only listen for new events raised by the event store. However, the event sourcing events tend to be very low level, and it may be necessary to generate specific integration events instead.

* 对事件存储所引发的每个事件执行任何操作的任务的解耦提供了灵活性和可扩展性。例如，处理事件存储所引发的事件只知道事件的性质和包含的数据。执行任务的方式与触发事件的操作分离。此外，多个任务可以处理每个事件。这可以与其他服务和系统进行简单的集成，只需要侦听由事件存储引发的新事件。然而，溯源事件往往是非常低的level，它可能是必要的，而不是产生特定的集成事件。

> Event sourcing is commonly combined with the CQRS pattern by performing the data management tasks in response to the events, and by materializing views from the stored events.
> Event-Sourcing通常都是配合[CQRS模式]来执行数据管理任务


## 实现Event-Sourcing模式的问题和顾虑

Consider the following points when deciding how to implement this pattern:

当在考虑实现Event-Sourcing模式的时候，需要考虑如下一些问题：

• The system will only be eventually consistent when creating materialized views or generating projections of data by replaying events. There is some delay between an application adding events to the event store as the result of handling a request, the events being published, and consumers of the events handling them. During this period, new events that describe further changes to entities may have arrived at the event store.

* 整个系统在创建物化视图或通过回放事件数据生成预测的时候都是不一致的，只是满足最终一致性的。应用程序将事件添加到事件存储区的过程与处理请求的结果、正在发布的事件以及处理它们的事件的消费者之间存在一些延迟。在此期间，描述实体的进一步更改的新事件可能已到达事件存储区。
> 可以参考[Data Consistency Primer]来了解最终一致性方面的信息

• The event store is the immutable source of information, and so the event data should never be updated. The only way to update an entity in order to undo a change is to add a compensating event to the event store, much as you would use a negative transaction in accounting. If the format (rather than the data) of the persisted events needs to change, perhaps during a migration, it can be difficult to combine existing events in the store with the new version. It may be necessary to iterate through all the events making changes so that they are compliant with the new format, or add new events that use the new format. Consider using a version stamp on each version of the event schema in order to maintain both the old and the new event formats.

* 事件存储是不可变的信息源，因此事件数据不应该被更新。更新实体以撤消更改的唯一方法是向事件存储添加补偿事件，就像在会计中使用负事务一样。如果持久化事件的格式（而不是数据）需要更改，可能在迁移过程中，将存储中的现有事件与新版本相结合很难。它可能需要遍历所有事件的变化使它们符合新的格式，或加用新的格式，新的事件。考虑在事件模式的每个版本上使用版本标记，以维护旧的和新的事件格式。

• Multi-threaded applications and multiple instances of applications may be storing events in the event store. The consistency of events in the event store is vital, as is the order of events that affect a specific entity (the order in which changes to an entity occur affects its current state). Adding a timestamp to every event is one option that can help to avoid issues. Another common practice is to annotate each event that results from a request with an incremental identifier. If two actions attempt to add events for the same entity at the same time, the event store can reject an event that matches an existing entity identifier and event identifier.

* 多线程应用程序和多个应用程序实例可以在事件存储区中存储事件。事件存储中事件的一致性是至关重要的，因为影响特定实体的事件顺序（实体发生变化的顺序影响其当前状态）。添加一个时间戳到每一个事件都是一个选项，可以帮助避免问题。另一个常见的做法是注释每一个事件的结果与增量标识符的请求。如果两个操作试图同时为同一实体添加事件，则事件存储可以拒绝与现有实体标识符和事件标识符相匹配的事件。


• There is no standard approach, or ready-built mechanisms such as SQL queries, for reading the events to obtain information. The only data that can be extracted is a stream of events using an event identifier as the criteria. The event ID typically maps to individual entities. The current state of an entity can be determined only by replaying all of the events that relate to it against the original state of that entity.

* 在阅读获取信息的事件上，是没有标准的方法或者一些诸如SQL查询的內建机制的。可以提取的唯一数据是使用事件标识符作为标准的事件流。事件ID通常映射到单个实体。一个实体的当前状态，只能通过回放所有涉及对该单位的原始状态的事件。

• The length of each event stream can have consequences on managing and updating the system. If the streams are large, consider creating snapshots at specific intervals such as a specified number of events. The current state of the entity can be obtained from the snapshot and by replaying any events that occurred after that point in time.

* 每个事件流的长度可以有管理和更新系统的后果。如果流较大，可以考虑每隔一定的时间间隔创建快照，例如指定的事件数。实体的当前状态可以从快照和重放，那个时间点之后发生的任何事件中获得。

> For more information about creating snapshots of data, see Snapshot on Martin Fowler’s Enterprise Application Architecture website and Master-Subordinate Snapshot Replication on MSDN.

> 想了解针对数据建立快照方面更多的信息，可以参考Martin Fowler的*Enterprise Application Architecture website*以及[Master-Subordinate Snapshot Replication on MSDN](http://msdn.microsoft.com/en-us/library/ff650012.aspx).

• Even though event sourcing minimizes the chance of conflicting updates to the data, the application must still be able to deal with inconsistencies that may arise through eventual consistency and the lack of transactions. For example, an event that indicates a reduction in stock inventory might arrive in the data store while an order for that item is being placed, resulting in a requirement to reconcile the two operations; probably by advising the customer or creating a back order.

* 即使事件获取将数据冲突更新的可能性降到最低，应用程序仍然必须能够处理可能通过最终一致性和事务缺乏而出现的不一致。例如，表示库存减少的事件可能到达数据存储区，而该项目的订单正在被放置，从而导致协调两个操作的要求；可能是通过通知客户或创建回订单。
* 
• Event publication may be “at least once,” and so consumers of the events must be idempotent. They must not reapply the update described in an event if the event is handled more than once. For example, if multiple instances of a consumer maintain an aggregate of a property of some entity, such as the total number of orders placed, only one must succeed in incrementing the aggregate when an “order placed” event occurs. While this is not an intrinsic characteristic of event sourcing, it is the usual implementation decision.

* 事件发布可能“至少一次”等活动的消费者必须是幂等的。他们必须重新更新了一个事件如果处理不止一次。例如，如果一个用户的多个实例维护某些实体的属性的集合，如订单总数，只有一个必须成功增加总当“订单”事件的发生。虽然这不是事件采购的固有特性，但它是通常的执行决策。

## 何时使用Event-Sourcing模式

Event-Sourcing模式很适合以下场景：

• When you want to capture “intent,” “purpose,” or “reason” in the data. For example, changes to a customer entity may be captured as a series of specific event types such as Moved home, Closed account, or Deceased.

* 当你想捕获数据中的“意图”、“目的”或“原因”时。例如，对客户实体的更改可以被捕获为一系列特定的事件类型，如移动的家庭、关闭的帐户或已死亡的事件。

• When it is vital to minimize or completely avoid the occurrence of conflicting updates to data.

* 当重要的是尽量减少或完全避免冲突的更新数据的发生。

• When you want to record events that occur, and be able to replay them to restore the state of a system; use them to roll back changes to a system; or simply as a history and audit log. For example, when a task involves multiple steps you may need to execute actions to revert updates and then replay some steps to bring the data back into a consistent state.

* 当开发者希望记录发生的事件，并能够重放它们以恢复系统的状态；使用它们回滚更改到系统；或简单地作为历史和审计日志。例如，当任务涉及多个步骤时，您可能需要执行返回更新的操作，然后重播一些步骤，使数据恢复到一致状态。

• When using events is a natural feature of the operation of the application, and requires little additional development or implementation effort.

* 当使用事件是应用程序运行的一个自然特性，并且需要很少的额外开发或实现工作。

• When you need to decouple the process of inputting or updating data from the tasks required to apply these actions. This may be to improve UI performance, or to distribute events to other listeners such as other applications or services that must take some action when the events occur. An example would be integrating a payroll system with an expenses submission website so that events raised by the event store in response to data updates made in the expenses submission website are consumed by both the website and the payroll system.

* 当您需要将输入或更新数据的过程与应用这些操作所需的任务脱钩时。这可能是为了改善UI性能，或者将事件分发给其他侦听器，如其他应用程序或服务，当事件发生时必须使用某些操作。一个例子是将一个工资单系统与一个费用提交网站相结合，以便由事件商店提出的响应于费用提交网站中的数据更新所引发的事件被网站和工资系统所消耗。

• When you want flexibility to be able to change the format of materialized models and entity data if requirements change, or—when used in conjunction with CQRS—you need to adapt a read model or the views that expose the data.

* 当你想能够如果要求改变物化模型和实体数据的格式，或使用与你需要适应一个阅读模式或观点，揭露数据cqrs连词。

• When used in conjunction with CQRS, and eventual consistency is acceptable while a read model is updated or, alternatively, the performance impact incurred in rehydrating entities and data from an event stream is acceptable.

* 当配合cqrs模式共同使用时，和最终一致性是可以接受的，一个读模式或者是更新的，另外，在补水的实体和数据从一个事件流所产生的性能的影响是可以接受的。

This pattern might not be suitable in the following situations:

Event-Sourcing模式在如下的情况下不太适用：

• Small or simple domains, systems that have little or no business logic, or non-domain systems that naturally work well with traditional CRUD data management mechanisms.
* 领域模型很小，或者较为简单，系统只有很少或没有业务逻辑，或非域系统，自然的工作以及与传统的CRUD数据管理机制。
• Systems where consistency and real-time updates to the views of the data are required.

* 系统对一致性和实时更新的数据视图要求较高的时候，不适合使用Event-Sourcing模式。
• Systems where audit trails, history, and capabilities to roll back and replay actions are not required.

* 系统不需要审计跟踪、历史和回滚和重放操作的功能的时候，因为复杂性的原因，不适合使用Event-Sourcing模式。

• Systems where there is only a very low occurrence of conflicting updates to the underlying data. For example, systems that predominantly add data rather than updating it.

* 只有底层数据相互冲突的更新发生的系统。例如，主要添加数据而不是更新数据的系统。

## 使用举例

A conference management system needs to track the number of completed bookings for a conference so that it can check whether there are seats still available when a potential attendee tries to make a new booking. The system could store the total number of bookings for a conference in at least two ways:

会议管理系统需要跟踪完成预订的会议，它可以检查是否还有空座位时，一个潜在的与会者试图使一个新的预订数量。该系统可以存储至少一个会议的预订总数至少两种方式：

• The system could store the information about the total number of bookings as a separate entity in a database that holds booking information. As bookings are made or cancelled, the system could increment or decrement this number as appropriate. This approach is simple in theory, but can cause scalability issues if a large number of attendees are attempting to book seats during a short period of time. For example, in the last day or so prior to the booking period closing.

* 该系统可以存储的预订总额的信息作为一个单独的实体在数据库中持有预订信息。由于预订或取消，系统可以增加或减少这个数字适当。这种方法在理论上是简单的，但可能会导致可扩展性问题，如果大量的与会者正在试图预订的座位在很短的时间内。例如，在最后一天左右预订期结束之前。
* 该系统可以存储信息的预订和取消的事件中，商店举行活动。它可以通过重播这些事件的座位数计算。这种方法可以更具可扩展性由于事件的不变性。系统只需要能够从事件存储区读取数据，或将数据附加到事件存储区。关于预订和取消事件的信息没有被修改过的。

• The system could store information about bookings and cancellations as events held in an event store. It could then calculate the number of seats available by replaying these events. This approach can be more scalable due to the immutability of events. The system only needs to be able to read data from the event store, or to append data to the event store. Event information about bookings and cancellations is never modified.


Figure 2 shows how the seat reservation sub-system of the conference management system might be implemented by using event sourcing.
图2显示了如何使用事件源来实现会议管理系统的座席预订子系统。

![](leanote://file/getImage?fileId=58bd4feec6f6b95818000000)


Figure 2.Using event sourcing to capture information about seat reservations in a conference management system
图2在会议管理系统中使用事件资源获取座位预订信息

保留两个座位的动作顺序如下：
The sequence of actions for reserving two seats is as follows:
1. The user interface issues a command to reserve seats for two attendees. The command is handled by a separate command handler (a piece of logic that is decoupled from the user interface and is responsible for handling requests posted as commands).
2. 用户界面发出一个命令，为两位与会者预留座位。该命令由一个单独的命令处理程序处理（一个与用户界面脱钩的逻辑，负责处理作为命令发送的请求）。
2. An aggregate containing information about all reservations for the conference is constructed by querying the events that describe bookings and cancellations. This aggregate is called SeatAvailability, and is contained within a domain model that exposes methods for querying and modifying the data in the aggregate.
3. 集合包含信息的会议预订是通过查询，预订和取消的事件描述了。这总叫seatavailability，并包含一个暴露的方法查询和修改数据的集合域模型内。
> 需要考虑的一些优化是使用快照（因此您不需要查询和重放事件的完整列表以获取聚合的当前状态），并在内存中维护聚合的缓存副本。
> Some optimizations to consider are using snapshots (so that you don’t need to query and replay the full list of events to obtain the current state of the aggregate), and maintaining a cached copy of the aggregate in memory.
3. The command handler invokes a method exposed by the domain model to make the reservations.
4. 命令处理程序调用由域模型公开的方法以使保留。
4. The SeatAvailability aggregate records an event containing the number of seats that were reserved. The next time the aggregate applies events, all the reservations will be used to compute how many seats remain.
5. 总的seatavailability事件的记录包含座位保留数。下一次聚合应用事件时，所有的预订将被用来计算有多少座位仍然存在。
5. The system appends the new event to the list of events in the event store.
6. 该系统增加新的事件，在事件存储事件列表。
If a user wishes to cancel a seat, the system follows a similar process except that the command handler issues a command that generates a seat cancellation event and appends it to the event store
As well as providing more scope for scalability, using an event store also provides a complete history, or audit trail, of the bookings and cancellations for a conference. The events recorded in the event store are the definitive and only source of truth. There is no need to persist aggregates in any other way because the system can easily replay the events and restore the state to any point in time.

如果用户希望取消座位，该系统遵循类似的过程除了命令处理程序问题产生取消座位的事件并将其添加到事件存储命令以及可扩展性提供了更多的范围，利用事件的商店还提供了一个完整的历史，或审计线索，为会议的预订和取消。事件商店中记录的事件是真实的唯一来源。没有必要以任何其他方式坚持聚集，因为系统可以很容易地重放事件并将状态恢复到任何时间点。
> You can find more information about this example in the chapter Introducing Event Sourcing in the patterns & practices guide CQRS Journey on MSDN.

> 开发者可以在[Introducing Event Sourcing](http://msdn.microsoft.com/en-us/library/jj591559.aspx)中更详细的了解这个例子。


## 相关的其他模式

当考虑实现Event-Sourcing模式的时候，也可以参考如下相关模式：

• Command and Query Responsibility Segregation (CQRS) Pattern. The write store that provides the immutable source of information for a CQRS implementation is often based on an implementation of the Event Sourcing pattern. The Command and Query Responsibility Segregation pattern describes how to segregate the operations that read data in an application from the operations that update data by using separate interfaces.
* **[CQRS模式]()**.CQRS实现中提供不可变的信息源的写存储通常就是基于Event-Sourcing模式的一种实现。CQRS模式描述了如何将操作，读取数据，应用程序的操作，通过使用单独的更新数据的接口来分离职能。
• Materialized View Pattern. The data store used in a system based on event sourcing is typically not well suited to efficient querying. Instead, a common approach is to generate pre-populated views of the data at regular intervals, or when the data changes. The Materialized View pattern shows how this can be achieved.

* **[Materialized-View模式]()**.在Event-Sourcing模式中所使用的数据仓库，通常来说是不利于查询的。通常提高查询效率的方式，就是每隔一定的时间，根据数据仓库生成预填充的视图来提高查询效率。Materialized-View模式描述了改功能是如何实现的。

• Compensating Transaction Pattern. The existing data in an event sourcing store is not updated; instead new entries are added that transition the state of entities to the new values. To reverse a change, compensating entries are used because it is not possible to simply reverse the previous change. The Compensating Transaction pattern describes how to undo the work that was performed by a previous operation.

* **[Compensating-Transaction模式]()**. 在实现Event-Sourcing模式中的数据仓库中的数据是不会执行更新操作的，相对来说，会通过增加额外的事件来执行回滚等操作，恢复实体的状态。Compensating-Transaction模式描述如何撤消由前一个操作执行的工作。

• Data Consistency Primer. When using event sourcing with a separate read store or materialized views, the read data will not be immediately consistent; instead it will be only eventually consistent. The Data Consistency Primer summarizes the issues surrounding maintaining consistency over distributed data.

* **[Data Consistency Primer]()**. 当使用Evnet-Sourcing时，读数据仓库或者Materialized视图是不会保证实时一致的。相对来说，他们会保证最终一致性。Data Consistency Primer中讲述了分布式数据保证一致性的诸多问题。
• Data Partitioning Guidance. Data is often partitioned when using event sourcing in order to improve scalability, reduce contention, and optimize performance. The Data Partitioning Guidance describes how to divide data into discrete partitions, and the issues that can arise.

* **[Data Partitioning Guidance]()**. 在使用Evnet-Sourcing的时候，为了提升扩展性，优化性能，减少争用，会考虑对事件存储进行分区。Data Partitioning Guidance中描述了如何将数据进行分区，以及分区中可能产生的问题等。
