# Best Practices on deploying DSP-Endpoints

DSP spec states
that [authorization](https://eclipse-dataspace-protocol-base.github.io/DataspaceProtocol/HEAD/#authorization) is
optional. However, participants usually require their endpoints to be protected and publicly reachable. Clients (usually
Consumers) have an interest to automatically discover DSP-endpoints and infer knowledge about their expected behavior.
For this, the metadata endpoint is particularly relevant because it hints to the Consumer what version of the DSP and
profiles to expect allowing the Consumer to terminate the interaction early.

Deployment of endpoints is a detail that is at the discretion of each participant. The DSP-spec accounts for this with
the concept of the `<base>` URL which can include subdomains and URL-segments. For example
`https://connector.mycorp.com/some/path` is a valid `<base>` URL. However, the spec places implicit constraints on it.
If precautions are not taken under certain circumstances, there are two effects that impede interoperability.

1. A DSP-endpoint is self-aware of its exposure location, because it needs to send protocol messages that include its
   own callback endpoints. Examples are the `ContractRequestMessage`'s `callbackAddress` property and the `endpointURL`
   the `Dataset`.
2. The `version` endpoint is bound to the root of the domain as per RFC8615. This is very much unlike the other DSP
   endpoints that are allowed to contain arbitrary URL-segments (both for the `<base>` and `/:callback` endpoints).

## Scenarios

### Scenario 1: Participant has single Agent serving single version

In this case, the subprotocols' endpoints (Catalog Protocol, Contract Negotiation Protocol, Transfer Process Protocol)
are all served from the same `<base>`. The component implementing the endpoints is self-contained and only needs its own
`<base>` as input to construct the `endpointURL` and `callbackAddress`. The metadata document can be hosted if the
application exposing the DSP-endpoint is deployed with direct access to the domain.

*`domain/.well-known/dspace-version` serving directions to the single DSP base endpoint.*

```json
{
  "protocolVersions": [
    {
      "version": "2025-1",
      "path": "/arbitrary/difference/between/domain/and/2025-1/endpoint/",
      "binding": "HTTPS"
    }
  ]
}
```

If the `CatalogService` is registered in the did document, the `service` entry looks like

```json
{
  "@context": [
    "https://w3id.org/dspace/2025/1/context.jsonld",
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "some:id",
  "type": "CatalogService",
  "serviceEndpoint": "https://<base>/catalog"
}
```

### Scenario 2: Participant has single Agent serving multiple versions

This scenario is similar to [scenario 1](#scenario-1-participant-has-single-agent-serving-single-version) with the
difference that two entries are returned to two separate `<base>` URLs - one for each protocol version.

- `domain/.well-known/dspace-version` serving directions to navigate to endpoints for specific versions.

```json
{
  "protocolVersions": [
    {
      "version": "2025-1",
      "path": "/arbitrary/difference/between/domain/and/2025-1/endpoint/",
      "binding": "HTTPS"
    },
    {
      "version": "2024-1",
      "path": "/arbitrary/difference/between/domain/and/2024-1/endpoint/",
      "binding": "HTTPS"
    }
  ]
}
```

If the `CatalogService` is registered in the did document, the `service` entry still looks like in scenario 1. The
Participant indicates that this is the preferred catalog endpoint version. As the term `CatalogService` is only defined
in the context of DSP-version `2025/1`, the Consumer should assume that it serves that. Regardless, the Client still has
the option to navigate back to the well-known endpoint, filter by their preferred version and navigate back.

```json
{
  "@context": [
    "https://w3id.org/dspace/2025/1/context.jsonld",
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "some:id",
  "type": "CatalogService",
  "serviceEndpoint": "https://<base>/catalog"
}
```

### Scenario 3: Participant has single DSP-endpoint behind an API-Gateway

API Gateways are infrastructure components that serve as single public entry point to an organization's internal
network which is by default not exposed to the public internet. An API Gateway thus incorporates multiple applications
and differentiates them via subpaths. Two example configuration rules of such an API gateway would be:

1. `https://gateway.mycorp.com/first-business-application/` redirects to `https://my-business-app.mycorp.com/api/`
2. `https://gateway.mycorp.com/dsp-participant-agent/` redirects to `https://dsp-participant-agent.mycorp.com/`

To the public internet, the well-known metadata endpoint would (if running in the Participant Agent itself)
be exposed to the public at`https://gateway.mycorp.com/dsp-participant-agent/.well-known/dspace-version` violating
RFC8615. Dataspace Participants would be required to configure an additional redirect:

3. `https://gateway.mycorp.com/.well-known/dspace-version` redirects to
   `https://dsp-participant-agent.mycorp.com/well-known/dspace-version`

The returned version endpoint information would be:

```json
{
  "protocolVersions": [
    {
      "version": "2025-1",
      "path": "/arbitrary/difference/between/domain/and/2025-1/endpoint/",
      "binding": "HTTPS"
    },
    {
      "version": "2024-1",
      "path": "/arbitrary/difference/between/domain/and/2024-1/endpoint/",
      "binding": "HTTPS"
    }
  ]
}
```

indicating paths that are relative to the domain of the URL this payload was served by. A client will assume that the
`2025-1` endpoint is available at `https://gateway.mycorp.com/arbitrary/difference/between/domain/and/2025-1/endpoint/`
which would be unavailable as the DSP-endpoints are registered with a first segment `dsp-participant-agent`. So
configuring an additional redirect rule (see rule 3) is not feasible.

In that case, the did doc would be

```json
{
  "@context": [
    "https://w3id.org/dspace/2025/1/context.jsonld",
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "some:id",
  "type": "CatalogService",
  "serviceEndpoint": "https://gateway.mycorp.com/dsp-participant-agent/arbitrary/difference/between/domain/and/2025-1/endpoint/catalog"
}
```

with the `serviceEndpoint` internally redirecting to
`https://dsp-participant-agent.mycorp.com/arbitrary/difference/between/domain/and/2025-1/endpoint/`
(see redirect rule 1) which succeeds.

Exposing a separate microservice implementing the metadata endpoint or a static document are options too. Their API
response would have to be aware of the Participant Agent's deployment on an API Gateway.

### Scenario 4: Participant has multiple DSP-endpoints behind an API-Gateway

