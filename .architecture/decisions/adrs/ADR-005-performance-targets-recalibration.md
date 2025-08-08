# ADR-005: Performance Targets Recalibration

**Status**: Accepted  
**Date**: 2025-08-08  
**Decision Makers**: Collaborative Architecture Review Team (5 specialists)  
**Related**: ADR-001 (Hybrid Architecture), Gemini PoC Testing Results

## Context

The collaborative architecture review included empirical performance testing through a proof-of-concept (PoC) implementation. The findings revealed that the original performance targets, while aspirational, required adjustment to reflect technical realities:

### PoC Findings:
- **Web (Three.js)**: 60fps achieved on modern hardware, performance degradation on older machines
- **Mobile (React Native Skia)**: Stable 45-50fps on mid-range devices with simplified 3D models
- **Technical Reality**: Consistent 60fps mobile rendering with complex 3D mahjong boards is challenging

The review team, led by Frontend Architect David Kim and Mobile Architect Sarah Patel, recommended realistic target adjustment with comprehensive mitigation strategies.

## Decision

We will **recalibrate performance targets** to reflect achievable benchmarks while maintaining exceptional user experience:

### Recalibrated Performance Targets

| Performance Area | Original Target | New Target | Mitigation Strategy |
|------------------|----------------|------------|-------------------|
| **Backend Response** | <70ms (95%ile) | <70ms (95%ile) | ✅ Redis L2 + proactive refresh |
| **Web Rendering** | 60fps | 60fps | ✅ Performance Budget + WebGL optimization |
| **Mobile Rendering** | 60fps | **45-50fps** | ⚠️ Performance Budget + Low-Fidelity Mode |
| **Network Recovery** | Standard | **<2 seconds** | ✅ Command queue + exponential backoff |

### Mobile Performance Mitigation Strategy

1. **Performance Budget Framework**
   - Strict polygon count limits
   - Texture size constraints
   - Draw call optimization
   - Shader complexity limits

2. **Low-Fidelity Mode**
   - Automatic fallback to 2D representation
   - Triggered when device cannot maintain target fps
   - User-selectable option for battery conservation
   - Maintains full gameplay functionality

3. **Adaptive Quality System**
   - Dynamic quality adjustment based on device performance
   - Battery-aware rendering optimization
   - Frame rate monitoring and auto-adjustment

4. **Device Performance Profiling**
   - Runtime device capability detection
   - Performance tier classification
   - Optimal settings auto-selection

## Implementation

```typescript
// Performance Budget Enforcement
class MobilePerformanceBudget {
  private readonly MAX_POLYGONS = 5000;
  private readonly MAX_TEXTURE_SIZE = 512;
  private readonly MAX_DRAW_CALLS = 50;
  private readonly TARGET_FPS = 45;
  
  private currentFPS = 60;
  private qualityLevel = 'high';
  
  monitorPerformance(): void {
    setInterval(() => {
      this.currentFPS = this.measureFPS();
      
      if (this.currentFPS < this.TARGET_FPS) {
        this.degradeQuality();
      } else if (this.currentFPS > this.TARGET_FPS + 10) {
        this.improveQuality();
      }
    }, 1000);
  }
  
  private degradeQuality(): void {
    switch (this.qualityLevel) {
      case 'high':
        this.qualityLevel = 'medium';
        this.reducePoly

nCount(0.7);
        break;
      case 'medium':
        this.qualityLevel = 'low';
        this.reduceTextureQuality(0.5);
        break;
      case 'low':
        this.enableLowFidelityMode();
        break;
    }
  }
  
  private enableLowFidelityMode(): void {
    this.switchTo2DMode();
    this.showUserNotification('Switched to Low-Fidelity Mode for better performance');
  }
}
```

## Rationale

### Why This Decision Makes Sense

1. **Evidence-Based**: Grounded in empirical PoC testing results
2. **User-Focused**: Maintains smooth gameplay experience on all devices
3. **Realistic**: Achievable targets prevent development frustration
4. **Future-Proof**: Fallback mechanisms ensure broad device compatibility
5. **Battery-Conscious**: Optimized for extended mobile gaming sessions

### Alternative Considered

- **Maintain 60fps Target**: Rejected due to PoC evidence of infeasibility
- **Drop 3D Entirely**: Rejected as it reduces visual appeal and immersion
- **Web-Only Release**: Rejected as mobile is critical for target market

## Consequences

### Positive
- **Realistic Expectations**: Development team can confidently meet targets
- **Broader Device Support**: Older devices can still run the application
- **Battery Optimization**: Extended gameplay sessions on mobile
- **User Choice**: Players can prioritize performance vs. visual quality
- **Technical Feasibility**: Proven achievable through PoC testing

### Negative
- **Reduced Mobile Visual Quality**: Compared to original aspirational targets
- **Additional Complexity**: Low-Fidelity Mode requires extra development effort
- **Performance Monitoring Overhead**: Runtime performance tracking required

### Risks
- **User Perception**: Some users may perceive lower mobile fps as inferior quality
- **Competitive Disadvantage**: If competitors achieve higher mobile performance
- **Implementation Complexity**: Adaptive quality system adds technical complexity

## Success Metrics

- **Mobile FPS**: Sustained 45-50fps on mid-range devices (Samsung Galaxy A50, iPhone XR)
- **Battery Life**: <20% additional drain during 1-hour gaming session
- **User Satisfaction**: >80% mobile users rate visual quality as "Good" or "Excellent"
- **Device Coverage**: Application runs acceptably on 95%+ of target devices

## Implementation Plan

1. **Week 9-10** (Phase 1): Performance Budget framework implementation
2. **Week 21-22** (Phase 3): Low-Fidelity Mode development
3. **Week 24-25** (Phase 3): Adaptive quality system integration
4. **Week 26-27** (Phase 4): Mobile device performance validation

## Compliance

- **Owner**: Frontend Architect (David Kim) + Mobile Architect (Sarah Patel)
- **Validation**: Continuous mobile device farm testing
- **Success Criteria**: Target FPS achieved on 95%+ test devices
- **Review Schedule**: Weekly performance validation during Phase 3

---

**Status**: Active Implementation  
**Approved By**: Architecture Review Board ✅  
**Impact**: Medium (User experience optimization)