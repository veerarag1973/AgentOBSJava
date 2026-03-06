# AgentOBS Java SDK — Phase-wise Implementation Plan

**Document:** AGENTOBS-JAVA-IMPL-PLAN-0001  
**Reference spec:** RFC-0001-AGENTOBS v2.0  
**Source SDK:** Python `agentobs` v1.0.5  
**Date:** March 6, 2026  
**Status:** Active — Draft

---

## Executive Summary

This document is the authoritative implementation plan for porting the AgentOBS Python SDK
to Java. It mirrors the full feature surface of the Python reference implementation —
all 36 event types, 11 namespaces, HMAC audit chains, PII redaction, OTLP export, provider
integrations, and CLI tooling — adapted to idiomatic Java design patterns with zero mandatory
dependencies beyond the JDK.

---

## Java Version Decision

**Target: Java 21 LTS**

| Criterion | Rationale |
|---|---|
| Industry adoption | Java 21 (released Sept 2023) is the current LTS; Spring Boot 3.2+, Quarkus 3.6+, Micronaut 4.3+ all target Java 21 as their baseline |
| Virtual Threads (JEP 444) | Replaces Python async I/O for exporter flush without blocking OS threads |
| Records (JEP 395) | Direct equivalent of Python `@dataclass(frozen=True)` — all value objects become records |
| Sealed classes (JEP 409) | Model closed type hierarchies (e.g., `SensitivityLevel`) |
| Pattern matching `switch` (JEP 441) | Clean dispatch over payload variants |
| Text blocks (JEP 378) | Readable JSON in tests and templates |
| `instanceof` pattern matching (JEP 394) | Eliminates unsafe casts in serialization |
| Minimum compatibility | Java 21 has ≥60% enterprise share for new projects in 2025-2026 surveys |

**Note:** The core module (`agentobs-core`) will compile with `--release 17` as a compatibility
floor. Only `agentobs-exporters-http` and `agentobs-cli` will require Java 21 features (virtual
threads). This gives consumers on Java 17+ full core access.

---

## Build Tooling

**Primary: Apache Maven 3.9+**  
**Module layout: Maven Multi-Module Project**

Maven was chosen over Gradle because:
- Dominant in enterprise Java (>70% of enterprise projects)
- Declarative BOMs (Bill of Materials) map cleanly to Python's optional extras
- Native support for multi-module reactor builds
- Stable, auditable dependency management

All modules share a parent POM at the root that pins all dependency versions.

---

## Module Structure

```
agentobs-java/                           (root — parent POM only)
├── agentobs-core/                       Phase 1–4   — zero JDK-external dependencies
├── agentobs-security/                   Phase 5     — HMAC signing (javax.crypto, stdlib)
├── agentobs-privacy/                    Phase 6     — PII redaction (stdlib)
├── agentobs-validation/                 Phase 7     — JSON Schema validation (optional: networknt)
├── agentobs-exporters-jsonl/            Phase 8     — JSONL file exporter (stdlib NIO)
├── agentobs-exporters-console/          Phase 8     — Console pretty-print (stdlib)
├── agentobs-exporters-http/             Phase 9     — OTLP + Webhook (java.net.http, JDK 11+)
├── agentobs-exporters-datadog/          Phase 9     — Datadog APM exporter
├── agentobs-exporters-grafana/          Phase 9     — Grafana Loki exporter
├── agentobs-otel-bridge/               Phase 10    — OpenTelemetry SDK bridge (optional: otel-sdk)
├── agentobs-integration-langchain4j/    Phase 11    — LangChain4j callback (optional)
├── agentobs-integration-spring-ai/      Phase 11    — Spring AI integration (optional)
├── agentobs-integration-openai/         Phase 11    — OpenAI Java SDK integration (optional)
├── agentobs-governance/                 Phase 12    — Governance + consumer registry
├── agentobs-cli/                        Phase 13    — CLI tool (picocli)
└── agentobs-bom/                        Phase 14    — Bill of Materials POM
```

**Dependency rule:** `agentobs-core` has **zero** dependencies outside the JDK.
Every other module depends on `agentobs-core` but may introduce optional runtime dependencies.

---

## Key Java Design Decisions

### D1 — Records for All Value Objects

Python's `@dataclass(frozen=True)` → Java `record`.

```java
// Python: @dataclass(frozen=True) class TokenUsage:
public record TokenUsage(
    int inputTokens,
    int outputTokens,
    int totalTokens,
    @Nullable Integer cachedTokens,
    @Nullable Integer reasoningTokens,
    @Nullable Integer imageTokens
) {
    // Compact constructor for validation
    public TokenUsage {
        if (inputTokens < 0) throw new SchemaValidationException("inputTokens", inputTokens, "must be non-negative");
        if (outputTokens < 0) throw new SchemaValidationException("outputTokens", outputTokens, "must be non-negative");
        if (totalTokens < 0) throw new SchemaValidationException("totalTokens", totalTokens, "must be non-negative");
    }
}
```

### D2 — AutoCloseable for Span Context Managers

Python's `__enter__`/`__exit__` → Java `AutoCloseable` + try-with-resources.

```java
// Python: with tracer.span("chat", model="gpt-4o") as span:
try (SpanContext span = tracer.span("chat").model("gpt-4o").start()) {
    span.setAttribute("temperature", 0.7);
    // ... LLM call ...
} // auto-closes: records duration, emits event
```

### D3 — ThreadLocal for Span Stack

Python's `threading.local()` → Java `ThreadLocal<Deque<SpanContext>>`.

The active span stack is held in a `ThreadLocal` so nested spans work
correctly across threads including virtual threads (each carries its own
thread-local state).

### D4 — Fluent Builder Pattern for Event

Python's keyword-argument constructors → Java Builder pattern.

```java
Event event = Event.builder()
    .eventType(EventType.TRACE_SPAN_COMPLETED)
    .source("my-app@1.0.0")
    .orgId("org_acme")
    .payload(Map.of("model", "gpt-4o", "latency_ms", 340.5))
    .tags(Tags.of("env", "production"))
    .build(); // validates eagerly; throws SchemaValidationException on failure
```

### D5 — Functional Interface for Exporter Protocol

Python's structural `Protocol` → Java `@FunctionalInterface`.

```java
@FunctionalInterface
public interface Exporter {
    void exportBatch(List<Event> events) throws ExportException;
}
```

### D6 — Enum with String Value for EventType

Python's `class EventType(str, Enum)` → Java `enum` with `value` field + `fromValue()`.

```java
public enum EventType {
    TRACE_SPAN_STARTED("llm.trace.span.started"),
    TRACE_SPAN_COMPLETED("llm.trace.span.completed"),
    // ...36 total
    ;
    private final String value;
    public String value() { return value; }
    public static EventType fromValue(String v) { ... }
    @Override public String toString() { return value; }
}
```

### D7 — Sealed Hierarchy for Sensitivity

Python's `class Sensitivity(str, Enum)` → Java `enum` with ordered comparison.

```java
public enum Sensitivity {
    LOW(0), MEDIUM(1), HIGH(2), PII(3), PHI(4);
    private final int order;
    public boolean isAtLeast(Sensitivity other) { return this.order >= other.order; }
}
```

### D8 — CompletableFuture for Async Exporters

Python's `async def export_batch` → Java `CompletableFuture<Void>` with virtual-thread executor.

```java
// Async HTTP send — uses virtual thread pool (Java 21)
CompletableFuture<Void> sendAsync(List<Event> events) {
    return CompletableFuture.runAsync(
        () -> sendSync(events),
        Executors.newVirtualThreadPerTaskExecutor()
    );
}
```

### D9 — Canonical JSON via Sorted LinkedHashMap

Python's `json.dumps(sort_keys=True, separators=(",", ":"))` → Java: sort map keys
into a `TreeMap` before serialization; use a minimal JSON writer (stdlib only,
no Jackson in core).

