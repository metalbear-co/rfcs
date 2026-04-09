- Feature Name: mirrord_up
- Start Date: 2026-04-09
- Last Updated: 2026-04-09 
- RFC PR: [metalbear-co/rfcs#0000](https://github.com/metalbear-co/rfcs/pull/0000)
- RFC reference:
  - [metalbear-co/rfcs#0000](https://github.com/metalbear-co/rfcs/pull/0000)

## Summary
[summary]: #summary

`mirrord up` runs and manages multiple mirrord sessions with a single `mirrord up` command, sourcing configuration for all the services from a single `mirrord-up.yaml` file. Think `docker compose` for mirrord.

## Motivation
[motivation]: #motivation

Some of our customers (e.g. Monday) use bespoke mirrord cli wrappers and `mirrord.json` generators to achieve similar functionality. The goal of `mirrord up` is to add first-class support for this use case.

Since `mirrord.json` supports a *lot* of configuration options, way more than is needed for the majority of use cases, _the design philosophy of this feature should be configuration of convention._ As Aviram said,

> We want it to work as simple as possible for 99% of our users but still have enough customization for the remaining 1%.

As such, certain configuration options from `mirrord.json` may not be exposed from `mirrord-up.yaml`, especially in the first few releases.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Start by creating a `mirrord-up.yaml` file:

```yaml
defaults:
  accept_invalid_certificates: true
  operator: true
  telemetry: true

services:
  user-auth-service:
    target:
      path: deployment/dashboard-app
    env:
      override:
        NODE_ENV: "development"
        AUTH_SERVICE_URL: "http://localhost:8080"
        TOKEN_EXPIRY: "3600"
    mode: steal
    http_filter:
      header_filter: "x-session: session2"

    ignore_ports: [9090, 9091, 15090]
    run: 
      command: ["node", "app.js"]

  stage-user-dashboard-app:
    target:
      path: deployment/dashboard-app-123
    env:
      override:
        APP_NAME: "dashboard-app-123"
        NODE_ENV: "development"

    mode: steal
      http_filter:
        header_filter: "x-session: session1"
      messages: # optional, for sqs/kafka message filters
        type: kafka # or sqs
        filter:
            - "baggage": ".*mirrord-session=alice.*"
      mode: steal
        # Available modes: 
        # 1. mirror (default mode)
        #    + optional http_filter (without filter, everything is mirrored)
        # 2. replace mode => copy target + scale down
        # 3. steal
        #    + optional http_filter. Without http_filter, steal is 
        #    automatically filtered by baggage and key
    ignore_ports: [9091, 15090]
    run: 
      type: container 
      command: ["docker", "run", "-p", "8080:8080", "-p", "8443:8443", "user-auth-service:latest"]
```

Then, run `mirrord up`. For each entry in `services`, a mirrord config will be generated and a corresponding session will be spun up. The sessions all run in the foregound, in parallel, and are stopped whenever any one of them exits.

### Supported fields

#### `defaults`
Common configuration options, applied to all defined services. Currently 3 options are supported:
- `accept_invalid_certificates`
- `operator`
- `telemetry`

All 3 map directly to their `mirrord.json` counterparts.

#### `services`
A map from service ids to a `ServiceConfig`. Each entry in this map defines and configures a mirrord process that will be run as part of the session.

##### `services.*.target`
Specifies the target of the session. Has 2 fields: `path` and `namespace`, which map directly to their `mirrord.json` counterparts.

##### `services.*.env`
Specifies the environment variable configuration for the given service. Maps directly (1:1) to `feature.env`

##### `services.*.mode`
Specifies the incoming network mode of the service. Has 3 available options:
- `mirror`: Incoming traffic is mirrored
- `steal`: Incoming traffic is stolen
- `replace`: Incoming traffic is stolen and the target is copied and scaled down.

##### `services.*.http_filter`
Specifies the HTTP filtering configuration for the given service. Maps directly to `feature.network.incoming.http_filter`

##### `services.*.ignore_ports` 
List of ports that should be ignored in incoming traffic. Maps directly to `feature.network.incoming.ignore_ports`

##### `services.*.messages` 
Specifies queue splitting configuration (specifics undecided as of now).

##### `services.*.run` 
Specifies the command that should be run with mirrord. Has 2 fields:
- `command`: Array of strings containing the command to be run and its CLI arguments.
- `type`: can be either `exec` or `container`, defaults to `exec`. Specifies how mirrord should be run (i.e. with `mirrord exec` or `mirrord container`)


### CLI args

#### `-f`, `--config-file`
Allows specifying a different config file, e.g. `mirrord up -f mirrord-up2.yaml`

#### `--key`
Allows specifying a custom session key. When not supplied, the OS username is used.


## Reference-level explanation (MVP)
[reference-level-explanation]: #reference-level-explanation

`mirrord up` works by parsing `mirrord-up.yaml` and generating a mirrord config for each of the defined services. Notably, no actual `mirrord.json` files are generated, instead a `LayerConfig` is created for each service and serialized into an environment variable, much like how the CLI passes the resolved config to the intproxy and layer.

Once the configs have been generated, a mirrord process is started for each service. The CLI captures logs from all of them and prints them to the console. All child processes are killed when one of them exits or the parent CLI process receives `SIGINT`.


## Prior art
[prior-art]: #prior-art

### Monday
I'm not too familiar with the specifics here but Monday has an ad-hoc solution for solving this same problem. It's worth looking deeper into it to ensure `mirrord up` covers all those use cases.

### Docker Compose
While not directly related to mirrord, the shape of the feature described in this RFC is very similar to that of Docker Compose. We may use it as a reference with regards to UX issues.


## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Queue splitting: how should the configuration be done in `mirrord-up.yaml`? Ideally we want something that's simpler than a 1:1 mapping with `feature.split_queues` in `mirrord.json`, but as of now it's unclear how that could/should be done.

- Interaction with the services: right now we just print stdout and stderr to the CLI (prefixed with the service ids), but this is a little ugly and does not allow interacting with stdin.


## Future possibilities
[future-possibilities]: #future-possibilities

### Background / Detached mode
Post-MVP, we should add a detached mode that functions similarly to `docker compose -d`, i.e. the session detaches from the current terminal and runs independently in the background. The toplevel CLI process should act as a daemon and expose a local API over a unix socket/named pipe to allow controlling/killing the session and extracting information (logs, status, etc).

### `mirrord up init` / Automatic generation of `mirrord-up.yaml`
This command should scan the project directory, look for common manifests, try to extract service definitions and (interactively) generate a `mirrord-up.yaml`. Sources that we should consider looking into:
- `package.json`
- `CMD/ENTRYPOINT` from Dockerfiles and similar
- `docker-compose.yaml`

This can also run automatically whenever no `mirrord-up.yaml` is found.



