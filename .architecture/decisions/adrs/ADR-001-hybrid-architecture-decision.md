# ADR-001: Adoption of Hybrid Architecture for Taiwan Mahjong Game

**Status**: ADOPTED  
**Date**: 2025-08-07  
**Decision Makers**: System Architecture Team, Product Development Team  
**Supersedes**: Original pure event-driven architecture approach  

---

## Summary

We are adopting a **hybrid architectural approach** that combines Command Pattern, State Machines, Request-Response, and Selective Event-Driven patterns for the Taiwan Mahjong online game, replacing the originally planned pure event-driven architecture.

## Context

The Taiwan Mahjong online game project initially planned to use a pure event-driven architecture within a modular monolith design. However, during the architecture feasibility analysis, several concerns were identified with the pure event-driven approach for game logic implementation:

### Original Challenges
- **Complex event orchestration** for Taiwan Mahjong game rules and scoring calculations
- **Debugging complexity** for asynchronous event chains in game state management
- **Performance concerns** regarding real-time response requirements (<100ms)
- **State consistency issues** in multi-player synchronization
- **Development complexity** that could impact the 8-month delivery timeline

### Domain-Specific Requirements
- Taiwan Mahjong actions (摸牌, 打牌, 吃碰槓胡) are **irreversible once committed** - no undo/redo functionality needed
- Game flow follows predictable state transitions (發牌→進行→結算)
- Real-time multiplayer coordination requires deterministic action processing
- Complex Taiwan scoring system (台數計算) needs server-side validation
- Support for 10,000+ concurrent users with <100ms response time requirement

## Decision

We will implement a **hybrid architecture** that uses different patterns for different concerns:

### 1. Command Pattern for Game Actions
- **Usage**: All Taiwan Mahjong actions (摸牌, 打牌, 吃碰槓胡)
- **Benefits**: 
  - Deterministic execution and validation
  - Network synchronization through command serialization
  - Game replay capability through command history
  - Server-side action validation prevents cheating

### 2. State Machines for Game Flow
- **Usage**: Game state transitions (發牌→進行→結算) and room lifecycle management
- **Benefits**: 
  - Prevents invalid state transitions
  - Clear game flow representation
  - Easier debugging and maintenance
  - Deterministic state management

### 3. Request-Response for Real-time Features
- **Usage**: Player actions, game synchronization, immediate responses
- **Benefits**: 
  - Lower latency (target <70ms, requirement <100ms)
  - Simpler debugging than asynchronous event chains
  - Predictable performance characteristics
  - Better error handling and recovery

### 4. Selective Event-Driven for Non-Critical Operations
- **Usage**: Chat system, analytics, notifications, background processing
- **Benefits**: 
  - Maintains loose coupling for non-critical features
  - Asynchronous processing doesn't block game actions
  - Existing event infrastructure can be reused

## Architecture Integration

### Modular Monolith Compatibility
- **Perfect fit**: Hybrid patterns integrate seamlessly with existing module boundaries
- **No technology stack changes**: Current Node.js + TypeScript stack supports all patterns
- **Enhanced module communication**: Clear boundaries between synchronous and asynchronous operations

### Module Design
- **Game Engine Module**: Commands + State Machines for core game logic
- **Room Management Module**: State Machines for room lifecycle
- **Real-time Communication**: Request-Response for immediate player actions
- **User Management**: Traditional request-response patterns
- **Analytics & Social**: Event-driven for non-blocking processing

## Benefits Realized

### 1. **Timeline Improvement**
- **1 month faster delivery**: 7 months instead of 8 months
- **Reduced development complexity** in critical game logic implementation
- **Faster debugging and testing** of synchronous patterns

### 2. **Cost Reduction**
- **12-15% cost savings**: $40,000-$60,000 reduction
- **Higher team productivity**: +15% efficiency from reduced complexity
- **Less specialized expertise required**: No need for advanced event-driven architecture specialists

### 3. **Performance Enhancement**
- **Lower latency**: Target <70ms response time (exceeding <100ms requirement)
- **Better scalability**: Optimized patterns for 10,000+ concurrent users
- **Improved reliability**: 99.9%+ system uptime through deterministic execution

