---
title: "OAuth Transaction Tokens Best Current Practice"
category: info

docname: draft-araut-oauth-transactiontokens-bcp-00
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  github: ashayraut/oauth-transactiontokens-best-current-practice
  latest: https://example.com/LATEST

author:
 -
    fullname: ASHAY RAUT
    organization: Amazon
    email: asharaut@amazon.com

normative:

informative:

...

--- abstract

This document provides best current practices for implementing and deploying OAuth 2.0 Transaction Tokens as specified in draft-ietf-oauth-transaction-tokens. Transaction Tokens (Txn-Tokens) enable workloads in a trusted domain to preserve and propagate user identity and authorization context across service boundaries during the processing of external programmatic requests. This BCP addresses practical deployment considerations including token service architecture, size management, propagation patterns, validation strategies, and operational monitoring that are essential for secure and effective implementation in production environments.


--- middle

# Introduction

## Background
Modern distributed systems built on microservice architectures face a fundamental challenge: maintaining security context as requests traverse multiple service boundaries. When an external actor initiates an API request, the user identity and authorization context must be preserved and made available to all downstream internal microservices involved in processing that request. Without a standardized mechanism, organizations resort to ad-hoc solutions that introduce security vulnerabilities, operational complexity, and interoperability challenges.

The OAuth 2.0 Transaction Tokens specification (draft-ietf-oauth-transaction-tokens) addresses this challenge by defining a token format and exchange protocol that enables secure context propagation across internal microservices within trusted domains. However, the specification focuses on protocol mechanics rather than deployment practices. Real-world implementations face additional challenges including latency constraints, token size limitations, schema evolution, propagation reliability, and operational monitoring.

## Purpose of the BCP
This Best Current Practice document provides implementers with guidance derived from production deployments of Txn-Token systems. It addresses practical considerations that fall outside the scope of the protocol specification but are critical for successful deployment. The recommendations in this document are based on operational experience with large-scale microservice environments where hundreds of internal microservices must coordinate security context propagation across complex call chains.

This BCP is intended for:
- Organizations implementing Transaction Token Services
- internal microservice developers integrating Txn-Token support
- Security architects designing authorization systems
- Operations teams monitoring token propagation


# Conventions and Definitions

{::boilerplate bcp14-tagged}
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.
This document uses terminology from draft-ietf-oauth-transaction-tokens including "Transaction Token" (Txn-Token), "Transaction Token Service", "trusted domain", and "authorization context".


# Best Current Practices
## Transaction Token Service Implementation
### Service Architecture
Organizations SHOULD setup Transaction Token Service (TTS) which hosts functionality to issue token, replace token and other Txn-Token related functionality. Organizations SHOULD prefer architectures where the authorization service that authenticates and authorizes external actors, invokes the TTS for getting Transaction Token as part of authentication and authorization requests and pass Txn-token as well with it. This way, it avoids an explicit calls from external endpoint to TTS and lesser code changes in external services. Additionally, all internal microservices that want to replace tokens SHOULD connect directly to TTS. This architecture strikes balance between the options to either have authorization service host all TTS functionality or external services needing to connect TTS.

### Context Selection
Transaction Token Services MUST include all mandatory claims defined in draft-ietf-oauth-transaction-tokens. However, services SHOULD NOT include all optional contexts by default. Optional contexts such as transaction context (tctx) or custom claims MUST be added only when explicitly requested.

Organizations SHOULD provide client libraries that offer interfaces for requesting Txn-Tokens with specific optional contexts. This approach prevents token bloat while ensuring that services can obtain the context they require. When an optional context cannot be added due to parsing errors, format violations, or unavailability, the Transaction Token Service MUST NOT fail the token issuance. Instead, it SHOULD issue the token without that specific context and MAY log the condition for operational monitoring.

## Token Size Management
### Size Limits
Transaction Token Services MUST NOT issue tokens larger than 4KB. While HTTP specifications do not mandate maximum header sizes, common web server implementations impose limits to prevent Denial of Service attacks. Apache defaults to 8KB maximum header size, but organizations must account for other headers in the same request. A 4KB limit for Txn-Tokens provides reasonable headroom while preventing operational issues.