A lightweight JSON serializer is implemented in `agentobs-core` using only
`java.util`, `java.io`, and `java.lang`. Jackson (`com.fasterxml.jackson`) is
an optional dependency in `agentobs-exporters-*`.

### D10 — HMAC via javax.crypto (stdlib)

Python's `import hmac; import hashlib` → `javax.crypto.Mac` + `java.security.MessageDigest`.
Both are part of the JDK — no Bouncy Castle or other crypto library required.

### D11 — Configuration via System Properties + Environment Variables

Python's `os.environ.get("AGENTOBS_EXPORTER")` → Java:
1. `System.getenv("AGENTOBS_EXPORTER")`
2. `System.getProperty("agentobs.exporter")` (JVM flag override)
3. Programmatic `AgentObs.configure(...)` (highest priority)

### D12 — Null Safety Annotations

Use `@Nullable` and `@NonNull` from `org.jetbrains.annotations` (compile-time
only, zero runtime dep) on all public APIs so IDEs and static analysis tools
flag null misuse. No runtime `null` checks are added for performance.

---

## Python → Java Type Mapping Reference

| Python construct | Java equivalent |
|---|---|
| `@dataclass(frozen=True)` | `record` |
| `__slots__` | `record` or `final` fields |
| `MappingProxyType` | `Collections.unmodifiableMap()` |
| `threading.local()` | `ThreadLocal<T>` |
| `typing.Protocol` | `interface` |
| `contextlib` context manager | `AutoCloseable` |
| `str` subclass `Enum` | `enum` with `String value` field |
| `frozenset` | `Set.of(...)` / `Collections.unmodifiableSet()` |
| `re.compile(pattern)` | `Pattern.compile(pattern)` |
| `hashlib.sha256()` | `MessageDigest.getInstance("SHA-256")` |
| `hmac.new(key, msg, sha256)` | `Mac.getInstance("HmacSHA256")` |
| `warnings.warn(...)` | SLF4J `log.warn(...)` |
| `asyncio.Queue` | `LinkedBlockingQueue<Event>` |
| `async def` / `await` | `CompletableFuture<T>` (virtual threads) |
| `optionals` install extras | Maven `<optional>true</optional>` deps |
| `__init__.py` exports | `module-info.java` `exports` directives |

---

## Phase Overview

| Phase | Name | Module(s) Produced | Java 17+ / 21 |
|---|---|---|---|
| 0 | **Project Scaffold** | Root POM, directory structure, CI | Both |
| 1 | **Core — ULID** | `agentobs-core` (ULID only) | 17+ |
| 2 | **Core — Event Envelope** | `agentobs-core` (Event, Tags, EventType) | 17+ |
| 3 | **Core — Exceptions + Validation** | `agentobs-core` | 17+ |
| 4 | **Core — Namespace Payload Types** | `agentobs-core` (all 11 namespaces) | 17+ |
| 5 | **Configuration Layer** | `agentobs-core` (AgentObsConfig) | 17+ |
| 6 | **Tracer + Span + Agent API** | `agentobs-core` (Tracer, Span, AgentRun) | 17+ |
| 7 | **Event Stream** | `agentobs-core` (EventStream, Exporter) | 17+ |
| 8 | **Security — HMAC Audit Chains** | `agentobs-security` | 17+ |
| 9 | **Privacy — PII Redaction** | `agentobs-privacy` | 17+ |
| 10 | **Built-in Exporters** | `agentobs-exporters-jsonl`, `-console`, `-http` | 17+ / 21 |
| 11 | **OTLP + OTel Bridge** | `agentobs-exporters-http`, `agentobs-otel-bridge` | 21 |
| 12 | **Governance + Consumer Registry** | `agentobs-governance` | 17+ |
| 13 | **Provider Integrations** | `agentobs-integration-openai`, et al. | 21 |
| 14 | **Framework Integrations** | `agentobs-integration-langchain4j`, `-spring-ai` | 21 |
| 15 | **Schema Validation** | `agentobs-validation` | 17+ |
| 16 | **CLI Tooling** | `agentobs-cli` | 21 |
| 17 | **BOM + Distribution** | `agentobs-bom`, Maven Central artifacts | 17+/21 |
| 18 | **Hardening + Docs** | Javadoc, examples, 1.0.0 release | — |

---

## Phase 0 — Project Scaffold

**Goal:** Create the Maven multi-module project skeleton, CI pipeline, and coding standards.

### Directory Layout

```
agentobs-java/
├── .github/
│   └── workflows/
│       ├── ci.yml              # Maven build + tests on push
│       └── release.yml         # Maven Central publish on tag
├── pom.xml                     # Parent POM — versions, plugin management
├── agentobs-bom/
│   └── pom.xml                 # BOM POM (no code)
├── agentobs-core/
│   ├── pom.xml
│   └── src/main/java/io/agentobs/...
│   └── src/test/java/io/agentobs/...
└── ... (one directory per module)
```

### Parent POM Key Decisions

```xml
<groupId>io.agentobs</groupId>
<artifactId>agentobs-parent</artifactId>
<version>0.1.0-SNAPSHOT</version>
<packaging>pom</packaging>

<!-- Compile core at Java 17 bytecode; CLI and HTTP modules at 21 -->
<properties>
  <maven.compiler.release>17</maven.compiler.release>
  <java.version>21</java.version>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

### Dependency Version Lock (parent POM `<dependencyManagement>`)

| Artifact | Version | Scope | Notes |
|---|---|---|---|
| `com.fasterxml.jackson.core:jackson-databind` | 2.17.x | optional | Exporters only |
| `io.opentelemetry:opentelemetry-api` | 1.40.x | optional | OTel bridge |
| `info.picocli:picocli` | 4.7.x | compile | CLI only |
| `org.slf4j:slf4j-api` | 2.0.x | compile | Logging facade |
| `com.networknt:json-schema-validator` | 1.4.x | optional | Schema validation |
| `org.junit.jupiter:junit-jupiter` | 5.11.x | test | All modules |
| `org.assertj:assertj-core` | 3.26.x | test | All modules |
| `org.mockito:mockito-core` | 5.12.x | test | Integration tests |

### CI Pipeline (GitHub Actions)

```yaml
# ci.yml — runs on every push and PR
steps:
  - name: Build + unit tests (Java 17)
    run: mvn -pl agentobs-core,agentobs-security,agentobs-privacy verify
  - name: Full build (Java 21)
    run: mvn verify
  - name: CheckStyle / SpotBugs
    run: mvn checkstyle:check spotbugs:check
  - name: Javadoc
    run: mvn javadoc:aggregate
```

### Acceptance Criteria

- `mvn clean install -DskipTests` succeeds on Java 17 for core modules
- `mvn verify` runs all tests on Java 21
- `mvn checkstyle:check` passes — Google Java Style enforced

---

## Phase 1 — Core: ULID Generator

**Module:** `agentobs-core`  
**Python source:** `agentobs/ulid.py`

### Specification (RFC-0001 §6)

- 128-bit identifier: 48-bit millisecond timestamp + 80-bit random
- Crockford Base32 encoding, 26 characters
- First character MUST be in `[0-7]` (timestamp MSB constraint)
- Monotonic within same millisecond: increment random segment on collision
- Zero external dependencies: uses `java.security.SecureRandom` + `System.currentTimeMillis()`

### Key Classes

```
io.agentobs.core.ulid/
  UlidGenerator.java       — stateful, thread-safe generator (singleton)
  Ulid.java                — value type wrapping the 26-char string
  UlidException.java       — extends AgentObsException
```

### Implementation Notes

```java
public final class UlidGenerator {
    private static final String ALPHABET = "0123456789ABCDEFGHJKMNPQRSTVWXYZ";
    private static final SecureRandom RANDOM = new SecureRandom();
    
