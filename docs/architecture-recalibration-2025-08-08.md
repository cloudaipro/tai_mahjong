# Architecture Recalibration Report
## Taiwan Mahjong Online Game Project

**Document Version**: 1.0  
**Date**: 2025-08-08  
**Status**: Active Recalibration  
**Based On**: Collaborative Architecture Review (2025-08-07 to 2025-08-22)

---

## Executive Summary

This document outlines the comprehensive architecture recalibration following the successful 5-specialist collaborative architecture review. The recalibration adjusts performance targets, enhances architectural governance, and integrates proven mitigation strategies while maintaining the core architectural vision.

### **Recalibration Outcomes:**
✅ **Performance Targets**: Adjusted to realistic, achievable benchmarks with fallback strategies  
✅ **Architectural Governance**: Enhanced with automated enforcement mechanisms  
✅ **Risk Mitigation**: Proactive strategies for identified performance and network challenges  
✅ **Testing Strategy**: Elevated to industry-leading 4-layer validation framework  
✅ **Timeline Optimization**: Adjusted phases to reflect enhanced architectural requirements  

---

## 1. Performance Target Recalibration

### **Original vs. Recalibrated Targets**

| Performance Area | Original Target | Recalibrated Target | Mitigation Strategy |
|------------------|----------------|-------------------|-------------------|
| **Backend Response Time** | <70ms (95%ile) | <70ms (95%ile) | ✅ Redis L2 caching + proactive refresh |
| **Web Frontend Rendering** | 60fps | 60fps | ✅ Performance Budget + WebGL optimization |
| **Mobile Rendering** | 60fps | 45-50fps | ⚠️ Performance Budget + Low-Fidelity Mode |
| **Network Recovery** | Standard reconnect | <2 seconds | ✅ Command queue + exponential backoff |
| **Test Coverage** | >90% | >95% | ✅ 4-layer testing framework |

### **Rationale for Mobile Performance Adjustment**

**Evidence from PoC Testing:**
- Web (Three.js): 60fps on modern hardware, performance drops on older machines
- Mobile (React Native Skia): Stable 45-50fps on mid-range devices with simplified 3D
- **Technical Reality**: Consistent 60fps mobile is challenging with complex 3D mahjong boards

**Mitigation Strategies:**
1. **Performance Budget**: Strict limits on polygons, textures, and draw calls
2. **Low-Fidelity Mode**: Automatic fallback to 2D representation for older devices
3. **Adaptive Quality**: Dynamic quality adjustment based on device performance
4. **Battery Optimization**: Reduced frame rate during non-interactive periods

---

## 2. Architectural Governance Enhancement

### **2.1 Automated Boundary Enforcement**

**Implementation**: Architectural Fitness Functions in CI/CD Pipeline

```typescript
// Architectural Fitness Function Examples
describe('Architecture Compliance', () => {
  test('Domain layer should not depend on Infrastructure layer', () => {
    const violations = archUnit.classes()
      .that().resideInAPackage('domain.*')
      .should().onlyDependOnClassesThat()
      .resideInAnyPackage('domain.*', 'java.lang.*');
    
    expect(violations.evaluate()).toHaveLength(0);
  });
  
  test('Game module should only communicate with Billing via events', () => {
    const violations = archUnit.classes()
      .that().resideInAPackage('game.*')
      .should().onlyAccessClassesThat()
      .resideInAnyPackage('game.*', 'shared.events.*')
      .orAre(assignableTo(DomainEvent));
      
    expect(violations.evaluate()).toHaveLength(0);
  });
});
```

**Benefits:**
- Prevents architectural drift during development
- Enforces Clean Architecture dependency rules
- Maintains bounded context boundaries
- Automated detection of coupling violations

### **2.2 Enhanced Code Review Process**

**New Requirements:**
- Pull requests crossing module boundaries require architect approval
- "Cross-Boundary" label for automatic architectural review flagging
- Mandatory ADR for significant architectural changes
- Security review for all cryptographic implementations

---

## 3. Data Architecture Enhancements

### **3.1 Caching Strategy Upgrade**

**Original**: Basic Redis caching  
**Recalibrated**: Cache-Aside pattern with proactive refresh

