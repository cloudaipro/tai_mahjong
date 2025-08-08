# Architectural Governance Framework
## Taiwan Mahjong Online Game Project

**Document Version**: 1.0  
**Date**: 2025-08-14
**Author**: Alex Chen (System Architect)
**Status**: DRAFT

---

## 1. Introduction & Purpose

This document outlines the architectural governance framework for the Taiwan Mahjong Online Game project. Its purpose is to ensure that the system's architecture remains clean, maintainable, and scalable as the project evolves and the team grows.

Adherence to these principles and practices is mandatory to prevent architectural drift, manage technical debt, and ensure our modular monolith can be gracefully migrated to microservices in the future, if and when required.

---

## 2. Core Architectural Principles

1.  **Adhere to Bounded Contexts**: All code must respect the logical boundaries defined by the project's Domain-Driven Design (DDD) contexts (e.g., `Game`, `Player`, `Billing`, `Authentication`).
2.  **Enforce the Dependency Rule**: As per Clean Architecture, dependencies must only point inwards. Domain and Application layers must not have any dependencies on Infrastructure or Presentation layers.
3.  **Synchronous for Critical Path, Asynchronous for Others**: Core game actions (e.g., drawing a tile, declaring a win) must be synchronous request-response calls to ensure immediate consistency. Non-critical operations (e.g., updating leaderboards, analytics) should be handled via asynchronous events.
4.  **Immutable State**: All state changes, both on the frontend (Redux) and backend (Domain Events), should be treated as immutable.
5.  **Configuration as Code**: All infrastructure and deployment configurations must be managed in version control (e.g., Terraform, Kubernetes YAML).

---

## 3. Enforcing Module Boundaries

The primary risk to our modular monolith is unintended coupling between modules. We will enforce these boundaries using a combination of automated checks and code review practices.

### 3.1. Technical Enforcement: Architectural Fitness Functions

We will implement automated tests that verify our architectural rules. These "fitness functions" will run as part of our CI/CD pipeline. A build will fail if any of these rules are violated.

**Example Fitness Function (using a tool like ArchUnit for Java, or a custom script for TypeScript/Node.js):**

```typescript
// Pseudocode for a fitness function test

import { Architecture } from 'arch-guard';

const arch = Architecture.load('src');

test('Domain layer should not depend on Infrastructure layer', () => {
  const rule = arch.layers()
    .where('domain.*')
    .shouldNotDependOn('infrastructure.*');
  
  expect(rule.check()).toBe(true);
});

test('Game module should not directly access Billing module database tables', () => {
  const rule = arch.modules()
    .where('game')
    .shouldNotHaveDependencyOn('billing.database');
    
  expect(rule.check()).toBe(true);
});

test('Communication between Game and Billing must be via asynchronous events', () => {
    // This test would check for direct function calls and allow only event emissions
    const rule = arch.modules()
        .where('game')
        .canOnlyCommunicateWith('billing')
        .via('EventBus');

    expect(rule.check()).toBe(true);
});
```

### 3.2. Code Review Enforcement

*   All pull requests must be reviewed by at least one other developer.
*   Pull requests that cross module boundaries or modify core architectural components **must** be reviewed by the designated architect for that domain.
*   A "Cross-Boundary" label will be used on pull requests to flag them for architectural review.

---

## 4. Architectural Decision Records (ADRs)

All significant architectural decisions must be documented in an ADR within the `.architecture/decisions/adrs/` directory. This includes:

*   Introducing a new technology or library.
*   Changing a core architectural pattern.
*   Defining a new module or bounded context.
*   Altering the communication method between modules.

An ADR provides a historical record of *why* a decision was made, which is invaluable for future maintenance and onboarding.

---

## 5. Service and Configuration Management

*   **Service Discovery**: Initially, within the monolith, services (Application Services in DDD) will be discovered via dependency injection. The DI container will be configured at application startup.
*   **Configuration**: All configuration will be managed via environment variables and configuration files. There will be a single, version-controlled `config.ts` file for each environment that defines all settings. **No secrets are to be stored in version control.** Secrets will be managed via a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager).

---

## 6. Next Steps

1.  **Tool Selection**: Select and configure the tool for implementing the architectural fitness functions within the CI/CD pipeline.
2.  **Initial Ruleset**: Implement the first set of fitness functions based on our core principles.
3.  **Team Training**: Conduct a session with the development team to review this governance framework and the importance of adhering to it.