    // Thread-safe monotonic state — use AtomicLong for last ms, AtomicLong for last rand
    private final AtomicLong lastMs = new AtomicLong(0);
    private final AtomicLong lastRand = new AtomicLong(0);
    private final Object lock = new Object();

    public String generate() { ... }    // returns 26-char string
    public static boolean isValid(String s) { ... }  // regex + first-char check
}
```

**Crockford decode map:** Pre-computed `Map<Character, Integer>` including
lowercase and `I→1, L→1, O→0` aliases for decode; strict charset for validation.

### Acceptance Criteria

- `UlidGenerator.generate()` returns 26-char Crockford Base32 string
- First character always in `[0-7]`
- 1000 sequential calls within same millisecond produce monotonically increasing ULIDs
- `Ulid.isValid("NOT_A_ULID")` returns `false`
- Thread-safety: 100 concurrent threads × 1000 generates produce 100,000 distinct ULIDs
- No external JAR dependency — `agentobs-core` ships zero transitive runtime deps

---

## Phase 2 — Core: Event Envelope

**Module:** `agentobs-core`  
**Python source:** `agentobs/event.py`, `agentobs/types.py`

### Key Classes

```
io.agentobs.core/
  Event.java               — immutable envelope (builder pattern)
  Event.Builder.java        — nested static builder
  Tags.java                — immutable String→String map
  EventType.java           — enum of 36 canonical types + fromValue()
  SchemaVersion.java        — SCHEMA_VERSION = "2.0" constant
```

### Event Immutability Contract

```java
public final class Event {
    // All fields are private final
    private final String schemaVersion;      // always "2.0"
    private final String eventId;           // ULID, auto-generated
    private final EventType eventType;
    private final String timestamp;          // ISO-8601 µs UTC
    private final String source;            // name@semver
    private final Map<String, Object> payload;  // unmodifiable
    // Optional fields...
    private final @Nullable String traceId;
    private final @Nullable String spanId;
    private final @Nullable String parentSpanId;
    private final @Nullable String orgId;
    private final @Nullable String teamId;
    private final @Nullable String actorId;
    private final @Nullable String sessionId;
    private final @Nullable Tags tags;
    private final @Nullable String checksum;
    private final @Nullable String signature;
    private final @Nullable String prevId;

    // No public constructor — use builder
    private Event(Builder b) { ... }

    // Canonical JSON serialization (sorted keys, null omitted, compact)
    public String toJson() { ... }
    public Map<String, Object> toDict() { ... }
    public static Event fromJson(String json) { ... }
    public static Event fromDict(Map<String, Object> dict) { ... }
    public void validate() throws SchemaValidationException { ... }
    
    public static Builder builder() { return new Builder(); }
}
```

### Canonical JSON Serializer

Implemented in `io.agentobs.core.json.CanonicalJsonWriter` — a minimal,
zero-dependency JSON writer:

- `Map<K,V>` → written with keys sorted via `TreeMap`
- `null` values → **omitted** (not written as `null`)
- `String` → escaped per JSON spec
- `Number` → integer or decimal without trailing zeros
- `List<?>` → written as JSON array
- `boolean` → `true`/`false`
- Datetime strings → passed through as-is (already formatted as ISO-8601)

### EventType Enum — All 36 Types

```java
public enum EventType {
    // llm.trace.*
    TRACE_SPAN_STARTED("llm.trace.span.started"),
    TRACE_SPAN_COMPLETED("llm.trace.span.completed"),
    TRACE_SPAN_FAILED("llm.trace.span.failed"),
    TRACE_AGENT_STEP("llm.trace.agent.step"),
    TRACE_AGENT_COMPLETED("llm.trace.agent.completed"),
    TRACE_REASONING_STEP("llm.trace.reasoning.step"),
    // llm.cost.*
    COST_TOKEN_RECORDED("llm.cost.token.recorded"),
    COST_SESSION_RECORDED("llm.cost.session.recorded"),
    COST_ATTRIBUTED("llm.cost.attributed"),
    // llm.cache.*
    CACHE_HIT("llm.cache.hit"),
    CACHE_MISS("llm.cache.miss"),
    CACHE_EVICTED("llm.cache.evicted"),
    CACHE_WRITTEN("llm.cache.written"),
    // llm.eval.*
    EVAL_SCORE_RECORDED("llm.eval.score.recorded"),
    EVAL_REGRESSION_DETECTED("llm.eval.regression.detected"),
    // llm.guard.*
    GUARD_INPUT_CHECKED("llm.guard.input.checked"),
    GUARD_OUTPUT_BLOCKED("llm.guard.output.blocked"),
    GUARD_OUTPUT_PASSED("llm.guard.output.passed"),
    // llm.fence.*
    FENCE_CONSTRAINT_APPLIED("llm.fence.constraint.applied"),
    FENCE_RETRY_TRIGGERED("llm.fence.retry.triggered"),
    FENCE_LIMIT_REACHED("llm.fence.limit.reached"),
    // llm.prompt.*
    PROMPT_RENDERED("llm.prompt.rendered"),
    PROMPT_SAVED("llm.prompt.saved"),
    PROMPT_PROMOTED("llm.prompt.promoted"),
    // llm.redact.*
    REDACT_PII_DETECTED("llm.redact.pii.detected"),
    REDACT_PII_REMOVED("llm.redact.pii.removed"),
    // llm.diff.*
    DIFF_PROMPT_CHANGED("llm.diff.prompt.changed"),
    DIFF_RESPONSE_CHANGED("llm.diff.response.changed"),
    // llm.template.*
    TEMPLATE_REGISTERED("llm.template.registered"),
    TEMPLATE_RENDERED("llm.template.rendered"),
    TEMPLATE_DEPRECATED("llm.template.deprecated"),
    // llm.audit.*
    AUDIT_KEY_ROTATED("llm.audit.key.rotated"),
    AUDIT_CHAIN_VERIFIED("llm.audit.chain.verified"),
    AUDIT_CHAIN_TAMPERED("llm.audit.chain.tampered"),
    AUDIT_ACCESS_GRANTED("llm.audit.access.granted"),
    AUDIT_ACCESS_DENIED("llm.audit.access.denied");
    
    private final String wireValue;
    // ... fromValue(), toString(), isRegistered(), namespaceOf()
}
```

### Tags

```java
public final class Tags {
    private final Map<String, String> data;  // Collections.unmodifiableSortedMap
    
    public static Tags of(String... kvPairs) { ... }   // varargs: "k1","v1","k2","v2"
    public static Tags from(Map<String, String> map) { ... }
    public String get(String key) { ... }
    public Map<String, String> toMap() { ... }
    // equals, hashCode, toString
}
```

### Acceptance Criteria

- `Event.builder().eventType(...).source("svc@1.0.0").payload(Map.of("k","v")).build()` succeeds
- `event.toJson()` produces alphabetically sorted JSON, omits nulls
- `Event.fromJson(event.toJson()).equals(event)` — lossless round-trip
- `event.toJson()` matches byte-for-byte across JVM restarts (determinism test)
- `Tags` rejects empty keys/values with `SchemaValidationException`

---

## Phase 3 — Core: Exception Hierarchy + Validation

**Module:** `agentobs-core`  
**Python source:** `agentobs/exceptions.py`, `agentobs/validate.py`

### Exception Hierarchy

```
AgentObsException (extends RuntimeException)
├── SchemaValidationException(field, received, reason)
├── UlidException(detail)
├── SerializationException(eventId, reason)
├── DeserializationException(reason, sourceHint)
├── EventTypeException(eventType, reason)
├── SigningException(reason)           — moved to agentobs-security
├── VerificationException(eventId)     — moved to agentobs-security
├── ExportException(reason, cause)
└── SchemaVersionException(version, accepted)
```

**Design rule:** Every exception carries structured fields — never bare string-only
messages. All exceptions extend `RuntimeException` (unchecked) because the Python
SDK also treats them as runtime failures, not checked exceptions.

### Validation

`EventValidator` in `agentobs-core` performs structural validation matching the
JSON Schema in `schemas/v1.0/schema.json`:

```java
public final class EventValidator {
    private static final Pattern ULID_RE = Pattern.compile("^[0-7][0-9A-HJKMNP-TV-Z]{25}$");
    private static final Pattern SOURCE_RE = Pattern.compile("^[a-zA-Z][a-zA-Z0-9._\\-]*@\\d+\\.\\d+\\.\\d+(?:[.\\-][a-zA-Z0-9.]+)?$");
    private static final Pattern TIMESTAMP_RE = Pattern.compile("^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{6}Z$");
    private static final Pattern TRACE_ID_RE = Pattern.compile("^[0-9a-f]{32}$");
    private static final Pattern SPAN_ID_RE = Pattern.compile("^[0-9a-f]{16}$");
    private static final Pattern CHECKSUM_RE = Pattern.compile("^sha256:[0-9a-f]{64}$");
    private static final Pattern SIGNATURE_RE = Pattern.compile("^hmac-sha256:[0-9a-f]{64}$");

