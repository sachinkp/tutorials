# Messaging Systems
- Producer and consumer don't have to be online or readily available at the same time
- Load Leveling : brings predictability when traffic spikes and dont overconsume resources.
 resources.
 – Lord, balancing each competing consumer acquires lock on message

 – fan out message processing using pub sub model


## SQS

In SQS workload, API produces events, workers pull and consume these events.
SQS is fundamentally a pull base system. consumers poll and polling frequency determines throughput

SQS is great when you don't poll aggressively

For example.
,
60 workfor example,
60 workers Polling every seconds can consume 
80 * 86400 seconds per day = 2.6 million polls per worker per month


Producer → SQS Queue → Consumer
-- Message waits safely in the queue
--- Messages stored redundantly
-- Consumer pulls, visibility timeout begins, and consume must process the message before the visibility timeout
-- Message is deleted after success

Types
-- Standard SQS- Order not guaranteed, dup possible, at least once delivery
-- FIFI - Guaranteed Order, Dupe detection, exact once processing

## SNS(Simple Notification Service) PUB SUB
- Publisher → SNS Topic → Subscribers
    - Publisher sends message to a Topic
    - SNS delivers message to all subscribers
    - Push-based/SNS pushes messages
    - Stateless/No message storage


## Rabbit MQ 
rabbit MQ pushes messages to consumers 
in rabbit MQ consumers are notified when new message is arrived.
Advantage.
On demand message processing reduces messaging processing cost

Use Rabbit MQ when you have uncertain burst in load and need dynamic scaling


## kafka
 
 High throughput.
 Multiple independent consumers 
 Replayability 
 Long retention window

 Good for fire and forget
 Need high throughput and retention matters
 Retain message history, easy to replay history


## Event Grid vs Event Hub vs Service Bus
- Event Grid is for lightweight eventing, 
- Event Hubs for large-scale telemetry, 
- Service Bus for critical business messages