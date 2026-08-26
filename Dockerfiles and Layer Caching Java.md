---
date: 2025-08-21
type: study-node
topic: "Dockerfiles and Layer Caching Java"
---

# Dockerfiles and Layer Caching Java

In Java projects (Gradle/Maven), layer caching allows us to separate heavy dependency downloads from fast source code compilation. By copying `build.gradle` or `pom.xml` and running dependency resolution *before* copying `src/`, Docker caches the dependencies and avoids re-downloading them on every source code change.

## Connections
Parent: [[Docker Kernel Isolation]]