    public void validate(Event event) throws SchemaValidationException { ... }
}
```

`validate()` is called automatically inside `Event.Builder.build()`.

### Acceptance Criteria

- `Event.builder()...build()` with missing `source` → `SchemaValidationException` with `field="source"`
- All pattern violations surface with the offending field name and value
- `SchemaValidationException.getField()` / `.getReceived()` / `.getReason()` accessible

---

## Phase 4 — Core: Namespace Payload Types

**Module:** `agentobs-core`  
**Python source:** `agentobs/namespaces/` (11 modules, 42 payload classes)

### Package Layout

```
io.agentobs.core.namespaces/
  trace/
    GenAiSystem.java          — enum (openai, anthropic, ... _custom)
    GenAiOperationName.java   — enum (chat, text_completion, ...)
    SpanKind.java             — enum (CLIENT, SERVER, INTERNAL, ...)
    TokenUsage.java           — record
    ModelInfo.java            — record
    CostBreakdown.java        — record
    PricingTier.java          — record
    ToolCall.java             — record
    ReasoningStep.java        — record (content NEVER stored — hash only)
    DecisionPoint.java        — record
    SpanPayload.java          — class with builder
    AgentStepPayload.java     — class with builder
    AgentRunPayload.java      — class with builder
  cost/
    CostRecordedPayload.java
    CostSessionPayload.java
    CostAttributedPayload.java
  cache/
    CachePayload.java
  eval/
    EvalScorePayload.java
    EvalRegressionPayload.java
  guard/
    GuardPayload.java
  fence/
    FencePayload.java
  prompt/
    PromptPayload.java
  redact/
    RedactPayload.java
  diff/
    DiffPayload.java
  template/
    TemplatePayload.java
  audit/
    AuditPayload.java
```

All payload types implement `PayloadSerializable`:

```java
public interface PayloadSerializable {
    Map<String, Object> toDict();
    // static fromDict(Map<String, Object>) in each class
}
```

### Critical Design: ReasoningStep (RFC-0001 §8.2)

```java
public record ReasoningStep(
    int stepIndex,
    int reasoningTokens,
    @Nullable Double durationMs,
    @Nullable String contentHash   // SHA-256 hex — raw content NEVER stored
) {
    public ReasoningStep {
        if (contentHash != null && !contentHash.matches("[0-9a-f]{64}"))
            throw new SchemaValidationException("contentHash", contentHash,
                "must be 64 lowercase hex characters");
    }
}
```

### SpanPayload (RFC-0001 §8.1)

```java
public final class SpanPayload implements PayloadSerializable {
    // REQUIRED
    private final String spanId;        // 16 hex chars
    private final String traceId;       // 32 hex chars
    private final String spanName;
    private final GenAiOperationName operation;
    private final SpanKind spanKind;
    private final String status;        // "ok" | "error" | "timeout"
    private final long startTimeUnixNano;
    private final long endTimeUnixNano;
    private final double durationMs;
    // OPTIONAL
    private final @Nullable String parentSpanId;
    private final @Nullable ModelInfo model;
    private final @Nullable TokenUsage tokenUsage;
    private final @Nullable CostBreakdown cost;
    private final List<ToolCall> toolCalls;         // never null, default []
    private final @Nullable String finishReason;
    private final @Nullable String error;
    private final @Nullable String errorType;
    private final Map<String, Object> attributes;   // never null, default {}
}
```

### Acceptance Criteria

- All 42 payload classes are constructable from their `toDict()` output (`fromDict(toDict()) == original`)
- `TokenUsage` with `inputTokens=-1` throws `SchemaValidationException`
- `ModelInfo` with `system=_custom` and no `customSystemName` throws immediately
- `ReasoningStep` does not expose raw reasoning content

---

## Phase 5 — Configuration Layer

**Module:** `agentobs-core`  
**Python source:** `agentobs/config.py`

### Classes

```java
// Thread-safe mutable singleton
public final class AgentObsConfig {
    private volatile String exporter = "console";
    private volatile @Nullable String endpoint;
    private volatile @Nullable String orgId;
    private volatile String serviceName = "unknown-service";
    private volatile String env = "production";
    private volatile String serviceVersion = "0.0.0";
    private volatile @Nullable String signingKey;
    private volatile @Nullable RedactionPolicy redactionPolicy;
    private volatile ExportErrorPolicy onExportError = ExportErrorPolicy.WARN;
    
    // Load from env at class-load time; configure() overrides
}

public final class AgentObs {
    public static AgentObsConfig configure() { ... }
    
    // Fluent configuration API
    public static AgentObsConfig configure(Consumer<AgentObsConfig> configurator) {
        configurator.accept(CONFIG);
        return CONFIG;
    }
    
    public static void configure(String exporter, String serviceName) { ... }
}
```

### Environment Variable / System Property Mapping

| Env var | System property | Config field |
|---|---|---|
| `AGENTOBS_EXPORTER` | `agentobs.exporter` | `exporter` |
| `AGENTOBS_ENDPOINT` | `agentobs.endpoint` | `endpoint` |
| `AGENTOBS_ORG_ID` | `agentobs.orgId` | `orgId` |
| `AGENTOBS_SERVICE_NAME` | `agentobs.serviceName` | `serviceName` |
| `AGENTOBS_ENV` | `agentobs.env` | `env` |
| `AGENTOBS_SERVICE_VERSION` | `agentobs.serviceVersion` | `serviceVersion` |
| `AGENTOBS_SIGNING_KEY` | `agentobs.signingKey` | `signingKey` |
| `AGENTOBS_ON_EXPORT_ERROR` | `agentobs.onExportError` | `onExportError` |

**Priority:** System property > Environment variable > programmatic `configure()` > default.

### Acceptance Criteria

- `AgentObs.configure(c -> c.setExporter("jsonl").setServiceName("my-svc"))` mutates singleton
- `AGENTOBS_EXPORTER=jsonl` is loaded at startup
- System property overrides env var
- `configure()` is thread-safe under concurrent writers

---

## Phase 6 — Tracer, Span, and Agent API

**Module:** `agentobs-core`  
**Python source:** `agentobs/_tracer.py`, `agentobs/_span.py`

### Classes

```
io.agentobs.core.tracer/
  AgentObs.java                — primary entry point
  Tracer.java                  — tracer singleton + factory
  SpanContext.java             — implements AutoCloseable
  SpanContextBuilder.java      — fluent builder
  AgentRunContext.java         — implements AutoCloseable
  AgentStepContext.java        — implements AutoCloseable
  SpanStack.java               — ThreadLocal<Deque<SpanContext>>
