---
title: "The Death of the Handshake: Inside MCP's Stateless Evolution and MRTR"
description: "An architectural deep-dive into the Model Context Protocol (MCP) 2026-07-28 specification. Discover how stateless execution and Multi-Round-Trip Requests (MRTR) solve cloud scaling, load balancing, and firewall challenges."
published-at: "https://www.linkedin.com/pulse/death-handshake-inside-mcps-stateless-evolution-mrtr-stanislav-deviatov/"
author: "Stanislav Deviatov"
date: "Jul-2026"
language: "en"
---

# The Death of the Handshake: Inside MCP's Stateless Evolution and MRTR

## 1. Introduction: The Production Pivot

When the Model Context Protocol (MCP) emerged across 2024 and 2025, its primary achievement was standardization. Early specification milestones, most notably version `2024-11-05` and the subsequent `2025-11-25` Tasks release, succeeded in establishing a universal language for Large Language Model (LLM) agents, tools, and resources. However, early specifications still reflected the architectural patterns of local desktop integrations and stateful standard input/output (`stdio`) pipelines.

As organizations migrated MCP workloads from desktop IDEs to enterprise cloud infrastructure, a systemic bottleneck emerged: **session coupling**. In traditional MCP deployments, a client and server had to negotiate a stateful lifecycle initiated by an explicit `initialize` / `initialized` handshake. The server assigned a persistent session identity (`Mcp-Session-Id`), and all subsequent tool invocations assumed an unbroken, stateful connection.

In distributed production environments (characterized by ephemeral container scaling such as AWS Lambda and Google Cloud Run, reverse proxies, load balancers, and aggressive HTTP/2 multiplexing), stateful session coupling proved fragile. Load balancers routinely severed idle connections, round-robin routers dropped requests lacking sticky session affinity, and horizontal scale-out forced servers to maintain expensive distributed session caches.

The Model Context Protocol specification `2026-07-28` addresses this fundamental architectural friction. It represents a paradigm shift from **stateful session connections** to **stateless, self-contained message processing**. Furthermore, it re-engineers human-in-the-loop and dynamic execution workflows via **Multi-Round-Trip Requests (MRTR)**, eliminating the networking hazards of bidirectional server-to-client callbacks.

This article provides an exhaustive architectural breakdown of the `2026-07-28` specification. We will dissect the removal of protocol handshakes, explore the mechanics of Multi-Round-Trip Requests, analyze the deprecation registry, and evaluate practical strategies for production readiness.

> [!NOTE]
> For a complete chronological analysis of earlier specification milestones leading up to this release, our companion guide [MCP Version History. The Evolution of AI Connectivity](mcp-version-history-dec-2025.md) has been fully actualized to include the `2026-07-28` specification.

---

## 2. The Battle Against State: Why Handshakes Fail at Scale

To understand the necessity of the `2026-07-28` redesign, we must examine the failure modes of the legacy stateful lifecycle when deployed behind enterprise edge proxies.

### 2.1 The Fragility of Session Coupling

In the legacy protocol flow, an MCP client could not invoke a tool until it completed a multi-step ceremony:

1.  Client sends `initialize` with its protocol version and client capabilities.
2.  Server returns an `InitializeResult` acknowledging capabilities and assigning session state.
3.  Client sends an `initialized` notification.
4.  Subsequent requests carry implicit state or rely on an `Mcp-Session-Id` header across HTTP/SSE transport channels.

```
Legacy Stateful Flow (Pre-2026-07-28)
Client                               Server
  |---- initialize (Capabilities) ---->|  [Allocates Session State]
  |<--- InitializeResult --------------|
  |---- initialized ------------------>|
  |---- tools/call (Session ID) ------>|  [Requires Sticky Session]
```

When operating across horizontally scaled clusters, this architecture introduced severe operational constraints:

*   **Sticky Session Dependency:** If a request arrived at Worker Node B while the session state resided in Worker Node A's local memory, the call failed with a session mismatch or required complex distributed caching (e.g., Redis cluster lookups on every JSON-RPC frame).
*   **Connection Dropping:** Reverse proxies and API gateways typically enforce idle connection timeouts. If an agent paused for reasoning or awaited external input, the underlying connection terminated, invalidating the session and requiring a complete re-initialization sequence.

### 2.2 Self-Contained Requests and the `_meta` Envelope

Version `2026-07-28` removes the `initialize` / `initialized` handshake entirely. The concept of an explicit session lifecycle is eliminated. Instead, every JSON-RPC request is architected to be **stateless and self-contained**.

To achieve stateless execution without sacrificing protocol negotiation, the specification leverages the request-level `_meta` property. Every request carries the client's identity, protocol version, and active capabilities inside `_meta`:

```json
{
  "jsonrpc": "2.0",
  "id": 101,
  "method": "tools/call",
  "params": {
    "name": "execute_query",
    "arguments": {
      "statement": "SELECT * FROM financial_ledger WHERE status = 'PENDING'"
    },
    "_meta": {
      "protocolVersion": "2026-07-28",
      "client": {
        "name": "EnterpriseAuditorAgent",
        "version": "3.1.0"
      },
      "capabilities": {
        "experimental": {},
        "sampling": {}
      }
    }
  }
}
```

