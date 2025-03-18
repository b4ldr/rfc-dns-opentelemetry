# DNS Tracing Schema for OpenTelemetry with Zone Transfer Support

**Authors**: [Your Name]  
**Date**: [Date]

**Status**: Draft

## Abstract

This RFC proposes a custom tracing schema for DNS software based on OpenTelemetry, with an extension to cover DNS zone transfer operations (e.g., AXFR, IXFR). It defines a set of standard attributes, spans, and tracing concepts to capture telemetry data from DNS clients, resolvers, authoritative servers, and zone transfer processes. The goal is to enhance observability for DNS operations, monitor zone transfers, troubleshoot issues, and gain insights into DNS traffic patterns. The proposed schema extends OpenTelemetry to support DNS query handling and zone transfer events.

---

## 1. Introduction

DNS software plays a key role in the functioning of the internet by resolving domain names into IP addresses and ensuring reliable propagation of DNS records. DNS zone transfers are an essential part of this process, where primary DNS servers send zone data to secondary servers to maintain consistency. These transfers, such as AXFR (full zone transfer) or IXFR (incremental zone transfer), are critical for maintaining DNS data across distributed servers.

As DNS operations grow in complexity and scale, observability becomes crucial. OpenTelemetry provides a framework for collecting and analyzing traces, metrics, and logs. This RFC proposes an extension to OpenTelemetry’s tracing schema, specifically designed to support DNS query processing and zone transfer operations, enabling better monitoring, troubleshooting, and performance insights.

## 2. Goals and Non-goals

### Goals:
- Define a custom tracing schema for DNS software with support for zone transfers.
- Improve observability of DNS query resolution and zone transfer operations.
- Enable seamless troubleshooting of DNS issues, including query performance, zone propagation delays, and failures in zone transfers.
- Provide a standard format for capturing DNS-related telemetry that can integrate with existing OpenTelemetry-compatible tools and systems.

### Non-goals:
- Define telemetry schema for other types of observability data like metrics or logs.
- Provide deep integration with DNS-specific protocols outside the scope of zone transfers (e.g., DNSSEC, DoH).

## 3. DNS Tracing Concept

DNS tracing tracks the lifecycle of DNS queries, including zone transfer events. Each trace will consist of **spans** that represent individual operations in both DNS query resolution and zone transfer processes.

- **Trace**: Represents the complete lifecycle of a DNS operation, including query resolution and zone transfer.
- **Span**: Represents a specific operation, such as DNS query handling, DNS zone transfer initiation, or DNS response processing.
- **Context**: Contains metadata about the query or transfer, including zone names, source/destination IPs, query types, and error codes.

## 4. Key Tracing Concepts for DNS Software (Including Zone Transfers)

### Span Attributes:
- **`dns.query_name`**: The domain name being queried (e.g., `www.example.com`).
- **`dns.query_type`**: The type of DNS query (e.g., `A`, `AAAA`, `MX`, `CNAME`).
- **`dns.query_class`**: The class of the DNS query (usually `IN` for Internet).
- **`dns.source_ip`**: The IP address of the DNS client making the request.
- **`dns.destination_ip`**: The IP address of the DNS server (could be a recursive resolver or authoritative server).
- **`dns.resolver_type`**: Indicates if the query was handled by a recursive resolver or an authoritative DNS server.
- **`dns.query_latency`**: The time taken for the query to be resolved (from submission to response).
- **`dns.response_code`**: The response code returned by the DNS server (e.g., `NOERROR`, `NXDOMAIN`, `SERVFAIL`).
- **`dns.error_message`**: If an error occurs (e.g., `timeout`, `connection refused`), the error message should be recorded.
- **`dns.cache_hit`**: Indicates whether the DNS query was resolved from a cache (boolean value).
- **`dns.is_recurse`**: Indicates if the DNS query was handled by a recursive resolver (boolean value).
- **`dns.ttl`**: The time-to-live (TTL) value of the DNS record returned in the response.
- **`dns.zone_transfer_type`**: The type of zone transfer (e.g., `AXFR` or `IXFR`).
- **`dns.zone_transfer_status`**: The status of the zone transfer (e.g., `SUCCESS`, `FAIL`, `TIMEOUT`).
- **`dns.zone_name`**: The name of the zone being transferred (e.g., `example.com`).
- **`dns.zone_transfer_latency`**: The latency involved in transferring the zone.
- **`dns.zone_transfer_source_ip`**: The source IP of the DNS server initiating the zone transfer (usually the primary DNS server).
- **`dns.zone_transfer_destination_ip`**: The destination IP of the secondary DNS server receiving the zone data.

### Trace Structure:

- **Root Span**: Represents the DNS query initiation or the initiation of a zone transfer.
- **Child Spans**: These include spans for DNS resolver steps, zone transfer events, or responses from authoritative servers.
- **End-to-End Flow**: Traces will include spans for each DNS query resolution step and zone transfer phase (initiation, transfer, and completion).

### Example of a DNS Query and Zone Transfer Trace:

- **Trace 1** (DNS Query for `www.example.com`):
  - **Span 1**: `DNS_Query_Sent` (Root Span)
    - Attributes:
      - `dns.query_name: "www.example.com"`
      - `dns.query_type: "A"`
      - `dns.source_ip: "192.168.1.10"`
      - `dns.destination_ip: "8.8.8.8"`
      - `dns.query_latency: 2ms`
  - **Span 2**: `DNS_Resolver_Hop`
    - Attributes:
      - `dns.query_name: "www.example.com"`
      - `dns.query_type: "A"`
      - `dns.query_latency: 10ms`
  - **Span 3**: `DNS_Response_Received`
    - Attributes:
      - `dns.query_name: "www.example.com"`
      - `dns.response_code: "NOERROR"`
      - `dns.cache_hit: true`
      - `dns.query_latency: 5ms`

- **Trace 2** (Zone Transfer for `example.com`):
  - **Span 1**: `Zone_Transfer_Init` (Root Span)
    - Attributes:
      - `dns.zone_name: "example.com"`
      - `dns.zone_transfer_type: "AXFR"`
      - `dns.zone_transfer_source_ip: "192.168.1.1"`
      - `dns.zone_transfer_destination_ip: "192.168.1.2"`
      - `dns.zone_transfer_status: "STARTED"`
  - **Span 2**: `Zone_Transfer_Processing`
    - Attributes:
      - `dns.zone_name: "example.com"`
      - `dns.zone_transfer_latency: 100ms`
      - `dns.zone_transfer_status: "IN_PROGRESS"`
  - **Span 3**: `Zone_Transfer_Completed`
    - Attributes:
      - `dns.zone_name: "exa

