---
title:  "Micrometer vs Otel for metrics in Spring ecosystem"
date:   2025-11-28 14:07:50 +0530
categories:
  - blog
tags:
  - Spring Boot
  - Otel
---


## Introduction and Problem Statement
###  Micrometer: The Facade

With Spring Boot 3, the Spring team introduced the **Micrometer Observation API**
Micrometer is often described as "SLF4J for metrics." In the Spring ecosystem, Micrometer is not merely a library; it is the default instrumentation engine embedded within the Spring Framework itself.
- **Instrumentation Philosophy:** Micrometer relies on "Manual Instrumentation" and "Library Instrumentation." Spring Boot Actuator auto-configures `MeterBinders`—specialized beans that hook into specific framework events (e.g., an HTTP request handling event or a JDBC connection pool lease). This ensures that metrics are generated only when the framework actually processes a request.
- **Data Model:** Micrometer uses a dimensional metrics model. A metric consists of a name and a set of key-value pairs called tags. This allows for rich querying, such as aggregating all requests by URI or splitting them by status code.
- **Aggregation:** Crucially, Micrometer performs **client-side aggregation**. When a timer records a duration, Micrometer immediately updates the relevant histogram buckets, summaries, or max values in memory. The backend (e.g., Prometheus) typically scrapes these pre-calculated aggregates. This reduces the volume of data transmitted over the network but places the burden of aggregation CPU cost on the application itself.
- **Spring Awareness:** Micrometer is deeply "Spring-aware." It understands the Spring bean lifecycle, the `ApplicationContext`, and Spring's event publication mechanisms. This allows it to capture highly specific metrics, such as the number of scheduled tasks in a `@Scheduled` annotation or the size of a specific Spring Cache abstraction, which an external agent might miss or misinterpret.

### OpenTelemetry: The Unified Ecosystem

OpenTelemetry operates as a platform-agnostic ecosystem.
- **Instrumentation Philosophy:** OpenTelemetry emphasizes "Zero-Code" instrumentation via the Java Agent. The agent attaches to the JVM at startup using the `-javaagent` flag and uses the Java Instrumentation API to modify bytecode on the fly. It injects telemetry-gathering code into known libraries (like Apache HttpClient, JDBC drivers, or Servlet containers).
- **Data Model:** OTel uses a hierarchical resource model. Metrics are associated with a "Resource" (the entity producing the telemetry, e.g., a specific pod, service, or host). The model supports complex data types like Exponential Histograms, which offer better compression and accuracy for high-cardinality latency data than Micrometer's fixed buckets.
- **Aggregation:** While OTel SDKs can perform client-side aggregation, the architecture is heavily designed around the **OpenTelemetry Collector**—an intermediary service that receives raw or semi-aggregated data, processes it (filtering, batching, attribute modification), and exports it to the final backend. Moving some processing overhead off the application.
- **Language Agnosticism:** OTel's primary strength is consistency. It enforces strict "Semantic Conventions" for metric names and attributes. This ensures that a dashboard built for a Node.js service works seamlessly for a Java service, a significant advantage in polyglot microservice environments.

### Metrics Duplication

The duplication of metrics arises from the simultaneous operation of these two independent instrumentation layers. When `spring-boot-starter-actuator` is present alongside the `opentelemetry-javaagent` (or the starter with default settings), both systems attempt to measure the same operations.

Both layers measure the same event (an HTTP request), but they process it separately, aggregate it in different memory spaces, and export it as two distinct metrics. This effectively doubles the storage cost, network bandwidth, and processing overhead without adding significant informational value. Furthermore, this duplication extends to other layers, such as JDBC, where Micrometer tracks connection pool usage via `hikaricp.connections` while OTel tracks it via `db.client.connections`.

## What will be captured?

### JVM Metrics (Memory, GC, Threads)

The JVM is the runtime engine for Spring Boot, making these metrics vital for operational health.

**Micrometer (via `JvmMemoryMetrics`, `JvmGcMetrics`):**

- **Memory:** Captures `jvm.memory.used`, `jvm.memory.committed`, and `jvm.memory.max`. It breaks these down by memory pool (Eden, Survivor, Old Gen, Metaspace, Code Cache) using tags like `area` (heap/non-heap) and `id` (pool name). This granularity is essential for diagnosing memory leaks or tuning heap sizes.   
- **Garbage Collection:** Captures `jvm.gc.pause` (Timer), `jvm.gc.memory.promoted`, and `jvm.gc.memory.allocated`. A key advantage of Micrometer is that it treats GC pauses as **Timers**, not just counters. This allows operators to calculate the 95th or 99th percentile of GC pause durations, providing a much clearer picture of latency impact than a simple average. It also tags metrics with the specific GC cause (e.g., `Allocation Failure`, `G1 Evacuation Pause`).   
- **Threads:** Captures `jvm.threads.live`, `jvm.threads.peak`, and `jvm.threads.daemon`. Crucially, it provides breakdowns by thread state (Runnable, Blocked, Waiting) via `jvm.threads.states`, which is invaluable for diagnosing thread starvation or deadlock scenarios.   
    