Organizations SHOULD implement monitoring on token size to detect trends toward the limit. Services that consistently approach size limits indicate either excessive context inclusion or the need for context relocation strategies.

### Context Relocation
When authorization context exceeds 4KB, Transaction Token Services SHOULD implement a relocation endpoint. The service stores oversized contexts in a separate data store using the Txn-Token identifier as the primary key. The Txn-Token itself contains only a reference to the relocated context.

Client libraries for token validation SHOULD transparently handle context relocation. When an internal microservice requests a context that has been relocated, the library fetches it from the relocation endpoint. This pattern mirrors Policy Information Points in Attribute-Based Access Control (ABAC) systems where additional attributes are retrieved at runtime.

Context relocation introduces additional latency and failure modes. Organizations SHOULD treat relocation as an exception rather than the normal case. Monitoring SHOULD track relocation frequency to identify services that consistently require excessive context.

## Token Lifetime and Expiration
Transaction tokens SHOULD have a time-to-live of less than 5 minutes. Organizations SHOULD determine appropriate lifetimes by working backward from latency Service Level Agreements (SLAs) defined for external endpoints.

Short token lifetimes reduce the window for token compromise and limit the impact of token mix-up scenarios. However, lifetimes must accommodate the longest expected call chains in the SOA. Organizations SHOULD measure actual request processing times and set token lifetimes to exceed the 99th percentile by a reasonable margin.

When tokens expire during request processing, services MUST NOT automatically request new tokens. Expired tokens indicate either excessively long call chains or performance problems that require investigation. Services SHOULD fail requests with expired tokens and emit telemetry for operational monitoring.

## Schema Governance
### Context Visibility
Organizations MUST govern the contexts added to Txn-Tokens. Once a context appears in a Txn-Token, it becomes visible to all services in the call chain. Services may develop dependencies on these contexts in ways not anticipated by the context provider. This phenomenon follows Hyrum's Law: with sufficient consumers, all observable behaviors of a system will be depended upon by somebody.

Before adding a new context to Txn-Tokens, organizations MUST consider the implications of making that context universally visible. If a context represents an identifier or concept known only to services early in the call chain, adding it to the Txn-Token exposes it to all downstream services. Those services may develop business logic dependencies on the context, not just security dependencies.

When the format or semantics of a widely-visible context must change, the organization faces a painful migration process. All services that depend on the context must be identified, updated, and deployed. Organizations SHOULD prefer adding new contexts with different names rather than changing existing contexts when semantic changes are required.

### Backward Compatibility
Organizations MUST implement backward compatibility tests for Txn-Token contexts. Automated tests SHOULD verify that changes to context format or structure do not break existing consumers. These tests SHOULD run as part of the continuous integration pipeline for the Transaction Token Service.

Backward compatibility testing becomes increasingly important as the number of services consuming Txn-Tokens grows. Without automated verification, format changes risk cascading failures across the SOA.

## Token Propagation
### Propagation Control
Organizations MUST prevent Txn-Tokens from propagating outside the trusted domain. While tokens contain encrypted sensitive data, organizations SHOULD implement explicit controls to block external propagation. Propagation libraries MUST detect when an internal microservice attempts to include a Txn-Token in a request to an external endpoint and MUST remove the token from that request.

This defense-in-depth approach protects against misconfiguration and implementation errors. Even if token encryption remains secure, preventing external propagation eliminates entire classes of potential vulnerabilities.

### Propagation Denylist
In addition to preventing external propagation, organizations SHOULD maintain a denylist of internal services that MUST NOT participate in token propagation. This addresses scenarios where specific services within the trusted domain should not receive or forward tokens due to architectural constraints, security requirements, or known incompatibilities.

#### Use Cases for Denylisting

- Services undergoing decommissioning that cannot be updated to handle tokens correctly
- Internal services that proxy to external endpoints (defense in depth)
- Services with known token handling bugs that cannot be immediately fixed
- Shared infrastructure services where token propagation creates unintended authorization coupling

#### Implementation

The denylist SHOULD be:
- Evaluated before any token propagation logic executes
- Configured at service startup (not changeable per-request)
- Checked case-insensitively against the service's canonical identifier
- Applied as a fail-safe — denylisted services return no-op results for all propagation operations

