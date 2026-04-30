# Distributed Transaction Aggregator (DTA) - Workflow-like Saga orchestration

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

Distributed Transaction Aggregator is not a new design pattern, it's a specific implementation of the combination of Saga/Orchestrator and Fork-join distributed transaction patterns. The purpose of this design is to establish clear boundaries and responsibilities per component as part of your business transactions (Transactions), with a specific mix of fork-join/orchestration (Aggreggator component), turning the open aspects of these patterns into a **workflow process**.

Long story short: It's a stage-based orchestration, combining joint operations to establish a durable and traceable workflow progression.

The main goal is to break a transaction into a combination of micro-transactions composed by smaller steps. Each step must have all the tasks that can run in parallel. The aggregator component for each step, should evaluate the state of all the tasks and decide to move next or call out an error state.

The total time is the sum of all the steps (aggregators) times. The more parallel tasks and normalized amount of aggregators, the best:
![image](https://user-images.githubusercontent.com/8673745/213585812-01330065-13d7-48fd-996e-f4e3521877be.png)

When to use this approach:
+ To break a (async) complex transaction having long processes into small units
+ When you have a "pipeline" like process - **Ensure sequencing is respected**
+ You need to make sure all the steps are executed and need stronger visibility and reactions to the system state (compensation actions, aggregators statuses)
+ A single transaction is composed by different tecnologies (Ex: send order to C# service, register into bank in a c++ service, then send an confirmation e-mail via a phyton API)
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
- You can't break your transaction into steps with parallel tasks
- You can perform all of the steps of your transaction in batches - no need for transactional (preferable to use a job/workflow orchestration solution)

Drawbacks:
- Adds complexity to manage the state of transactions
- The progress through steps depends on state management/extra checks
- Adds considerable network traffic (communications) in order to orchestrate the distributed task execution. You'll need to manage the communication between all components
- With this approach, your transaction becomes non-linear. Think about the Chain of Responsibility pattern. It's hard maintain

### The pattern components

The main components of this pattern are:

#### Transaction

A (business) transaction is a combined and complex **set of tasks** in order to perform an **action**.

Think about an example of an e-commerce Order transaction. An order represents a purchase of a product.

It's composed by multiple tasks - smaller processes and transactions (save order in DB, notify logistics service, freeze stock) and aggregators (smaller state machines to decide wether to move forward, wait or compensation action).

Optional for implementation:
- A (business) transaction entity can be created in your system and the steps can be tracked by using notifications from the Aggregators components, to keep track of the whole business transaction at a higher-level without relying on tracking the aggregators. This is not related to your business transaction status (ORDER_PENDING, ORDER_DISPATCHED).

#### Aggregator (Step)

Acts like join node, and must know the required tasks in order to call the next step. Each **aggregator** represents a single **step**, and is responsible for deal with the state related to all the tasks that belongs to its respective step. Each **Aggregator/Step** must be composed by at least one **Task**.

The recovery logic must be placed on the Aggregator component as triggers for compensation actions (although local retries and the actual business logic for the compensation should remain in the domain systems). It must be completely responsible for evaluating the step state and decide to move forward or recover some falty state. This component has the same principle of a mediator object.

The main responsibilities of the Aggregator are:
- Ensure clear sequencing: In case a step is not finished yet (all the taks within step completed) and another task is executed from the next step, the transaction must enter an error state and trigger a compensation action
- Decide to move forward to the next step: Only move to the next step once all the tasks within the current step are completed (state machine behavior)
- System recovery: In case of errors or timeouts, the aggregator can have retry policies and trigger the compensation action once recovery was not possible
- Trigger next actions: As oppose to Coreography design, the Aggregator is responsible for triggering the steps, since it's a variation of the Orchestration approach - As a trade-off, the Aggregator shouldn't know how to integrate to business systems, process business logic or payloads, so it should be an agnostic trigger (event or direct integration, but without business related data), so your transaction continues. In practice, that means you will need the Tasks (business systems actions) to fetch data from the previous state to move forward with your transaction

Aggregators must-haves:
- Durable state
- Idempotency
- Timeout
- Compensation action (same as saga)
- Notifications between Aggregators to trigger a new stage

#### Task

A unit of work. This component is responsible for the **execute** and **notify** actions, so the aggregator can proceed with the other steps of a transaction. A step (aggregator) can contain one or more tasks.

As an alternative design, you can centralize the notification to the aggregators by relying on your state machine to map the business statuses to Aggregator steps (with the trade-offs of using a central notification component):

Example:
Task: Freeze stock
Actions: Set status quantity to "on-hold" - whatever it means in your design (database tables, decrease quantity, 2-phase commit)
Business status of your system: STOCK_ON_HOLD
When moving to this status, notify the Aggregator that the task "Freeze stock" has been completed

NOTE: The aggregator should never react to business-specific concepts, like STOCK_RELEASE, PAYMENT_COMPLETED, it should be task-oriented, which means: STARTED, COMPLETED or FAILED (compensation action)

### Putting all together

The relationship between components:
- A **Transaction** can be a conceptual composition of all the aggregators sequencing or an actual entity in the system composed by the status of all the aggregators
- An **Aggregator** coordinates the tasks (tracks stage status, trigger next stage or compensation action)
- The **Tasks** executes business logic
- The **Tasks** owns domain integration
- The **Aggregator** tracks task lifecycle only (COMPLETED, FAILED - STARTED not mandatory)

#### Sequence Diagram:
NOTE: Looking for a real-world implementation to map as part of this proposal and case. The following suggestion is still theoretical, so some components need optimization to polish this material.

![pattern-diagram-Sequence](https://user-images.githubusercontent.com/8673745/213585083-ceb95021-6c14-4cc9-9769-0de903f2dd6b.png)
TODO: evaluate if it's possible to remove OnBegin Event, for a lightweight processing. The less communications and events, the better

### Use cases and examples: 
Transaction: Order fullfilment

Aggregators:
- Order Placement Aggregator
 - [Aggregator status: Started]
 - Transactions/Tasks (your internal systems logic)
   - Freeze stock
   - Trigger payment
   - Process payment response (async or sync, doesn't matter)
   - Decrease or release stock
   - Move to status (PAYMENT_COMPLETED or PAYMENT_FAILED) - these are your business statuses, the Aggregator component doesn't compute these
 - [Aggregator status: Completed or Failed]
- Order Dispatcher Aggregator
 - [Aggregator status: Started]
 - Transactions/Tasks (your internal systems logic)
   - Start logistics delivery trip
   - Wait for confirmation/courier acceptance
   - Move to status (COURIER_STARTED or COURIER_FAILED) - business statuses
 - [Aggregator status: Completed or Failed]
- Order Tracking Aggregator
  - [Aggregator status: Started]
  - Transactions/Tasks (your internal systems logic)
   - Order trip events (reacting to events from courier)
   - Move to status (DELIVERY_COMPLETED or DELIVERY_FAILED) - business statuses
 - [Aggregator status: Completed or Failed]
- Complete order
  - [Aggregator status: Started]
  - Transactions/Tasks (your internal systems logic)
    - Trigger ratings flow
    - Notifies internal systems (billing, marketing, customer history data)
  - [Aggregator status: Completed or Failed]

### Related patterns and ideas:
- Aggregator Pattern
- Saga Pattern
- Chain of responsibility (process chain)
- Fork-join

### Contributors: 
|Name|Email|GitHub|
| -------- | -------- | -------- |
|Victor da Hora|vhhora@gmail.com|https://github.com/vhora|
