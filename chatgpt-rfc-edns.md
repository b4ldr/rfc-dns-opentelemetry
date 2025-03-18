# RFC: OpenTelemetry EDNS Extension for Distributed Tracing in DNS Transactions

## Status

This document proposes an **EDNS (Extension Mechanisms for DNS)** extension to carry **OpenTelemetry distributed tracing context** (trace IDs, span IDs, and other relevant metadata) during DNS transactions between upstream and downstream DNS servers.

## Abstract

Distributed tracing is essential for observing and troubleshooting modern distributed systems. DNS plays a vital role in service discovery and communication between services in such systems. However, existing DNS infrastructure does not support the propagation of tracing context like trace IDs or span IDs. This RFC proposes an **EDNS extension** to allow DNS queries and responses to carry OpenTelemetry tracing context, enabling end-to-end observability of DNS interactions within distributed systems.

## Motivation

In microservices architectures, DNS queries are frequently used for service discovery, communication, and inter-service coordination. However, DNS itself lacks native support for tracking the flow of requests across multiple services. Without distributed tracing, it becomes difficult to trace requests that pass through various DNS resolvers, which are critical in understanding performance bottlenecks, failures, and service dependencies in a distributed environment.

The goal of this RFC is to define an EDNS extension to propagate OpenTelemetry trace context within DNS queries and responses. This enables end-to-end observability and correlation of DNS lookups with the overall request/response lifecycle in distributed systems.

## Design Overview

This RFC proposes a new EDNS option specifically designed for **OpenTelemetry** distributed tracing. The OpenTelemetry context, such as trace IDs, span IDs, and related metadata, will be included as part of the DNS wire format through the EDNS option, which will be embedded in both DNS queries and responses.

### 1. **New EDNS Option for OpenTelemetry Context**

To propagate distributed tracing context within DNS transactions, we define a new **EDNS option**. This option will contain the following fields:

- **Trace ID**: A unique identifier for the trace.
- **Span ID**: A unique identifier for the span, representing a unit of work within the trace.
- **Parent Span ID**: The identifier for the parent span, to track the relationship between requests.
- **Sampling Decision**: A flag indicating whether the trace should be sampled.
- **Metadata**: Any additional relevant metadata (e.g., DNS query type, service name, etc.).

### 2. **EDNS Option Format**

The new EDNS option will include the tracing context as a part of the DNS packet. The **EDNS option** format for the tracing context is as follows:

| Field | Length (bytes) | Description |
|-------|----------------|-------------|
| Option Code | 2 bytes | The EDNS option code for OpenTelemetry tracing context (proposed value: `0x1234`). |
| Option Length | 2 bytes | The length of the option data (excluding the option code and length fields). |
| Trace ID | 16 bytes | The unique trace ID (16-byte value) that represents the entire trace. |
| Span ID | 8 bytes | The unique span ID (8-byte value) for the current operation. |
| Parent Span ID | 8 bytes | The ID of the parent span (8-byte value) if present. |
| Sampling Decision | 1 byte | A flag indicating whether this trace is sampled (0 = no, 1 = yes). |
| Metadata | Variable | Any additional context or metadata as a variable-length field (e.g., service name, query type, etc.). |

#### Option Code
- The **Option Code** for the OpenTelemetry tracing extension will be assigned as `0x1234` (or another value upon IANA assignment).

#### Option Length
- The **Option Length** specifies the total length of the tracing context data that follows (excluding the Option Code and Option Length fields themselves).

#### Trace ID, Span ID, and Parent Span ID
- These fields carry the distributed tracing context. The **Trace ID** (16 bytes) is the unique identifier for the trace, while the **Span ID** (8 bytes) identifies the specific unit of work. The **Parent Span ID** (8 bytes) is used to maintain the parent-child relationships between spans.

#### Sampling Decision
- The **Sampling Decision** (1 byte) indicates whether this trace should be sampled for observability. A value of `1` means sampled, while `0` means not sampled.

#### Metadata
- **Metadata** can be additional context information such as the DNS query type (e.g., `A`, `AAAA`, `MX`) or service-specific data, which may help with more granular trace analysis.