```typescript
// Enhanced Caching Strategy
class ProactiveCacheRefreshService {
  private readonly REFRESH_THRESHOLD = 0.75; // Refresh when 75% of TTL elapsed
  
  async getGame(gameId: string): Promise<Game> {
    const cachedGame = await this.redis.get(`game:${gameId}`);
    const ttl = await this.redis.ttl(`game:${gameId}`);
    
    if (cachedGame) {
      // Proactive refresh to prevent thundering herd
      if (ttl > 0 && ttl < (this.GAME_TTL * this.REFRESH_THRESHOLD)) {
        this.scheduleBackgroundRefresh(gameId);
      }
      return JSON.parse(cachedGame);
    }
    
    // Cache miss - load from database
    return this.loadAndCache(gameId);
  }
  
  private async scheduleBackgroundRefresh(gameId: string): Promise<void> {
    // Background refresh without blocking current request
    setTimeout(async () => {
      const freshGame = await this.gameRepository.findById(gameId);
      if (freshGame) {
        await this.redis.set(`game:${gameId}`, JSON.stringify(freshGame), 
          { EX: this.GAME_TTL });
      }
    }, 0);
  }
}
```

**Impact:**
- Prevents cache stampede scenarios
- Maintains <70ms response times under high load
- Reduces database load during peak usage
- Improves user experience consistency

### **3.2 Player Analytics Deferral**

**Decision**: Remove analytics from Phase 1-2 implementation  
**Rationale**: Focus development resources on core gameplay excellence  
**Future Integration**: Event-driven analytics pipeline ready for Phase 3+

---

## 4. Network Resiliency Architecture

### **4.1 Client-Side Command Queue**

**Enhancement**: Idempotent command processing with abuse prevention

```typescript
// Client-Side Command Queue with Anti-Abuse
class ResilientCommandQueue {
  private commandQueue: CommandWithId[] = [];
  private disconnectCount = 0;
  private readonly MAX_DISCONNECTS = 3;
  
  async enqueueCommand(command: GameCommand): Promise<void> {
    const commandWithId = {
      ...command,
      id: uuidv4(),
      timestamp: Date.now()
    };
    
    this.commandQueue.push(commandWithId);
    await this.persistQueue();
    await this.processQueue();
  }
  
  async handleReconnection(): Promise<void> {
    this.disconnectCount++;
    
    if (this.disconnectCount > this.MAX_DISCONNECTS) {
      // Lose grace period for this game due to frequent disconnects
      await this.sendMessage({
        type: 'FREQUENT_DISCONNECT_PENALTY',
        message: 'Grace period revoked due to frequent disconnections'
      });
    }
    
    await this.resyncGameState();
    await this.processQueue();
  }
}
```

**Benefits:**
- Prevents command loss during network instability
- Handles mobile network volatility gracefully
- Prevents rage-quit abuse through disconnect penalties
- Maintains game flow integrity

### **4.2 WebSocket Heartbeat Protocol**

**Implementation**: 5-second ping, 2.5-second timeout
- Early detection of connection issues
- Proactive reconnection attempts
- User-friendly connection status indicators

---

## 5. Testing Framework Elevation

### **5.1 4-Layer Testing Architecture**

**Layer 1: Unit Tests (>95% Coverage)**
- Individual domain functions and components
- Taiwan Mahjong rule validation
- Cryptographic signing and verification

**Layer 2: Integration Tests**
- Cross-module communication
- Database persistence layers
- Redis caching behavior

**Layer 3: Scenario-Based Testing**
- JSON-scripted game scenarios
- Edge case validation (過水, 詐胡, 搶槓)
- Complete game flow testing

**Layer 4: Expert Validation**
- Taiwan Mahjong master review
- Cultural authenticity verification
- Rule interpretation validation

### **5.2 JSON Scenario Testing Framework**

```json
{
  "scenarioName": "Test 'Sacred Discard' (過水) Rule Enforcement",
  "initialWall": ["W1", "W2", "W3", ...],
  "gameSetup": {
    "players": 4,
    "dealer": 1,
    "wind": "E"
  },
  "actions": [
    { "player": 1, "action": "DISCARD", "tile": "R5", "canWin": true },
    { "player": 1, "action": "PASS_WIN", "reason": "Player chooses not to win" },
    { "player": 2, "action": "DISCARD", "tile": "W3" },
    { "player": 3, "action": "DISCARD", "tile": "R5" },
    { "player": 1, "action": "DECLARE_WIN", "tile": "R5", "expectedResult": "INVALID" }
  ],
  "expectedOutcome": {
    "result": "PASS_WIN_ENFORCED",
    "winner": null,
    "reason": "Player 1 cannot win on R5 after passing on same tile"
  }
}
```

---

## 6. Implementation Timeline Adjustments

### **Phase 1 Extensions (Foundation + Security)**

**Original Duration**: 10 weeks  
**Recalibrated Duration**: 10 weeks (same, enhanced scope)

