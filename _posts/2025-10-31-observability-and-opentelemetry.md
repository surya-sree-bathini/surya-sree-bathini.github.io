---
title:  "Observability and OpenTelemetry"
date:   2025-10-31 14:15:50 +0530
categories:
  - blog
tags:
  - Spring Boot
  - Otel
---

Observability is the ability to understand what’s happening inside your systems by collecting and analyzing **metrics, logs, and traces**.
- **Traces:** Shows the full journey of a request as it moves through all your different services.
- **Metrics:** Numerical measurements over time (e.g., CPU usage, request count, error rates).
- **Logs:** Simple, timestamped text records of specific events (e.g., "User logged in," "Database connection failed").

As modern applications become more distributed (microservices, cloud-native, etc.), observability becomes crucial for debugging, performance monitoring, and gaining insights.
This is where **OpenTelemetry (OTel)** comes in, a unified, open-source framework for collecting telemetry data from your applications and infrastructure.

Unlike normal software development terminology, the terms in observability are a bit different
1. **Data**: it refers to the telemetry your system produces; **metrics**, **logs**, **traces**, or a combination of all three.
2. **Backend**: This is not your application backend, but a storage unit and a querying engine that collects, stores, and retrieves your telemetry data for analysis.
3. **Instrumentation:**  The process of configuring your application to be able to **generate, collect, and export telemetry data** (traces, logs, and metrics). 

# What Is OpenTelemetry (OTel)?

OpenTelemetry provides a **standardized way** to generate, collect, and export telemetry data (traces, metrics, and logs). It removes the need to use vendor-specific SDKs for observability.
At a high level, OTel is composed of:
- **APIs & SDKs** – for instrumenting code and generating telemetry data.
- **Collectors** – for receiving, processing, and exporting telemetry to various backends.
- **Exporters** – for sending data to storage or visualization systems like Prometheus, Grafana, or Signoz.

OpenTelemetry uses the **OpenTelemetry Protocol (OTLP)** — a vendor-neutral, high-performance protocol for transmitting telemetry data between SDKs, Collectors, and backends.
The diagram below shows how telemetry flows through an OTel pipeline; from your application code to visualization tools.

<img src="otel.excalidraw.png" alt="otel" style="width:100%;">

- **On the Left (Your App):** This is your application (e.g., a Java service). This is where data is born. You can generate this data automatically or by adding custom code.
- **In the Middle (The Collector):** This is the **OTel Collector**. It’s a powerful agent that acts as a central hub. It receives data from your apps, processes it (like batching, filtering, or adding info), and then exports it to one or more destinations.
- **On the Right (Your Tools):** This is where you actually _see_ and _query_ your data. These are your **Observability Tools** (like Grafana, SigNoz, Prometheus) and the **Storage Systems** they use (like ClickHouse, Elasticsearch).

# Instrumenting Your Application

There are a few different ways to instrument your application to generate telemetry data.
1. **Annotations & Manual Instrumentation** - You can add OTel annotations (like `@WithSpan`) in your code to mark key functions or operations for tracing. Manual instrumentation gives you full control over what gets traced or measured.
 2. **SDK Configuration** - The **OpenTelemetry SDK** provides configuration options to set up exporters (e.g., OTLP, Prometheus), context propagation, sampling, and resource attributes. This setup defines _how and where_ telemetry data is sent.
 3. **Out-of-the-Box Instrumentations**: OTel offers many **auto-instrumentations** for popular libraries and frameworks (HTTP clients, JDBC, gRPC, etc.). These automatically capture spans and metrics without changing your code.

