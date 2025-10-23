---
# layout: post
title:  "Dependency Management Overview Spring Boot"
date:   2025-10-23 11:47:50 +0530
categories:
  - blog
tags:
  - Backend Dev
  - Spring Boot
---

Effective dependency management ensures consistent builds, predictable runtime behavior, and simplified maintenance across backend modules.
This document outlines best practices and common pitfalls observed in multi-module Maven projects, along with an optimized approach for version and plugin management.

---

## Common Challenges in Multi-Module Maven Projects

### 🔹 Lack of Centralized Version Control

Different modules sometimes define their own dependency versions, leading to potential conflicts and instability.
For example, inconsistent versions of the same library can result in runtime errors such as `ClassNotFoundException` or `NoSuchMethodError`.

### 🔹 Hardcoded Dependency Versions

When dependency versions are duplicated across multiple child POMs, upgrading a library requires manual edits in several places, increasing maintenance overhead and error risk.

### 🔹 Inconsistent Java Versions

Parent POM and child modules occasionally define different Java versions.
Such mismatches can cause build or runtime failures — containerization may mask these inconsistencies but doesn’t eliminate the risk.

### 🔹 Redundant Configurations

Child POMs often repeat properties and plugin definitions already present in the parent POM, making them cluttered and harder to maintain.

---

## Optimized Dependency Management Structure

The recommended approach consolidates all dependency and plugin management into the **root POM**, serving as the single source of truth.

### Centralized Dependency Management

All dependency versions should be defined in the parent’s `<dependencyManagement>` section.
Child modules simply declare dependencies without specifying versions.

```xml
<!-- Parent pom.xml -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>1.18.30</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<!-- Child pom.xml -->
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
</dependency>
```

### Centralized Plugin Management

Define all plugin versions and configurations once in the parent `<pluginManagement>` section.
Child modules can then use the plugins without redefining them.

### Using a BOM (Bill of Materials)

BOMs ensure consistent versions across related dependencies (e.g., Spring Boot stack).

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.5</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

This automatically aligns all Spring-related dependencies to compatible versions.
You don’t need to manually manage Spring Boot dependencies (such as `spring-boot-starter-web`, `spring-boot-starter-test`, `spring-boot-starter-data-jpa`, or `spring-boot-starter-security`) in the `<dependencyManagement>` section.
With the BOM in place, child modules can directly include these dependencies **without specifying version numbers**, as the BOM ensures all of them remain consistent and compatible.


### Core Module vs Supporting Modules

* Only the **core module** — the one responsible for packaging the main executable JAR — should include build plugins such as the `spring-boot-maven-plugin`.
* **Supporting modules** (e.g., libraries, starters, or utility modules) **should not include** these executable packaging plugins, as they are not meant to produce runnable artifacts.
* However, if a supporting module requires specific plugins for its own build or testing purposes (e.g., test plugins), it may include those as needed.
<!-- {: .notice} -->

### Special Cases (e.g., Flyway)

Avoid explicitly managing versions for dependencies that are already handled by imported BOMs (such as Spring Boot).
Let the BOM control these to prevent conflicts.

---

## 4. Maven Dependency Resolution Overview

Maven resolves versions using the following priority:

1. Version explicitly declared in the dependency
2. Version defined in `<dependencyManagement>`
3. Version imported from a BOM
4. Transitive dependency version

If multiple versions appear, Maven selects the nearest one in the dependency tree.
Centralized management helps ensure consistent resolution.

---

## Adding a New Dependency (Recommended Steps)

1. **Add Version Property (Parent POM):**

```xml
<properties>
  <guava.version>33.0.0-jre</guava.version>
</properties>
```

2. **Define Dependency in `<dependencyManagement>`:**

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>${guava.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

3. **Use Dependency in Child Module:**

```xml
<dependency>
  <groupId>com.google.guava</groupId>
  <artifactId>guava</artifactId>
</dependency>
```

4. **Verify the Dependency Tree:**

```bash
mvn dependency:tree
```

This helps ensure version alignment and identify potential conflicts early.

---

## Key Benefits of This Approach

* **Single Source of Truth:** All versions are managed centrally.
* **Consistency:** No version mismatches across modules.
* **Simplified Upgrades:** One place to update library versions.
* **Clean POMs:** Child modules remain minimal and focused.
* **Reduced Risk:** Lower chance of runtime or build-time dependency conflicts.

---

## Summary

By centralizing dependency and plugin management, projects become **cleaner, more maintainable, and resilient** to growth.
This structure ensures stable builds, easier upgrades, and smoother onboarding for new developers.

---

### 📘 Recommended Reading

* [Maven Dependency Management Guide (Apache)](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
* [Spring with Maven BOM](https://www.baeldung.com/spring-maven-bom)