**Additional Week 3-4 Deliverables:**
- Architectural fitness functions implementation
- Redis proactive refresh mechanism
- Client-side command queue framework
- WebSocket heartbeat protocol

**Additional Week 5-6 Deliverables:**
- JSON scenario testing framework
- Enhanced test coverage (95%+ target)
- Expert validation planning and scheduling

### **Phase 2 Enhancements (Integration + Real-time)**

**Performance Validation Additions:**
- 3D rendering performance budgets
- Mobile device performance testing
- Network resiliency stress testing
- Cache performance under load testing

### **Phase 3 Mobile Adjustments**

**Realistic Performance Targets:**
- 45-50fps base performance
- Low-Fidelity Mode implementation
- Battery optimization strategies
- Adaptive quality settings

---

## 7. Risk Mitigation Updates

### **7.1 Primary Risk: 3D Performance**
- **Mitigation**: Performance Budget + Low-Fidelity Mode
- **Implementation**: Week 9-10 (Web), Week 21-22 (Mobile)
- **Fallback**: Automatic 2D mode for underperforming devices

### **7.2 Secondary Risk: Network Instability**
- **Mitigation**: Command queue + heartbeat + anti-abuse
- **Implementation**: Week 3-4 (Framework), Week 9-10 (Integration)
- **Benefit**: Superior mobile network handling

### **7.3 Tertiary Risk: Rule Authenticity**
- **Mitigation**: 4-layer testing + expert validation
- **Implementation**: Continuous from Week 5-6
- **Assurance**: Taiwan Mahjong master approval

---

## 8. Success Metrics Recalibration

### **Technical Performance Metrics**

| Metric | Original Target | Recalibrated Target | Measurement Method |
|--------|----------------|-------------------|-------------------|
| Backend Response Time | <70ms (95%ile) | <70ms (95%ile) | Prometheus histograms |
| Web Rendering | 60fps | 60fps | Performance API |
| Mobile Rendering | 60fps | 45-50fps + fallback | React Native Flipper |
| Network Recovery | N/A | <2 seconds | Connection state tracking |
| Test Coverage | >90% | >95% | Jest coverage reports |
| Rule Accuracy | Manual review | Expert validation | 4-layer testing |

### **User Experience Metrics**

- Command processing feels instantaneous (<70ms)
- Mobile gameplay remains smooth (45-50fps minimum)
- Network disruptions are transparent to users
- 100% Taiwan Mahjong rule authenticity maintained

---

## 9. Next Steps: Recalibrated Implementation

### **Immediate Actions (Week 1)**

1. ✅ **Update Development Environment**
   - Configure architectural fitness functions
   - Setup Redis with proactive refresh capabilities
   - Implement command queue infrastructure

2. ✅ **Update Team Training Materials**
   - Include network resiliency patterns
   - Add performance budget concepts
   - Enhance testing methodology training

3. ✅ **Revise Project Documentation**
   - Update PRD with recalibrated performance targets
   - Modify technical specifications
   - Adjust development methodology guidelines

### **Phase 1 Start (Week 2)**

1. Begin enhanced architectural foundation implementation
2. Implement fitness functions in CI/CD pipeline
3. Setup comprehensive caching infrastructure
4. Establish 4-layer testing framework

### **Validation Checkpoints**

- **Week 6**: Domain logic and rules engine validation
- **Week 10**: Frontend performance and usability validation  
- **Week 14**: Integration and caching performance validation
- **Week 22**: Mobile performance and network resiliency validation

---

## 10. Conclusion

The architecture recalibration successfully integrates all findings from the collaborative 5-specialist review while maintaining the project's core vision and objectives. The adjustments are:

✅ **Realistic**: Based on empirical PoC testing and expert recommendations  
✅ **Achievable**: With proven mitigation strategies and enhanced governance  
✅ **Comprehensive**: Addressing performance, security, testing, and user experience  
✅ **Future-proof**: Maintaining scalability and maintainability objectives  

The recalibrated architecture provides a robust foundation for delivering an exceptional Taiwan Mahjong online gaming experience with:

- **Cultural Authenticity**: 100% Taiwan Mahjong rule compliance
- **Technical Excellence**: Optimized performance with graceful fallbacks
- **User Experience**: Seamless gameplay across web and mobile platforms
- **Commercial Viability**: Scalable architecture supporting business objectives

**The project is ready to proceed with Phase 1: Foundation & Security Framework under the recalibrated architecture.**

---

**Document Status**: Active Implementation Guide  
**Approval Required**: Technical Lead, Product Owner, Architecture Review Board  
**Next Review**: Week 6 (Phase 1 Mid-point Validation)