## Getting Started with Java & Spring
In this blog at least, we will talk in terms of spring and Java for Observability. But OTel is language agnostic, meaning it can be used on any stack.
### Java Agent
The easiest way to start is with the **Java OTEL Agent**. This is a single `.jar` file you attach to your application when it runs. You don't have to change a single line of your code!
This is what I mean by out of the box instrumentation. In the picture we have Java agent sitting inside the JVM, meaning it uses the same processes as your application.
Click [here](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar) to download the jar.
1. Add `-javaagent:path/to/opentelemetry-javaagent.jar` and other config to your JVM startup arguments and launch your app:
    - Directly on the startup command:
        ```bash
        java -javaagent:path/to/opentelemetry-javaagent.jar -Dotel.service.name=your-service-name -jar myapp.jar
        ```
    - Via the `JAVA_TOOL_OPTIONS` and other environment variables:
		```bash
		export JAVA_TOOL_OPTIONS="-javaagent:path/to/opentelemetry-javaagent.jar" 
		export OTEL_SERVICE_NAME="your-service-name" 
		java -jar myapp.jar
		```
 For more information on Java Agent refer this [official documentation](https://opentelemetry.io/docs/zero-code/java/agent/).

### Spring OTel Starter and Micrometer

When using **Spring Boot 3.x or later**, observability is tightly integrated through **Micrometer** and the **Spring OTel Starter**.

**Micrometer**  
Micrometer provides a metrics abstraction layer that allows developers to record and publish metrics without depending on a specific monitoring system. As for my knowledge spring Boot’s Micrometer integration is spring aware, so it knows what are different metrics, dependencies and basically spring concepts. 

**Spring OTel Starter**  
The Spring OTel Starter connects Spring Boot applications with OpenTelemetry. It automatically configures OpenTelemetry components, instruments common Spring libraries, and bridges Micrometer metrics and tracing data into the OpenTelemetry data model for export.

```xml
<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>io.opentelemetry.instrumentation</groupId>
                <artifactId>opentelemetry-instrumentation-bom</artifactId>
                <version>2.21.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
</dependencyManagement>

  
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    endpoint: http://localhost:4318
```
Both micrometer and starter way of OTel are configured inside your application. These are what I call in-app instrumentation.
###  Annotations & API

- **Annotations:** OTel provides an `@WithSpan` annotation. You can add this to any method in your code. OTel will then automatically create a new "span" (a part of a trace) for that method, letting you see exactly how long your custom business logic took.
- **SDK Config & Modifying the API:** For full control, you can use the **OTel SDK** and **API** directly. This lets you manually create your own traces or add custom attributes. The Micrometer or Spring OTel Starter configuration(like setting `management.metrics.export.otlp.enabled=true` in Spring) lets you control _how_ the SDK behaves—like telling it _where_ to send its data (the "OTLP exporter") or how often to sample requests.
Both annotations and API configuration are part of your code just like all the other business logic. That is why we have annotations and manual instrumentation sitting alongside the code in JVM. 

Until now we have seen this part of the architecture.
<img src="instrumenting-application.png" alt="otel-in-app" style="width:100%;">

# Otel Collector

In last section we saw how to generate or configure your application such that we can expose the telemetry data of the application. Traditionally, each library handled its own exporting—either to storage backends or local zPages.
This approach has several limitations:
- Exporters and zPages must be reimplemented in every language.
- Some languages (e.g., Ruby, PHP) cannot efficiently perform in-process aggregation.
- Users must manually configure exporters and redeploy apps to change telemetry settings.
- Misconfiguration is common, and many teams hesitate to embed observability setup directly in application code.

To resolve the issues above, you can run OpenTelemetry Collector.
The OpenTelemetry Collector processes telemetry data (metrics, logs, and traces) through a well-defined pipeline composed of **receivers**, **processors**, and **exporters**.

<img src="otel-collector.png" alt="otel-collector" style="width:100%;">

## The Collector Pipeline

**Receivers**  
Receivers are responsible for **ingesting telemetry data** into the Collector. They can either **pull** data from an application or which **receive** data that the application **pushes** to the Collector. Examples include the `otlp`, `prometheus`, and `jaeger` receivers.

**Processors**  
Processors perform **intermediate processing** on the data before export. Common tasks include batching, filtering, attribute enrichment, or sampling. For example, the `batch` processor groups telemetry for efficient export, while the `attributes` processor adds or modifies metadata.

**Exporters**  
Exporters send the processed telemetry data to its final destination — typically a backend such as **Prometheus**, **Jaeger**, **Tempo**, or **OpenSearch**. Exporters define where and how your telemetry data is stored and visualized.

In the diagram we see that there are many receivers, processors and exporters within the same collector. It is to imply that different providers have their own set of receivers and exporters, for example Prometheus. But it does not imply every collector has it.

Collector can have one or more receivers which can receive the data from one or more applications. Processors can be one or more parallel or chained within the pipeline. Similarly, we can have one or more exporters which export to single or multiple backends.

## Collector Deployment Modes

<img src="collector-deployments.png" alt="otel-collector-modes" style="width:100%;">

##### Running as an agent

The agent runs as a standalone process in the same VM or container, independent of the application. For example it can be a **sidecar container** in our docker orchestration or as a **DaemonSet** in Kubernetes, ensuring one agent per node or host. In our diagram we can see the otel collector (agent mode) directly sending data to backends. 

##### Running as a gateway

In **gateway mode**, the Collector runs as a **centralized service**, usually behind a load balancer, outside of the application nodes. Instrumented applications send telemetry data to this gateway, which performs further processing.

##### Hybrid approach

In a **hybrid setup**, both modes are combined. Each node runs an **agent collector** close to the application, and all agents forward their processed data to a **central gateway collector**. This model offers the best of both worlds: **efficient local data capture** and **centralized control** over telemetry flow and configuration. It’s the most common architecture in production-grade observability systems. Which is what we see in the diagram as well, all the agents sending data to the centralized collector as a gateway.

To read more about collector refer [official documentation](https://opentelemetry.io/docs/collector/).

#### Non application telemetry

Collecting telemetry data is not limited to the application itself. You can also instrument **non-application telemetry sources** such as **managed databases, APIs, proxies, Kubernetes components, and cloud provider services**.  

These systems generate their own telemetry, which provide valuable insights into **infrastructure health, network behavior, and service dependencies**.

For example, telemetry from a managed database can help identify slow queries or connection pool issues, while data from a proxy or Kubernetes node can reveal latency, traffic patterns, or resource bottlenecks. When combined with application-level telemetry, these signals provide a more complete view of system performance and reliability.

# The Backends

**Storage Systems (The Backends)**

This is the destination where all your telemetry data (metrics, traces, and logs) is sent and stored. Its job is to handle a massive amount of incoming data and store it efficiently so it can be queried later. Like Time series databases, columnar databases etc.

**Observability Tools (The Frontends)**

These are the user interfaces you log into. Their job is to connect to one or more of the storage systems, query them for data, and present it to you in a human-friendly way (graphs, flame charts, dashboards, etc.).

#### How They Work Together

- Your **OTel Collector** (the gateway) exports data.
- It sends that data to a **Storage System** (e.g., Prometheus for metrics, ClickHouse for traces).
- Your **Observability Tool** (e.g., Grafana) is configured to read data _from_ that storage system to build your dashboards.
- Alternatively, an all-in-one tool like **SigNoz** acts as _both_ the storage and the UI, so the OTel Collector just sends data directly to it.

For instance, your app → Collector → Prometheus (metrics) → Grafana dashboard.

# TLDR;

OpenTelemetry standardizes telemetry collection across languages.
Instrumentation (agent, SDK, auto/manual) generates the data.
Collectors receive, process, and export data to backends.
Backends + dashboards provide visibility into system performance.  
Together, they form a unified observability ecosystem.