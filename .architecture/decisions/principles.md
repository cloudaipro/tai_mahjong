# Architectural Principles for Taiwan Mahjong Online Game

> "Good programmers write code that humans can understand." — Martin Fowler

## Core Principles

### 1. Cultural Authenticity First
**Description**: Every technical decision must honor the authentic Taiwan Mahjong tradition and cultural context.

**Rationale**: This game represents a cherished cultural tradition. Technical convenience cannot compromise the authenticity of rules, terminology, or gameplay experience.

**Wisdom Quote**: "Culture is the widening of the mind and of the spirit." — Jawaharlal Nehru

**Application Guidelines**:
- Use traditional Chinese terminology in code comments and UI
- Implement 100% accurate Taiwan 16-tile Mahjong rules
- Preserve cultural nuances like scoring (台數) and special combinations
- Validate all game logic with domain experts familiar with traditional play

### 2. Server-Authoritative Design
**Description**: All game-critical logic and state management must be executed and validated on the server.

**Rationale**: Multiplayer card games are inherently vulnerable to cheating. Trust must be earned through transparent, server-controlled gameplay.

**Wisdom Quote**: "Trust, but verify." — Ronald Reagan

**Application Guidelines**:
- Never trust client-side game state for authoritative decisions
- Validate every player action on the server before execution
- Use client prediction only for immediate UI feedback
- Implement comprehensive anti-cheat detection and logging

### 3. Clarity Over Cleverness
**Description**: Write code that prioritizes readability and maintainability over performance optimizations or clever shortcuts.

**Rationale**: Complex game logic requires long-term maintenance. Clear code reduces bugs and enables faster feature development.

**Wisdom Quote**: "Programs must be written for people to read, and only incidentally for machines to execute." — Harold Abelson

**Application Guidelines**:
- Choose descriptive variable and function names that reflect Taiwan Mahjong terminology
- Break complex game logic into small, testable functions
- Comment complex algorithms with both English and Chinese explanations
- Prefer explicit type definitions over type inference

### 4. Evolvability Through Modularity
**Description**: Design systems as composable modules that can evolve independently as requirements change.

**Rationale**: Gaming requirements evolve rapidly based on user feedback and market demands. Modular architecture enables quick adaptation.

**Wisdom Quote**: "The best way to get a project done faster is to start sooner." — Jim Highsmith

**Application Guidelines**:
- Separate game rules engine from UI and networking layers
- Design plugin-based architecture for AI opponents
- Create configurable rule variations for future game modes
- Use event-driven architecture for loose coupling between systems

### 5. Performance with Purpose
**Description**: Optimize for performance only where it directly impacts user experience, with measurable goals.

**Rationale**: Premature optimization wastes development time. Real-time gaming has specific performance requirements that must be met.

**Wisdom Quote**: "Premature optimization is the root of all evil." — Donald Knuth

**Application Guidelines**:
- Target sub-100ms response time for game actions
- Optimize network bandwidth for mobile users
- Profile and measure before optimizing
- Document performance requirements and trade-offs

### 6. Defensive Programming
**Description**: Assume inputs are invalid, networks will fail, and users will behave unexpectedly.

**Rationale**: Online gaming faces unique challenges including network instability, malicious users, and edge cases not found in single-player games.

**Wisdom Quote**: "Expect the best, plan for the worst, and prepare to be surprised." — Denis Waitley

**Application Guidelines**:
- Validate all inputs with clear error messages
- Implement graceful degradation for network failures
- Handle edge cases in Taiwan Mahjong rules explicitly
- Log sufficient information for debugging production issues

### 7. Cross-Platform Consistency
**Description**: Ensure identical game experience and behavior across web, Android, and iOS platforms.

**Rationale**: Players may switch between devices and expect seamless continuation of their experience.

**Wisdom Quote**: "The details are not the details. They make the design." — Charles Eames

**Application Guidelines**:
- Share game logic code between platforms
- Test all features on each target platform
- Maintain consistent UI layouts and interactions
- Synchronize user data and game state across devices

### 8. Observable Systems
**Description**: Build comprehensive logging, monitoring, and analytics into every system component.

**Rationale**: Online games require continuous monitoring to ensure smooth operation and to understand player behavior.

**Wisdom Quote**: "You can't manage what you can't measure." — Peter Drucker

**Application Guidelines**:
- Log all game events with sufficient context for analysis
- Monitor system performance and alert on degradation
- Track user behavior for product improvement insights
- Implement distributed tracing for complex request flows

## Implementation Guidelines

### Code Organization
- Group related functionality into cohesive modules
- Follow TypeScript strict mode for type safety
- Use consistent naming conventions across all platforms
- Maintain separation between pure functions and side effects

### Documentation Requirements
- Document all public APIs with TSDoc comments
- Maintain up-to-date architectural decision records
- Create visual diagrams for complex system interactions
- Write user-facing documentation in Traditional Chinese

### Testing Strategy
- Unit test all game logic with Taiwan Mahjong rule scenarios
- Integration test real-time multiplayer scenarios
- Performance test under realistic load conditions
- Cultural validation by Taiwan Mahjong experts

### Security Practices
- Never expose sensitive game state to clients
- Use HTTPS/WSS for all communications
- Implement rate limiting on all API endpoints
- Regular security audits and penetration testing

## Review Process

### Architectural Reviews
- All major technical decisions require ADR (Architectural Decision Record)
- Multi-perspective review by architecture team members
- Regular review of principles alignment in code reviews
- Quarterly assessment of principle effectiveness

### Exception Handling
- Document any deviations from these principles with justification
- Time-limited exceptions must include remediation timeline
- Regular review of exceptions for pattern identification
- Update principles based on learned experiences

### Principles Evolution
- Annual review of all architectural principles
- Community feedback integration from development team
- Adaptation based on new technology capabilities
- Alignment with changing business requirements

## Success Metrics

### Technical Excellence
- Code review feedback consistently references these principles
- Reduced bug count in Taiwan Mahjong rule implementation
- Faster onboarding time for new team members
- Improved system maintainability scores

### Cultural Authenticity
- Zero complaints about rule inaccuracies from Taiwan Mahjong experts
- Positive feedback on cultural representation
- High player satisfaction scores from Taiwanese users
- Successful validation by traditional gaming communities

### System Reliability
- 99.9% uptime achievement
- Sub-100ms response time consistency
- Zero security incidents related to game cheating
- Successful cross-platform feature parity

---

*These principles serve as our north star for technical decision-making. They should be referenced in every architectural discussion and code review to ensure we build a system that honors Taiwan Mahjong tradition while delivering exceptional technical excellence.*

**Document Status**: Draft v1.0  
**Review Schedule**: Monthly for first 6 months, then quarterly  
**Next Review Date**: September 7, 2025  
**Approved By**: [Pending architecture team review]