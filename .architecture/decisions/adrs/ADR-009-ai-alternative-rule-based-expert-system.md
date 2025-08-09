# ADR-009: AI Alternative - Rule-Based Expert System

## Status
**Accepted** ✅

## Context

The Taiwan Mahjong project originally included AI components for single-player practice mode, tutorial systems, and premium AI coaching features. However, implementing machine learning-based AI introduces significant complexity, uncertainty, and maintenance challenges:

### Problems with AI Approach
- **Implementation Uncertainty**: ML models require training data, hyperparameter tuning, and unpredictable behavior
- **Cultural Risk**: AI might learn incorrect patterns inconsistent with authentic Taiwan Mahjong traditions  
- **Testing Complexity**: Non-deterministic behavior makes comprehensive testing difficult
- **Resource Requirements**: Training infrastructure, model deployment, and ongoing retraining needs
- **Maintenance Burden**: Model drift, performance degradation, and complex debugging

### Requirements to Preserve
- Practice opponents at multiple skill levels (beginner/intermediate/advanced)
- Tutorial and learning systems for Taiwan Mahjong rules
- Hint/coaching functionality for premium features
- Authentic Taiwan Mahjong gameplay behavior
- <70ms response time performance targets

## Decision

**Replace AI components with a Rule-Based Expert System** that combines deterministic decision-making with sophisticated behavioral modeling.

### Core Architecture

#### 1. Rule-Based Decision Engine
```typescript
interface NpcDecisionEngine {
  evaluateGameState(state: GameState): ContextAnalysis;
  generateMoveOptions(context: ContextAnalysis): WeightedMove[];
  selectOptimalMove(moves: WeightedMove[], profile: NpcProfile): GameAction;
  validateCulturalAuthenticity(move: GameAction, context: GameContext): boolean;
}
```

#### 2. Weighted Probability System
```typescript
interface WeightedMove {
  action: GameAction;
  strategic_value: number;
  risk_assessment: number;
  cultural_authenticity: number;
  final_weight: number;
}
```

#### 3. NPC Behavioral Profiles
```typescript
interface NpcProfile {
  name: string;
  difficulty_level: 'beginner' | 'intermediate' | 'advanced';
  defensive_weight: number;        // 0.0 - 1.0
  aggressive_weight: number;       // 0.0 - 1.0  
  risk_tolerance: number;          // 0.0 - 1.0
  cultural_patterns: AuthenticPattern[];
  rule_complexity_limit: number;   // Max evaluation depth
}
```

### Implementation Strategy

#### 1. YAML-Based Rule Configuration
```yaml
# beginner-profile.yml
profile:
  name: "Beginner Opponent"
  difficulty: "beginner"
  defensive_weight: 0.8
  aggressive_weight: 0.3
  risk_tolerance: 0.2
  
rules:
  - priority: 1
    condition: "can_declare_hu"
    action: "declare_hu"
    weight: 1.0
    
  - priority: 2  
    condition: "safe_discard_available"
    action: "discard_safe_tile"
    weight: 0.9
```

#### 2. Graduated Difficulty Levels

**Beginner Profile:**
- Simple heuristics only
- Always discard terminals first
- Basic tile safety rules
- No complex strategic planning

**Intermediate Profile:**
- Defensive play patterns
- Basic 台數 (scoring) optimization
- Conservative claiming strategies
- Hand reading fundamentals

**Advanced Profile:**
- Master-level rule complexity
- Sophisticated 台數 calculations
- Aggressive claiming strategies  
- Complex hand reading and probability analysis

#### 3. Cultural Authentication Layer
- **Taiwan Expert Validation**: All rule sets validated by traditional Taiwan Mahjong masters
- **Authentic Pattern Encoding**: Region-specific playing styles and strategies
- **Cultural Terminology**: Use traditional Taiwan Mahjong terms and concepts
- **Traditional Wisdom**: Encode classical strategic principles

### Feature Implementation

#### 1. Practice Mode Opponents
```typescript
class PracticeOpponent {
  constructor(
    private profile: NpcProfile,
    private ruleEngine: NpcDecisionEngine,
    private authenticator: CulturalAuthenticator
  ) {}
  
  async makeMove(gameState: GameState): Promise<GameAction> {
    const context = this.ruleEngine.evaluateGameState(gameState);
    const options = this.ruleEngine.generateMoveOptions(context);
    const move = this.ruleEngine.selectOptimalMove(options, this.profile);
    
    // Ensure cultural authenticity
    if (!this.authenticator.validateMove(move, context)) {
      return this.authenticator.suggestAlternative(move, context);
    }
    
    return move;
  }
}
```

#### 2. Hint System (Reusing Same Engine)
```typescript
class PlayerHintSystem {
  constructor(private expertEngine: NpcDecisionEngine) {}
  
  generateHint(playerHand: Tile[], gameState: GameState): MoveRecommendation {
    // Reuse the same rule engine that powers opponents
    const context = this.expertEngine.evaluateGameState(gameState);
    const options = this.expertEngine.generateMoveOptions(context);
    
    return {
      recommended_move: options[0].action,
      explanation: this.generateExplanation(options[0]),
      alternative_moves: options.slice(1, 3)
    };
  }
}
```

#### 3. Interactive Tutorial System
```typescript
interface TutorialScenario {
  scenario_id: string;
  description: string;
  initial_state: GameState;
  expected_moves: GameAction[];
  learning_objectives: string[];
  cultural_context: string;
}
```