When a service is on the denylist, propagation libraries MUST:
- Not extract tokens from incoming requests
- Not attach tokens to outgoing requests
- Not generate placeholder tokens
- Log that propagation was skipped due to denylist membership

### Propagation Libraries
Organizations SHOULD provide standardized propagation libraries that handle token lifecycle within an internal microservice workload processing. These libraries MUST extract the Txn-Token from the incoming HTTP header, store it in request-scoped memory, add the token to outgoing request headers, and clear it from memory when request processing completes.

Standardized libraries provide several benefits. First, they enforce propagation controls including external blocking to avoid the token flowing outside your trust boundary. Second, they can be used to consistently emit telemetry about token initiation, propagation, and validation. Third, they provide a centralized point for implementing fallback behaviors when tokens are missing.

### Multi-Language Propagation Considerations
Different programming languages have fundamentally different concurrency models that affect propagation library design. Organizations supporting polyglot architectures MUST account for these differences.

#### Language-Specific Token Storage

| Language | Recommended Storage Mechanism | Cross-Async Behavior |
|----------|-------------------------------|---------------------|
| Java | ThreadLocal + framework-specific transaction context | Requires explicit StateCaptor or framework agent for cross-thread propagation |
| Python | `ContextVar` | Natively propagates across `async`/`await` boundaries — no explicit cross-thread logic needed |
| NodeJS | `AsyncLocalStorage` | Natively propagates across async callbacks and promises — no explicit cross-thread logic needed |
| Go | `context.Context` | Propagates naturally via context passing — no thread-local issues |

#### Feature Parity Expectations

Not all languages need identical feature sets. Organizations SHOULD prioritize:

| Capability | Priority |
|-----------|----------|
| Token storage + propagation | **Required** in all languages |
| Domain allowlist | **Required** in all languages |
| Disable switch | **Required** in all languages |
| Token validation | High priority (Java first, others as needed) |
| Token issuance | Server-side only — language of TTS |
| Async preservation | Based on async workload patterns per language |
| Token modification/override | Based on intermediate service patterns |

#### Telemetry Consistency

Despite different implementations, all language SDKs MUST emit semantically equivalent metrics with consistent naming conventions. This enables cross-language monitoring dashboards and comparable adoption tracking.

### Cross-Thread Propagation
Modern services use thread pools, async executors, and non-blocking frameworks where request processing spans multiple threads. Token storage based on thread-local variables does NOT automatically propagate to child threads. Organizations MUST address this.

#### The Problem

When a service spawns child threads (via thread pools, async executors, or reactive frameworks), the token in the parent thread's storage is not automatically inherited. Without explicit propagation, child threads have no token — breaking the authentication chain for any downstream calls they make.

#### Solution: Capture-Restore Pattern

Propagation libraries SHOULD implement a state capture pattern:

```
Parent Thread                          Child Thread
|                                      |
+-- Capture: snapshot current token    |
|                                      |
|                                      +-- Restore: set captured token
|                                      |   into child's storage
|                                      |
|                                      +-- Execute: child runs with
|                                      |   propagated token
|                                      |
|                                      +-- Cleanup: clear child's storage
|                                      |   (prevent leak to next task)
```

#### Key Behaviors

- The captured token is a **snapshot** at capture time — subsequent changes on the parent thread are NOT reflected in the child
- Null/empty tokens SHOULD be propagated gracefully (no error)
- Cleanup MUST always run regardless of success or failure (prevents leaks on reused pool threads)
- The capture mechanism SHOULD be auto-discovered (e.g., via service loader or framework hooks) — no explicit wiring by service developers

#### Fire-and-Forget Pattern

For fire-and-forget async patterns (where the parent thread completes and responds before the child finishes):
- Parent thread clearing its storage at request end MUST NOT affect the child thread's copy
- The token storage implementation MUST support reference semantics where the child retains access even after the parent's cleanup
- This is a critical design consideration — naive "clear all" implementations will break fire-and-forget

