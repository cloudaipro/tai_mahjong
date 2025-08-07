# ADR-001: Technology Stack Selection for Taiwan Mahjong Online Game

## Status
Proposed

## Context
The Taiwan Mahjong online game project requires a comprehensive technology stack that can support:
- Cross-platform deployment (Web, Android, iOS)
- Real-time multiplayer gameplay with WebSocket connections
- Complex game logic and Taiwan Mahjong rules engine
- Scalable backend architecture for 10,000+ concurrent users
- Responsive UI/UX across different screen sizes

Based on the PRD requirements and technical analysis, we need to make strategic decisions about our core technologies.

## Decision Drivers
- **Cross-platform Consistency**: Unified codebase and user experience across platforms
- **Real-time Performance**: Sub-100ms response time for game actions
- **Scalability**: Support for large user base and concurrent games
- **Development Efficiency**: Faster time-to-market with maintainable code
- **Cultural Authenticity**: Proper support for Traditional Chinese and Taiwan-specific features
- **Team Expertise**: Leverage existing JavaScript/TypeScript knowledge

## Decision

### Frontend Technology Stack
**Web Platform:**
- **Framework**: React.js 18+ with TypeScript
- **3D Rendering**: Three.js for 3D mahjong table visualization
- **State Management**: Redux Toolkit with RTK Query
- **Real-time Communication**: Socket.IO client
- **Styling**: Tailwind CSS with custom Taiwan Mahjong theme

**Mobile Platforms:**
- **Framework**: React Native with TypeScript
- **Cross-platform**: Expo managed workflow initially, eject if needed
- **Navigation**: React Navigation v6
- **State Management**: Redux Toolkit (shared with web)

### Backend Technology Stack
**Core Services:**
- **Runtime**: Node.js with TypeScript
- **Framework**: Express.js with Helmet for security
- **Real-time**: Socket.IO for WebSocket connections
- **Authentication**: JWT with refresh tokens

**Data Layer:**
- **Primary Database**: PostgreSQL 14+
- **Caching**: Redis 6+ for session management and game states
- **Message Queue**: Bull Queue with Redis

**Infrastructure:**
- **Containerization**: Docker with multi-stage builds
- **Orchestration**: Kubernetes for production
- **Monitoring**: Prometheus + Grafana

### Game Engine Architecture
- **Rules Engine**: TypeScript classes for Taiwan Mahjong logic
- **AI System**: Decision tree algorithm for different difficulty levels
- **Scoring Engine**: Comprehensive 台数 (tai) calculation system
- **Anti-cheat**: Server-side validation of all game actions

## Consequences

### Positive
- **Unified Development**: TypeScript across frontend and backend reduces context switching
- **Component Reusability**: Shared components between React and React Native
- **Strong Ecosystem**: Rich package ecosystem and community support
- **Type Safety**: Compile-time error detection reduces runtime bugs
- **Scalability**: Node.js + PostgreSQL + Redis proven for real-time games
- **Cultural Support**: Excellent Unicode and internationalization support

### Negative
- **Bundle Size**: React applications can be larger than native apps
- **Performance**: JavaScript performance limitations for extremely demanding operations
- **Mobile Limitations**: React Native may not access all platform-specific features
- **Complexity**: Multiple technologies require broader team expertise

### Neutral
- **Learning Curve**: Team needs to learn Three.js and Socket.IO specifics
- **Tooling**: Need to establish proper TypeScript, testing, and deployment workflows

## Implementation Phases

### Phase 1: Core Foundation (Weeks 1-4)
- Set up TypeScript configurations for all platforms
- Implement basic React.js web application structure
- Create PostgreSQL schema for user and game data
- Set up Redis for caching and sessions

### Phase 2: Game Engine (Weeks 5-8)
- Develop Taiwan Mahjong rules engine in TypeScript
- Implement scoring (台数) calculation system
- Create WebSocket-based real-time communication
- Build basic 3D game table with Three.js

### Phase 3: Mobile Development (Weeks 9-12)
- Set up React Native project with Expo
- Port web components to mobile-compatible versions
- Implement touch-friendly game interactions
- Test cross-platform state synchronization

## Alternatives Considered

### Alternative 1: Flutter + Dart
- **Pros**: Truly native performance, single codebase
- **Cons**: Team has no Dart experience, limited web support

### Alternative 2: Unity + C#
- **Pros**: Excellent 3D capabilities, cross-platform
- **Cons**: Overkill for card game, larger app sizes, no web deployment

### Alternative 3: Native Development
- **Pros**: Maximum platform optimization
- **Cons**: Triple development cost, maintenance complexity

### Alternative 4: Go Backend
- **Pros**: Better performance, simpler deployment
- **Cons**: Team expertise in JavaScript, fewer gaming libraries

## Validation Criteria

### Technical Validation
- [ ] Real-time game actions respond within 100ms
- [ ] Application loads within 5 seconds on 4G connection
- [ ] Handles 1,000+ concurrent users per server instance
- [ ] 99.9% uptime in production environment

### Business Validation
- [ ] Development timeline meets 8-month target
- [ ] Cross-platform code reuse exceeds 70%
- [ ] Taiwan Mahjong rules accuracy verified by domain experts
- [ ] User acceptance testing shows >4.5/5 satisfaction

## References
- Taiwan Mahjong Online Game PRD v1.0
- Technical Architecture Requirements (Section 6)
- Performance Requirements (Section 9.1)
- Cross-platform Compatibility Requirements (Section 6.3)

---
**Decision Date**: August 7, 2025  
**Review Date**: September 7, 2025  
**Participants**: Product Team, Technical Architecture Team  
**Status**: Awaiting technical team review and stakeholder approval