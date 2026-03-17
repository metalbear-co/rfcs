- Feature Name: `sqs_auto_discovery`
- Start Date: 2026-03-16
- Last Updated: 2026-03-16
- RFC PR: [metalbear-co/rfcs#21](https://github.com/metalbear-co/rfcs/pull/21)

## Summary
[summary]: #summary

Reduce the time and coordination required to enable SQS splitting. The mirrord operator will be able to observe which SQS queues a selected workload actually uses, then expose that information to a mirrord admin. The discovered queue names can later be used when creating `MirrordWorkloadQueueRegistry` resources and configuring SQS splitting sessions.

## Motivation
[motivation]: #motivation

Today, enabling SQS splitting requires an admin to know which queues belong to which workloads. In practice, that mapping is often knowledge held by the application team (if anyone), not the platform team operating mirrord. As a result, rollout becomes slow and communication-heavy:

1. The admin identifies a workload that should support SQS splitting.
2. The admin asks the owning team which queues it consumes.
3. The application team investigates code, configuration, and runtime behavior.
4. Only then can the admin create the matching `MirrordWorkloadQueueRegistry`.

This slows down adoption of SQS splitting and turns what should be an operator task into a cross-team coordination exercise.

Automatic discovery improves this workflow by letting the operator learn queue usage from real traffic. The admin still decides how to configure SQS splitting, but no longer needs to manually discover the queue-to-workload mapping first.

### Use cases

1. A platform engineer wants to prepare a workload for SQS splitting without waiting on the application team to enumerate every queue.
2. An application team is unsure which queues are still used in production and wants operational evidence before enabling splitting.
3. An organization rolling out mirrord across many services wants a repeatable, low-touch workflow for SQS onboarding.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

An admin marks a workload for SQS autodiscovery. The operator then ensures that each pod of that workload runs with a lightweight proxy sidecar.

The sidecar sits on the path of outbound SQS requests and does three things:

1. Receives the SQS request before it leaves the pod.
2. Extracts queue information from the request.
3. Forwards the request to AWS so the application continues to behave normally.

Separately, the operator periodically collects the discovered queue names from the sidecars and exposes them back to the admin. The exact API surface is still open in this RFC: the data may be attached directly to the workload, or surfaced through `MirrordWorkloadQueueRegistry`.

From the admin's point of view, the feature is intentionally simple:

1. Opt a workload into autodiscovery.
2. Let the workload run normally for some time.
3. Inspect the discovered queue names.
4. Use that information when creating the final SQS splitting configuration.

This feature is advisory. It does not enable SQS splitting by itself, and it does not change message-routing behavior. It only shortens the path from "we want SQS splitting here" to "we know how to configure it."

This proposal also has minimal interaction with the rest of mirrord. It uses operator-managed workload patching and sidecar injection, but does not otherwise change traffic stealing, file ops, DNS, or environment handling outside the SQS-specific pod modifications described below.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The design has four major parts:

1. Detect workloads that should participate in autodiscovery.
2. Inject a proxy sidecar into those workloads.
3. Redirect SQS requests through the sidecar.
4. Periodically collect the discovered queue names from the sidecars.

### Detection of workloads marked for autodiscovery

This RFC leaves the control-plane shape open, but there are two concrete options.

#### Option A: mark workloads directly

Example:

```yaml
metadata:
  labels:
    operator.metalbear.co/sqs-splitting: '{"autoDiscovery": true}'
```

In this model, the admin annotates or labels the workload directly, and the operator reconciles all SQS-splitting-compatible workloads that carry the marker.

This keeps opt-in close to the workload and makes autodiscovery easy to enable before any `MirrordWorkloadQueueRegistry` exists.

#### Option B: mark `MirrordWorkloadQueueRegistry`

Example:

```yaml
spec:
  sqs:
    autoDiscovery: true
```

In this model, the admin sets a flag on the registry, and the operator reconciles registries instead of raw workloads.

This keeps autodiscovery inside the existing SQS configuration model, but it means the admin must create registry objects earlier in the workflow.

#### Where the output should go

The same choice affects how the operator exposes discovered queues:

1. A workload annotation, for example `operator.metalbear.co/sqs-splitting: '{"discoveredQueues": [...]}'`
2. A new `MirrordWorkloadQueueRegistry` status field such as `.status.sqsDetails.discoveredQueues`

The RFC does not yet choose between these shapes. The main tradeoff is whether autodiscovery should be modeled as workload metadata or as part of the registry lifecycle.

### Sidecar injection

To support autodiscovery, the operator must be able to inject a sidecar container in addition to the existing environment-variable patching.
The cleanest way to express that is to extend `MirrordClusterWorkloadPatchRequest` and `MirrordClusterWorkloadPatch` with a new `sidecars` field.

```rust
pub struct MirrordClusterWorkloadPatchRequestSpec {
    pub workload_ref: PatchedWorkloadRef,
    pub env_vars: Vec<EnvVar>,
    pub replicas: Option<i32>,

    /// New field
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub sidecars: Vec<Sidecar>,
}

/// Custom type to represent a container in the patching resources.
/// k8s-openapi `Container` does not implement `Hash`/`Eq`.
#[derive(Clone, Debug, Deserialize, Serialize, JsonSchema, Hash, PartialEq, Eq)]
#[schemars(schema_with = "...")] // needs a custom schema due to usage of `serde_json::Value`
pub struct Sidecar {
    pub name: String,
    #[serde(flatten)]
    pub fields: BTreeMap<String, serde_json::Value>,
}
```

The `name` field is important because it gives the patch request controller a stable way to detect conflicts between injected sidecars.

For each selected workload, the operator would:

1. Allocate a random local port for the sidecar's SQS listener.
2. Build the sidecar container spec.
3. Issue a patch request that:
   - injects the environment variables required to redirect SQS traffic to the sidecar
   - injects the sidecar container itself

### Handling of SQS requests

The sidecar must be able to inspect SQS requests and still forward them to the correct AWS endpoint. There are two candidate mechanisms for redirecting traffic from the application to the sidecar:

1. `AWS_ENDPOINT_URL_SQS`
2. `HTTP(S)_PROXY`

Both require some application-level adjustments, so the question is not whether code changes are needed, but which set of changes is more acceptable.

#### Option A: `AWS_ENDPOINT_URL_SQS`

`AWS_ENDPOINT_URL_SQS` is the SDK-supported mechanism for overriding the target endpoint for requests to SQS.
That makes it the more natural fit for this feature, because the application is still talking directly to "an SQS endpoint", just one that happens to be the local sidecar.

With this approach, the SDK sends requests similar to the following:

```http
POST / HTTP/1.1
content-type: application/x-amz-json-1.0
x-amz-target: AmazonSQS.GetQueueUrl
x-amzn-query-mode: true
content-length: 20
x-amz-user-agent: aws-sdk-js/3.1008.0
user-agent: aws-sdk-js/3.1008.0 ua/2.1 os/linux#6.17.0-14-generic lang/js md/nodejs#20.18.0 api/sqs#3.1008.0 m/E,n
host: 127.0.0.1:8888
amz-sdk-invocation-id: <...>
amz-sdk-request: attempt=1; max=3
x-amz-date: 20260316T114858Z
x-amz-content-sha256: 92ffd1bfe9cfde1a3a28bda764b4c6758415669ada01c5ea74e33aad7212428f
authorization: AWS4-HMAC-SHA256 Credential=<...>/<...>/eu-central-1/sqs/aws4_request, SignedHeaders=amz-sdk-invocation-id;amz-sdk-request;content-length;content-type;host;x-amz-content-sha256;x-amz-date;x-amz-target;x-amz-user-agent;x-amzn-query-mode, Signature=<...>
connection: keep-alive

{"QueueName":"test"}
```

Important properties of this request:

1. The URI is just `/`, and the `host` header points at the sidecar.
2. The request is still SigV4-signed.
3. The operation is carried in `x-amz-target`, for example `AmazonSQS.GetQueueUrl`.

To forward the request, the sidecar must reconstruct the original AWS endpoint. The required region is already present in the credential scope inside the `authorization` header:

`Credential=<...>/<...>/eu-central-1/sqs/aws4_request`

From that, the sidecar can infer the upstream endpoint, for example `sqs.eu-central-1.amazonaws.com`.

The main complication is that many SQS SDK workflows eventually switch from queue names to queue URLs. For autodiscovery to remain on the request path, the application must avoid treating the returned queue URL as a new endpoint. In the Node.js SDK v3, for example:

```javascript
const client = new SQSClient({
  credentials: defaultProvider(),
  useQueueUrlAsEndpoint: false,
});
```

Without this adjustment, two failure modes appear:

1. If the application already has a prefetched queue URL, later requests bypass the sidecar entirely.
2. If the application fetches the queue URL through the sidecar, AWS will return an **HTTPS** URL based on the sidecar endpoint, and later requests will fail.

The second case is theoretically fixable by mutating the response body (change schema from `https` to `http` in queue URL, if the response contains one).
The first is not. Because of that, application changes are required for this option.

#### Option B: `HTTP(S)_PROXY`

`HTTP(S)_PROXY` is the more generic mechanism: applications are told to route outbound HTTP(S) traffic through a local proxy.
In theory that lets the sidecar observe SQS without reconstructing the original endpoint.

In practice, this path is more invasive:

1. Not all AWS SDKs respect system proxy settings. Node.js SDK v3 is one example.
2. SQS traffic is HTTPS, so the SDK will usually send a `CONNECT` request and still expect end-to-end TLS.
3. The sidecar would need to terminate TLS while impersonating `sqs.<region>.amazonaws.com`.
4. That in turn requires distributing a custom CA and teaching the application to trust it.

Because many SDKs rely on embedded CA bundles, simply mounting a certificate file is often not enough. The application may need explicit trust-store configuration. For example, with the Node.js SDK v3:

```javascript
const mirrordProxyCaPath = process.env.MIRRORD_PROXY_CA_PATH;
const mirrordProxyCa = readFileSync(mirrordProxyCaPath, "utf8");
const sqsClient = addProxyToClient(
  new SQSClient({ credentials: defaultProvider() }),
  {
    agentOptions: {
      ca: [mirrordProxyCa], // overrides the default trust roots
    },
  }
);
```

This option might seem a bit more complex, but it does have a very nice property - we can establish a generic pattern for enabling mfT to be man-in-the-middle for outbound HTTPS traffic. While we don't have any use cases for it right now, it does seem very handy.

### Data collection

In addition to the request-forwarding listener, the sidecar runs a second HTTP server on a random high port, bound to `0.0.0.0` inside the pod. This endpoint exposes the set of queue names observed by that sidecar.

The operator periodically polls these sidecars and aggregates the results for the owning workload. Discovery is therefore best-effort and observational:

1. Only queues that were actually used while autodiscovery was enabled will be reported.
2. Newly deployed pods may observe different queues over time.
3. The exposed data should be treated as a helpful starting point, not a formal source of truth.

For the first iteration, this second API can be left as plain HTTP inside the cluster. If customers later require stronger guarantees, the design can be extended with operator-issued certificates:

1. The operator generates a random certificate.
2. The sidecar receives the certificate through environment variables.
3. The sidecar accepts only connections that present the expected certificate.
4. The operator presents that certificate when polling sidecars.

## Drawbacks
[drawbacks]: #drawbacks

Discovery requires application changes regardless of which request-redirection mechanism is chosen.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Prior art
[prior-art]: #prior-art

This design follows two well-established patterns:

1. Sidecar-based traffic observation inside Kubernetes pods.
2. Endpoint or proxy overrides for SDK-driven outbound traffic.

Internally, it also aligns with existing mirrord operator behavior: the operator already patches workloads, injects environment variables, and manages runtime changes on behalf of the admin. This proposal extends that model to SQS-specific discovery rather than introducing a separate control plane.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

1. How should the admin opt a workload into autodiscovery: a new workload marker, or a field in `MirrordWorkloadQueueRegistry`?
2. Where should discovered queue names be published: workload metadata, or `MirrordWorkloadQueueRegistry` status?
3. Which traffic-redirection mechanism should we standardize on for the first implementation: `AWS_ENDPOINT_URL_SQS` or `HTTP(S)_PROXY`?

## Future possibilities
[future-possibilities]: #future-possibilities

The most natural extension is to reuse the same sidecar not only for discovery, but also for the SQS split itself.
Because the sidecar already observes SQS requests and runs with the workload's identity,
it could eventually become the control plane for message routing.

One possible future flow for `AmazonSQS.ReceiveMessage` would be:

1. Read the queue URL from the request body.
2. Determine the temporary output queue that matches the active split filters.
3. Rewrite the request to target that queue.
4. Re-sign the modified request with the available credentials.