**OpenTelemetry (via OTel Java Instrumentation):**

- **Memory:** Captures `jvm.memory.used`, `jvm.memory.committed`, and `jvm.memory.limit`. OTel has recently aligned its naming closer to Micrometer's but strictly follows "Semantic Conventions." It uses attributes like `jvm.memory.pool.name` and `jvm.memory.type` (heap/non_heap).   
- **Garbage Collection:** Captures `jvm.gc.duration` (Histogram). This aligns with the OTel philosophy of using histograms for duration data. While powerful, the default bucket boundaries in OTel might not always be optimized for the sub-millisecond pauses seen in modern GCs like ZGC or Shenandoah.
- **Threads:** Captures `jvm.thread.count`. While useful, the default set in OTel historically provided less detailed breakdown of thread states compared to Micrometer's default binders, although this gap is closing with newer semantic conventions.

**Comparison Insight:** Micrometers binders are often updated in lock-step with Spring Boot releases, ensuring compatibility with new JVM features (like Virtual Threads in Java 21). The `jvm.gc.pause` metric in Micrometer is particularly valuable for performance tuning. OTel's JVM metrics are robust but serve as a general-purpose baseline rather than a specialized JVM diagnostic tool.

### HTTP/Web Metrics

This layer measures the traffic entering the application.

**Micrometer (`http.server.requests`):**

- **Mechanism:** Uses `WebMvcMetricsFilter` (for blocking) or `WebFluxMetricsFilter` (for reactive).
- **Dimensions (Tags):** `uri` (templated, e.g., `/users/{id}`), `method` (GET/POST), `status` (200, 500), `exception` (class name), `outcome` (SUCCESS, SERVER_ERROR).
- **Richness:** Micrometer automatically handles path templating to avoid cardinality explosions. It uses the Spring MVC mapping to ensure that `/users/1` and `/users/2` are aggregated under the single tag value `/users/{id}`. This is a critical feature for preventing memory exhaustion in the metrics backend.   

**OpenTelemetry (`http.server.request.duration`):**

- **Mechanism:** Servlet Filter instrumentation or Java Agent bytecode injection at the Servlet container level.
- **Dimensions (Attributes):** `http.method`, `http.status_code`, `http.route`, `http.scheme`, `net.host.name`.
- **Richness:** OTel strictly follows the HTTP Semantic Conventions. The `http.route` attribute serves the same purpose as Micrometer's `uri` tag.
- **Risk:** If the OTel Agent fails to correctly identify the framework's routing template (which can happen with custom routing logic or complex Spring configurations), it might fall back to using the raw URL path. This results in "High Cardinality," where every unique ID in a URL creates a new metric time series, potentially crashing the monitoring backend or incurring massive costs.   

**Comparison Insight:** Micrometer's `http.server.requests` is functionally equivalent to OTel's `http.server.request.duration`. However, Micrometer's integration with **Spring Security** is often tighter. Micrometer can easily tag metrics with the authenticated user's role or principal if customized, which is significantly harder to achieve with the OTel Agent that sits outside the Spring Security context.

### JDBC and Database Metrics

This area highlights the most significant divergence between the "Agent" and "Library" approaches.

**Micrometer (`hikaricp.connections`, `jdbc.connections`):**
- **Source:** Spring Boot auto-configures `DataSourcePoolMetadataProviders`. If the default HikariCP is used, Micrometer binds directly to the `HikariPoolMXBean`.
- **Metrics:** `hikaricp.connections.active`, `hikaricp.connections.idle`, `hikaricp.connections.pending`, `hikaricp.connections.usage` (Timer).
- **Analysis:** These metrics are extremely accurate for monitoring the _health of the connection pool_. They tell you if the pool is exhausted or if threads are waiting for connections. However, Micrometer does **not** record per-query latency by default. It treats the database largely as a black box resource.   

**OpenTelemetry (`db.client.connections`):**

- **Source:** The Java Agent intercepts JDBC driver calls (e.g., `java.sql.Connection`, `java.sql.PreparedStatement`).
- **Metrics:** `db.client.connections.usage`, `db.client.connections.max`.
- **Tracing Superpower:** The OTel Agent automatically generates **Tracing Spans** for every SQL query (`SELECT * FROM users`). These spans contain the actual sanitized SQL statement as an attribute (`db.statement`). While this is tracing data, not metric data, modern backends can derive metrics from these traces (e.g., "Average latency for query X").
- **Analysis:** For _connection pooling metrics_, Micrometer is more reliable because it queries the pool (HikariCP) directly. The OTel Agent tries to infer usage by observing open/close calls, which can sometimes drift from the pool's internal truth. However, for _query performance_ (time taken per SQL statement), OTel's auto-instrumentation is far superior.