### Propagation Reliability
Organizations SHOULD monitor propagation success rates across the SOA. Unless propagation success reaches 100% for a given call chain, services cannot reliably enforce authorization policies based on Txn-Token contents. Services MUST implement reasonable fallback behaviors when tokens are absent.

Propagation libraries MAY implement automatic token initiation when an incoming request lacks a Txn-Token. The library requests a placeholder token from the Transaction Token Service indicating that no context was received at the current service. This placeholder enables downstream services to identify where propagation broke in the call chain, facilitating operational debugging.

### Placeholder Token Design
Placeholder tokens serve as diagnostic markers that identify WHERE propagation broke in a call chain. They are NOT authorization tokens and MUST NOT be used for access decisions.

#### Placeholder Token Contents

Placeholder tokens SHOULD contain:
- `issuerServiceName` — the service that generated the placeholder (where the break was detected)
- `clientName` — the upstream caller that failed to propagate a real token
- `issuedAt` — timestamp of generation
- `reasonCode` — why the real token is missing

#### Reason Codes

| Code | Meaning |
|------|---------|
| `TOKEN_NOT_PRESENT` | No token in incoming request headers |
| `TOKEN_GENERATION_ERROR` | Token generation/issuance failed |
| `UNSUPPORTED_REGION` | Service is in a region that doesn't support Txn-Tokens |
| `UNKNOWN` | Reason could not be determined |

#### Placeholder Token Properties

- Placeholder tokens MUST NOT be cryptographically signed — they have no security value
- Validators encountering placeholder tokens SHOULD emit per-issuer metrics to identify which services are generating them
- APIs that extract details from placeholder tokens SHOULD be explicitly named to signal unreliability (e.g., `getUnsafePlaceholderDetails()`)
- Placeholder tokens SHOULD be distinguishable from real tokens via a format version identifier (O(1) check, no full decode needed)

#### Auto-Generation at Potential Entry Points

Services that might be the first in a call chain (potential initiators) SHOULD automatically generate a placeholder token if no token arrives. This distinguishes "I am the entry point and no token was issued" from "someone upstream broke propagation."

### Token Mix-up Prevention
Token mix-up represents the most severe propagation risk. Mix-up occurs when a token intended for one request is incorrectly attached to a different request. If token T1 is meant for request R1 and token T2 for request R2, but T2 is sent with R1 due to a propagation bug, actors may access data they are not authorized to see.

Token mix-up scenarios are difficult to detect because they may not cause obvious failures. The request succeeds but with incorrect authorization context. Organizations MUST implement request-scoped token storage in propagation libraries to prevent mix-up. Tokens MUST be associated with specific request contexts and MUST NOT be stored in shared or global state.

Organizations SHOULD implement testing strategies that deliberately attempt to cause token mix-up under concurrent load. These tests verify that propagation libraries correctly isolate tokens across concurrent requests.

### Token Leak Detection
Beyond preventing mix-up during concurrent requests, organizations MUST implement leak detection at request boundaries. A "leak" occurs when a token from a previous request remains in storage when a new, unrelated request arrives — meaning one request's credentials could bleed into another's authorization context.

#### Detection Mechanism

At the start of every incoming request, propagation libraries SHOULD check whether a leftover Txn-Token is already present in request-scoped storage before the new request's token is stored:

```
New request arrives
  -> Check: is a token already in storage?
  -> YES -> Leak detected! Log safe identifiers + emit metric
  -> NO -> Clean state, proceed normally
```

#### Detection vs Prevention

Leak detection is distinct from leak prevention:
- **Detection**: Identifies that a leak occurred (observability)
- **Prevention**: Mechanisms that ensure cleanup happens (see request lifecycle cleanup)

Both are required. Detection catches cases where prevention mechanisms fail.

#### Metrics

Organizations SHOULD emit a binary metric at request start:
- Value `0` = clean state (no leftover token)
- Value `1` = leak detected (token from previous request still present)

Detection SHOULD log safe, non-sensitive identifiers of both the leftover token and the incoming token for debugging — never raw token values.

#### Remediation

Upon detecting a leak, propagation libraries SHOULD override the leaked token with the correct incoming token. Detection is observability-only — it SHOULD NOT block or fail the request. The incoming request's token takes precedence.

