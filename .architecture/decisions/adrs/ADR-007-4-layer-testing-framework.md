# ADR-007: 4-Layer Testing Framework for Taiwan Mahjong Rules

**Status**: Accepted  
**Date**: 2025-08-08  
**Decision Makers**: Lisa Wang (Game Systems Architect)  
**Related**: ADR-001 (Hybrid Architecture), Taiwan Mahjong Domain Requirements

## Context

The collaborative architecture review emphasized that **100% Taiwan Mahjong rule authenticity** is paramount to project success. Any deviation from traditional 16-tile Taiwan Mahjong rules would damage player trust and market acceptance.

Game Systems Architect Lisa Wang conducted analysis showing:

### Complexity Factors:
- **144-tile system** with flowers, winds, and dragons
- **Traditional 台數計算** with 20+ scoring patterns  
- **Edge cases**: 過水 (sacred discard), 詐胡 (false win), 搶槓 (robbing kong)
- **Cultural nuances**: Regional variations and interpretation differences
- **Rule interactions**: Complex combinations and precedence rules

### Risk Assessment:
- Standard unit testing insufficient for rule complexity
- Manual testing cannot cover all edge cases
- Expert knowledge required for cultural authenticity
- Automated testing needed for continuous validation

## Decision

We will implement a **comprehensive 4-layer testing framework** to ensure 100% Taiwan Mahjong rule accuracy:

## Layer 1: Unit Tests (>95% Coverage)

**Scope**: Individual functions and domain objects in isolation  
**Framework**: Jest with comprehensive Taiwan Mahjong test suites  
**Target**: >95% code coverage for all domain logic

```typescript
// Example Unit Tests
describe('Taiwan Mahjong Tile Validation', () => {
  test('should identify valid wind tiles', () => {
    expect(TileType.isWind('E')).toBe(true);
    expect(TileType.isWind('S')).toBe(true);
    expect(TileType.isWind('W')).toBe(true);
    expect(TileType.isWind('N')).toBe(true);
    expect(TileType.isWind('R1')).toBe(false);
  });
  
  test('should correctly calculate All Pungs (碰碰胡) score', () => {
    const hand = [
      ['R1', 'R1', 'R1'], // Red 1 Pung
      ['G5', 'G5', 'G5'], // Green 5 Pung  
      ['B9', 'B9', 'B9'], // Blue 9 Pung
      ['E', 'E', 'E'],    // East Wind Pung
      ['W', 'W']          // West Wind Pair
    ];
    
    const score = ScoringEngine.calculateTaishu(hand);
    expect(score.patterns).toContain('ALL_PUNGS');
    expect(score.taishu).toBeGreaterThanOrEqual(3);
  });
});

describe('Game State Transitions', () => {
  test('should prevent 詐胡 (false win) attempts', () => {
    const game = createTestGame();
    const invalidHand = [
      ['R1', 'R2'], ['G3', 'G4'], ['B5', 'B6'], ['E'], ['S', 'S']
    ];
    
    const result = game.executeHupai(playerId, invalidHand);
    expect(result.isFailure()).toBe(true);
    expect(result.error).toEqual('INVALID_WIN_ATTEMPT');
  });
});
```

## Layer 2: Integration Tests

**Scope**: Cross-module interactions and data persistence  
**Focus**: How domain, application, and infrastructure layers work together

```typescript
describe('Game Integration Tests', () => {
  test('should persist game state changes through repository', async () => {
    const game = await gameService.createGame(players);
    await gameService.executeMopai(game.id, player1.id);
    
    const persistedGame = await gameRepository.findById(game.id);
    expect(persistedGame.getCurrentPlayer()).toBe(player1.id);
    expect(persistedGame.getPlayerHand(player1.id)).toHaveLength(17);
  });
  
  test('should handle concurrent player actions correctly', async () => {
    const game = await gameService.createGame(players);
    
    // Simulate simultaneous Pung and Chow attempts
    const pungResult = gameService.executePeng(game.id, player2.id, ['R5']);
    const chowResult = gameService.executeChow(game.id, player3.id, ['R4', 'R6']);
    
    const results = await Promise.all([pungResult, chowResult]);
    
    // Pung should win (higher priority)
    expect(results[0].isSuccess()).toBe(true);
    expect(results[1].isSuccess()).toBe(false);
  });
});
```

