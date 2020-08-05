---
title: Event Sourcing
date: 2019-07-25 18:46:23
tags:
hidden: true
---
It is also possible to recreate the status of the order as an aggregate of all of the events that occurred on the stream.

we use event sourcing as our main log of truth for historical transactions.

All events that were written to the log are immutable, and changes to a stream are handled by appending new events to the log that contains an updated state for the stream.

Events are written to streams

We think of streams as a specific domain type (e.g. Customer Order), and the events that are applied to the stream are the Domain Events.

The stream name has the syntax of StreamName-{Id}

Our microservices can subscribe to these projection streams and react to the events that are appended to the stream. 

Idempotent Writes event?

Teams at Jet typically use a message queue, like Kafka, to communicate between microservices. Kafka guarantees at-least once delivery of messages, which means that a microservice can receive the same message one or more times. 

we write idempotent microservices


-----

we distribute requests acorss web servers. 

figure out

send request to least busy servers

round-robin is simple
randomness



violate privacy
finite size
promote to master
---

session
* share storage
* sticky session 