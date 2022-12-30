[DRAFT]

# Distributed Transaction Aggregator (DTA)

Long story short: It's basically a distributed 'Task.WhenAll()', but instead of deal with threads on a single machine, we are dealing with completely decoupled units of work (workers).

When use this approach (benefits):
- To break a complex transaction having long processes into small units
- You'll be able to checkpoint your transaction, and perform recovery logic only at the faulty task
- When you can perform steps of a transaction in parallel 
- When you need to escalate this processment into distributed machines
- A single transaction is composed by different tecnologies (Ex: send order to C# service, register into bank in a c++ service, then send an confirmation e-mail via a phyton API)
- The total time is the sum of all the steps (aggregators) times. The more parallel tasks and normalized amount of aggregators, the best

When not to use it:
- You can't break your transaction into a amount of steps with parallel tasks
- Adds complexity to manage the state of transactions
- With this approach, your transaction becomes non-linear. Think about the Chain of Responsibility pattern. It's hard maintain

The main idea of this pattern:
![image](https://user-images.githubusercontent.com/8673745/210102138-99e31d8d-2a08-4d0f-986c-0413e0919fd1.png)

![image](https://user-images.githubusercontent.com/8673745/210102156-5e57986c-4345-45b0-9e54-2e460f8f7006.png)

The main goal is to break a transaction into a combination of micro-transactions based on small steps. Each step must have all the tasks that can run in parallel. The aggregator component for each step, should evaluate the state of all the tasks and decide to move next or call out an error state.

The main components of this pattern are:

### Transaction

A transaction is a combined and complex set of tasks in order to perform an action.

Think about an example of an e-commerce Order transaction. An order represents a purchase of a product.

### Aggregator

Acts like a semaphore, and must know the required tasks in order to call the next step. Each aggregator represents a single step, and is responsible for deal with the state related to all the tasks that belongs to its respective step.

### Task

A unit of work. This component is responsible for execute your process and notify the aggregator in order to proceed with the other steps of a transaction. A step (aggregator) can contain one or more tasks.

## A use case example:

![image](https://user-images.githubusercontent.com/8673745/210114851-b13a0b7e-1302-4953-ae18-ffa5dcd8b23c.png)