```

### Primary API

```java
// Zero-config:
AgentObs.configure(c -> c.setExporter("console").setServiceName("my-app"));

// Span API:
try (SpanContext span = AgentObs.tracer().span("chat").model("gpt-4o").start()) {
    span.setAttribute("temperature", 0.7);
    String result = callLlm(prompt);
    span.setStatus("ok");
}

// Agent API:
try (AgentRunContext run = AgentObs.tracer().agentRun("research-agent")) {
    try (AgentStepContext step = run.agentStep("search")) {
        step.setAttribute("query", "what is RAG?");
    }
    try (AgentStepContext step = run.agentStep("summarize")) {
        // ...
    }
}
// → emits AgentRunPayload with aggregated stats on outer close
```

### SpanContext Lifecycle

```
SpanContextBuilder.start()
  │  ── pushes to ThreadLocal SpanStack
  │  ── records startTimeUnixNano = System.nanoTime() + epoch offset
  │  ── generates spanId (ULID→hex)
  │  ── inherits traceId from parent or generates new one
  ▼
try-body executes
  │  ── span.setAttribute(k, v)
  │  ── span.setTokenUsage(usage)
  │  ── span.recordError(e)
  ▼
SpanContext.close()
  ── records endTimeUnixNano
  ── computes durationMs
  ── sets status = "ok" (or "error" if exception was recorded)
  ── builds SpanPayload
  ── builds Event(TRACE_SPAN_COMPLETED, payload, source, ...)
  ── applies RedactionPolicy (if configured)
  ── applies HMAC signing (if signingKey configured)
  ── dispatches to Exporter via EventStream
  ── pops from SpanStack
```

### Auto-Populated Fields on Every Span

| Field | Source |
|---|---|
| `spanId` | `UlidGenerator.generate()` converted to hex |
| `traceId` | Inherited from parent via `SpanStack`; new ULID hex if root |
| `eventId` | `UlidGenerator.generate()` |
| `timestamp` | `Instant.now()` formatted as `yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'` |
| `schemaVersion` | `"2.0"` |
| `source` | `config.getServiceName() + "@" + config.getServiceVersion()` |

### Acceptance Criteria

```java
AgentObs.configure(c -> c.setExporter("console"));
try (SpanContext span = AgentObs.tracer().span("test").model("gpt-4o").start()) {
    span.setAttribute("test_key", "test_value");
}
// → spans complete with no exceptions
// → console prints valid event JSON with event_type = "llm.trace.span.completed"
```

---

## Phase 7 — Event Stream

**Module:** `agentobs-core`  
**Python source:** `agentobs/stream.py`, `agentobs/_stream.py`

### API

```java
public final class EventStream {
    // Immutable sequence
    public static EventStream of(List<Event> events) { ... }
    public static EventStream empty() { ... }
    
    // Constructors
    public static EventStream fromFile(Path path) throws IOException { ... }
    public static EventStream fromQueue(BlockingQueue<Event> queue) { ... }
    
    // Filtering (returns new EventStream)
    public EventStream filter(Predicate<Event> predicate) { ... }
    public EventStream filterByType(EventType type) { ... }
    public EventStream filterByType(String typePrefix) { ... }
    public EventStream filterByTags(String key, String value) { ... }
    
    // Export
    public void drain(Exporter exporter) { ... }
    public CompletableFuture<Void> drainAsync(AsyncExporter exporter) { ... }
    public void route(Exporter exporter, Predicate<Event> predicate) { ... }
    
    // Accessors
    public int size() { ... }
    public List<Event> events() { ... }   // unmodifiable
    public Stream<Event> stream() { ... } // java.util.stream.Stream
}

@FunctionalInterface
public interface Exporter {
    void exportBatch(List<Event> events) throws ExportException;
}

@FunctionalInterface
public interface AsyncExporter {
    CompletableFuture<Void> exportBatchAsync(List<Event> events);
}
```

### Internal Dispatch Stream

`io.agentobs.core.tracer.DispatchStream` — the live, mutable write-side of the stream.
On each `emit(Event)` call:
1. Apply `RedactionPolicy` (if configured) — mutates payload before export
2. Apply HMAC signing (if `signingKey` configured)
3. Call `Exporter.exportBatch(List.of(event))`
4. If export fails: honour `onExportError` policy (`WARN|RAISE|DROP`)

### Acceptance Criteria

- `EventStream.of(events).filter(...).drain(exporter)` calls `exportBatch` with only matching events
- `EventStream.fromFile(path)` loads JSONL correctly; skips blank lines
- `route()` sends events to the right exporter based on predicate

---

## Phase 8 — Security: HMAC Audit Chains

**Module:** `agentobs-security`  
**Python source:** `agentobs/signing.py`

### Algorithm (RFC-0001 §11)

```
checksum  = "sha256:"       + SHA-256(canonical_payload_json_bytes).hex
sig_input = event_id + "|" + checksum + "|" + (prevId ?? "")
signature = "hmac-sha256:" + HMAC-SHA256(sig_input_bytes, orgSecret_bytes).hex
```

### Classes

```java
public final class EventSigner {
    // Sign a single event; returns new Event with checksum, signature, prevId
    public static Event sign(Event event, String orgSecret, @Nullable Event prevEvent) { ... }
    
    // Verify single event signature
    public static boolean verify(Event event, String orgSecret) { ... }
    public static void assertVerified(Event event, String orgSecret) throws VerificationException { ... }
}

public final class AuditStream {
    public AuditStream(String orgSecret, String source) { ... }
    public Event append(Event event) { ... }      // signs + links prev_id
    public ChainVerificationResult verify() { ... }
    public void rotateKey(String newSecret) { ... }  // inserts AUDIT_KEY_ROTATED event
}

public record ChainVerificationResult(
    boolean valid,
    @Nullable String firstTampered,
    List<String> gaps,
    int tamperedCount
) {}
```

### Security Requirements

- `orgSecret` MUST NEVER appear in exception messages, `toString()`, or logs
- All comparisons use `MessageDigest.isEqual()` (constant-time) to prevent timing attacks
- Empty/whitespace `orgSecret` is rejected immediately before any crypto operation
- `SigningException` carries only a `reason` string — never the key value

### Acceptance Criteria

- `EventSigner.sign(event, "test-key", null)` produces event with `checksum` + `signature`
- `EventSigner.verify(signed, "wrong-key")` returns `false`
- `AuditStream` correctly detects deletion (gap), reordering, and tampered content
- Timing-attack test: verification time does not statistically correlate with position of first mismatch

---

## Phase 9 — Privacy: PII Redaction

**Module:** `agentobs-privacy`  
**Python source:** `agentobs/redact.py`

### Sensitivity Enum

```java
public enum Sensitivity {
    LOW(0), MEDIUM(1), HIGH(2), PII(3), PHI(4);

    public boolean isAtLeast(Sensitivity threshold) {
        return this.order >= threshold.order;
    }
}
```

### PII Type Registry

```java
public final class PiiTypes {
    public static final Set<String> KNOWN = Set.of(
        "credit_card", "date_of_birth", "email", "financial_id",
        "ip_address", "medical_id", "name", "phone", "ssn", "address"
    );
}
```

### Redactable Wrapper

```java
public final class Redactable {
    private final Object value;           // intentionally Object, not exposed via API
    private final Sensitivity sensitivity;
    private final Set<String> piiTypes;

    // Constructor — value is kept but NEVER surfaced in toString/equals/hashCode
    public Redactable(Object value, Sensitivity sensitivity) { ... }
    public Redactable(Object value, Sensitivity sensitivity, Set<String> piiTypes) { ... }

    public Sensitivity getSensitivity() { return sensitivity; }
    public Set<String> getPiiTypes() { return piiTypes; }

    // Security: value is HIDDEN
    @Override public String toString() { return "<Redactable:" + sensitivity.name().toLowerCase() + ">"; }
    @Override public int hashCode() { return sensitivity.hashCode(); }  // NOT value-based
    // equals: identity only
}
```

