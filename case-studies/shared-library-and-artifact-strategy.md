# Shared Library and Artifact Strategy

## 1. Executive Summary
As the ERP/LMS platform scaled into multiple independently deployed microservices, repeated utility code and cross-cutting logic started creating delivery risk. Teams were shipping similar DTOs, exception mappers, security filters, and configuration conventions in each service, but with small deviations that caused integration defects and operational inconsistency.  

To address this, engineering introduced a shared library architecture with disciplined artifact publishing, semantic versioning, and service-specific packaging choices (fat jar and thin jar) based on runtime context. This decision reduced duplication, improved consistency, and accelerated onboarding, while introducing new tradeoffs around release governance, version compatibility, and operational control.

## 2. Business Context
The platform served institutional workflows across admissions, enrollment, fee processing, timetable operations, learning modules, and notifications. Product velocity required multiple teams to deliver in parallel while maintaining strict reliability for academic and financial operations.

Business pressure surfaced in three areas:
- **Faster feature rollout:** shared models and validations needed to evolve without forcing each team to rebuild foundational components.
- **Regulated correctness:** admission and fee-related services required consistent rule interpretation and error semantics.
- **Operational predictability:** incidents spanning multiple services needed uniform logging, tracing hooks, and failure handling patterns.

The shared library initiative was positioned as a platform capability to protect delivery speed while improving cross-service consistency.

## 3. Initial Architecture Challenges

### Code Duplication
Common classes (request/response models, mappers, utility helpers, auth guards) were copied service-to-service. Small edits in one service rarely propagated cleanly to others, creating functional drift.

### Inconsistent Implementations
Even when solving the same problem, teams implemented different validation messages, error code mappings, pagination DTOs, and token parsing approaches. This harmed API contract reliability and debugging speed.

### Maintenance Overhead
A single policy change (for example, password rules or exception contract format) required repeated edits across services, each with separate testing and rollout windows. Engineering effort was spent on synchronization rather than product outcomes.

### Release Management Issues
Because shared logic was duplicated, there was no single source of truth. During production incidents, platform teams could not quickly identify which services were aligned to the latest behavior and which were not.

## 4. Shared Library Architecture

### Purpose
Centralize stable, reusable backend building blocks so services consume consistent foundations while keeping domain logic inside each microservice.

### Benefits
- Reduced duplicate code and divergence risk.
- Standardized DTO contracts, exceptions, and security primitives.
- Lower onboarding friction for new teams and contractors.
- Faster rollout of platform-level improvements.

### Package Structure
The shared artifact was intentionally modular at package level to avoid a monolithic “god library”:
- `common.dto` – base DTOs, pagination wrappers, validation payloads.
- `common.exception` – exception hierarchy, error envelopes, global error mapping contracts.
- `common.security` – token utilities, auth interceptors, role/claim abstractions.
- `common.config` – reusable configuration binders and defaults.
- `common.util` – deterministic utility functions and framework-neutral helpers.

Each package was scoped to cross-service concerns only; domain-specific models remained in owning services.

### Versioning Strategy
Semantic versioning was applied with explicit compatibility policy:
- **MAJOR:** breaking API/contract changes.
- **MINOR:** backward-compatible enhancements.
- **PATCH:** bug fixes and non-breaking refinements.

Additionally:
- Services pinned explicit versions (no floating `latest`).
- Breaking changes required migration notes and deprecation windows.
- Artifact release notes documented changed classes, behavioral impact, and upgrade guidance.

## 5. Dependency Management Strategy
Dependency governance combined technical and process controls:
- Centralized dependency declarations via parent BOM/version catalog.
- Strict transitive dependency review to avoid hidden runtime conflicts.
- Exclusion rules for conflicting logging/security libraries.
- Compatibility matrix tracking shared-library version vs service runtime stack.
- Controlled adoption policy: critical services upgraded first in pre-production lanes before fleet-wide rollout.

This reduced accidental drift and made upgrades auditable.

## 6. Build Process Overview

### Maven/Gradle Build Lifecycle
The shared module followed a standard lifecycle: compile, unit test, package, publish. Quality gates ran before publication to artifact repositories.

### Artifact Generation
Build output produced versioned JAR artifacts with metadata (version, build number, commit reference). Optional source/Javadoc artifacts were published to support internal discoverability and debugging.

### Artifact Consumption
Service pipelines resolved the shared artifact from the internal repository. Version upgrades were explicit pull requests so changes could be reviewed, tested, and promoted environment-by-environment.

## 7. Fat Jar vs Thin Jar

