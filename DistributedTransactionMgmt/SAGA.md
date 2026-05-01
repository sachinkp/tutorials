# SAGA
## When to use?
You need to ensure data consistency in a distributed system without tight coupling.
You need to roll back or compensate if one of the operations in the sequence fails.

## Design Pattern
A saga is a sequence of local transactions(tasks) where each service performs its operation and initiates the next step through events or messages. If a step in the sequence fails, the saga performs **compensating transactions** to undo the completed steps

A Transaction -> Unit of Work -> subdeivided into a sequence of Tasks
 - Each task is triggered by a Command message and executed as a local transaction
 - One local transaction generates an event/command to trigger next Task
 - if any local transaction fails, triggers a compensating transaction in rverse order to rollback
 - An intermediate step is pivot transaction, once you pass this step transaction can be rolled back.  

 ## Approaches