### Trust Boundary Handling
When requests cross trust boundaries within the organization, propagation libraries MUST either block token propagation or replace token contents with appropriately scoped contexts. Organizations SHOULD define trust boundaries explicitly and configure propagation libraries with boundary detection logic.

At trust boundaries, services MAY request new Txn-Tokens from the Transaction Token Service with contexts appropriate for the target trust domain. This approach maintains context propagation while respecting security boundaries.

### Runtime Disable Switch
Organizations MUST provide a runtime kill switch that can disable token propagation without requiring service restarts. In production incidents where token propagation is contributing to failures, operators need immediate relief.

#### Requirements

- Each propagation solution (e.g., per framework integration) SHOULD have an independent disable switch
- Disable switches SHOULD be configurable via JVM system properties, environment variables, or dynamic configuration
- Changes SHOULD take effect within a bounded interval (e.g., 180 seconds maximum polling interval)
- All solutions MUST default to ENABLED on initialization
- Disabling propagation SHOULD NOT cause request failures — requests proceed without tokens

#### Operational Behavior When Disabled

When the disable switch is active:
- Incoming token extraction returns empty (tokens in headers are ignored)
- Outgoing request header injection is skipped
- Token storage operations become no-ops
- Telemetry SHOULD still emit metrics indicating the disabled state

#### Safety Constraints

- Disable switches SHOULD require explicit opt-in registration before polling takes effect
- A disabled propagation path MUST NOT generate errors or exceptions — it simply becomes invisible
- Organizations SHOULD alert when a disable switch has been active for extended periods (indicating a forgotten workaround)

### Token Context Modification
Intermediate services in a call chain sometimes need to modify token contexts for downstream calls without requesting a completely new token from the TTS. Organizations SHOULD support a token modification mechanism for this use case.

#### Use Cases

- Adding line-of-business (LOB) context for specific downstream services
- Narrowing authorization scope before calling a less-trusted downstream
- Adding operational metadata (e.g., "doing business as" identifiers) required by specific downstream APIs

#### Modification vs Exchange

| Aspect | Token Modification | Token Exchange (re-issuance) |
|--------|-------------------|------------------------------|
| When | Automatic during propagation | Explicit service code call |
| Who decides | Configuration (per downstream target) | Service developer |
| Network call | Yes (signed supplement from TTS) | Yes (full token from TTS) |
| Result | Original token + appended supplement | Brand new token |
| Latency impact | Lower (supplement is smaller) | Higher (full issuance) |

Organizations SHOULD prefer modification over exchange when only adding context, and prefer exchange when replacing or significantly altering token contents.

#### Configuration Model

Token modification SHOULD be configurable per downstream target:
- Which downstream services trigger modification
- What context to add/override for each target
- Failure mode: propagate placeholder token, propagate original token, or fail the request

#### Security Requirements

- Modifications MUST be cryptographically signed by the TTS
- The intermediate service's identity MUST be embedded in the modification (traceability)
- Only explicitly authorized namespaces (e.g., LOB) SHOULD be modifiable — general-purpose context modification introduces security risks

### Cache Considerations
The introduction of Txn-token provides more information now to the entire microservice architecture graph. There are Services in the graph that cache data to avoid calling dependent services multiple times. Now, they SHOULD consider Txn-Token contexts to be included in the cache keys. If not included, there is a risk that incorrect data is vended out or cache hit is impacted because the dependent services might be using the Txn-Token contexts for computing the results which might get cached.

Organizations SHOULD provide guidance to workload developers on cache key construction when Txn-Tokens are involved. Cache invalidation strategies MUST account for context changes that affect cached data.

## Token Validation
### Validation Libraries
Organizations SHOULD provide standardized validation libraries that handle signature verification, decryption, and token parsing. These libraries SHOULD synchronize cryptographic keys in the background, ensuring that services always have current keys for verification and decryption.

Validation libraries SHOULD decode Txn-Tokens into strongly-typed objects appropriate for the implementation language. This approach prevents parsing errors and provides compile-time verification of context access patterns.

### Key and Schema Caching
Validation libraries MUST cache cryptographic keys and token schemas locally. Remote lookups on every validation request introduce unacceptable latency.

