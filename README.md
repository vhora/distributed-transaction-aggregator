# Distributed Transaction Aggregator (DTA)

![Badge](https://img.shields.io/badge/status-work%20in%20progress-yellow)
![Badge](https://img.shields.io/badge/topics-architecture-blue)
![Badge](https://img.shields.io/badge/topics-distributed%20services-blue)
![Badge](https://img.shields.io/badge/topics-cloud-blue)

### Table of content
   * [Why do I need DTA?](#why-do-i-need-dta)
   * [The pattern components](#the-pattern-components)
   * [Putting all together](#putting-all-together)
   * [Use cases and examples](#use-cases-and-examples)
   * [Related patterns and ideas](#related-patterns-and-ideas)
   * [Contributors](#contributors)

### Why do I need DTA?

Long story short: It's basically a distributed 'Task.WhenAll()', but instead of dealing with threads on a single machine, we are dealing with completely decoupled and distributed units of work.

The main goal is to break a transaction into a combination of micro-transactions based on small steps. Each step must have all the tasks that can run in parallel. The aggregator component for each step, should evaluate the state of all the tasks and decide to move next or call out an error state.

The total time is the sum of all the steps (aggregators) times. The more parallel tasks and normalized amount of aggregators, the best:
![image](https://user-images.githubusercontent.com/8673745/213585812-01330065-13d7-48fd-996e-f4e3521877be.png)

When to use this approach:
+ To break a (async) complex transaction having long processes into small units
+ A single transaction is composed by different tecnologies (Ex: send order to C# service, register into bank in a c++ service, then send an confirmation e-mail via a phyton API)
+ When you have a "pipeline" like process 
+ When you have bottlenecks in your transaction, and parallel processing could improve your transaction performance
+ When you can perform steps of a transaction in parallel
+ When you need to escalate processment into distributed machines

Benefits:
+ In case of error, you'll not loose the entire step. It could happen in a multi-thread processment
![image](https://user-images.githubusercontent.com/8673745/213585572-9eed26f2-5d8f-42f7-a2a5-d5689dd8d41f.png)
+ You'll be able to "checkpoint" your transaction, and perform recovery logic only at the faulty task
![image](https://user-images.githubusercontent.com/8673745/213585776-5b4d11ab-084e-4ba2-9ff3-aaa0967bde2f.png)
+ You'll be able to identify the status of the transaction. In case of error, the transaction will be stalled at one specific aggregator/step
+ Small and distibuted units of work (Tasks)

When **not** to use it:
- Your transaction needs to be sync
- You can't break your transaction into a amount of steps with parallel tasks
- You can perform all of the steps of your transaction in batches - no need for transactional (preferable to use a job/workflow orchestration solution)

Drawbacks:
- Adds complexity to manage the state of transactions
- Adds considerable network traffic (communications) in order to orchestrate the distributed task execution. You'll need to manage the communication between all components
- With this approach, your transaction becomes non-linear. Think about the Chain of Responsibility pattern. It's hard maintain

### The pattern components

The main components of this pattern are:

#### Transaction

A transaction is a combined and complex **set of tasks** in order to perform an **action**.

Think about an example of an e-commerce Order transaction. An order represents a purchase of a product.

#### Aggregator (Step)

Acts like a semaphore, and must know the required tasks in order to call the next step. Each **aggregator** represents a single **step**, and is responsible for deal with the state related to all the tasks that belongs to its respective step. Each **Aggregator/Step** must be composed by at least one **Task**.

All of the recovery logic must be placed on the Aggregator component. It must be completely responsible for evaluate the step state and decide to move forward or recover some falty state. This component has the same principle of a mediator object.

#### Task

A unit of work. This component is responsible for **execute** your process and **notify** the aggregator in order to proceed with the other steps of a transaction. A step (aggregator) can contain one or more tasks.

### Putting all together
#### Sequence Diagram:

![pattern-diagram-Sequence](https://user-images.githubusercontent.com/8673745/213585083-ceb95021-6c14-4cc9-9769-0de903f2dd6b.png)
TODO: evaluate if it's possible to remove OnBegin Event, for a lightweight processing. The less communications and events, the better

### Use cases and examples:
TODO

### Related patterns and ideas:
- Aggregator Pattern
- Saga Pattern
- Chain of responsibility (process chain)
- Semaphore

### Contributors: 
|Name|Email|GitHub|
| -------- | -------- | -------- |
|Victor da Hora|vhhora@gmail.com|https://github.com/vhora|