### Performance Guarantees

#### 1. Response Time Optimization
- **Hard Limit**: <70ms decision making
- **Rule Chain Depth**: Maximum 10 evaluation levels
- **Lookup Tables**: O(1) access for common scenarios
- **Memory Caching**: YAML rules loaded once at startup

#### 2. Scalability Design
```typescript
class OptimizedRuleEngine {
  private ruleCache: Map<GameStateHash, WeightedMove[]>;
  private lookupTables: StrategyLookupTables;
  
  // O(1) for common positions
  getCachedMoves(stateHash: string): WeightedMove[] | null {
    return this.ruleCache.get(stateHash);
  }
  
  // Bounded evaluation time
  evaluateWithTimeout(state: GameState, maxMs: number): GameAction {
    // Implementation with timeout guarantee
  }
}
```

## Benefits

### Technical Benefits
- **100% Deterministic**: Predictable, testable behavior
- **No Training Required**: Rules defined by domain experts
- **Lightweight**: Minimal computational overhead
- **Maintainable**: Human-readable YAML configuration
- **Version Control**: All logic tracked in source code

### Cultural Benefits  
- **Expert-Validated**: Rules approved by Taiwan Mahjong masters
- **Authentic Patterns**: Traditional playing styles preserved
- **Cultural Nuances**: Regional variations properly encoded
- **Educational Value**: Players learn authentic strategies

### Business Benefits
- **Reduced Risk**: No ML uncertainty or model failure
- **Lower Costs**: No training infrastructure or ongoing model maintenance  
- **Faster Development**: Deterministic implementation and testing
- **Easier Debugging**: Clear rule tracing and troubleshooting

## Testing Strategy

### 1. Deterministic Unit Tests
```typescript
describe('Advanced Opponent', () => {
  it('should declare hu when winning hand available', () => {
    const gameState = createWinningScenario();
    const move = advancedOpponent.makeMove(gameState);
    expect(move.type).toBe('DECLARE_HU');
  });
  
  it('should discard safe tile when opponent is ready', () => {
    const gameState = createOpponentReadyScenario();
    const move = advancedOpponent.makeMove(gameState);
    expect(isSafeTile(move.tile, gameState)).toBe(true);
  });
});
```

### 2. Scenario-Based Testing
- **Edge Cases**: 過水 (passing), 詐胡 (false win), 搶槓 (robbing kong)
- **Cultural Scenarios**: Traditional Taiwan Mahjong situations
- **Performance Tests**: <70ms response time validation
- **Behavioral Tests**: Profile-specific strategy verification

### 3. Expert Validation Layer
- Taiwan Mahjong masters review opponent behavior
- Cultural authenticity validation
- Traditional pattern verification
- Educational value assessment

## Alternatives Considered

### Machine Learning AI
**Rejected** - Too complex, unpredictable, and culturally risky

### Simple Random Logic  
**Rejected** - Insufficient strategic depth and learning value

### Pre-recorded Game Sequences
**Rejected** - Limited variety and poor adaptability

## Implementation Plan

### Phase 1: Core Engine (2 weeks)
- Basic rule engine infrastructure
- YAML configuration system
- Simple beginner profile implementation

### Phase 2: Advanced Profiles (3 weeks)  
- Intermediate and advanced rule sets
- Weighted probability system
- Cultural authentication layer

### Phase 3: Feature Integration (2 weeks)
- Hint system implementation
- Tutorial scenario engine
- Performance optimization

### Phase 4: Expert Validation (1 week)
- Taiwan Mahjong master review
- Cultural authenticity verification
- Final adjustments and approval

## Consequences

### Positive
- ✅ **Eliminates AI complexity** while preserving functionality
- ✅ **Ensures cultural authenticity** through expert validation
- ✅ **Provides deterministic behavior** for reliable testing
- ✅ **Reduces long-term maintenance** burden
- ✅ **Enables rapid iteration** on opponent behavior

### Negative
- ❌ **Static behavior** - opponents won't "learn" from gameplay
- ❌ **Manual rule updates** required for strategy improvements
- ❌ **Limited emergent behavior** compared to ML approaches

### Neutral
- 🔄 **Development effort** shifts from ML expertise to game logic design
- 🔄 **Maintenance model** changes from model retraining to rule refinement

## Compliance

### ADR Alignment
- **ADR-003A**: Aligns with DDD approach through domain expert rule encoding
- **ADR-003B**: Meets <70ms performance targets through optimization design
- **ADR-007**: Supports 4-layer testing framework with deterministic behavior

### Project Requirements
- ✅ **Cultural Authenticity**: Expert-validated Taiwan Mahjong patterns
- ✅ **Multi-level Practice**: Graduated difficulty through rule complexity
- ✅ **Performance Targets**: <70ms response time guarantees  
- ✅ **Educational Value**: Authentic strategy learning through hint system

## Decision Owners
- **Primary**: Game Systems Architect (Lisa Wang)
- **Supporting**: Systems Architect (Alex Chen)
- **Validation**: Taiwan Mahjong Cultural Experts

## References
- `docs/ai-alternative-analysis.md` - Original technical analysis
- `docs/ai-alternatives-collaborative-meeting-2025-08-09.md` - Collaborative discussion
- `reference/` - Taiwan Mahjong rules and cultural documentation

---

**ADR Status**: Accepted ✅  
**Implementation Status**: Ready for Phase 1 development  
**Next Review**: Upon Phase 1 completion