### RedactionPolicy

```java
public final class RedactionPolicy {
    private final Sensitivity minSensitivity;
    private final String redactedBy;

    public RedactionResult apply(Event event) {
        // Recursively walk payload; replace Redactable values meeting threshold
        // with "[REDACTED:<level>]"; count replacements
    }
}

public record RedactionResult(Event event, int redactionCount) {}
```

### Acceptance Criteria

- `Redactable.toString()` never exposes the wrapped value
- `policy.apply(event)` replaces all `Redactable` fields at or above minSensitivity
- Nested payload structures are fully scanned recursively
- `containsPii(event)` returns `false` after applying a PII-level policy

---

## Phase 10 — Built-in Exporters

### Phase 10A — JSONLExporter

**Module:** `agentobs-exporters-jsonl`

```java
public final class JsonlExporter implements Exporter {
    private final Path outputPath;
    // Uses java.nio.file.Files.newBufferedWriter with StandardOpenOption.APPEND
    // One JSON line per event; flush after each batch

    @Override
    public synchronized void exportBatch(List<Event> events) throws ExportException { ... }
}
```

Default path: `agentobs_events.jsonl` relative to working directory.

### Phase 10B — ConsoleExporter

**Module:** `agentobs-exporters-console`

```java
public final class ConsoleExporter implements Exporter {
    // Pretty-print with ANSI colours in TTY; plain JSON when non-TTY
    // Uses System.out (configurable PrintStream)
    @Override
    public void exportBatch(List<Event> events) { ... }
}
```

### Phase 10C — OTLPExporter + WebhookExporter

**Module:** `agentobs-exporters-http` (requires Java 11+ `java.net.http.HttpClient`)

```java
public final class OtlpExporter implements AsyncExporter {
    private final String endpoint;   // https://collector:4318/v1/traces
    private final HttpClient client; // java.net.http.HttpClient
    
    // Format selection:
    //   event WITH trace_id → resourceSpans (OTLP span format)
    //   event WITHOUT trace_id → resourceLogs (OTLP log record format)
    
    @Override
    public CompletableFuture<Void> exportBatchAsync(List<Event> events) { ... }
}

public final class WebhookExporter implements AsyncExporter {
    // HTTP POST List<Event> as JSON array to endpoint
    // SSRF guard: validate URL scheme (https only in production) + reject private IPs
    private final String endpoint;
    private final Map<String, String> headers;
}
```

**SSRF protection** (matching Python implementation):
- Validate URL scheme is `https://` (or `http://` with explicit `allowPrivateAddresses=true`)
- Reject URLs resolving to private/loopback IPs unless explicitly whitelisted

### Phase 10D — Datadog + Grafana Exporters

**Modules:** `agentobs-exporters-datadog`, `agentobs-exporters-grafana`

```java
// Datadog uses stdlib HttpClient — no datadog-agent-java required
public final class DatadogExporter implements AsyncExporter { ... }

// Grafana Loki — Loki HTTP API (push endpoint)
public final class GrafanaLokiExporter implements AsyncExporter { ... }
```

### Acceptance Criteria

- `JsonlExporter` writes one valid JSON line per event to the target file
- `OtlpExporter` produces well-formed OTLP/JSON for both span and log-record paths
- SSRF guard rejects `http://169.254.169.254/...` (AWS metadata endpoint)
- All HTTP exporters support configurable connection timeout and retry count

---

## Phase 11 — OpenTelemetry Bridge

**Module:** `agentobs-otel-bridge`  
**Optional dependency:** `io.opentelemetry:opentelemetry-api`

```java
// Converts Event → real OTel SDK span using globally configured TracerProvider
public final class OtelBridgeExporter implements Exporter {
    private final io.opentelemetry.api.trace.Tracer tracer;
    
    // Applies gen_ai.* semantic convention attributes
    // Sets span context from event.traceId / event.spanId / event.parentSpanId
    @Override
    public void exportBatch(List<Event> events) { ... }
}
```

Also provides `AgentObsOtelContextPropagator` for W3C `traceparent` header injection/extraction.

---

## Phase 12 — Governance + Consumer Registry

**Module:** `agentobs-governance`  
**Python source:** `agentobs/governance.py`, `agentobs/consumer.py`, `agentobs/deprecations.py`

### EventGovernancePolicy

```java
public final class EventGovernancePolicy {
    private final Set<String> blockedTypes;
    private final Set<String> warnDeprecated;
    private final List<Function<Event, Optional<String>>> customRules;
    private final boolean strictUnknown;
    
    public void checkEvent(Event event) throws GovernanceViolationException { ... }
}
```

### ConsumerRegistry

```java
public final class ConsumerRegistry {
    // Thread-safe — backed by CopyOnWriteArrayList
    public ConsumerRecord register(String toolName, List<String> namespaces, String schemaVersion) { ... }
    public void assertCompatible() throws IncompatibleSchemaException { ... }
}
```

### DeprecationRegistry

```java
public final class DeprecationRegistry {
    public void markDeprecated(String eventType, String since, String sunset, @Nullable String replacement) { ... }
    public Optional<DeprecationNotice> get(String eventType) { ... }
    public void warnIfDeprecated(String eventType) { ... }  // uses SLF4J warn
}
```

---

## Phase 13 — Provider Integrations

**Python source:** `agentobs/integrations/`

Each integration auto-extracts `TokenUsage`, `ModelInfo`, and `CostBreakdown` from provider
responses and populates the active `SpanContext`.

### OpenAI Integration

**Module:** `agentobs-integration-openai`  
**Optional dependency:** `com.theokanning.openai-gpt3-java:client` or `io.github.sashirestela:simple-openai`

```java
public final class OpenAiInstrumentation {
    // Wraps the OpenAI Java client's chat completion response
    public static NormalizedResponse normalize(ChatCompletionResponse response) { ... }
    
    // Field mapping (matches Python SDK):
    // response.model              → ModelInfo.name
    // usage.promptTokens          → TokenUsage.inputTokens
    // usage.completionTokens      → TokenUsage.outputTokens
    // usage.cachedTokens          → TokenUsage.cachedTokens
    // usage.reasoningTokens       → TokenUsage.reasoningTokens
}
```

### Additional Provider Normalizers

| Module | Provider | Key extraction |
|---|---|---|
| `agentobs-integration-anthropic` | Anthropic Java SDK | `inputTokens`, `outputTokens`, `cacheCreationInputTokens` |
| `agentobs-integration-groq` | Groq SDK | `promptTokens`, `completionTokens` |
| `agentobs-integration-ollama` | Ollama Java client | `promptEvalCount`, `evalCount` |
| `agentobs-integration-together` | Together AI SDK | Standard OpenAI-compatible fields |

### Built-in Pricing Table

`io.agentobs.core.pricing.PricingTable` — static lookup for known model prices
(per input/output token per million). Updated as a resource file `pricing.json` bundled in the JAR.

---

## Phase 14 — Framework Integrations

### LangChain4j Integration

**Module:** `agentobs-integration-langchain4j`  
**Optional dependency:** `dev.langchain4j:langchain4j-core`

```java
// Implements dev.langchain4j.data.message callback interface
public final class AgentObsLangChain4jListener implements ... {
    // Automatically instruments every model call within a LangChain4j chain
    // Creates span on model-call start; populates token usage on completion
}
```

### Spring AI Integration

**Module:** `agentobs-integration-spring-ai`  
**Optional dependency:** `org.springframework.ai:spring-ai-core`