### Fat Jar
**Definition:** Single deployable JAR containing service code plus most runtime dependencies.  
**Internal Structure:** Application classes + embedded third-party libraries, often under nested classpath layout.  
**Advantages:** Simplified deployment, predictable runtime classpath, easy portability.  
**Disadvantages:** Larger artifact size, slower transfer/startup in some environments, duplicate libraries across services.  
**Deployment Impact:** Best for environments prioritizing immutable self-contained deployment units and minimal host-level dependency management.

### Thin Jar
**Definition:** Minimal application JAR that excludes most dependencies, resolved externally at runtime or container/image build stage.  
**Internal Structure:** Primarily service classes and metadata, with dependency descriptors referencing external libraries.  
**Advantages:** Smaller service artifact, better layer caching in container builds, centralized dependency reuse.  
**Disadvantages:** Higher operational dependency on runtime classpath correctness, potential startup failures if dependency resolution is misconfigured.  
**Deployment Impact:** Works well in mature platform environments with controlled artifact mirrors, deterministic dependency provisioning, and strong runtime governance.

## 8. Why Different Services Needed Different Packaging Approaches
Service contexts varied:
- Legacy VM-based deployments with limited orchestration favored **fat jars** for reliability and straightforward rollback.
- Containerized services with optimized image layers and shared base runtimes favored **thin jars** for faster build/pull cycles.
- Latency-sensitive services balanced startup predictability vs image size based on autoscaling behavior.

A single packaging standard would have over-optimized for one environment and penalized others.

## 9. Copy Dependencies Concept

### What Problem It Solves
“Copy dependencies” creates a controlled local dependency directory during build/deploy so thin-jar services run with an explicit, versioned classpath bundle without embedding everything into one artifact.

### How Dependencies Are Managed
- Build pipeline resolves declared dependencies from internal repositories.
- Required JARs are copied into a deterministic directory structure.
- Startup scripts/container entrypoints reference that dependency path.
- Checksums and lock-like mechanisms ensure repeatable deployments.

### Deployment Considerations
- Deployment packages must include both app artifact and dependency directory.
- Rollback requires compatibility between app binary and copied dependency set.
- Operational tooling must validate missing or conflicting libraries before startup.

## 10. Engineering Tradeoffs

### Maintainability
Shared code improved maintainability for platform-wide concerns, but required stronger ownership and review discipline to prevent uncontrolled API growth.

### Release Coupling
Centralized libraries reduced duplication but introduced coupling: poorly planned changes could block multiple teams. Version governance and deprecation policy were essential.

### Version Compatibility
Supporting multiple services on staggered upgrade timelines required compatibility commitments and longer support windows for older shared-library versions.

### Operational Complexity
Thin-jar and copy-dependency flows improved artifact efficiency but increased runtime configuration complexity. Fat jars simplified operations at the cost of artifact bloat.

## 11. Impact on Development Productivity
- Faster service bootstrapping using standardized base components.
- Less repetitive coding for DTOs, validation contracts, and exception handling.
- Shorter review cycles due to familiar shared patterns.
- Reduced incident triage time because error and security behavior became more predictable.

Net effect: more engineering time shifted from infrastructure repetition to business capability delivery.

## 12. Impact on Microservices Ecosystem
- Improved cross-service contract consistency.
- Better platform governance through controlled dependency updates.
- Higher confidence in rolling out cross-cutting changes (security patches, logging enhancements).
- Stronger service interoperability, especially where workflows crossed ERP and LMS boundaries.

At ecosystem level, the platform moved from “independent but inconsistent services” toward “autonomous services with shared engineering standards.”

## 13. Lessons Learned
- Shared libraries succeed only with clear scope boundaries and ownership.
- Backward compatibility policy must be enforced, not implied.
- Release notes and migration guides are critical engineering artifacts.
- Packaging strategy should align with operational maturity, not preference alone.
- Treat shared artifacts as products with roadmap, SLAs, and support expectations.

## 14. Backend Engineering Insights
- Reuse is an architecture decision, not just a code organization technique.
- Contract consistency can be a bigger scaling factor than raw throughput.
- Dependency governance is foundational for microservice stability.
- Standardized error/security layers materially improve observability and incident response.
- Artifact strategy (fat vs thin) directly influences deployment reliability and cost.

## 15. Future Improvements
- Split the shared library into separately versioned modules to minimize unnecessary transitive load.
- Add automated compatibility testing against representative service versions.
- Introduce deprecation telemetry to track API usage before removals.
- Expand platform tooling to auto-suggest safe shared-library upgrades per service.
- Establish architecture review checkpoints for new shared components to prevent over-centralization.
