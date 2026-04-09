- Feature Name: `unified_queue_splitting_crd`
- Start Date: 2026-04-07
- Last Updated: 2026-04-07
- RFC PR: [metalbear-co/rfcs#26](https://github.com/metalbear-co/rfcs/pull/26)

## Summary
[summary]: #summary

Replace the existing queue splitting Custom Resource Definitions (`MirrordWorkloadQueueRegistry`, `MirrordKafkaTopicsConsumer`, `MirrordKafkaClientConfig`) with a single `MirrordSplitConfig` CRD and a shared queue splitting flow in the operator. The new model keeps broker-specific details where needed, but standardizes how workloads, client configuration, and queue references are described across SQS, Kafka, RabbitMQ, and future supported brokers (including Google Cloud Pub/Sub and Azure Service Bus).

## Motivation
[motivation]: #motivation

Queue splitting has evolved separately for different broker types. That creates three practical problems:

1. Users need to reason about several different resources for what is conceptually one feature.
2. The operator contains duplicated queue splitting logic and duplicated configuration handling.
3. Some capabilities are available for some broker types but not for others. For example, SQS and RabbitMQ support regex-based environment variable selection today, while Kafka does not.

This fragmentation also makes future work slower. Adding a new broker type or a new configuration capability means touching several resource definitions and keeping their behavior aligned.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Queue splitting is configured through a single `MirrordSplitConfig` resource per workload. The resource describes:

1. Which workload is being patched for queue splitting.
2. How the workload restart should behave.
3. How temporary resource names should be generated.
4. How the broker client(s) should be configured.
5. Which logical queues (queues, topics, pull subscriptions, ...) can be split for that workload.

`MirrordPropertyList` remains the generic way to provide broker-specific configuration.
`MirrordSplitConfig` references those property lists where needed instead of introducing new per-broker-flavor CRDs.

From the user's point of view, the most important change is that queue splitting becomes "one workload, one config".

As usual, every logical queue entry has a stable `id`, and that `id` is what the developer references from their mirrord config.
The operator resolves the actual queue names, consumer groups, application IDs, or exchanges by reading the target workload's pod template.

There can be at most one `MirrordSplitConfig` for a workload. If the operator discovers more than one, it may fail queue splitting sessions for that workload rather than guessing which configuration should win.

For example:

```yaml
apiVersion: queues.mirrord.metalbear.co/v1
kind: MirrordSplitConfig
metadata:
  name: checkout-queues
  namespace: stage-1
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout
  clientConfigs:
    kafka: checkout-kafka-client
  queues:
    - id: orders
      kind: kafka
      appConfig:
        topic:
          - env: ORDERS_TOPIC
        groupId:
          - env: ORDERS_GROUP_ID
    - id: delayed-retries
      kind: sqs
      appConfig:
        queue:
          - env: ^ORDERS_RETRY_QUEUE_
```

Queue splitting status is exposed to users through `mirrord queues` (aliased `mirrord qs`) subcommands:

- `mirrord queues list`
- `mirrord queues get --name <splitconfig name>`

The exact contents of that status output are still to be defined, but the intent is that queue splitting state is inspected through the dedicated queue-splitting commands rather than through `mirrord operator status`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `MirrordSplitConfig`

`MirrordSplitConfig` replaces the queue-specific CRDs with one schema that covers all supported queue kinds. The resource contains:

1. `targetRef`: the workload to patch.
2. `restart`: optional restart behavior for the workload.
3. `drainTimeout`: optional message drain timeout for the workload.
4. `tmpNameTemplate`: optional template for generated temporary resource names.
5. `clientConfigs`: optional default client configuration per queue kind.
6. `queues`: the logical queues that users can reference from mirrord config.

### Target workload

The target workload is specified in the required `.spec.targetRef` field.

`targetRef` contains three required fields:

1. `apiVersion`: API version of the target resource.
2. `kind`: kind of the target resource.
3. `name`: name of the target resource.

The namespace is inferred from the namespace of the `MirrordSplitConfig` itself. This is flexible enough to target any supported Kubernetes resource in the same namespace.

Example:

```yaml
targetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: workload-name
```

### Restart policy

Restart behavior is configured in the optional `.spec.restart` section.

`restart` contains three optional fields:

1. `strategy`: restart strategy to use, either `standard` or `isolatePods`. Defaults to the global operator setting.
2. `timeout`: timeout in seconds (`u32`) for reaching the `waitForPods` condition after the restart starts. Defaults to the global operator setting.
3. `waitForPods`: success condition for the restart. This controls how many patched and ready target pods are required before mirrord sessions may start.

`waitForPods` can be either:

1. A `u32`. `0` means sessions may start immediately.
2. `all`. This means there can be no unpatched pod, and at least one patched pod must be ready.

Examples:

```yaml
# Use isolatePods and wait for all pods to be patched.
# Timeout after 180 seconds.
restart:
  strategy: isolatePods
  timeout: 180
  waitForPods: all

---
# Do not wait for any pod to be patched.
# Use global defaults for other settings.
restart:
  waitForPods: 0
```

### Drain timeout

Message drain timeout (in seconds, as `u32`) is configured in the optional `.spec.drainTimeout` field.

If the field is not present, idle splits will live indefinitely while the fallback queues have unconsumed messages.

Examples:

```yaml
# Disable message drain. Split will terminate as soon as there are no active split sessions.
drainTimeout: 0

---
# Wait for at most 120 seconds for the fallback queues to be drainged while there are no active split sessions.
drainTimeout: 120
```

### Temporary resource name template

Temporary resource names are configured through the optional `.spec.tmpNameTemplate` field. The default value is:

```text
mirrord-tmp-{{RANDOM}}{{FALLBACK}}{{ORIGINAL}}
```

The template supports three special blocks:

1. `{{RANDOM}}`: random lowercase ASCII characters.
2. `{{FALLBACK}}`: `-fallback-` for fallback resources (e.g. queues for unmatched messages) and `-` for user temporary resources.
3. `{{ORIGINAL}}`: the original resource name.

### Broker client configuration

Default broker client configuration is specified in the optional `.spec.clientConfigs` section.

This section has one optional field per supported queue kind:

1. `sqs`
2. `kafka`
3. `rmq`
4. `googlePubSub`
5. `azureServiceBus`
6. ...

Each field contains the name of a `MirrordPropertyList` resource that holds broker-specific client configuration.
How that property list is interpreted remains specific to the queue kind.

This section is optional because for some queue kinds we can derive their client configuration from the operator environment.

Example:

```yaml
clientConfigs:
  kafka: mirrord-librdkafka-client
  # SQS client config can be derived from the operator environment.
  # sqs: ...
```

### Queue definitions

Splittable queues are configured in the required `.spec.queues` field.

`queues` is an array. Each item defines one logical queue ID that users can reference from mirrord config. One logical queue ID may resolve to more than one physical queue/topic/subscription name.

Each queue entry supports the following fields:

1. `id` (required): arbitrary queue ID used in the user's mirrord config.
2. `kind` (required): one of `sqs`, `kafka`, `rmq`, `googlePubSub`, `azureServiceBus`, ...
3. `clientConfig` (optional): override for `.spec.clientConfigs.<kind>`. If present, the operator uses this client configuration for this queue entry instead of the kind-wide default.
For example, this enables support of workloads that consume from multiple message broker clusters.
4. `queueConfig` (optional): name of a `MirrordPropertyList` containing additional queue-specific configuration.
For example, this enables setting marker tags on created temporary queues.
5. `appConfig` (required): describes how the target application is configured to consume the queue.

`appConfig` can contain multiple keys, each corresponding to a different queue-kind-specific property that is needed in order to consume messages:
1. `kafka`:
    * `topic` (required)
    * `groupId` (either `groupId` or `appId` must be present)
    * `appId` (either `groupId` or `appId` must be present)
2. `sqs`:
    * `queue` (required)
3. `rmq`:
    * `queue` (required)
    * `exchange` (optional)
4. `googlePubSub`:
    * `subscription` (required)
    * `projectId` (optional)
5. `azureServiceBus`: TBD, most likely `queue` OR `topic` and `subscription`

Each key is mapped to an array of references.
Each reference points to one or more environment variables in the target pod's template,
and supports the following fields:

1. `env` (either `env` or `envLike` must be present): exact environment variable name.
2. `envLike` (either `env` or `envLike` must be present): regular expression matching environment variable names ([fancy-regex](https://docs.rs/fancy-regex/latest/fancy_regex/)).
3. `fallback` (optional): fallback value if the variable is not found. Only valid when `env` is used,
because without `env` we don't have any variable name to use when injecting our patched value.
4. `valueSelector` (optional): [JAQ](https://github.com/01mf02/jaq) selector used to extract the property value(s) from the variable(s) value(s).
This allows for accessing properties that are contained in structured data.
For example, selector `.[]` can be used to extract the names of both topics specified in JSON `{"ordersTopicName": "topicName1", "viewsTopicName": "topicName2"}`.
5. `containers` (optional): list of containers that have this reference.
Defaults to all containers of the target workload, excluding known infra (e.g. mesh) sidecars.

Examples:

```yaml
# The application consumes one Kafka topic.
# The topic name and consumer group ID are passed in two separate variables.
# The workload has only one container.
queues:
  - id: registration-events
    kind: kafka
    appConfig:
      topic:
        - env: REGISTRATION_EVENTS_TOPIC
      groupId:
        - env: KAFKA_GROUP_ID

---
# The application consumes one Kafka topic.
# The topic name and consumer group ID are stored in one structured variable.
# Only the `consumer` container contains the relevant env var.
queues:
  - id: registration-events
    kind: kafka
    appConfig:
      topic:
        - env: REGISTRATION_EVENTS
          valueSelector: .topic
          containers: ["consumer"]
      groupId:
        - env: REGISTRATION_EVENTS
          valueSelector: .groupId
          containers: ["consumer"]

---
# The application consumes two Kafka topics: one with the standard consumer API
# and one with the Kafka Streams API. The Kafka Streams topic uses a special client.
queues:
  - id: registration-events
    kind: kafka
    appConfig:
      topic:
        - env: REGISTRATION_EVENTS_TOPIC
      groupId:
        - env: KAFKA_GROUP_ID
  - id: subscription-events
    kind: kafka
    clientConfig: mirrord-java-kafka-client
    appConfig:  
      topic:
        - env: SUBSCRIPTION_EVENTS
      appId:
        - env: KAFKA_STREAMS_APP_ID

---
# The application consumes multiple SQS queues. Their names are stored in
# multiple environment variables prefixed with `SQS_QUEUE_`. Each variable
# contains a map whose values are queue names.
queues:
  - id: all-sqs-queues
    kind: sqs
    appConfig:
      queue:
        - envLike: ^SQS_QUEUE_
          valueSelector: .[]
          containers: ["main", "sidecar"]
```

### Full example

```yaml
apiVersion: queues.mirrord.metalbear.co/v1
kind: MirrordSplitConfig
metadata:
  name: rfc-example
  namespace: workload-namespace
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: deployment-name
  restart:
    strategy: standard
    timeout: 120
    waitForPods: all
  drainTimeout: 120
  tmpNameTemplate: "mirrord-tmp{{FALLBACK}}{{RANDOM}}-{{ORIGINAL}}"
  clientConfigs:
    kafka: mirrord-librdkafka
    rmq: mirrord-rmq
  queues:
    - id: my-kafka-standard-topic
      kind: kafka
      appConfig:
        topic:
          - env: KAFKA_TOPIC
        groupId:
          - env: KAFKA_GROUP_ID
    - id: my-kafka-streams-topic
      kind: kafka
      clientConfig: mirrord-kafka-java
      appConfig:  
        topic:
          - envLike: ^KAFKA_STREAMS_TOPIC_
        appId:
          - env: KAFKA_APP_ID
    - id: my-sqs-queue
      kind: sqs
      queueConfig: mirrord-sqs-extra-tags
      appConfig:
        queue:
          - envLike: ^SQS_QUEUES_
            valueSelector: .[]
    - id: my-rmq-queue
      kind: rmq
      queueConfig: mirrord-my-rmq-queue
      appConfig:
        queue:
          - env: RMQ_QUEUE
            fallback: events-queue
            containers: ["rmq-consumer"]
        exchange:
          - env: RMQ_EXCHANGE
            fallback: events-exchange
            containers: ["rmq-consumer"]
    - id: my-google-pubsub-subscription
      kind: googlePubSub
      appConfig:
        subscription:
          - env: PUB_SUB_SUBSCRIPTION
```

### Queue splitting status

Queue splitting status is part of the `MirrordOperatorCrd` status, but it is intentionally surfaced through the queue-splitting CLI commands:

1. `mirrord qs list`
2. `mirrord qs get --name <splitconfig name>`

`mirrord operator status` does not include queue splitting status, because it's already cluttered.

The precise status payload is still TBD.

### Unified operator flow

The operator implementation is unified in the same way as the user-facing CRD:

1. There is one queue splitting controller instead of separate broker-specific controllers.
2. There is one session-owned internal resource, `MirrordClusterSplitSession`, instead of separate resources such as `MirrordSqsSession` and `MirrordClusterRmqSession`.
3. All temporary resources are tracked through `MirrordClusterExternalResource`, with owner references to either `MirrordClusterSplitSession` or `MirrordClusterWorkloadPatchRequest`.
4. Workload split state is tracked through `MirrordClusterWorkloadPatchRequest` and its metadata, as in the current SQS and RMQ implementations.

`MirrordClusterSplitSession`:

```yaml
apiVersion: mirrord.metalbear.co/v1
kind: MirrordClusterSplitSession
metadata:
  name: aaaaaaaaaaaaaaaa
spec:
  # Session target. Required.
  target:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
    container: my-container
  # Session namespace. Required.
  namespace: target-namespace
  # Session owner. Required.
  owner:
    userId: ==asd3kjndfkvjn
    hostname: razz-machine
    username: razz
    k8sUsername: dev
  # Session key. Optional for backwards compatibility.
  sessionKey: 2f1b09ed-8788-4a68-a5b5-fc6919c2cf05
  # Queues to split. Required.
  # Both `headersFilter` and `jqFilter` empty mean "match nothing".
  queues:
      # ID from MirrordSplitConfig. Required.
    - id: queue-id
      # Queue kind. Required.
      kind: kafka
      # Set of regex-based headers/attributes filters, optional.
      headersFilter:
        header-1: regex-1
        header-2: regex-2
      # JQ-style filter, optional.
      # Receives the message in a queue-kind-specific structured format.
      jqFilter: .Body.clientId == "my-client-id"
  status:
    # Created temporary resources, by queue ID.
    tmpResources:
        # Queue ID from `MirrordSplitConfig`.
      - queueId: queue-id
        topic: # Matches property key in `.spec.queues.queue_id.appConfig` in `MirrordSplitConfig`.
          orig-1: mirrord-tmp-asdjcnoeff-orig-1
          orig-2: mirrord-tmp-asddkjfhnv-orig-2
    # Environment variable updates, by container name.
    # Stores updates for all containers, as they might be needed for target copying.
    envUpdates:
      main:
        - name: MESSAGE_TOPIC_1
          value: mirrord-tmp-asdjcnoeff-orig-1
        - name: MESSAGE_TOPIC_2
          value: mirrord-tmp-asddkjfhnv-orig-2
      sidecar:
        - name: MESSAGE_TOPIC_1
          value: mirrord-tmp-asdjcnoeff-orig-1
        - name: MESSAGE_TOPIC_2
          value: mirrord-tmp-asddkjfhnv-orig-2
    # Filled in case the split fails:
    #
    # failed:
    #   code: 404
    #   reason: QueueIdNotFound
    #   message: queue ID `queue-id` was not found in MirrordSplitConfig `checkout-queues`
```

Queue splitting flow:
1. Initiator of the split (e.g. endpoint that creates the target copy, endpoint that creates a new session, multicluster envoy) creates `MirrordClusterSplitSession`.
The status either empty or prefilled with created `tmpResources`.
2. Controller resolves full splitting parameters.
3. Controller starts split.
4. (optional, if not prefilled) Controller creates user tmp resources. Preserves them in `MirrordClusterExternalResource`s (with owner references to `MirrordClusterSplitSession`).
Also stores them in `.status.tmpResources`.
5. (optional, if prefilled) Controller compares tmp resources found in `.status.tmpResources` against the resolved splitting properties.
6. Controller fills `.status.envUpdates`.
7. Initiator of the split observes filled env updates and resumes work.

All potential failures are communicated to the initiator through the `.status.failed` field.
If the `MirrordClusterSplitSession` is owned by a resource that supports setting failure status, the controller fills that status as well.

High availability is implemented following the SQS example. Splitting state is saved in metadata of the `MirrordClusterWorkloadPatchRequest`.
The request is also marked with a label that prevents the patch manager from deleting it.
The queue splitting controller is responsible for deleting the `MirrordClusterWorkloadPatchRequest` when its split expires.

### Migration and backward compatibility

The migration path favors a unified implementation without immediately dropping all existing configuration:

1. Old-style active splits are forcefully terminated by the new version of the operator.
2. Existing queue definitions from `MirrordWorkloadQueueRegistry` and `MirrordKafkaTopicsConsumer` are still read during migration.
3. The operator builds one in-memory unified view by combining those existing definitions with `MirrordSplitConfig`.
4. If the same queue ID is defined in both old-style and new-style configuration, the `MirrordSplitConfig` definition wins.

## Drawbacks
[drawbacks]: #drawbacks

1. Existing queue splitting users will eventually need to migrate to the new configuration model.
2. During the migration period, the operator must understand both the old resources and the new unified CRD.
3. A single CRD is broader than the current per-broker resources, so validation rules that used to be implicit in separate schemas must now be enforced carefully.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why this design

A single CRD is the simplest representation of what queue splitting already is conceptually: one workload-level feature with broker-specific details. The design reduces duplication, makes capabilities easier to keep consistent across queue kinds, and provides a clearer extension point for new broker types.

Keeping `MirrordPropertyList` as the carrier for broker-specific properties also avoids inventing a second generic configuration mechanism.
The unified CRD decides when those properties are needed; the queue-specific code still decides how to interpret them.

Restricting the model to at most one `MirrordSplitConfig` per workload avoids ambiguous merges between multiple configs that all share the same target.
Failing the session is safer than silently choosing one configuration over another.

### Alternative: use resource metadata

Instead of creating a new CRD for queue splitting config, we can follow meshes example and put the configuration in the metadata of the workload itself.
Each of the `MirrordSplitConfig` top-level fields could be its own annotation on the workload.

Pros:
* We've heard from a customer that they don't like CRDs
* We would follow a known pattern (mesh)

Cons:
* mirrord-specific config would be more tightly coupled with the cluster infra

### Alternative: use operator config map

Instead of creating a new CRD for queue splitting config, we can have the user configure all queues in the operator helm chart.
The configuration would then be mounted on the operator pods through a ConfigMap.

Pros:
* Less entities for the customer to worry about

Cons:
* Customer would no longer be able to generate configuration dynamically
* Mounted ConfigMaps are not refreshed instantly

### Impact of not doing this

If we keep the current model, queue splitting remains harder to reason about for users and harder to maintain in the operator.
The immediate cost is duplication and inconsistent capabilities.
The longer-term cost is slower delivery whenever we extend queue splitting to new brokers or add a new feature to all existing brokers.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

1. What should be included in queue splitting status, and how much of it should be exposed through `mirrord qs list` versus `mirrord qs get`?

## Future possibilities

1. Use the `MirrordSplitConfig` CRD as a general configuration for "splitting" incoming data streams - both queue messages and HTTP.
Add an option to specify message/request filter templates for sessions.