```java
// Spring AI Advisor pattern — wraps any AdvisedChatClient
public final class AgentObsAdvisor implements ... {
    // Emits TRACE_SPAN_STARTED on request, TRACE_SPAN_COMPLETED on response
}

// Spring Boot Auto-configuration
@AutoConfiguration
public class AgentObsAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public Tracer agentObsTracer(AgentObsProperties properties) { ... }
}
```

### Spring Boot Actuator Integration

Exposes `/actuator/agentobs` endpoint showing:
- Current configuration
- Exporter health status
- Consumer registry state
- Event throughput metrics

---

## Phase 15 — Schema Validation

**Module:** `agentobs-validation`  
**Optional dependency:** `com.networknt:json-schema-validator`

```java
public final class JsonSchemaValidator {
    // Loads schemas/v1.0/schema.json from classpath
    // Falls back to structural validation (no external dep) if networknt is absent
    public void validate(Event event) throws SchemaValidationException { ... }
}
```

Matches Python's dual-mode: full JSON Schema Draft 2020-12 with `networknt`;
structural stdlib fallback without.

---

## Phase 16 — CLI Tooling

**Module:** `agentobs-cli`  
**Dependency:** `info.picocli:picocli` (shadows into a self-contained fat JAR)

```
agentobs-cli.jar check-compat events.jsonl
agentobs-cli.jar validate events.jsonl
agentobs-cli.jar audit-chain events.jsonl --org-secret $SECRET
agentobs-cli.jar migration-roadmap
agentobs-cli.jar version
```

| Command | Function |
|---|---|
| `check-compat` | Validates JSONL file against current schema; exits non-zero on failure |
| `validate` | Strict JSON Schema Draft 2020-12 validation (requires networknt on classpath) |
| `audit-chain` | Verifies HMAC chain integrity; reports first tampered event and all gaps |
| `migration-roadmap` | Prints v1→v2 deprecation roadmap table |
| `version` | Prints SDK version and schema version |

Distributed as:
1. Runnable fat JAR (`agentobs-cli-VERSION-all.jar`)
2. Native binary via GraalVM native-image (GitHub Actions artifact)
3. Homebrew formula (Phase 18)

---

## Phase 17 — BOM and Maven Central Distribution

**Module:** `agentobs-bom`

```xml
<!-- BOM POM — consumers import this to pin all agentobs versions -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.agentobs</groupId>
      <artifactId>agentobs-core</artifactId>
      <version>${agentobs.version}</version>
    </dependency>
    <dependency>
      <groupId>io.agentobs</groupId>
      <artifactId>agentobs-security</artifactId>
      <version>${agentobs.version}</version>
    </dependency>
    <!-- ... all modules -->
  </dependencies>
</dependencyManagement>
```

**Consumer usage:**

```xml
<!-- pom.xml — import BOM once, then add modules without version -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.agentobs</groupId>
      <artifactId>agentobs-bom</artifactId>
      <version>0.1.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- Core always included — zero deps -->
  <dependency>
    <groupId>io.agentobs</groupId>
    <artifactId>agentobs-core</artifactId>
  </dependency>
  <!-- Optional: HMAC signing -->
  <dependency>
    <groupId>io.agentobs</groupId>
    <artifactId>agentobs-security</artifactId>
  </dependency>
</dependencies>
```

---

## Phase 18 — Hardening, Documentation, and 1.0.0 Release

### Quality Gates

| Metric | Target |
|---|---|
| Unit test coverage | ≥ 90% line + branch (`agentobs-core`, `-security`, `-privacy`) |
| Integration test coverage | ≥ 80% |
| Javadoc coverage | 100% on public API |
| Static analysis | Zero SpotBugs HIGH/CRITICAL; zero PMD violations |
| OWASP dependency check | No known CVEs in `agentobs-core` transitive closure |
| Performance | 10,000 spans/sec sustained on a single core (JMH benchmark) |

### Test Categories

| Category | Scope |
|---|---|
| Unit | All value objects, validators, ULID generator, canonical JSON |
| Integration | Full span → event → export round-trip; agent nesting |
| Security | HMAC sign/verify; timing-attack resistance; secret non-leakage |
| Privacy | Redaction at every sensitivity level; nested payload scanning |
| Schema | JSON round-trip determinism; all 36 event types |
| Compatibility | Events produced by Java SDK validate against Python SDK's schema |
| Performance | JMH: ULID generation, JSON serialization, span dispatch throughput |

### Documentation

- README with 5-minute quickstart (mirrors Python README structure)
- Javadoc hosted on GitHub Pages
- `docs/` directory with architecture guide, namespace reference, migration guide
- Example projects for Spring Boot, Quarkus, and plain Java

---

## Cross-Cutting Concerns

### Thread Safety

| Component | Mechanism |
|---|---|
| `UlidGenerator` | `synchronized` block on monotonic state |
| `AgentObsConfig` | `volatile` fields + `ReentrantReadWriteLock` on batch mutations |
| `SpanStack` | `ThreadLocal<Deque<SpanContext>>` |
| `ConsumerRegistry` | `CopyOnWriteArrayList` |
| `DeprecationRegistry` | `ConcurrentHashMap` |
| `JsonlExporter` | `synchronized exportBatch` + `BufferedWriter.flush()` |

### Logging

SLF4J `slf4j-api` is the **only** mandatory logging dependency across all modules.
No SLF4J binding is pulled in by the SDK — consumers provide their own (Logback, Log4j2, etc.).

Logging levels:
- `TRACE` — span enter/exit (high volume, off by default)
- `DEBUG` — config load, exporter initialization
- `INFO` — major lifecycle events (first span emitted, exporter connected)
- `WARN` — deprecated event types, export errors in `WARN` mode
- `ERROR` — unrecoverable export failures, signing failures

### Null Safety Strategy

- `@NonNull` on all parameters that must not be null
- `@Nullable` on all optional fields
- `Optional<T>` is **not** used in core value types (records use `@Nullable` for cleaner records)
- `Optional<T>` is used in query APIs (e.g., `ConsumerRegistry.find(...)`)

### Package Naming

Base package: `io.agentobs`

| Maven module | Java package |
|---|---|
| `agentobs-core` | `io.agentobs.core` |
| `agentobs-security` | `io.agentobs.security` |
| `agentobs-privacy` | `io.agentobs.privacy` |
| `agentobs-exporters-*` | `io.agentobs.exporters.*` |
| `agentobs-otel-bridge` | `io.agentobs.otel` |
| `agentobs-integration-*` | `io.agentobs.integrations.*` |
| `agentobs-governance` | `io.agentobs.governance` |
| `agentobs-cli` | `io.agentobs.cli` |

### Java Module System (JPMS)

All core modules ship `module-info.java` descriptors for strong encapsulation:

```java
// agentobs-core/src/main/java/module-info.java
module io.agentobs.core {
    exports io.agentobs.core;
    exports io.agentobs.core.namespaces.trace;
    exports io.agentobs.core.namespaces.cost;
    // ... all namespace packages
    
    // Internal — not exported
    // io.agentobs.core.json    (canonical writer)
    // io.agentobs.core.ulid    (internals)
}
```

---

## Feature Parity Matrix