#### Key Caching Requirements

- Keys MUST be pre-fetched into local cache before validation requests arrive
- Background refresh SHOULD run on a daemon thread at a fixed interval (e.g., every 1 hour) with jitter (+/-10%) to avoid thundering herd
- Cache miss for a required key SHOULD NOT trigger a synchronous remote fetch — the token validation fails fast with a clear error code
- Multiple concurrent keys MUST be supported to enable zero-downtime rotation (old key and new key both valid during transition)

#### Schema Caching Requirements

- Schemas define how token contexts are decoded. They MUST be cached locally.
- Background refresh interval SHOULD be longer than key refresh (e.g., every 6 hours) since schemas change less frequently
- A fresh schema parser instance SHOULD be used per deserialization to avoid cross-contamination between independently evolving schemas
- Schema not in cache at validation time -> fail with a specific error code (e.g., `MISSING_SCHEMA`) — do NOT block waiting for a fetch

#### Key Refresh Resilience

- Background refresh failures MUST be caught and logged — they MUST NOT terminate the refresh scheduler
- If the key management service is down, the last successfully fetched keys remain valid until they expire
- Organizations SHOULD alert on consecutive refresh failures exceeding a threshold

#### Environment-Specific Strategies

| Environment | Strategy |
|-------------|----------|
| Long-running services | Background daemon thread with scheduled refresh |
| Serverless / Lambda | Eager fetch on cold start; no background thread (short-lived process) |
| Edge / resource-constrained | Configurable refresh intervals with larger TTLs |

### Error Handling
Validation libraries SHOULD emit standardized error codes for common failure conditions including expired tokens, malformed tokens, and signature verification failures. These error codes enable consistent operational monitoring across the SOA.

### Fallback Policies
Services SHOULD NOT automatically fail requests when Txn-Tokens are missing or invalid. Organizations MUST define fallback policies that balance security with user experience. Fallback policies MAY include serving redacted data, limiting functionality, or requesting step-up authentication.

The appropriate fallback depends on the sensitivity of the requested operation. Services accessing highly sensitive data MAY require valid Txn-Tokens and fail requests when tokens are absent. Services providing less sensitive functionality SHOULD implement graceful degradation.

### Verification Strategy Modes
Organizations deploying Txn-Token validation MUST support at least three progressive enforcement modes that can be configured per-service and per-request:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `SHADOW_MODE` | Token is decoded and validated but failures are only logged — never enforced. Requests always proceed. | Initial rollout, testing, measuring impact before enforcement |
| `TOKEN_PRESENCE_MODE` | Token must be present in the request but no cryptographic verification is performed. Absence causes failure. | Intermediate enforcement — ensures propagation works before trusting content |
| `STRICT_VERIFICATION_MODE` | Full cryptographic verification. Invalid or missing tokens cause request rejection. | Production enforcement after confidence is established |

Organizations SHOULD default to SHADOW_MODE when first deploying validation and progressively tighten enforcement. The default mode SHOULD be configurable at both the service level and per-request level (e.g., based on API sensitivity or client identity).

Validation libraries SHOULD emit distinct metrics for each mode so organizations can track:
- How many requests WOULD fail under stricter enforcement
- Per-client and per-error-code failure rates in shadow mode
- Readiness percentage before promoting to stricter enforcement

Error logs during shadow mode SHOULD be sampled (e.g., 50%) to prevent log flooding while still providing visibility into issues.

Organizations MUST document and communicate enforcement promotion timelines to downstream service teams. Surprise enforcement changes cause outages.

## Telemetry and Monitoring
### Adoption Monitoring
Transaction token adoption in large SOA environments takes time. Organizations SHOULD implement comprehensive telemetry to monitor adoption progress, propagation reliability, and validation patterns.

When organizations provide standardized libraries for token initiation, propagation, and validation, telemetry logic SHOULD be embedded in those libraries. This approach ensures consistent telemetry across all services without requiring individual workload implementations.

### Telemetry Aggregation
Services SHOULD aggregate telemetry locally before transmitting to centralized monitoring systems. Local aggregation reduces network overhead and enables higher-frequency sampling without overwhelming monitoring infrastructure.

