[DRAFT]

# Distributed Transaction Aggregator (DTA)

Long story short: It's basically a distributed 'Task.WhenAll()', but instead of deal with threads on a single machine, we are dealing with completely decoupled units of work (workers).

When use this approach (benefits):
- To break long processes into small units
- When you can perform tasks in parallel but need to escalate this processment into distributed machines
- A single transaction is composed by different tecnologies (Ex: send order to C# service, register into bank in a c++ service, then send an confirmation e-mail via a phyton API)


The main idea of this pattern:
![image](https://user-images.githubusercontent.com/8673745/210102138-99e31d8d-2a08-4d0f-986c-0413e0919fd1.png)

![image](https://user-images.githubusercontent.com/8673745/210102156-5e57986c-4345-45b0-9e54-2e460f8f7006.png)

The main goal is to break a transaction into a combination of micro-transactions based on small steps. Each step must have all the tasks that can run in parallel. The aggregator component for each step, should evaluate the state of all the tasks and decide to move next or call out an error state.

The main components of this pattern are:

Aggregator

Acts like a semaphore, and must know the required tasks in order to call the next step. Each aggregator represents a single step, and is responsible for deal with the state related to all the tasks that belongs to its respective step.