| Python Feature | Java Equivalent | Module | Phase |
|---|---|---|---|
| `Event` envelope | `Event` + `Event.Builder` | `agentobs-core` | 2 |
| `Tags` | `Tags` | `agentobs-core` | 2 |
| `EventType` (36 types) | `EventType` enum | `agentobs-core` | 2 |
| `ULID` generator | `UlidGenerator` | `agentobs-core` | 1 |
| `Event.to_json()` / `from_json()` | `Event.toJson()` / `Event.fromJson()` | `agentobs-core` | 2 |
| `Event.validate()` | `Event.validate()` / `EventValidator` | `agentobs-core` | 3 |
| All 42 payload classes | All payload records/classes | `agentobs-core` | 4 |
| `configure()` | `AgentObs.configure()` | `agentobs-core` | 5 |
| `tracer.span()` | `AgentObs.tracer().span()` | `agentobs-core` | 6 |
| `tracer.agent_run()` | `AgentObs.tracer().agentRun()` | `agentobs-core` | 6 |
| `tracer.agent_step()` | `AgentObs.tracer().agentStep()` | `agentobs-core` | 6 |
| `EventStream` | `EventStream` | `agentobs-core` | 7 |
| `sign()` / `verify()` / `AuditStream` | `EventSigner` / `AuditStream` | `agentobs-security` | 8 |
| `Redactable` / `RedactionPolicy` | `Redactable` / `RedactionPolicy` | `agentobs-privacy` | 9 |
| `ConsoleExporter` | `ConsoleExporter` | `agentobs-exporters-console` | 10 |
| `JSONLExporter` | `JsonlExporter` | `agentobs-exporters-jsonl` | 10 |
| `OTLPExporter` | `OtlpExporter` | `agentobs-exporters-http` | 10 |
| `WebhookExporter` | `WebhookExporter` | `agentobs-exporters-http` | 10 |
| `DatadogExporter` | `DatadogExporter` | `agentobs-exporters-datadog` | 10 |
| `GrafanaExporter` | `GrafanaLokiExporter` | `agentobs-exporters-grafana` | 10 |
| `OTelBridgeExporter` | `OtelBridgeExporter` | `agentobs-otel-bridge` | 11 |
| `EventGovernancePolicy` | `EventGovernancePolicy` | `agentobs-governance` | 12 |
| `ConsumerRegistry` | `ConsumerRegistry` | `agentobs-governance` | 12 |
| `DeprecationRegistry` | `DeprecationRegistry` | `agentobs-governance` | 12 |
| OpenAI integration | `OpenAiInstrumentation` | `agentobs-integration-openai` | 13 |
| LangChain integration | `AgentObsLangChain4jListener` | `agentobs-integration-langchain4j` | 14 |
| JSON Schema validation | `JsonSchemaValidator` | `agentobs-validation` | 15 |
| CLI (`check-compat`, `validate`, `audit-chain`) | `agentobs-cli` | `agentobs-cli` | 16 |
| Pydantic model layer | N/A — Java's type system provides equivalent | — | — |
| `models.py` `EventModel` | `Event` record (Java records are already typed) | `agentobs-core` | 2 |

---

## Open Design Questions

1. **Async span API:** Should `SpanContext` also implement `AsyncCloseable` for environments
   using Project Reactor or CompletableFuture chains? Recommendation: provide
   `AgentObs.tracer().spanAsync(...)` returning `Mono<SpanContext>` as a separate
   opt-in `agentobs-integration-reactor` module in Phase 13+.

2. **GraalVM native-image metadata:** Records serialized via the canonical JSON writer
   work natively; reflection-based Jackson in exporters requires `reflect-config.json`.
   Generate via `agentobs-native-hints` module in Phase 18.

3. **Quarkus / Micronaut extensions:** Defer to community after 1.0.0. Both frameworks
   can consume the plain Maven artifacts; optional `@ApplicationScoped` beans can be
   provided in extension modules.

4. **Kafka consumer:** Python SDK supports `EventStream.from_kafka()`. Java equivalent
   would be in `agentobs-integration-kafka` using the official Kafka Java client.
   Targeted for Phase 13 alongside other provider integrations.

---

## Release Cadence

| Milestone | Version | Phases Included |
|---|---|---|
| Developer Preview 1 | `0.1.0-alpha` | Phases 0–4 (core + namespaces) |
| Developer Preview 2 | `0.2.0-alpha` | Phases 5–7 (config + tracer + stream) |
| Beta 1 | `0.3.0-beta` | Phases 8–10 (security + privacy + exporters) |
| Beta 2 | `0.4.0-beta` | Phases 11–13 (OTel + governance + provider integrations) |
| Release Candidate | `1.0.0-rc1` | Phases 14–16 (framework integrations + CLI) |
| General Availability | `1.0.0` | Phase 17–18 (BOM + hardening + docs) |

---

## Appendix A — Directory Structure (Target)

```
agentobs-java/
├── pom.xml
├── agentobs-bom/pom.xml
├── agentobs-core/
│   ├── pom.xml
│   └── src/
│       ├── main/java/io/agentobs/core/
│       │   ├── AgentObs.java
│       │   ├── Event.java
│       │   ├── Tags.java
│       │   ├── EventType.java
│       │   ├── SchemaVersion.java
│       │   ├── exception/
│       │   │   ├── AgentObsException.java
│       │   │   ├── SchemaValidationException.java
│       │   │   ├── UlidException.java
│       │   │   ├── SerializationException.java
│       │   │   ├── DeserializationException.java
│       │   │   ├── EventTypeException.java
│       │   │   └── ExportException.java
│       │   ├── ulid/
│       │   │   └── UlidGenerator.java
│       │   ├── json/
│       │   │   └── CanonicalJsonWriter.java
│       │   ├── config/
│       │   │   ├── AgentObsConfig.java
│       │   │   └── ExportErrorPolicy.java
│       │   ├── tracer/
│       │   │   ├── Tracer.java
│       │   │   ├── SpanContext.java
│       │   │   ├── SpanContextBuilder.java
│       │   │   ├── AgentRunContext.java
│       │   │   ├── AgentStepContext.java
│       │   │   └── SpanStack.java
│       │   ├── stream/
│       │   │   ├── EventStream.java
│       │   │   ├── Exporter.java
│       │   │   └── AsyncExporter.java
│       │   ├── namespaces/
│       │   │   ├── trace/   (13 classes)
│       │   │   ├── cost/    (3 classes)
│       │   │   ├── cache/   (1 class)
│       │   │   ├── eval/    (2 classes)
│       │   │   ├── guard/   (1 class)
│       │   │   ├── fence/   (1 class)
│       │   │   ├── prompt/  (1 class)
│       │   │   ├── redact/  (1 class)
│       │   │   ├── diff/    (1 class)
│       │   │   ├── template/(1 class)
│       │   │   └── audit/   (1 class)
│       │   ├── actor/
│       │   │   └── ActorContext.java
│       │   └── validate/
│       │       └── EventValidator.java
│       └── test/java/io/agentobs/core/
│           └── ... (mirrors main structure)
├── agentobs-security/
│   └── src/main/java/io/agentobs/security/
│       ├── EventSigner.java
│       ├── AuditStream.java
│       └── ChainVerificationResult.java
├── agentobs-privacy/
│   └── src/main/java/io/agentobs/privacy/
│       ├── Sensitivity.java
│       ├── PiiTypes.java
│       ├── Redactable.java
│       ├── RedactionPolicy.java
│       └── RedactionResult.java
└── ... (remaining modules)
```

---

## Appendix B — Java 17 vs Java 21 Feature Usage

| Feature | Java version | Used in |
|---|---|---|
| Records | 16 (final) | All value objects and payload types |
| Sealed classes | 17 (final) | Exception hierarchy, SensitivityLevel |
| Pattern matching `instanceof` | 16 (final) | JSON writer, payload dispatch |
| Pattern matching `switch` | 21 (final) | Payload variant dispatch, CLI command routing |
| Virtual threads | 21 | HTTP exporter send, CLI audit-chain verification |
| Text blocks | 15 (final) | Test fixtures, default templates |
| `Map.of()` / `Set.of()` | 9 | PiiTypes, namespace registries |
| `Stream` API enhancements | 16 | EventStream filtering |
| `HttpClient` | 11 | OTLPExporter, WebhookExporter |

---

*End of AgentOBS Java SDK Phase-wise Implementation Plan*  
*Author: Sriram | Date: March 6, 2026 | Status: Active Draft*
