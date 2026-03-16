- Feature Name: `sqs_auto_discovery`
- Start Date: 2026-03-16
- Last Updated: 2026-03-16
- RFC PR: [metalbear-co/rfcs#0000](https://github.com/metalbear-co/rfcs/pull/0000)

## Summary
[summary]: #summary

Improve user experience when configuring SQS splitting.
mirrord Operator is able to automatically detect SQS queues consumed by target workloads,
and on demand exposes this data to the mirrord admin. The admin can later use it when configuring SQS splitting.

## Motivation
[motivation]: #motivation

mirrord admin usually does not now which SQS queues are consumed by which workloads.
This knowledge is required to create `MirrordWorkloadQueueRegistry` and enable SQS splitting sessions.
Because of this, setting up SQS splitting takes a long time, as it requires communication between teams.
Speeding up this process translates to speeding up mirrord adoption in the organization.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Operator reconciles workloads marked for SQS autodiscovery (see [detection](#detection-of-workloads-marked-for-autodiscovery)).
For each such workload, the operator ensures that all of workload's pods run with proxy sidecars injected.
The sidecars:

1. Intercept all SQS requests made by the application
2. Inspect the requests and extract queue names
3. Transparently pass the requests to their original destination

The operator periodically queries sidecars for names of consumed queues, and exposes this data to the admin (see [detection](#detection-of-workloads-marked-for-autodiscovery)).

This new functionality does not directly interact with any other mirrord feature.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

There are 4 major parts of the implementation:
1. Detection of workloads marked for autodiscovery
2. Sidecar injection
3. Handling of SQS requests
4. Data collection

### Detection of workloads marked for autodiscovery

There are two options to consider:
1. New label, e.g. `operator.metalbear.co/sqs-splitting: "{\"autoDiscovery\": true}"`. The admin adds the label to the workloads.
The operator reconciles all SQS-splitting-compatible workloads with this label.
2. New `MirrordWorkloadQueueRegistry` flag, e.g. `.spec.sqs.autoDiscovery: true`.
The admin sets the flag. The operator reconciles all registries.

This also determines how the operator later exposes the data to the admin:
1. New annotation on the workload, e.g. `operator.metalbear.co/sqs-splitting: "{\"discoveredQueues\": [...]}"`
2. New `MirrordWorkloadQueueRegistry` status field, e.g. `.status.sqsDetails.discoveredQueues`

### Sidecar injection

First, we need to add a new field to `MirrordClusterWorkloadPatchRequest` and `MirrordClusterWorkloadPatch`, allowing them to inject sidecar containers into pods.

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

`name` field on the sidecar allows for detecting conflicts in the patch request controller.

With this, the operator can easily patch the workload's pods:
1. Generate a random port for the sidecar to accept SQS requests on.
2. Prepare sidecar spec.
3. Issue a patch request that:
    * Injects extra environment variables into the original containers (see [next section](#handling-of-sqs-requests))
    * Injects the sidecar container

### Handling of SQS requests

In order to inspect the requests, we redirect them to the sidecar.
Here we have two options as well: `AWS_ENDPOINT_URL` and `HTTP(S)_PROXY`.

Note that **both** options require adjustments to the application code.

#### `AWS_ENDPOINT_URL`

This is the official way of telling AWS SDKs to use a different endpoint, so we can expect all their SDKs to respect it.
The SDK will send requests like this:

```
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

Couple of notes:
1. URI is just `/`, and `host` header is the sidecar's socket address.
2. Request is signed (`authorization` header).
3. SQS operation kind is speficied in the header - `x-amz-target: AmazonSQS.GetQueueUrl`.

In order to pass the request to its original destination, we need to reconstruct it's URL from the `Credential` entry in the `authorization` header.
The entry contains the region:
`Credential=<...>/<...>/eu-central-1/sqs/aws4_request` -> `sqs.eu-central-1.amazonsqs.com`

One important note is that SQS queues are mostly accessed by their URLs.
Because of this, the **application code must be adjusted** - SDK must be configured not to use queue URL as endpoint.

```javascript
const client = new SQSClient({
    credentials: defaultProvider(),
    useQueueUrlAsEndpoint: false,
});
```

If the SDK is not configured properly, following scenarios will fail:
1. Application uses prefetched queue URL -> request bypasses our proxy.
2. Application fetches queue URL -> proxy passes the request to AWS -> AWS returns queue URL like `https://127.0.0.1:<port>` -> application later makes an HTTPS request to the proxy -> request fails.

While the second case can be fixed by editing the response body, the first case cannot be fixed in any way.
Application code adjustment is required.

#### `HTTP(S) PROXY`

This is a standard way of telling applications to direct HTTP(S) requests to a systemwide proxy.
Not all AWS SDKs respect it (e.g. Node.js SDK v3), so the **application code must be adjusted**.

In this solution the main issue is that AWS SDK will always use HTTPS.
Even if we make it use the sidecar as a plain HTTP proxy, the SDK will still enforce TLS by making a `CONNECT` request.

In order to make it work, we need to terminate TLS **in the proxy sidecar**.
Doing that will require injecting a dummy CA into the application's truststore,
as we will need to pretend that we're `sqs.<region>.amazonaws.com`.
Since AWS SDKs use embedded lists of Mozilla-approved CAs,
we will not be able to simply inject the dummy CA through the filesystem.
This is the second required **adjustment of the application code**.

Full example with Node.js SDK v3:
```javascript
  const mirrordProxyCaPath = process.env.MIRRORD_PROXY_CA_PATH;
  const mirrordProxyCa = readFileSync(mirrordProxyCaPath, "utf8");
  const sqsClient = addProxyToClient(
    new SQSClient( {credentials: defaultProvider() }),
    {
      agentOptions: {
        ca: [mirrordProxyCa], // this overrides *all* certs trusted by default
      }
    }
  );
```

### Data collection

Sidecar containers run a second HTTP server (random high port, public IP `0.0.0.0`).
This second server allows for querying observed queue names.
Operator periodically queries the sidecars for data.

For the first version, we can leave this second API plain.
Should a relevant request come from the users, we can add TLS:
1. Operator generates a random certificate.
1. Proxy sidecar gets the certificate through env.
2. After accepting the TCP connection, sidecar runs TLS connect (accepting only the certificate found in env).
3. After making the TCP connection, operator runs TLS accept (offering only the previously generated certificate).

## Drawbacks
[drawbacks]: #drawbacks

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Prior art
[prior-art]: #prior-art

## Unresolved questions
[unresolved-questions]: #unresolved-questions

1. How should the admin mark workloads for autodiscovery? Should it be a new label on the workload or a field in the `MirrordWorkloadQueueRegistry`?
2. Similarly, where should we put autodiscovery output? Should it be an annotation on the workload or a field in the `MirrordWorkloadQueueRegistry` status?
3. Should we use `AWS_ENDPOINT_URL` or `HTTP(S)_PROXY`?

## Future possibilities
[future-possibilities]: #future-possibilities

A natural extension of this approach is to use the proxy sidecar for message routing during the SQS split itself.
Since the sidecar controls SQS requests and runs with the same service account, it can be used to patch the `AmazonSQS.ReceiveMessage` requests on the fly:

1. Retrieve the queue URL from the request body.
2. Determine the URL of the correct temporary output queue, according to the filters.
3. Change the queue URL in the request body.
4. Resign the request using the available credentials.
