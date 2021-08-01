# Notes
RabbitMQ is a message broker

RabbitMQ, and messaging in general, uses some jargon

- Producing means nothing more than sending. 
A program that sends messages is a producer 

- A queue is the name for a post box which lives inside RabbitMQ. 
Although messages flow through RabbitMQ and your applications, they can only be stored inside a queue. 
A queue is only bound by the host's memory & disk limits, it's essentially a large message buffer. 
Many producers can send messages that go to one queue, and many consumers can try to receive data from one queue. 

- Consuming has a similar meaning to receiving. A consumer is a program that mostly waits to receive messages

## Tech
This tutorial uses AMQP 0-9-1, which is an open, general-purpose protocol for messaging.
```
go get github.com/streadway/amqp
```
Also, do not remember to set up docker and rabbitmq from it
```
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```

## Work Queues
The main idea behind Work Queues (aka: Task Queues)
is to avoid doing a resource-intensive task 
immediately and having to wait for it to complete.
Instead we schedule the task to be done later. 
We encapsulate a task as a message and send it to a queue. A worker process running in the background will pop the tasks and eventually execute the job.
When you run many workers the tasks will be shared between them.

## Round-robin dispatching(Циклическая диспетчеризация)
One of the advantages of using a Task Queue is the ability to easily parallelise work. If we are building up a backlog of work,
we can just add more workers and that way, scale easily.

By default, RabbitMQ will send each message to the next consumer, in sequence. On average every consumer will get the same number of messages.
This way of distributing messages is called round-robin.

## Message acknowledgment

we don't want to lose any tasks.
If a worker dies, we'd like the task to be delivered to another worker.

In order to make sure a message is never lost, RabbitMQ supports message acknowledgments. An ack(nowledgement) is sent back by the consumer to tell RabbitMQ that a particular message has been received, 
processed and that RabbitMQ is free to delete it.

If a consumer dies (its channel is closed, connection is closed, or TCP connection is lost) without sending an ack,
RabbitMQ will understand that a message wasn't processed fully and will re-queue it. If there are other consumers online at the same time, 
it will then quickly redeliver it to another consumer. That way you can be sure that no message is lost, 
even if the workers occasionally die.

## Message durability

When RabbitMQ quits or crashes it will forget the queues and messages unless you tell it not to. Two things are required to make sure that messages aren't lost: 
we need to mark both the queue and messages as durable.

This durable option change needs to be applied to both the producer and consumer code

#### Note on message persistence

Marking messages as persistent doesn't fully guarantee that a message won't be lost. 
Although it tells RabbitMQ to save the message to disk, there is still a short time window when RabbitMQ has accepted 
a message and hasn't saved it yet. Also, RabbitMQ doesn't do fsync(2) for every message -- it may be just saved to cache and not really written to the disk. 
The persistence guarantees aren't strong, but it's more than enough for our simple task queue. If you need a stronger guarantee then you can use publisher confirms.

## Fair dispatch

In order to defeat that we can set the prefetch count with the value of 1. 
This tells RabbitMQ not to give more than one message to a worker at a time.
 Or, in other words, don't dispatch a new message to a worker until it has processed and acknowledged the previous one. 
 Instead, it will dispatch it to the next worker that is not still busy.
 
####  Note about queue size

If all the workers are busy, your queue can fill up. You will want to keep an eye on that, and maybe add more workers, or have some other strategy.