```
Modern Stateless Flow (2026-07-28+)
Client                               Server Cluster (Node A, B, C)
  |---- tools/call + _meta ------------>|  [Any stateless node processes]
  |<--- CallToolResult -----------------|  [Zero session state required]
```

By embedding execution context directly within each message envelope, the protocol decouples processing from connection persistence. Any stateless worker container behind a standard load balancer can immediately inspect `_meta.protocolVersion` and process the payload without querying a central session registry.

---

## 3. Multi-Round-Trip Requests (MRTR): Unidirectional Orchestration

A critical architectural friction point in previous MCP specifications was the reliance on **bidirectional communication**. Under earlier standards, if a tool execution required additional context from the client (such as user confirmation, sampling an LLM, or eliciting dynamic parameters), the server initiated a reverse JSON-RPC request upstream over the active connection.

### 3.1 The Security and Networking Cost of Bidirectionality

Bidirectional server-initiated requests introduced three major engineering risks:

1.  **Firewall and Ingress Blocks:** Enterprise firewalls and Zero-Trust networks readily allow outbound client requests but strictly scrutinize or block reverse server-initiated streams.
2.  **Transport Deadlocks:** On synchronous transports or single-threaded execution pools, a server waiting on a reverse callback could easily deadlock if the client was concurrently waiting for the server's HTTP response.
3.  **Stateful Routing Complexity:** Handling server-to-client requests required keeping full duplex transport channels (such as WebSockets or persistent SSE pipelines) permanently open.

### 3.2 Wire-Level Mechanics of SEP-2322

The `2026-07-28` specification solves this via **Multi-Round-Trip Requests (MRTR)**, standardized under **SEP-2322**. MRTR transforms all interactive workflows into a strictly **unidirectional, client-driven request/retry loop**.

When a tool requires external input or user elicitation, the server pauses its execution and returns an immediate response with `resultType: "input_required"`. This payload includes two critical elements:

*   `inputRequests`: An array of structured prompts or parameter requests that the client must satisfy.
*   `requestState`: An opaque, base64-encoded or encrypted state token generated by the server encapsulating the frozen execution context.

#### Step 1: Initial Client Request
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "deploy_release",
    "arguments": {
      "environment": "production",
      "version": "v4.2.0"
    },
    "_meta": {
      "protocolVersion": "2026-07-28"
    }
  }
}
```

#### Step 2: Server Returns Input Required (MRTR Pause)
Instead of executing a reverse callback, the server returns a terminal HTTP 200 response containing the `input_required` payload:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "input_required",
    "requestState": "eyJ0YXNrSWQiOiJkZXAtOTkwMSIsInN0ZXAiOiJhdXRoX2NoZWNrIn0=",
    "inputRequests": [
      {
        "id": "mfa_token",
        "type": "text",
        "title": "MFA Authorization Required",
        "description": "Provide the hardware security key token to deploy to production."
      }
    ]
  }
}
```

#### Step 3: Client Retries with Input Responses
Once the client obtains the necessary input from the user or secure store, it re-issues the `tools/call` request. It attaches the original `requestState` along with an `inputResponses` map:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "deploy_release",
    "arguments": {
      "environment": "production",
      "version": "v4.2.0"
    },
    "requestState": "eyJ0YXNrSWQiOiJkZXAtOTkwMSIsInN0ZXAiOiJhdXRoX2NoZWNrIn0=",
    "inputResponses": {
      "mfa_token": "849201"
    },
    "_meta": {
      "protocolVersion": "2026-07-28"
    }
  }
}
```

```
Multi-Round-Trip Requests (MRTR) Execution Pattern
Client                                           Server
  |---- tools/call ------------------------------->|
  |<--- resultType: "input_required" + state ------|  [Connection Closes]
  |     (User resolves input request)              |
  |---- tools/call (retry + state + responses) --->|  [Resumes Execution]
  |<--- CallToolResult (Completed) ----------------|