Centralized monitoring systems SHOULD store telemetry in data warehouses that support analytical queries. Organizations SHOULD implement automated monitors that alert on significant changes in propagation rates, validation failures, or token expiration rates.

### Key Metrics
Organizations SHOULD monitor the following key metrics:
- Token initiation rate by service
- Propagation success rate by call chain
- Token expiration rate during request processing
- Validation failure rate by error type
- Token size distribution
- Context relocation frequency

These metrics provide visibility into Txn-Token health across the SOA and enable rapid identification of deployment issues.

### Token Validation Audit Trail
Beyond operational telemetry, organizations MUST implement token validation audit logging that creates an audit trail for every token validation attempt.

#### When to Log

A security event MUST be generated after **every** validation attempt — regardless of success or failure. This includes:
- Successful token validation (identity confirmed)
- Failed validation (expired, malformed, bad signature)
- Placeholder token encountered
- Token absent when required

#### What to Log

Security event entries SHOULD include:
- **Initiator identity** — the application/service that originally issued the token
- **Subject identity** — the authenticated entity (end user, system) the token represents
- **Requester line of business** — organizational context
- **Customer/entity identifier** — if present in the token
- **Token type** — real token vs placeholder vs auto-generated
- **Validation status** — success, failure code, or skip reason
- **Timestamp and request identifier**

For placeholder tokens, log the issuer service name and upstream client name to trace propagation breaks.

#### Override Context Priority

When token modification (override) contexts exist alongside the original decoded contexts, the override context SHOULD take precedence in the security event log. This ensures the audit trail reflects the effective authorization state, not the original state.

#### Failure Resilience

Security event logging failures MUST NOT interrupt request processing. If the logging system is unavailable:
- Emit a metric counting missed security events
- Sample error logs (e.g., 50%) to avoid flooding
- Never propagate the logging failure to callers

## Key Management
Organizations MUST implement secure key management practices for Txn-Token cryptographic operations. Key management SHOULD follow the guidelines in RFC 4107 "Guidelines for Cryptographic Key Management".

Transaction Token Services MUST support key rotation without service disruption. Validation libraries MUST support multiple concurrent keys to enable zero-downtime rotation. Organizations SHOULD automate key rotation on a regular schedule.

## Batch Processing Pattern
OAuth Transaction Tokens are designed to propagate security context through a call chain within a trust domain. To maintain a high security posture without the overhead of a global revocation infrastructure, these tokens are short-lived (typically minutes). In many modern architectures, a transaction may be asynchronous. For example, a request may be placed on a message queue (e.g., Kafka, RabbitMQ) and processed by a worker service hours or days later. By the time the worker resumes the transaction, the original Transaction Token has expired.

Batch Token (Voucher): A long-lived, opaque, or encrypted token representing the transaction context during a period of rest.
Initiator: The internal microservice that receives a Transaction Token and requests a Batch Token before an asynchronous pause.
Rehydrator: The internal microservice that takes a Batch Token and exchanges it for a fresh, short-lived Transaction Token to resume processing.

### Initiation (Pausing the Transaction)

When a internal microservice determines that a transaction will exceed the TTL of the current Transaction Token (TraT), it SHOULD request a Batch Token from the Transaction Token Service (TTS).
The request to the TTS SHOULD include:
   * The current valid TraT.
   * The intended "use case ID" or "namespace" to constrain the token.

The TTS returns a Batch Token with a TTL suitable for the asynchronous delay (e.g., 24 hours to 7 days).

### Rehydration (Resuming the Transaction)

When a worker service (the Rehydrator) picks up the task, it MUST NOT use the Batch Token directly to call downstream services. Instead, it MUST exchange the Batch Token at the TTS for a fresh TraT.
The TTS SHALL:
   1.  Verify the Batch Token's signature and expiration.
   2.  Validate that the Rehydrator is authorized for the specific
       "use case ID" or "namespace" embedded in the Batch Token.
   3.  Issue a new, short-lived TraT containing the original
       claims (e.g., subject, original requester IP).

### Async Context Preservation

