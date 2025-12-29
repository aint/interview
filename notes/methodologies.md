# Software Development Methodologies

**Software development methodologies** provide frameworks, practices, and principles for organizing teams, designing systems, and delivering software effectively.

## Table of Contents
1. [Design Methodologies](#design-methodologies)
2. [Development Practices](#development-practices)
3. [Documentation Approaches](#documentation-approaches)
4. [Organizational Methodologies](#organizational-methodologies)

---

## Design Methodologies

### Domain-Driven Design (DDD)
- **Core principle**: Model software based on the business domain
- **Key concepts**:
  - **Ubiquitous Language**: Shared vocabulary between developers and domain experts
  - **Bounded Contexts**: Explicit boundaries where a domain model applies
  - **Entities**: Objects with unique identity
  - **Value Objects**: Immutable objects defined by their attributes
  - **Aggregates**: Cluster of entities and value objects treated as a single unit
  - **Domain Events**: Significant business occurrences
  - **Repositories**: Abstraction for accessing domain objects
- **Strategic patterns**: Context Mapping, Shared Kernel, Anti-Corruption Layer
- **Tactical patterns**: Aggregate Root, Domain Service, Factory
- **Use cases**: Complex business domains, large applications, microservices boundaries

### Use Case Driven Design
- **Core principle**: Organizes around user actions/use cases
- **Approach**: Each use case is a cohesive workflow from user perspective
- **Characteristics**:
  - Use cases define system behavior
  - Domain model supports use cases
  - User-centric organization
- **Use cases**: User-facing applications, applications with clear user workflows
- **Trade-offs**: Good for user-centric apps, but can blur domain boundaries

---

## Development Practices

### API-First Development
- **Principle**: Design APIs before implementation
- **Approach**: Contract-first (OpenAPI/Schema) as source of truth
- **Benefits**:
  - Frontend and backend teams work in parallel
  - Early validation of API design
  - Better API consistency
  - Mock servers for testing
- **Tools**: OpenAPI/Swagger, GraphQL Schema, gRPC Protocol Buffers

### Trunk-Based Development
- **Principle**: Short-lived branches, frequent integration to main/master
- **Key practices**:
  - Small, frequent commits to main branch
  - Feature flags for incomplete features
  - Branch lifetime: hours or days, not weeks
  - Continuous integration required
- **Benefits**:
  - Reduces merge conflicts
  - Faster feedback loops
  - Easier code reviews
  - Better collaboration
- **Anti-pattern**: Long-lived feature branches

### Contract-Driven Development
- **Principle**: API contracts (schemas) drive development
- **Approach**:
  - Define contracts first (OpenAPI, JSON Schema, Protocol Buffers)
  - Generate code from contracts
  - Validate against contracts in CI/CD
- **Benefits**:
  - Prevents breaking changes
  - Enables parallel development
  - Clear API boundaries
  - Automated contract testing

---

## Documentation Approaches

### Request for Comments (RFC)
- **Principle**: Document-driven approach for proposing changes
- **Process**: Written before implementation to gather feedback
- **Benefits**:
  - Aligns team on design decisions
  - Asynchronous review and discussion
  - Documents rationale and alternatives
- **Use cases**: Open-source projects, large organizations, architectural decisions

### README-Driven Development
- **Principle**: Write the README first before writing code
- **Benefits**:
  - Forces upfront thinking about API and UX
  - Documentation stays in sync with implementation
  - Easier onboarding for new contributors
- **Use cases**: Libraries, APIs, open-source projects

---

## Organizational Methodologies

### Team Topologies
- **Principle**: Team structure influences system architecture (Conway's Law)
- **Four team types**:
  - **Stream-Aligned Team**: Owns a stream of work end-to-end
  - **Complicated-Subsystem Team**: Handles complex subsystems requiring deep expertise
  - **Platform Team**: Provides internal services and tools for other teams
  - **Enabling Team**: Helps other teams acquire missing capabilities
- **Three interaction modes**:
  - **Collaboration**: Working together on same work
  - **X-as-a-Service**: One team provides services to another
  - **Facilitating**: One team helps another improve
- **Key principles**:
  - Team size: 5-9 people (two-pizza team)
  - Cognitive load: Limit what teams need to know
  - Thinnest viable platform: Minimal platform that enables stream-aligned teams
- **Benefits**: Autonomous teams, reduced dependencies, improved flow

---
