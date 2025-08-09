# ADR-001: Adopt Modular Monolith Architecture

## Status
**ACCEPTED** - August 7, 2025

## Context

The Taiwan Mahjong online game project requires an architectural decision for backend implementation. The initial plan considered microservices architecture, but after comprehensive analysis of project requirements, team constraints, and industry best practices, we need to make a definitive architectural choice.

### Project Characteristics:
- Multi-platform online multiplayer Taiwan Mahjong game
- Real-time gameplay with WebSocket communications
- Complex game rules and scoring (台數) system
- Target platforms: Web (React.js) + Mobile (React Native/Flutter)
- Team size: 8-10 developers
- Timeline: 8-9 months to production
- Currently in planning phase with no existing implementation

### Analysis Conducted:
- Comprehensive research on modular monolith vs microservices
- Manpower and timeline analysis
- Cost-benefit evaluation
- Risk assessment for both approaches

## Decision

We will adopt **Modular Monolith Architecture** for the Taiwan Mahjong online game backend implementation.

## Rationale

### 1. **Development Efficiency**
- **40-50% smaller team requirement** (8-10 vs 12-15 people)
- **2-3 months faster delivery** (8.5 vs 10-12 months)
- **50% lower development cost** ($340K-510K vs $660K-990K)
- Unified development workflow and debugging process

### 2. **Technical Suitability**
- **Real-time gaming benefits**: Single deployment unit reduces network latency
- **Game state consistency**: Easier to maintain consistent Taiwan Mahjong game states
- **Complex rule implementation**: Taiwan Mahjong rules better implemented as cohesive modules
- **WebSocket management**: Simplified real-time connection handling

### 3. **Risk Mitigation**
- **Lower operational complexity**: Simpler deployment and monitoring
- **Easier debugging**: Single process debugging vs distributed system troubleshooting
- **Reduced infrastructure costs**: Basic cloud setup vs complex Kubernetes orchestration
- **Team management**: Smaller, more manageable team structure

### 4. **Future Flexibility**
- **Migration path preserved**: Modular design enables future microservices extraction
- **Scalability options**: Can extract high-traffic modules when scaling demands it
- **Architecture evolution**: Designed with clear module boundaries for future decomposition

## Implementation Details

### Module Structure:
```
├── game-core/              # Taiwan Mahjong rules, scoring, rule-based opponents
├── room-management/        # Room creation, matchmaking
├── user-management/        # Authentication, profiles, friends
├── real-time-communication/# WebSocket handling, messaging
├── data-access/           # Database operations, caching
└── analytics/             # Statistics, reporting
```

### Technology Stack:
- **Runtime**: Node.js + TypeScript
- **Database**: PostgreSQL + Redis
- **Communication**: Internal event bus + Redis Pub/Sub
- **Deployment**: Single Docker container
- **Scaling**: Horizontal scaling with load balancers

### Migration Strategy:
- Extract services only when metrics indicate necessity:
  - >1000 concurrent WebSocket connections
  - >100 concurrent games
  - >70% memory usage or >60% CPU usage
  - Response times >500ms

## Alternatives Considered

### Microservices Architecture
**Rejected because:**
- Higher complexity not justified by current scale
- 95% higher development cost
- 2-3 months longer development timeline
- Operational overhead exceeds project requirements
- Risk of creating "distributed monolith" without clear domain boundaries

### Traditional Monolith
**Rejected because:**
- Limited scalability for gaming applications
- Difficult to maintain clean separation of concerns
- No clear evolution path for future growth
- Less suitable for complex game logic organization

## Consequences

### Positive:
- Faster development and time-to-market
- Lower development and operational costs
- Simplified deployment and monitoring
- Easier team coordination and debugging
- Strong foundation for Taiwan Mahjong game logic implementation

### Negative:
- Scaling limitations compared to microservices
- Single point of failure for entire application
- Potential performance bottlenecks under extreme load
- Limited technological diversity (single language/framework)

### Mitigation:
- Design modules with clear boundaries for future extraction
- Implement comprehensive monitoring and performance metrics
- Plan horizontal scaling strategy for high-traffic scenarios
- Use event-driven architecture within monolith to prepare for distribution

## Compliance

This decision aligns with:
- Project timeline and budget constraints
- Team size and skill requirements
- Taiwan Mahjong game performance requirements
- Industry best practices for gaming applications
- Migration flexibility for future scaling needs

## References

- Architecture Manpower & Timeline Analysis (docs/architecture-manpower-timeline-analysis.md)
- Modular Monolith Migration Strategy (docs/modular-monolith-migration-strategy.md)
- Taiwan Mahjong Game PRD (docs/台灣麻將線上遊戲PRD.md)
- Industry research on modular monolith vs microservices architectures

## Review Schedule

This architectural decision will be reviewed:
- **3 months post-launch**: Evaluate performance metrics and scaling requirements
- **6 months post-launch**: Assess user growth and infrastructure needs
- **12 months post-launch**: Consider microservices extraction if scaling thresholds are met

---

**Decision Made By**: System Architecture Team  
**Date**: August 7, 2025  
**Next Review**: Post-launch performance assessment