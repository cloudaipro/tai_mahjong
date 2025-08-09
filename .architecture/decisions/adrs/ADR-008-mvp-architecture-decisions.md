# ADR-008: MVP Architecture Decisions

**Status**: Accepted  
**Date**: 2025-08-08  
**Decision Makers**: Collaborative Architecture Review Team (8 specialists)  
**Related**: ADR-001 through ADR-007 (Foundation Architecture)

## Context

Following the comprehensive architecture recalibration, the project stakeholders requested an accelerated MVP (Minimum Viable Product) to validate market demand and core technical assumptions before proceeding with the full 8-month implementation.

A collaborative architecture session with 8 specialist architects was conducted to define MVP scope, architecture, and implementation strategy. The goal is to deliver a market-ready Taiwan Mahjong online gaming experience in 4 months (16 weeks) with reduced complexity while maintaining core value proposition.

## Decision

We will implement a **simplified MVP architecture** that preserves essential game authenticity while dramatically reducing scope and complexity.

### MVP Core Principles

1. **Authentic Taiwan Mahjong Experience**: 100% traditional 16-tile rule compliance (non-negotiable)
2. **Web-First Strategy**: Focus on web platform, defer native mobile apps
3. **Simplified Architecture**: Modular monolith with clean boundaries, defer microservices
4. **Market Validation Focus**: Prove concept viability with early adopters

### Technology Stack Simplification

```typescript
// MVP Technology Stack
const mvpStack = {
  backend: {
    runtime: 'Node.js 18+',
    framework: 'Express.js (simplified from DDD architecture)',
    language: 'TypeScript',
    architecture: 'Simplified modular monolith (3 core modules)'
  },
  database: {
    primary: 'PostgreSQL 14+',
    cache: 'Redis 7+ (basic caching only)',
    orm: 'Prisma (simplified schema - 8 tables vs 20+)'
  },
  frontend: {
    framework: 'React 18+ with TypeScript',
    state: 'Redux Toolkit (simplified state)',
    styling: 'Tailwind CSS',
    bundler: 'Vite',
    platform: 'Web only (defer React Native)'
  },
  realtime: {
    protocol: 'Socket.io (simpler than custom WebSocket)',
    fallback: 'Long polling'
  },
  deployment: {
    platform: 'Docker containers',
    cloud: 'AWS single region (US-West-2)',
    cdn: 'CloudFront',
    monitoring: 'Basic CloudWatch only'
  }
};
```

### Architecture Simplifications

| Full Project Architecture | MVP Architecture | Rationale |
|--------------------------|-----------------|-----------|
| **Multi-region deployment** | Single region (US-West-2) | Reduce complexity, serve early adopters |
| **Advanced security (ECDSA)** | JWT + HTTPS | Secure but simple authentication |
| **Comprehensive observability** | Basic monitoring | Essential metrics only |
| **React + React Native** | React web + PWA | Cross-platform via mobile web |
| **Advanced caching strategy** | Basic Redis cache | Simple but effective caching |
| **20+ database tables** | 8 core tables | Minimal viable schema |
| **Microservices ready** | Modular monolith | Maintain boundaries, defer splitting |

### Feature Scope Decisions

#### MUST HAVE (MVP Core)
- ✅ **Taiwan Mahjong 4-player multiplayer** (100% rule accuracy)
- ✅ **Real-time gameplay** via Socket.io
- ✅ **Web interface** with basic 3D game board
- ✅ **User authentication** (JWT-based)
- ✅ **Room management** (create, join, lobby)
- ✅ **Mobile web support** (PWA)
- ✅ **Connection recovery** (basic reconnection)

#### DEFERRED (Post-MVP)
- ❌ Native mobile apps (iOS/Android)
- ✅ Rule-based opponents for practice (Added via ADR-009)
- ❌ Ranking/ELO systems
- ❌ Tournament modes
- ❌ Advanced analytics
- ❌ Social features
- ❌ Monetization features
- ❌ Multi-language support

## Implementation Strategy

### Development Phases (16 Weeks)

**Phase 1: Foundation (Weeks 1-4)**
- Backend setup with simplified architecture
- Taiwan Mahjong rules engine (non-negotiable quality)
- Basic authentication and user management
- Core game entity models

**Phase 2: Real-time Multiplayer (Weeks 5-8)**
- Socket.io integration for real-time communication
- Room management and player matchmaking
- 4-player game session management
- Game state synchronization

**Phase 3: Frontend Interface (Weeks 9-12)**
- React application with authentication
- Basic 3D mahjong game board
- Tile interaction and game controls
- Mobile web optimization (PWA)

**Phase 4: MVP Completion (Weeks 13-16)**
- Integration testing and polish
- Performance optimization
- Security review and validation
- Launch preparation

### Team Composition (8-10 developers)
- 1x Tech Lead (System Architecture)
- 2x Backend Developers (Game logic + Real-time)
- 2x Frontend Developers (React + 3D interface)
- 1x Full-stack Developer (Integration + Mobile web)
- 1x QA Engineer (Testing + Taiwan Mahjong validation)
- 1x DevOps Engineer (Deployment + Basic monitoring)

## Performance Targets (MVP)

Relaxed from full project targets while maintaining user experience:

| Metric | Full Project Target | MVP Target | Rationale |
|--------|-------------------|------------|-----------|
| **Backend Response** | <70ms (95%ile) | <100ms (90%ile) | Acceptable for MVP validation |
| **Frontend Rendering** | 60fps | 60fps (modern browsers) | Maintain game experience quality |
| **Mobile Web** | 45-50fps native | 30fps web acceptable | PWA trade-off acceptable |
| **Concurrent Users** | 10,000+ | 500 players | Sufficient for MVP testing |
| **Uptime** | >99.9% | >99% | Early stage acceptable |

