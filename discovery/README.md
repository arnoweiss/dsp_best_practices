# Discovery of DSP endpoints in Practice

Participants usually require their endpoints to be protected and publicly reachable. Clients (usually
Consumers) have an interest to automatically discover DSP-endpoints and infer knowledge about their expected behavior.
For this, the [metadata endpoint](https://eclipse-dataspace-protocol-base.github.io/DataspaceProtocol/2025-1-RC4/#exposure-of-dataspace-protocol-versions) 
is particularly relevant because it hints to the Consumer what version of the DSP and profiles to expect, allowing the 
Consumer to terminate the interaction early if there's no commonly understood protocol.

Deployment of endpoints is a detail that is at the discretion of each participant. That's why it's necessary to
differentiate URLs and their relationship with each other.

- `<domain>` is the naked domain without URL segments or fragments.
- `<root>` is the URL on which the `/.well-known/dspace-version` endpoint is hosted. It should be equivalent to 
`<domain>` to satisfy RFC8615. If it does, clients can always find the metadata endpoint. Else, a Dataspace should
have conventions to enable discovery of that endpoint in a different manner - for instance by [registering a 
`DataService`](https://eclipse-dataspace-protocol-base.github.io/DataspaceProtocol/2025-1-RC4/#example-data-service-did-service-example) 
in their did document.
- `<base>` is the URL on which the protocol endpoints are hosted. These endpoints host specific profiles and versions
of the DSP. They are constructable from the metadata endpoint by concatenating `<root>` with the `path` property fetched
from the document.

A DSP-endpoint must be self-aware of its exposure location, because it needs to send protocol messages that include its
own callback endpoints. Examples are the `ContractRequestMessage`'s `callbackAddress` property and the `endpointURL`
the `DataService` exposes. Both serve the `<base>` URL of the relevant services. In deployment scenarios where the
DSP-endpoints are proxied through the network, the application logic must be made aware of this as it impacts the
payloads it will emit.