### 3. **DNS Query and Response Format**

The **EDNS** extension will be added to DNS queries and responses to propagate the OpenTelemetry tracing context. The standard DNS wire format will be modified to include the new EDNS option.

#### DNS Query with EDNS Trace Context

When a DNS client sends a query, the trace context will be included as part of the EDNS option in the query packet:


```plaintext
DNS Query:
  - Name: example.com
  - Type: A (or other record type)
  - EDNS Option (OpenTelemetry Trace Context):
      Option Code: 0x1234
      Option Length: N (size of the context)
      Trace ID: abc123... (16 bytes)
      Span ID: xyz456... (8 bytes)
      Parent Span ID: pqr789... (8 bytes)
      Sampling Decision: 1
      Metadata: service_name=ServiceA, query_type=A
```


#### DNS Response with EDNS Trace Context

When the DNS resolver (downstream daemon) processes the query and returns a response, it will include the same trace context in the response:

```plaintext
DNS Response:
  - Name: example.com
  - Type: A (or other record type)
  - EDNS Option (OpenTelemetry Trace Context):
      Option Code: 0x1234
      Option Length: N (size of the context)
      Trace ID: abc123... (16 bytes)
      Span ID: def789... (8 bytes)
      Parent Span ID: xyz456... (8 bytes)
      Sampling Decision: 1
      Metadata: service_name=ServiceB, query_type=A
```

### 4. **Interaction Between Upstream and Downstream DNS Servers**

1. **Upstream DNS Server (Initiating Query)**:
   - When an upstream DNS server (or DNS client) initiates a DNS query, it attaches the tracing context as part of the EDNS option in the query packet.
   - This context is sent to the downstream DNS resolver.

2. **Downstream DNS Resolver**:
   - Upon receiving the DNS query, the downstream DNS resolver extracts the tracing context from the EDNS option.
   - It then performs the DNS resolution and includes the same trace context in the response, which may be updated (for example, a new span ID is generated for the resolution step).
   - If the resolver forwards the query to another DNS server, it will include the same trace context, preserving the trace continuity.

3. **Trace Propagation**:
   - This ensures that DNS interactions are consistently correlated with the overall system trace, allowing for end-to-end visibility.

---

## Example of EDNS Trace Context in DNS Transactions

Consider the case where **Service A** queries a DNS resolver to resolve `example.com`:

1. **Service A (Client)** sends a DNS query to resolve `example.com` and includes the trace context in the EDNS extension.

2. The **DNS Resolver** (downstream) receives the query, processes it, and adds its own span to the trace, then responds with the trace context intact.

3. If needed, the **DNS Resolver** forwards the query to another resolver or authoritative server, preserving the tracing context in the EDNS extension.

---

## Security Considerations

- **Privacy**: DNS queries and responses may contain sensitive data (e.g., domain names). Therefore, the trace context should be transmitted over secure channels (e.g., DNS over HTTPS or DNS over TLS) to prevent data leakage.
- **Integrity**: The trace context should be validated to ensure that it has not been tampered with in transit. Signed DNS responses (DNSSEC) could help ensure integrity, and any traces should be correlated securely.
- **Data Sensitivity**: If tracing metadata includes sensitive information, it should be encrypted or obfuscated as needed.

---

## Backward Compatibility

This extension is backward compatible with DNS systems that do not support the EDNS OpenTelemetry tracing context. Servers and clients that do not recognize the new EDNS option will simply ignore it and continue processing DNS queries and responses as usual. However, OpenTelemetry-enabled systems will be able to take full advantage of the distributed tracing context.

---

## Conclusion

This RFC defines a new EDNS extension to propagate **OpenTelemetry tracing context** in DNS transactions, enabling **end-to-end observability** of DNS queries and responses in distributed systems. By embedding trace IDs, span IDs, and other metadata into DNS queries and responses, this extension will allow for better tracking, troubleshooting, and performance monitoring of DNS interactions within modern service-oriented architectures.

---

### Authors

- [Author Name]
- [Author Email]
- [Date]

---