### 4. **Risk Mitigation**
- **Reduced technical complexity** in game rule implementation
- **Easier maintenance** and debugging for development team
- **Better testability** with synchronous patterns
- **Clearer system behavior** and state management

## Implementation Strategy

### Phase 1: Foundation (Months 1-3)
- Implement Command Pattern for all Taiwan Mahjong actions
- Design and implement State Machines for game flow
- Establish architectural guidelines and team training
- Develop core game engine with hybrid patterns

### Phase 2: Integration (Months 3-5)
- Integrate State Machines with room management
- Implement Request-Response patterns for real-time features
- Performance testing and optimization
- Module boundary validation

### Phase 3: Optimization (Months 5-7)
- Fine-tune performance for <70ms response targets
- Complete Taiwan scoring system integration
- Cross-platform API finalization
- Load testing with 10,000+ concurrent users

### Phase 4: Launch (Month 7)
- Selective event-driven implementation for non-critical features
- Final integration testing and production deployment
- Beta testing and performance monitoring

## Consequences

### Positive Consequences
- **Faster time to market** by 1 month
- **Significant cost savings** and improved team productivity
- **Superior performance** exceeding original requirements
- **Lower technical risk** through proven patterns
- **Better maintainability** for long-term project success
- **Perfect alignment** with Taiwan Mahjong domain requirements

### Negative Consequences
- **Learning curve** for team members (1-2 weeks training)
- **Pattern integration complexity** requires careful design
- **Architectural documentation** needs to be comprehensive

### Mitigations
- **Comprehensive training program** for team members
- **Clear architectural guidelines** and design reviews
- **Senior developer oversight** for pattern implementation
- **Progressive rollout** across development phases

## Compliance and Constraints

### Taiwan Mahjong Domain Rules
- ✅ **No undo/redo required**: Commands are immutable once executed
- ✅ **Deterministic rule enforcement**: State machines prevent invalid moves
- ✅ **Accurate scoring**: Server-side validation through request-response
- ✅ **Multiplayer synchronization**: Command serialization ensures consistency

### Performance Requirements
- ✅ **Response time**: <70ms target, <100ms guaranteed
- ✅ **Concurrent users**: 10,000+ users supported
- ✅ **System availability**: 99.9%+ uptime through reliability improvements
- ✅ **Scalability**: Horizontal scaling maintained through modular monolith

### Technology Constraints
- ✅ **Current stack compatibility**: Node.js + TypeScript fully supports hybrid patterns
- ✅ **Database architecture**: PostgreSQL + Redis integration unchanged
- ✅ **Deployment strategy**: Single container deployment maintained
- ✅ **Future migration**: Architecture supports microservices evolution

## Success Metrics

### Technical Metrics
- Game action response time: <70ms average
- System uptime: >99.9%
- Zero invalid game state transitions
- Command processing throughput: >1000 commands/second per instance

### Business Metrics
- Development cost reduction: 12-15% ($40K-60K savings)
- Time to market improvement: 1 month earlier launch
- Team productivity increase: +15%
- Post-launch maintenance cost reduction: 20%+

### Quality Metrics
- Bug discovery rate reduction: 30%+ due to deterministic patterns
- Integration testing efficiency: 40%+ improvement
- Code review cycle time: 25%+ reduction
- Developer onboarding time: 50%+ faster

## Related Documents

- [台灣麻將線上遊戲PRD.md](./台灣麻將線上遊戲PRD.md) - Updated product requirements with hybrid architecture
- [hybrid-architecture-feasibility-analysis.md](./hybrid-architecture-feasibility-analysis.md) - Detailed feasibility analysis
- [architecture-manpower-timeline-analysis.md](./architecture-manpower-timeline-analysis.md) - Resource and timeline analysis
- [modular-monolith-migration-strategy.md](./modular-monolith-migration-strategy.md) - Migration strategy documentation

---

**Review and Approval**
- **System Architect**: ✅ Approved
- **Product Owner**: ✅ Approved  
- **Technical Lead**: ✅ Approved
- **Project Manager**: ✅ Approved

**Last Updated**: 2025-08-07  
**Next Review**: 2025-09-07 (Monthly architecture review)