## Layer 3: Scenario-Based Testing (JSON Scripts)

**Scope**: Complete game scenarios with predetermined outcomes  
**Format**: JSON-defined game scripts for reproducible testing

### JSON Scenario Format:
```json
{
  "scenarioName": "Test 'Sacred Discard' (過水) Rule Enforcement",
  "description": "Player cannot win on a tile they previously passed",
  "initialWall": [
    "R1", "R2", "R3", "R4", "R5", "R6", "R7", "R8", "R9",
    "G1", "G2", "G3", "G4", "G5", "G6", "G7", "G8", "G9",
    "B1", "B2", "B3", "B4", "B5", "B6", "B7", "B8", "B9",
    "E", "S", "W", "N", "RED", "GREEN", "WHITE",
    "F1", "F2", "F3", "F4", "F5", "F6", "F7", "F8"
  ],
  "gameSetup": {
    "players": 4,
    "dealer": 1,
    "wind": "E",
    "shuffled": false
  },
  "actions": [
    {
      "step": 1,
      "player": 1,
      "action": "DISCARD",
      "tile": "R5",
      "canWin": true,
      "description": "Player 1 discards R5, could win with it"
    },
    {
      "step": 2,
      "player": 1,
      "action": "PASS_WIN",
      "reason": "Player chooses not to declare win"
    },
    {
      "step": 3,
      "player": 2,
      "action": "DISCARD",
      "tile": "G3"
    },
    {
      "step": 4,
      "player": 3,
      "action": "DISCARD", 
      "tile": "R5",
      "description": "Same tile (R5) discarded by different player"
    },
    {
      "step": 5,
      "player": 1,
      "action": "DECLARE_WIN",
      "tile": "R5",
      "expectedResult": "INVALID",
      "description": "Should be blocked by 過水 rule"
    }
  ],
  "expectedOutcome": {
    "result": "SACRED_DISCARD_VIOLATION",
    "winner": null,
    "blocked_player": 1,
    "reason": "Player 1 cannot win on R5 after previously passing on the same tile"
  },
  "validation": {
    "game_continues": true,
    "current_player": 4,
    "sacred_discard_recorded": {
      "player": 1,
      "tile": "R5"
    }
  }
}
```

### Scenario Test Categories:
1. **Basic Gameplay**: Standard mopai, dapai, pengpai, gangpai, hupai
2. **Special Rules**: 過水, 詐胡, 搶槓, 海底撈月
3. **Scoring Patterns**: All 20+ traditional 台數計算 patterns
4. **Edge Cases**: Multiple simultaneous claims, unusual tile combinations
5. **Game Flow**: Complete games from start to finish

## Layer 4: Expert Validation

**Scope**: Cultural authenticity and rule interpretation validation  
**Process**: Professional Taiwan Mahjong masters review and approve

### Expert Validation Process:

1. **Master Selection**
   - Recognized Taiwan Mahjong tournament players
   - Cultural authenticity specialists
   - Regional rule variation experts

2. **Validation Sessions**
   - Live gameplay testing with masters
   - Rule interpretation discussions
   - Edge case scenario validation
   - Cultural context verification

3. **Approval Criteria**
   - 100% traditional rule compliance
   - Authentic 台數計算 implementation
   - Correct handling of all edge cases
   - Cultural terminology accuracy