**Comparison Insight:** If the requirement is simply to monitor "basic metrics" like whether the database is up and if the connection pool is sized correctly, Micrometer is sufficient and more accurate. If deep debugging of slow queries is required, OTel's tracing capabilities are indispensable.

## Performance and Overhead Analysis

### Micrometer Overhead

Micrometer operates as a library within the application's process.

- **CPU:** The overhead is extremely low. Recording a metric in Micrometer typically involves a hash map lookup (for tags) and an atomic increment (using `LongAdder` or `DoubleAdder`). This path is highly optimized for the JVM and generates zero garbage in the steady state.   
- **Memory:** Overhead is moderate but bounded. Micrometer stores aggregated statistics (e.g., histogram buckets) in memory. The memory footprint is proportional to the cardinality of the metrics (number of unique tag combinations). As long as cardinality is managed (e.g., by templating URIs), the memory usage remains stable and predictable.
- **Startup Time:** The impact is negligible. Micrometer initializes quickly along with the Spring context.

### OpenTelemetry Agent Overhead

The OTel Java Agent is a comprehensive bytecode manipulator, which inherently introduces more complexity.

- **CPU:** Research and benchmarks indicate that the OTel Agent can increase CPU usage by 10% to 40% under high load compared to a baseline, primarily due to the overhead of intercepting method calls, managing context propagation, and creating/processing span objects.
- **Memory:** The Agent requires higher heap usage. It must allocate objects for every span, event, and metric data point. It also maintains internal buffers for exporting data. The "Class Loading" overhead is also significant; the agent must scan and retransform classes as they are loaded, which can increase PermGen/Metaspace usage.   
- **Startup Time:** There is a noticeable delay in application startup (often increasing by 15-20%) because the agent must instrument thousands of classes during the boot phase.
- **Throughput:** In extremely high-throughput scenarios (millions of requests per second), the agent's overhead becomes a measurable bottleneck, potentially reducing maximum throughput by 5-10% compared to a Micrometer-only baseline.

### Memory Efficiency: The Histogram Dilemma

While Micrometer is CPU-efficient, OpenTelemetry's **Data Model** is often more Memory-efficient for **Histograms**.

- **Micrometer (Legacy):** To calculate a P99 latency, Micrometer traditionally uses `HdrHistogram` (High Dynamic Range Histogram) or fixed SLA buckets. `HdrHistogram` is accurate but memory-heavy (can be kilobytes per metric). Fixed buckets (e.g., counting requests < 10ms, < 100ms) are memory-light but inaccurate if the buckets are defined incorrectly.
- **OpenTelemetry (Modern):** OTel supports **Exponential Histograms**. This data structure uses base-2 logarithmic buckets. It can represent values from nanoseconds to hours with high precision using a compact, sparse array representation.
- **The Hybrid Win:** By using Micrometer to _record_ the data and the OTLP Bridge to _export_ it, users can potentially configure the bridge to translate Micrometer's recordings into OTel's efficient histogram format, provided the backend supports it.
Using `micrometer-registry-otlp` incurs the standard Micrometer overhead plus the cost of serializing data to OTLP format (Protobuf) and sending it over HTTP/gRPC.

**Architectural implication:** Using the Starter allows the user to record via Micrometer but bridge the data to the OTel SDK. If the backend supports OTLP Exponential Histograms, the bridge can potentially map Micrometer's recording to this efficient format, giving the user the best of both worlds (simple API, efficient data structure).

### Network Protocol: OTLP vs. Prometheus

- **Prometheus (Text):** Verbose. `http_server_requests_seconds_bucket{le="0.1"} 5`. Heavy string processing. Large payload sizes.
- **OTLP (Binary):** Uses Protocol Buffers (Protobuf). Compresses heavily. Data is sent in batches over a persistent HTTP/2 or gRPC connection.
- **Efficiency:** OTLP can reduce network bandwidth for telemetry by 10x-20x compared to text-based formats, which is significant for high-traffic microservices.

## Conclusion

The architectural conflict between Micrometer and OpenTelemetry in the Spring Boot ecosystem is resolved not by choosing one over the other, but by assigning them distinct roles based on their engineering strengths.

**Micrometer** excels as the **Instrumentation Layer**
**OpenTelemetry** excels as the **Transport Layer**

 The Recommended Architecture: "Micrometer + OTel Bridge"

 Note: Most of the insights are from Gemini Deep Research. There will be mistakes, but this is for my own learning.