## Quality & Testing Strategy

### Taiwan Mahjong Rule Accuracy (Non-Negotiable)
- Implement full 4-layer testing framework for game logic
- Expert validation required before launch
- 80% test coverage for game rules (same as full project)
- Comprehensive edge case testing (過水, 詐胡, 搶槓)

### General Quality Standards (Pragmatic)
- 60% overall test coverage (vs 95% full project)
- Basic integration testing
- Manual testing for user flows
- Code review for all changes

## Security (Essential Only)

```typescript
// MVP Security Framework
const mvpSecurity = {
  authentication: 'JWT + HTTP-only cookies',
  authorization: 'Basic RBAC (player/admin)',
  transport: 'HTTPS everywhere (Let\'s Encrypt)',
  validation: 'Input sanitization (Joi + express-validator)',
  rateLimit: 'Basic rate limiting (express-rate-limit)',
  cors: 'Strict CORS policy',
  antiCheat: 'Server-side validation only (no ML)'
};
```

Deferred: Advanced cryptographic signatures, ML-based cheat detection, comprehensive audit logging

## Success Criteria

### Technical Validation
- [ ] 100% Taiwan Mahjong rule compliance (expert validated)
- [ ] Stable 4-player real-time multiplayer
- [ ] <100ms backend response times
- [ ] 60fps web rendering on modern browsers
- [ ] Functional mobile web experience (PWA)

### Market Validation
- [ ] Average game session >20 minutes
- [ ] 40% user return rate within 7 days
- [ ] 80% game completion rate
- [ ] >4.0/5.0 user satisfaction rating
- [ ] Evidence of organic user growth

### Business Objectives
- [ ] Prove market demand for authentic Taiwan Mahjong
- [ ] Validate core technical architecture
- [ ] Generate user feedback for full product roadmap
- [ ] Build Taiwan Mahjong domain expertise

## Consequences

### Positive
- **Faster Time-to-Market**: 4 months vs 8 months to market validation
- **Reduced Risk**: Lower investment to test market assumptions
- **Focused Scope**: Clear priorities prevent feature creep
- **Learning Opportunity**: Real user feedback guides full development
- **Technical Validation**: Prove architecture decisions work

### Negative
- **Limited Feature Set**: May not fully represent final product vision
- **Technical Debt Risk**: Simplified implementations may need refactoring
- **Mobile Experience**: Web-based mobile experience vs native apps
- **Scalability Constraints**: Single-region deployment limits growth
- **Advanced Features Missing**: Limited rankings or social features (Rule-based opponents now included via ADR-009)

### Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **Rule Implementation Complexity** | Medium | High | Start with rules engine, expert validation |
| **Real-time Performance Issues** | Medium | High | Early performance testing, fallback options |
| **Team Coordination** | Low | Medium | Daily standups, clear component interfaces |
| **Scope Creep** | High | Medium | Strict feature lock after Week 2 |
| **3D Performance on Mobile Web** | High | Medium | Progressive enhancement, 2D fallback |

## Migration Path to Full Product

### Architecture Evolution Strategy
The MVP architecture is designed to evolve into the full product architecture:

1. **Module Boundaries**: Clean boundaries enable microservices migration
2. **Database Schema**: Designed for extension, not replacement
3. **API Patterns**: REST/GraphQL patterns ready for scale
4. **Authentication**: JWT foundation supports advanced auth
5. **Monitoring**: Basic monitoring expandable to full observability

### Feature Addition Strategy
Post-MVP features can be added incrementally:
- Native mobile apps (React Native code sharing)
- Advanced security (cryptographic signatures)
- Enhanced rule-based opponents (additional difficulty levels, cultural patterns)
- Social features (new modules)
- Analytics (separate data pipeline)

## Approval Process

### Stakeholder Sign-off Required
- [ ] **Product Owner**: MVP scope and success criteria
- [ ] **Technical Lead**: Architecture and implementation feasibility
- [ ] **Business Stakeholders**: Budget and timeline approval
- [ ] **Security Team**: MVP security framework acceptance
- [ ] **QA Lead**: Testing strategy and quality standards

### Go/No-Go Decision Points
- **Week 4**: Foundation architecture review
- **Week 8**: Multiplayer functionality demonstration
- **Week 12**: User interface and experience validation
- **Week 15**: Launch readiness assessment

## Implementation Authorization

**Decision**: ✅ **APPROVED for immediate implementation**

**Rationale**:
1. **Clear Value Proposition**: Authentic Taiwan Mahjong with simplified delivery
2. **Feasible Timeline**: 16 weeks achievable with focused scope
3. **Risk Management**: Comprehensive mitigation strategies identified
4. **Architecture Soundness**: Simplified but not compromised design
5. **Market Opportunity**: First-mover advantage in authentic Taiwan Mahjong online

**Next Steps**:
1. Secure stakeholder approval and budget allocation
2. Assemble MVP development team
3. Begin Phase 1: Foundation implementation
4. Schedule Taiwan Mahjong expert engagement
5. Setup AWS infrastructure and CI/CD pipeline

---

**Status**: Active Implementation  
**MVP Target Launch**: December 4, 2025  
**Review Schedule**: Bi-weekly progress reviews + Phase gate approvals  
**Success Tracking**: Weekly metrics dashboard with market validation KPIs