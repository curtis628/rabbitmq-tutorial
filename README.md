# RabbitMQ CheatSheet and Tutorials

This is intended as a quick reference for using [RabbitMQ](https://www.rabbitmq.com/). It also includes code snippets pulled directly from their [tutorials](https://www.rabbitmq.com/tutorials/) page.

## rabbitmqctl commands

Here are some helpful `rabbitmqctl` commands:
- To list queues     : `sudo rabbitmqctl list_queues`
- To list exchangings: `sudo rabbitmqctl list_exchanges`
- To list bindings   : `sudo rabbitmqctl list_bindings`
- To list messages in queues (forgotten ack): `sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged`

## Acknowledgments
- Consumer acknowledges message successfully processed in callback method, using: ch.basic_ack(delivery_tag = method.delivery_tag)
  - If worked killed while processing the message (and no ack sent), nothing is lost.
  - Messages only re-delivered when client quits

## Durability
- Acknowledgments protect message if consumer/client dies; durability protects the message if RabbitMQ **server** stops
- When creating the queue, use 'durable=True'
- When publishing messages, need to mark messages as persistent by supplying this in the basic_publish:
```python
  properties=pika.BasicProperties(
     delivery_mode = 2, # make message persistent
  )
```

## Routing
- By default, rabbitmq dispatches messages in roundrobin.
- In order to NOT dispatch a message to a worker until it's processed and acknowledge the previous one, use `basic.qos`: `channel.basic_qos(prefetch_count=1)`

## Core Concepts
- **Producer**: A user application that sends messages (but never directly to a queue; always sends to an exchange)
- **Queue**: A buffer that stores messages
  - Create named queue: channel.queue_declare(queue='hello')
  - Create Temporary Queue (random name): channel.queue_declare()
  - Create Temporary that should be deleted once consumer app disconnects: `channel.queue_declare(exclusive=True)`
- **Consumer**: A user application that receives messages
- **Exchanges**: It receives messages from producers, must know exactly what to do with it (append to a specific queue/multiple queues, discard)
- **Bindings**: The relationship between an exchange and a queue: `channel.queue_bind(exchange='logs', queue=result.method.queue)`
- **Exchange Types**: Rules for how exchange processes message. See section below.

## Exchange Types
Here are the main exchange types, along with some examples of how they are used

### Nameless
`basic_publish(exchange='', routing_key='hello'...)` --> send message to queue named 'hello'

### Fanout
Broadcast all messages it receives to all the queues the exchange is binded to

### Direct
Messaged routed based on single criteria. Messages routed to the queue whose "binding key" exactly matches the "routing key" of the message. `channel.queue_bind(..., routing_key='black')`
- can have different bindings go to the same queue: both 'black' and 'green' go to one queue, 'orange' to another
- can have multiple bindings. 'black' can be routed to multiple queues
- consumer "subscribes" binding queue to all keys it's interested in: ie: 'info' 'warning' and 'error'

### Topic
Messages routed on multiple criteria, achieved by using list of words delimited by dots and `*` and `#` special characters
- A topic exchange can't have an arbitrary "routing_key" - it must be a list of words, delimited by dots. Words can be anything (up to 255 bytes), but usually specify features connected to the message
  - examples: `stock.usd.nyse`, `nyse.vmw`, `quick.orange.rabbit`
- The consumer's "binding key" is in same form. Similar to `direct`, a message sent with a particular routing key will be delivered to all the queues that are bound with a matching binding key.
- However there are two important special cases for binding keys:
  - `*` (star) can substitute for exactly one word.
  - `#` (hash) can substitute for zero or more words.
- Topic exchange: is powerful and can behave like other exchanges.
  - When a queue is bound with "#" (hash) binding key - it will receive all the messages, regardless of the routing key - like in `fanout` exchange.
  - When special characters "*" (star) and "#" (hash) aren't used in bindings, the `topic` exchange will behave just like a `direct` one.