```

This design guarantees that all network traffic remains strictly client-initiated. It eliminates duplex deadlocks and ensures that state can be either serialized securely into `requestState` or stored in a persistent backend, allowing any server node in the cluster to resume the paused operation.

---

## 4. Protocol Governance: Feature Lifecycle & Strategic Deprecations

As protocols mature, shedding historical complexity becomes as important as adding capabilities. The `2026-07-28` specification introduces formal governance rules and streamlines the protocol footprint by deprecating redundant subsystems.

### 4.1 The 12-Month Feature Lifecycle Policy

To prevent ecosystem fragmentation across tool developers and LLM providers, version `2026-07-28` establishes a standardized **Feature Lifecycle Policy**:

*   **Experimental Stage:** Features introduced under experimental namespaces may change rapidly.
*   **Stable Stage:** Core protocol specifications maintain backward compatibility across minor revisions.
*   **Deprecated Stage:** Once a feature is marked deprecated in an official specification release, maintainers are guaranteed a **minimum 12-month deprecation lifecycle** before removal. Servers and clients must continue supporting deprecated capabilities during this window while emitting deprecation warnings.

### 4.2 Deconstructing the Deprecated APIs

The core subsystems formally deprecated in the `2026-07-28` specification and their modern architectural replacements are detailed below:

*   **Roots API (`roots/list`)**
    *   **Architectural Rationale:** Managing virtual filesystem boundaries via bidirectional synchronization created high latency and tight coupling between client and server storage layers.
    *   **Modern Replacement (2026-07-28+):** Explicit file content or path parameters passed directly into tool arguments.

*   **Sampling API (`sampling/createMessage`)**
    *   **Architectural Rationale:** Server-initiated LLM sampling loops violated statelessness, caused network ingress friction, and complicated billing and token governance.
    *   **Modern Replacement (2026-07-28+):** Client-managed LLM orchestration or structured tool responses that prompt the client agent to perform further reasoning.

*   **Logging API (`logging/setLevel`)**
    *   **Architectural Rationale:** Maintaining a custom protocol-level log transport duplicated mature enterprise observability infrastructure and added unnecessary protocol overhead.
    *   **Modern Replacement (2026-07-28+):** Standard `stderr` output streams or structured OpenTelemetry (OTel) log exporters.

*   **Dynamic Client Registration (RFC 7591)**
    *   **Architectural Rationale:** On-the-fly OAuth 2.0 dynamic registration introduced a significant attack surface and complicated edge authentication workflows.
    *   **Modern Replacement (2026-07-28+):** Static or dynamic **Client ID Metadata Documents** served via secure HTTPS URLs.

---

## 5. Performance Engineering: Caching & Determinism

In enterprise agent architectures, tool definitions and schema discovery often account for a substantial percentage of total LLM prompt tokens. Version `2026-07-28` introduces targeted optimizations to maximize KV-cache reuse across modern inference engines.

### 5.1 Deterministic Tool Lists and Prompt Caching

Previous specifications permitted servers to return tool lists (`tools/list`) in arbitrary or non-deterministic order. When serialized into LLM system prompts, any reordering of JSON tool schemas invalidated pre-computed prompt KV-caches, triggering full token re-evaluation.

Version `2026-07-28` mandates deterministic, stable ordering for all list endpoints (`tools/list`, `resources/list`, `prompts/list`). By ensuring byte-for-byte identical JSON serialization across identical server states, client frameworks can achieve high prompt cache hit ratios, reducing inference latency and cost.

### 5.2 The `CacheableResult` Interface

To further optimize network and token overhead, read-only and idempotent operations can implement the `CacheableResult` contract. Servers can augment response payloads with explicit cache governance fields:

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{"status": "OPERATIONAL", "uptime": "99.99%"}"
      }
    ],
    "ttlMs": 300000,
    "cacheScope": "public"
  }
}
```

*   `ttlMs`: Time-to-live in milliseconds, instructing the client or gateway how long the result remains valid.
*   `cacheScope`: Dictates whether the result is `public` (shareable across multiple client agents or tenants in a gateway) or `private` (restricted to the specific caller).

---

## 6. Strategic Recommendations for System Architects

For engineering teams building or migrating MCP implementations on top of the `2026-07-28` specification, we recommend four immediate architectural adjustments:

1.  **Decouple Execution Nodes from Session Storage:** Eliminate local memory dependencies for tool execution. Ensure that any long-running or interactive operation serializes its continuation requirements into `requestState` or persists state to a distributed store (e.g., PostgreSQL or Redis).
2.  **Adopt MRTR for All Human-in-the-Loop Operations:** Replace ad-hoc polling loops or bidirectional server callbacks with standard `InputRequiredResult` flows. This ensures compatibility across diverse client interfaces and simplifies network security policies.
3.  **Audit and Remove Deprecated API Usage:** Audit server codebases to phase out `sampling/*` and `roots/*` endpoints well before the 12-month lifecycle expiration. Transition logging pipelines directly to standard `stderr` or structured OpenTelemetry sinks.
4.  **Enforce Deterministic Serialization:** Verify that your MCP server frameworks serialize tools, arguments, and resource descriptions in a strictly consistent, sorted order to take full advantage of LLM prompt caching.

---

## 7. Conclusion

The Model Context Protocol `2026-07-28` specification represents a pivotal shift from an IDE-centric integration protocol to an enterprise-grade distributed standard. By removing the stateful `initialize` handshake, standardizing stateless execution envelopes via `_meta`, and introducing Multi-Round-Trip Requests (MRTR), the protocol eliminates its most significant scaling bottlenecks.

For enterprise architects, these refinements deliver immediate benefits: zero-downtime serverless deployments, robust ingress-friendly security topologies, and predictable token cache economics. As the AI ecosystem converges on MCP as the primary integration layer between LLMs and external systems, version `2026-07-28` provides the resilient, cloud-native foundation required for mission-critical autonomous agents.
