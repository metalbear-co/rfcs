- Feature Name: `rabbitmq`
- Start Date: 2026-01-25
- Last Updated: 2026-01-25
- RFC PR: [metalbear-co/rfcs#0002](https://github.com/metalbear-co/rfcs/pull/7)
- RFC reference:
  - [metalbear-co/rfcs#0002](https://github.com/metalbear-co/rfcs/pull/7)

## Summary
[summary]: #summary

Support RabbitMQ for operator's queue splitting feature.

## Motivation
[motivation]: #motivation

Queue splitting is a pretty neat feature, it would be nice to do so with RabbitMQ.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### RabbitMQ names and concepts.

- `vhost` - RabbitMQ natively is a multi-tenant system and thus supports internal segmentation where connections, exchanges, queues, bindings, user permissions, policies and some other things to **virtual hosts**. (similar to Kubernetes namespaces)
- `queue` - A buffer that will store messages until a consumer is ready to process them. there are 3 types of queues supported in RabbitMQ and those are `classic`, `quorum` and `stream`. Queue mainly has a type, name and key-value store of arguments and can be set to either durable or not. (durable makes so the queue survives broker restart) Arguments define certain behaviour the broker will ensure for example auto expiration of messages, overflows-behaviour, limits on message size and etc. (queue type is also defined as an argument named `x-queue-type`)
  * `classic` - The quintessential FIFO queue that will remove messages from the queue once one of the consumers acknowledges the message or is not-acknowledged and then either enqueued again to back of the queue or sent to another exchange via `x-dead-letter-exchange` argument.
  * `quorum` - Similar to classic queues but instead of using replication across different nodes the messages are shared between them based on the Raft consensus algorithm. This queue is always durable and is encouraged by rabbit to be considered the default choice when needing a replicated, highly available queue.
  * `stream` - Unlike classic and quorum queues a stream does not remove the message from the queue once it is acknowledged and is deleted after some period of time instead. This mean the streams are traversable and behaves more like a Kafka topic than a queue.
- `exchange` - The publishing mechanism of RabbitMQ, when you publish a message you must specify an exchange and a routing-key and with this routing-key and optional filters the exchange chooses the queue where the message should go, this can be one queue or multiple ones. Exchanges also have multiple types `direct`, `fanout`, `headers`, `topic` and `x-local-random` but more can be added via plugins.
  * `fanout` - ignores routing-key and copies messages to all bounded queues.
  * `direct` - directly matches routing-key and will copy messages for multiple matches.
  * `headers` - uses the message headers instead of routing-key and copies messages to all matching queues as specified in bindings.
  * `topic` - works similar to direct exchange but works with wildcards when specified in bindings.

### Rabbit Usage

As far as I've seen the most basic way of usage in rabbit is having a queue as a temporary buffer for messages with a async workflow where you send a request that will eventually be processed and acknowledged. The idea is that you have some task you need to be done but it's not critical that it's performed immediately or a bit later and without indicating any status to the original requester (fire and forget style request).

```
┌─────────────┐       ┌─────────────────────┐      ┌───────────────────┐         ┌──────────────────┐ 
│             │       │                     │      │                   │         │                  ├┐
│  Publisher  ├──────►│   Input Exchange    ├─────►│   Target Queue    ├────────►│  Target Pods     ││
│             │       │                     │      │                   │         │                  ││
└─────────────┘       └─────────────────────┘      └───────────────────┘         └┬─────────────────┘│
                                                                                  └──────────────────┘
```

In this graphic the idea is that some Publisher is aware of the existence of the Input Exchange only where as the Target knows of the Input Exchange and the Target Queue, as rabbit by default works on "get or create" method where when one of the Target Pods starts it may create the Target Queue and then also bind it to the Input Exchange (or maybe also create it if it's not done with Publisher beforehand)

There is more to it some patters are more like RPC requests where the Publisher embeds the reply_to argument (it can be done in many ways just the reply_to is the rabbit provided way) and then is expected by the target pod to send the reply to provided queue.

```
┌─────────────┐        ┌──────────────────┐        ┌───────────────────┐         ┌──────────────────┐     
│             │        │                  │        │                   │         │                  ├┐    
│  Publisher  ├───────►│  Input Exchange  ├───────►│   Target Queue    ├────────►│  Target Pods     ││    
│             │        │                  │        │                   │         │                  ││    
└──────▲──────┘        └──────────────────┘        └───────────────────┘         └┬─────────────────┘│    
       │                                                                          └───────┬──────────┘    
       │                                                                                  │ From: reply_to
       │                ┌───────────────┐                                                 │    argument   
       │                │               │                                                 │               
       └────────────────┤  Reply Queue  │◄────────────────────────────────────────────────┘               
                        │               │                                                                 
                        └───────────────┘                                                                 
```

The main benefit but also the problem for us is that the architecture can be very complex as well, exchanges can bind to other exchanges, queues can be "ephemeral" where they auto delete once they are no longer used and the architecture is very built for customisation with plugins and all. But also as mentioned before most probably use some sort of abstraction that hides that rabbitmq is even used so they use a very generic approach with one queue and probably an exchange to go with it.

### What we can do

With the first scenario it's quite simple to do the same as we've done with SQS where we create 2 new queues and a dummy exchange (the dummy exchange is needed if the code declares the exchange as part of the startup and is needed to not bind to original one)

```
┌─────────────┐        ┌──────────────────┐        ┌───────────────────┐         ┌──────────────────┐                        
│             │        │                  │        │                   │         │                  ├┐                       
│  Publisher  ├───────►│  Input Exchange  ├───────►│   Target Queue    │         │  Target Pods     ││                       
│             │        │                  │        │                   │         │                  ││                       
└─────────────┘        └──────────────────┘        └──┬────────────────┘         └┬─────────────────┘│                       
                                                      │                           └─────▲────────────┘                       
                                                      │                                 │                                    
                                                      │                                 │                                    
                                                    ┌─▼────────────────┐          ┌─────┴──────────┐                         
                                                    │                  │ Rest     │                │                         
                                                    │  Mirrord Shovel  ├─────────►│  Mirrord Sink  │◄───────────┐            
                                                    │                  │          │                │            │            
                                                    └──┬───────────────┘          └────────────────┘            │            
                                                       │                                                        │            
                                                       │Filtered                                                │            
                                                   ┌───▼───────────┐             ┌────────────────┐      ┌──────┴───────────┐
                                                   │               │             │                │      │                  │
                                                   │  Split Queue  ├────────────►│  Dev Instance  │      │  Dummy Exchange  │
                                                   │               │             │                │      │                  │
                                                   └──────▲────────┘             └────────────────┘      └──────┬───────────┘
                                                          │                                                     │            
                                                          └─────────────────────────────────────────────────────┘            
```

With managing our own "shovel" we get complete control over the message and we can perform body as well as header filters.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The base solution is doing pretty much the same as we do in SQS, we can create 2 or more new queues (one for unfiltered and any filtered queues depending on how many sessions are up), then we patch the target with the new queue for the unfiltered messages and we will be the consumer from original queue and pipe to the relevant queue.

### MirrordRabbitMQCluster

A custom resource to store connection strings so they can be used on a name basis when split definition is needed.

```yaml
apiVersion: queues.mirrord.metalbear.co/v1alpha
kind: MirrordRabbitMQCluster
metadata:
  name: cluster-definition-name-with-credenitals
spec:
  clientProperties: {}
  connectionString: amqp://guest:guest@rabbit:5672/%2F
  locale: en_US
  tls:
    identity: # client certificate authentication
    certChain: # custom certificates chain in PEM format
```

Here also it's the place for any tls config or custom clientProperties or some other locale than the default one.

### MirrordWorkloadQueueRegistry extension.

As part of the changes I assume best way is to extend the current `MirrordWorkloadQueueRegistry` resource, this is quite simple since the `queueType` is the tag for `SplitQueue` enum. The change should be the new arguments that are needed to define the queue and exchange binding if it is defined.

```yaml
apiVersion: queues.mirrord.metalbear.co/v1alpha
kind: MirrordWorkloadQueueRegistry
metadata:
  name: queue-definition
spec:
  queues:
    first-queue: 
      queueType: RMQ
      nameSource:
        envVar: QUEUE_NAME
      arguments:
        x-queue-type: quorum
      options:
        passive: false
        durable: true
        exclusive: false
        auto_delete: false
        nowait: false
      exchangeBindings:
        nameSource:
          envVar: EXCHANGE_NAME
        routingKey: ""
        arguments:
          some-logic-header: value
  ...
status:
  activeRmqSplits:
    queueNames:
      first-queue:
        exchangeName: dummy-exchange
        originalName: target-queue
        outputName: mirrord-sink
```

### MirrordRabbitMQSession

The main purpose of the rabbitmq session is to create and handle the lifespan of created queues and the state of their creation so the operator can assertively define the state. I want to store all needed data for the split in the session since I want to avoid management api and only use the declarative AMQP api only.

```yaml
apiVersion: queues.mirrord.metalbear.co/v1alpha
kind: MirrordRabbitMQSession
metadata:
  name: some-session-id-with-prefix-or-suffix
spec:
  cluster:
    clusterRef:
      name: cluster-definition-name-with-credenitals
      vhost: /
    // or inline specification
    connectionString: amqp://guest:guest@rabbit:5672/%2F
  queues:
  - queue: target-queue
    arguments:
      x-queue-type: quorum
    options:
      passive: false
      durable: true
      exclusive: false
      auto_delete: false
      nowait: false
  exchangeBindings:
  - exchange: dummy-exchange
    queue: target-queue
    routingKey: ""
    arguments:
      some-logic-header: value
  filters:
  - queues: [target-queue]
    messageFilter:
      x-header: my-value
status:
  sharedQueue:
    queue: mirrord-sink
    arguments:
      x-queue-type: classic
    options:
      durable: true
  queues:
  - queue: split-queue
    arguments:
      x-queue-type: quorum
    options:
      passive: false
      durable: true
      exclusive: false
      auto_delete: false
      nowait: false
  ...
```

The idea for the implementation is to be the single resource that is needed to be managed by the operator regarding the rabbitmq splitting, when deleted then corresponding queues should be drained and moved back to original one.

### Expected Workflow

The idea is that we have a loaded in-memory state of all the `MirrordWorkloadQueueRegistry` resources, once a session with the relevant queue-split feature is enabled the `MirrordRabbitMQSession` resource will be created with all the relevant paramagnets so each side can create the queue and bindings meaning we probably don't need to wait for the queue-split to actually start before starting the agent and continue the session since the api is mostly declarative with the queue and binding being assumed by the application that they may not exist even in normal workflow.

One thing is that different with rabbit and sqs is that rabbit also includes exchange bindings as part of the declaration, we don't want to bind the original exchange to our new created queues because in most cases it will cause message duplication in some cases my break the expected flow of messages through it. Simple solution is to create a dummy exchange that will be used just for the declarative bind instead of asking the user to make it optional in their code. (the exchange will not be used for messages just for the binding and will need to have the same arguments as original one has). This logic should be optional for us because not everyone will declare the exchanges bindings in their service.

To share the same unfiltered queue for multiple sessions via the `MirrordWorkloadQueueRegistry`'s status with activeRmqSplits where if another session is reusing the same queue it will first reference the `MirrordWorkloadQueueRegistry`'s store.

```
                                  ┌─────────────────┐                                                     
                                  │                 │                                                     
                                  │  Patch Target   │                                                     
                                  │                 │                                                     
                                  └────────▲────────┘                                                     
                                           │ Unfiltered                                                   
┌───────────────────┐         ┌────────────┴─────────────┐          ┌────────────────────┐                
│                   │ Create  │                          │ Filtered │                    │                
│  Mirrord Session  ├────────►│  MirrordRabbitMQSession  ├─────────►│ Update Env Values  │                
│                   │         │                          │          │    (on session)    │                
└───────────────────┘         └────────────┬─────────────┘          └────────────────────┘                
                                           │        ┌───────────────────────────────────────────┐         
                                  ┌────────▼────────┘   ┌────────────┐       ┌─────────────┐    │         
                                  │                     │            │       │             │    │         
                                  │  Create Queues      │  Filtered  │       │ Unfiltered  │    │         
                                  │                     │            │       │             │    │         
                                  └────────┬────────┐   └──────▲─────┘       └─────▲───────┘    │         
                                           │        └──────────┼───────────────────┼────────────┘         
        ┌────────────────┐         ┌───────▼────────┐          │                   │                      
        │                │         │                │          │                   │                      
        │  Target Queue  ├────────►│  Start Shovel  ├──────────┴───────────────────┘                      
        │                │         │                │                                                     
        └────────────────┘         └────────────────┘                                                     
```

### "Shovel"

The shovel should be a task that we spawn on the operator (or a separated cli for memory and performance sake if need be) and we should have some sort of internal api to it, to create the initial consumer and to update the filter logic so it will know what queue to send to. It's quite simple but also nuanced, the consumer should be quite configurable by the user because incorrect configuration may cause performance impact on the entire rabbit cluster (durable queues and non durable queues have very different resource constraints aka disk vs memory usage regarding queue sizes), I will add the pseudo code for what I expect to do.

When a new queue split is requested and there is no one existing it will do something like
```rust
let connection = ClientPool::get_connection(mirrord_rabbit_mq_cluster);

let publisher_channel = connection.create_channel();

// Check targets exist (or create them)
{
  publisher_channel.assert_queue(target_queue_filter.unfiltered_queue());

  for filtered in target_queue_filter.filtered_queues() {
    publisher_channel.assert_queue(filtered);
  }
}

known_filters[target_queue] = target_queue_filter; // the actual map of filters and what queue to go to

// By default channels don't return confirmation upon message delivery
publisher_channel.make_with_confirm_channel();

let consumer_channel = connection.create_channel();
let consumer = consumer_channel.consume(target_queue, target_queue_arguments);

while let Some(message) = consumer.next() {
  let queue_name = known_filters[target_queue].evaluate_destination(&message);

  let result = publisher_channel.publish("", queue_name, &message);

  match result {
    Confirm::Ack(..) => message.ack(),
    Confirm::Nack(err) => message.nack(err),
  }
}
```

And when doing a queue split when another one exists as well we only will need to update the filters.

```rust
let new_filtered_queue = known_filters[target_queue].merge_filters(target_queue_filter);

if new_filtered_queue.is_empty().not() {
  let connection = ClientPool::get_connection(mirrord_rabbit_mq_cluster);
  let channel = connection.create_channel();

  for filtered in new_filtered_queue {
    channel.assert_queue(filtered);
  }
}
```

## Drawbacks
[drawbacks]: #drawbacks

Main drawback of this solution is the fact that the operator is the one performing the shoveling action, the movement of the messages form inside the cluster and back inside of it. We need to pay very careful attention to not drop any messages at any point we need to make sure we recover correctly from any network issues or anything else because if we start dropping messages we will defiantly interfere with other users on the cluster.

One main impact on the cluster itself will be a performance one, since we are moving messages through the external api we are subject to multiple "transactions" we create where we want to guarantee delivery, and the replication that can happen if the queues are defined as highly available.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main though behind consuming and re-publishing the messages is the amount of control we have over the filters, it's very likely that payload filters are a thing that costumers will require since a lot of messages are simple json objects that may have the developer identifying information that is needed to create a correct filter so the messages will be routed to the correct developer.

## Prior art
[prior-art]: #prior-art

SQS queue splitting, most complaints Iv'e heard is the existence of the `MirrordWorkloadQueueRegistry`, though I think the way to solve it is to have a util (maybe inside mirrord wizard) that can connect to the management api and provide a way to create the registry entries with automatic filling of most fields.

The features should be very similar.


## Unresolved questions
[unresolved-questions]: #unresolved-questions

Streams, they theoretically should work the same but because of the persistent nature of them the movement of messages upon cleanup needs to be handled carefully.

Multiple matches, the question is what happens if we have multiple sessions and a message that matches the filter for both, should we do like the rabbit's exchange and just copy the message to both? or race and first one to be in the list wins?

## Future possibilities
[future-possibilities]: #future-possibilities

We can crate our own exchange plugin that does the routing and keep most of the logic in the rabbitmq cluster itself without doing the moving of the messages via api.

Another possible problem is hardcoded values at the user side so theoretically it should be easy enough to make AI traverse user's code and help them in the creation of `MirrordWorkloadQueueRegistry` resources and the possible changes to allow mirrord to work and actually replace the fields.