```typescript
// Expert Validation Integration
class ExpertValidationService {
  async scheduleValidationSession(
    gameVersion: string,
    validationAreas: ValidationArea[]
  ): Promise<ValidationSession> {
    const session = new ValidationSession({
      gameVersion,
      areas: validationAreas,
      experts: await this.selectExperts(validationAreas),
      scenarios: await this.prepareScenarios(validationAreas)
    });
    
    return await this.bookingService.schedule(session);
  }
  
  async recordExpertFeedback(
    sessionId: string,
    feedback: ExpertFeedback
  ): Promise<void> {
    await this.feedbackRepository.save({
      sessionId,
      expert: feedback.expert,
      areas: feedback.validatedAreas,
      issues: feedback.identifiedIssues,
      approval: feedback.overallApproval,
      recommendations: feedback.recommendations
    });
    
    if (!feedback.overallApproval) {
      await this.createFixTasksFromFeedback(feedback);
    }
  }
}
```

## Implementation Strategy

### Testing Framework Setup (Week 5-6)
```typescript
// Test Infrastructure
class TaiwanMahjongTestFramework {
  private unitTestSuite: JestTestSuite;
  private integrationTestSuite: TestContainerSuite;
  private scenarioTestRunner: JSONScenarioRunner;
  private expertValidationTracker: ExpertValidationTracker;
  
  async runAllLayers(): Promise<TestResults> {
    const results = {
      layer1: await this.runUnitTests(),
      layer2: await this.runIntegrationTests(),
      layer3: await this.runScenarioTests(),
      layer4: await this.getExpertValidationStatus()
    };
    
    return this.generateComprehensiveReport(results);
  }
}
```

### Continuous Validation Pipeline
- **Every Commit**: Layer 1 (Unit Tests) - must pass to merge
- **Daily**: Layer 2 (Integration Tests) - automated overnight runs  
- **Weekly**: Layer 3 (Scenario Tests) - comprehensive scenario validation
- **Monthly**: Layer 4 (Expert Review) - scheduled validation sessions

## Success Criteria

### Quantitative Metrics
- **Layer 1**: >95% code coverage of domain logic
- **Layer 2**: 100% critical integration paths tested
- **Layer 3**: 100+ comprehensive game scenarios validated
- **Layer 4**: Approval from 3+ recognized Taiwan Mahjong experts

### Qualitative Measures
- Zero rule accuracy issues identified in user feedback
- Expert validation approval for cultural authenticity
- Competitive tournament players accept rule implementation
- Community recognition as authentic Taiwan Mahjong

## Implementation Timeline

- **Week 5-6**: Framework setup, Layer 1 implementation
- **Week 7-8**: Layer 2 integration tests
- **Week 11-12**: Layer 3 scenario testing system
- **Week 12-13**: First expert validation session
- **Week 18-19**: Cultural authenticity review
- **Week 30-31**: Final expert approval

## Benefits

### Technical Benefits
- **Confidence**: 100% rule accuracy assurance
- **Regression Prevention**: Automated detection of rule violations
- **Documentation**: Tests serve as living rule specification
- **Maintainability**: Safe refactoring with comprehensive coverage

### Business Benefits
- **Player Trust**: Authentic Taiwan Mahjong implementation
- **Market Credibility**: Expert-validated cultural accuracy
- **Competitive Advantage**: Superior rule implementation vs. competitors
- **Risk Mitigation**: Prevents costly rule-accuracy related issues

## Consequences

### Positive
- Guarantees 100% Taiwan Mahjong rule authenticity
- Provides comprehensive regression testing
- Creates living documentation of rules
- Enables confident code refactoring
- Builds market credibility through expert validation

### Negative  
- Significant upfront investment in test framework development
- Ongoing maintenance of large test suite
- Expert validation requires budget and scheduling
- Complex scenario testing may slow development velocity

### Risks
- Expert availability for validation sessions
- Potential disagreements between experts on edge cases
- Test suite maintenance overhead as rules evolve
- Over-testing could slow development progress

## Compliance

- **Owner**: Game Systems Architect (Lisa Wang)
- **Success Criteria**: 100% expert approval + >95% automated coverage
- **Expert Budget**: Allocated for 6 validation sessions
- **Review Schedule**: Weekly test results validation

---

**Status**: Active Implementation  
**Approved By**: Architecture Review Board ✅  
**Priority**: Critical (Core Game Authenticity)