When preserving token contexts across asynchronous boundaries, organizations MUST use server-side signed preservation via the TTS. The TTS issues an ECDSA-signed preservation context that provides tamper detection and centralized audit. Client-side unsigned preservation MUST NOT be used — all async context preservation requires cryptographic integrity guarantees.

### Message Transport

For message-based async (SQS, SNS, Kafka), preserved context SHOULD be transported as a message attribute rather than embedded in the message body:
- Attribute name SHOULD be standardized (e.g., `x-transaction-token-preservation-context`)
- Data type: String (Base64-encoded binary)
- Idempotency: if the attribute already exists on the message, preserve the existing value (no overwrite)
- Message attribute limits (e.g., SQS max 10 attributes) MUST be respected — if the limit is reached, the failure SHOULD be logged but the original message MUST still be sent

#### Batch Token Scoping Enhancements

Batch Tokens (Vouchers) SHOULD include:
- A maximum chain depth counter to prevent infinite rehydration loops
- A total transaction lifetime that cannot be extended beyond a hard maximum regardless of rehydration count
- Namespace/use-case scoping so that a batch token obtained for one async workflow cannot be rehydrated in a different workflow context


# Security Considerations

## Token Mix-up Prevention
Token mix-up represents a critical security risk. Organizations MUST implement request-scoped token storage and MUST test for mix-up scenarios under concurrent load. Token mix-up can result in unauthorized data access without obvious system failures, making it particularly dangerous.

## Trust Boundary Controls
Organizations MUST define trust boundaries explicitly and MUST implement controls that prevent inappropriate token propagation across those boundaries. Failure to control propagation at trust boundaries can expose sensitive contexts to unauthorized services.

## Cache Security
Services that cache data based on Txn-Token contexts face security risks if cache keys do not incorporate all relevant contexts. Organizations MUST provide guidance on secure cache key construction and MUST audit caching services for correct context handling.

## Fallback Mechanisms
While fallback mechanisms improve availability, they can introduce security vulnerabilities if not carefully designed. Organizations MUST ensure that fallback policies do not inadvertently grant excessive access when Txn-Tokens are absent. Fallback policies SHOULD be explicitly documented and reviewed by security teams.

## Token Lifetime
Short token lifetimes reduce the window for token compromise but may cause operational issues if set too aggressively. Organizations MUST balance security considerations with operational requirements when setting token lifetimes.

## External Propagation
Preventing Txn-Tokens from leaving the trusted domain is critical. Organizations MUST implement multiple layers of defense including library-level controls, network-level filtering, and monitoring for external propagation attempts.

## Batch Processing Security Consideration

### Token Constraining

Batch Tokens MUST be sender-constrained or scoped to specific namespaces. This prevents a compromised service from "stealing" a Batch Token from a queue and successfully minting a
Transaction Token for an unrelated flow.

### Data Mutability and Consent

Asynchronous delays increase the risk that the underlying authorization context has changed (e.g., a user has revoked consent). The TTS SHOULD perform a "freshness check" during
rehydration for claims marked as mutable or sensitive.

### Infinite Exchange Prevention

To prevent a transaction from living indefinitely through repeated rehydrations, the TTS SHOULD implement a maximum chain depth or total transaction lifetime counter within the token metadata.

## Token Format Identification
Validation libraries MUST be able to quickly identify token type (real token vs placeholder vs handle) without performing full decryption. This enables:
- Fast-path rejection of placeholder tokens in strict enforcement mode
- Per-type metrics without expensive decode operations
- Efficient routing to format-specific decoders

# IANA Considerations

This document has no IANA actions.

# References
## Normative References
[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119) Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/RFC2119, March 1997.

[TXNTOKENS](https://www.ietf.org/archive/id/draft-ietf-oauth-transaction-tokens-06.html) Tulshibagwale, A., Hardt, D., and G. Fletcher, "OAuth 2.0 Transaction Tokens", draft-ietf-oauth-transaction-tokens-06 (work in progress).

## Informative References
[RFC4107](https://datatracker.ietf.org/doc/html/rfc4107) Bellovin, S. and R. Housley, "Guidelines for Cryptographic Key Management", BCP 107, RFC 4107, DOI 10.17487/RFC4107, June 2005.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
