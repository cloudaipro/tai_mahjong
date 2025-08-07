# Taiwan Mahjong: Hybrid Architecture Implementation Guidelines

**Document Version**: 1.0  
**Date**: 2025-08-07  
**Target Audience**: Development Team, Technical Leads  
**Status**: Active Implementation Guidelines

---

## Overview

This document provides comprehensive implementation guidelines for the hybrid architectural approach adopted for the Taiwan Mahjong online game project. The hybrid architecture combines Command Pattern, State Machines, Request-Response, and Selective Event-Driven patterns to optimize for performance, maintainability, and Taiwan Mahjong domain requirements.

## Architecture Decision Summary

Based on **ADR-001-hybrid-architecture-decision**, we use different patterns for different concerns:

- **Command Pattern**: Game actions (摸牌, 打牌, 吃碰槓胡)
- **State Machines**: Game flow and room lifecycle  
- **Request-Response**: Real-time player interactions
- **Selective Event-Driven**: Non-critical operations (chat, analytics, notifications)

---

## 1. Pattern Usage Guidelines

### When to Use Each Pattern

| Scenario | Pattern | Rationale | Example |
|----------|---------|-----------|---------|
| Taiwan Mahjong game actions | Command Pattern | Deterministic, serializable, replay-able | 摸牌, 打牌, 碰牌, 槓牌, 胡牌 |
| Game state transitions | State Machine | Prevents invalid states, clear flow | 發牌→進行→結算 |
| Immediate player responses | Request-Response | Low latency, predictable | Score calculation, move validation |
| Background processing | Event-Driven | Non-blocking, loose coupling | Chat messages, analytics |
| Room lifecycle | State Machine | Controlled transitions | Created→Waiting→Playing→Finished |
| User authentication | Request-Response | Synchronous validation | Login, profile updates |

### Pattern Selection Flowchart

```
Is it a Taiwan Mahjong game action (摸牌,打牌,etc.)?
├─ YES → Use Command Pattern
└─ NO
   ├─ Does it involve state transitions?
   │  ├─ YES → Use State Machine
   │  └─ NO
   │     ├─ Does it require immediate response (<100ms)?
   │     │  ├─ YES → Use Request-Response
   │     │  └─ NO → Use Event-Driven (if non-critical)
```

---

## 2. Command Pattern Implementation

### 2.1 Command Structure

```typescript
// Base command interface
interface GameCommand {
  readonly type: string;
  readonly playerId: string;
  readonly roomId: string;
  readonly timestamp: number;
  readonly sequenceId: number;
  
  // Validation method
  validate(gameState: GameState): ValidationResult;
  
  // Execution method  
  execute(gameState: GameState): GameState;
  
  // Serialization for network/storage
  serialize(): string;
}

// Taiwan Mahjong specific commands
class DrawTileCommand implements GameCommand {
  readonly type = 'DRAW_TILE';
  
  constructor(
    readonly playerId: string,
    readonly roomId: string,
    readonly tilePosition?: number // Optional for specific draws
  ) {
    this.timestamp = Date.now();
    this.sequenceId = generateSequenceId();
  }

  validate(gameState: GameState): ValidationResult {
    // Validate if player can draw tile
    // Check game phase, player turn, etc.
    return {
      isValid: gameState.currentPlayer === this.playerId &&
               gameState.phase === GamePhase.PLAYING,
      errorCode: isValid ? null : 'INVALID_TURN'
    };
  }

  execute(gameState: GameState): GameState {
    // Execute tile draw
    const newState = { ...gameState };
    const tile = newState.tileWall.pop();
    newState.players[this.playerId].hand.push(tile);
    newState.lastAction = this;
    return newState;
  }

  serialize(): string {
    return JSON.stringify({
      type: this.type,
      playerId: this.playerId,
      roomId: this.roomId,
      timestamp: this.timestamp,
      sequenceId: this.sequenceId
    });
  }
}
```

### 2.2 Command Handler Service

```typescript
class CommandHandlerService {
  private commandHistory: GameCommand[] = [];
  private validators: Map<string, CommandValidator> = new Map();
  
  async processCommand(command: GameCommand): Promise<CommandResult> {
    // 1. Validate command
    const validation = await this.validateCommand(command);
    if (!validation.isValid) {
      return { success: false, error: validation.errorCode };
    }
    
    // 2. Execute command
    const currentState = await this.getGameState(command.roomId);
    const newState = command.execute(currentState);
    
    // 3. Persist state and command
    await Promise.all([
      this.saveGameState(command.roomId, newState),
      this.saveCommandToHistory(command)
    ]);
    
    // 4. Broadcast to players (Request-Response pattern)
    await this.broadcastToRoom(command.roomId, {
      type: 'GAME_UPDATE',
      state: newState,
      lastCommand: command
    });
    
    this.commandHistory.push(command);
    return { success: true, newState };
  }
  
  // Replay functionality for debugging/analysis
  async replayGame(roomId: string, upToSequence?: number): Promise<GameState> {
    const commands = await this.getCommandHistory(roomId, upToSequence);
    let gameState = this.getInitialGameState(roomId);
    
    for (const command of commands) {
      gameState = command.execute(gameState);
    }
    
    return gameState;
  }
}
```

### 2.3 Taiwan Mahjong Command Types

```typescript
// Complete set of Taiwan Mahjong commands
enum MahjongCommandType {
  DRAW_TILE = 'DRAW_TILE',           // 摸牌
  DISCARD_TILE = 'DISCARD_TILE',     // 打牌  
  CHOW = 'CHOW',                     // 吃牌
  PONG = 'PONG',                     // 碰牌
  KONG = 'KONG',                     // 槓牌
  WIN = 'WIN',                       // 胡牌
  PASS = 'PASS',                     // 過牌
  DECLARE_READY = 'DECLARE_READY'    // 聽牌宣告
}

// Command factory
class MahjongCommandFactory {
  static createCommand(type: MahjongCommandType, params: any): GameCommand {
    switch (type) {
      case MahjongCommandType.DRAW_TILE:
        return new DrawTileCommand(params.playerId, params.roomId);
      case MahjongCommandType.DISCARD_TILE:
        return new DiscardTileCommand(params.playerId, params.roomId, params.tileId);
      case MahjongCommandType.PONG:
        return new PongCommand(params.playerId, params.roomId, params.targetTile);
      // ... other command types
      default:
        throw new Error(`Unknown command type: ${type}`);
    }
  }
}
```

---

## 3. State Machine Implementation

### 3.1 Game State Machine

```typescript
// Game state definitions
enum GameState {
  WAITING_FOR_PLAYERS = 'WAITING_FOR_PLAYERS',
  DEALING = 'DEALING',                    // 發牌階段
  PLAYING = 'PLAYING',                    // 遊戲進行
  SCORING = 'SCORING',                    // 計分階段  
  FINISHED = 'FINISHED'                   // 遊戲結束
}

enum GameEvent {
  PLAYERS_READY = 'PLAYERS_READY',
  DEALING_COMPLETE = 'DEALING_COMPLETE', 
  ROUND_COMPLETE = 'ROUND_COMPLETE',
  GAME_END = 'GAME_END'
}

// State machine configuration
class GameStateMachine {
  private transitions: Map<string, string[]> = new Map([
    [GameState.WAITING_FOR_PLAYERS, [GameState.DEALING]],
    [GameState.DEALING, [GameState.PLAYING]],
    [GameState.PLAYING, [GameState.SCORING, GameState.FINISHED]],
    [GameState.SCORING, [GameState.DEALING, GameState.FINISHED]],
    [GameState.FINISHED, [GameState.WAITING_FOR_PLAYERS]]
  ]);
  
  private currentState: GameState = GameState.WAITING_FOR_PLAYERS;
  
  async transition(event: GameEvent, context: GameContext): Promise<boolean> {
    const nextState = this.getNextState(this.currentState, event);
    
    if (!nextState) {
      throw new Error(`Invalid transition: ${this.currentState} -> ${event}`);
    }
    
    // Execute pre-transition actions
    await this.executePreTransition(this.currentState, nextState, context);
    
    const previousState = this.currentState;
    this.currentState = nextState;
    
    // Execute post-transition actions
    await this.executePostTransition(previousState, nextState, context);
    
    return true;
  }
  
  private async executePreTransition(
    from: GameState, 
    to: GameState, 
    context: GameContext
  ): Promise<void> {
    switch (to) {
      case GameState.DEALING:
        await this.prepareDealingPhase(context);
        break;
      case GameState.PLAYING:
        await this.startGameplayPhase(context);  
        break;
      case GameState.SCORING:
        await this.calculateScores(context);
        break;
    }
  }
}
```

### 3.2 Room State Machine

```typescript
enum RoomState {
  CREATED = 'CREATED',
  WAITING = 'WAITING',
  PLAYING = 'PLAYING', 
  PAUSED = 'PAUSED',
  FINISHED = 'FINISHED'
}

class RoomStateMachine {
  private stateHandlers: Map<RoomState, RoomStateHandler> = new Map();
  
  constructor() {
    this.stateHandlers.set(RoomState.CREATED, new CreatedStateHandler());
    this.stateHandlers.set(RoomState.WAITING, new WaitingStateHandler());
    this.stateHandlers.set(RoomState.PLAYING, new PlayingStateHandler());
    // ... other handlers
  }
  
  async handleEvent(roomId: string, event: RoomEvent): Promise<void> {
    const room = await this.getRoomState(roomId);
    const handler = this.stateHandlers.get(room.state);
    
    if (!handler) {
      throw new Error(`No handler for state: ${room.state}`);
    }
    
    const nextState = await handler.handle(event, room);
    if (nextState !== room.state) {
      await this.transitionRoom(roomId, nextState);
    }
  }
}
```

---

## 4. Request-Response Implementation

### 4.1 Real-time Communication Service

```typescript
class RealtimeCommunicationService {
  private wsConnections: Map<string, WebSocket> = new Map();
  private requestTimeout = 5000; // 5 second timeout
  
  // Send request and wait for response
  async sendRequest<T>(
    playerId: string, 
    request: GameRequest
  ): Promise<GameResponse<T>> {
    const connection = this.wsConnections.get(playerId);
    if (!connection) {
      throw new Error('Player not connected');
    }
    
    return new Promise((resolve, reject) => {
      const requestId = generateRequestId();
      const timeoutId = setTimeout(() => {
        reject(new Error('Request timeout'));
      }, this.requestTimeout);
      
      // Set up response handler
      const responseHandler = (response: GameResponse<T>) => {
        if (response.requestId === requestId) {
          clearTimeout(timeoutId);
          resolve(response);
        }
      };
      
      this.addResponseHandler(requestId, responseHandler);
      
      // Send request
      connection.send(JSON.stringify({
        ...request,
        requestId,
        timestamp: Date.now()
      }));
    });
  }
  
  // Broadcast to room with acknowledgment
  async broadcastToRoom(
    roomId: string, 
    message: GameMessage,
    requireAcknowledgment = false
  ): Promise<void> {
    const room = await this.getRoomInfo(roomId);
    const promises = room.players.map(playerId => {
      if (requireAcknowledgment) {
        return this.sendRequest(playerId, {
          type: 'BROADCAST',
          data: message
        });
      } else {
        return this.sendMessage(playerId, message);
      }
    });
    
    await Promise.all(promises);
  }
}
```

### 4.2 API Response Standards

```typescript
// Standardized response format
interface APIResponse<T = any> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  timestamp: number;
  requestId: string;
}

// Game-specific responses
interface GameActionResponse {
  success: boolean;
  gameState: GameState;
  validActions: string[];
  timestamp: number;
}

// Taiwan Mahjong scoring response
interface ScoringResponse {
  totalScore: number;
  scoreBreakdown: {
    baseScore: number;
    multipliers: TaiwanMultiplier[];
    bonuses: ScoreBonus[];
  };
  winnerDetails: WinnerInfo;
  paymentMatrix: PaymentInfo[];
}
```

---

## 5. Event-Driven Implementation (Selective)

### 5.1 Event System for Non-Critical Features

```typescript
// Event system for non-critical operations
class SelectiveEventSystem {
  private eventBus: EventBus;
  private eventHandlers: Map<string, EventHandler[]> = new Map();
  
  // Only use for non-critical features
  private nonCriticalEvents = new Set([
    'CHAT_MESSAGE',
    'PLAYER_ANALYTICS',
    'ACHIEVEMENT_UNLOCK',
    'SYSTEM_NOTIFICATION',
    'SOCIAL_UPDATE'
  ]);
  
  async publishEvent(event: GameEvent): Promise<void> {
    // Critical events should not use this system
    if (!this.nonCriticalEvents.has(event.type)) {
      throw new Error(`Critical event ${event.type} should not use event system`);
    }
    
    // Asynchronous processing for non-critical events
    setImmediate(async () => {
      try {
        await this.processEventAsync(event);
      } catch (error) {
        // Log error but don't affect game flow
        this.logger.error('Non-critical event processing error:', error);
      }
    });
  }
  
  private async processEventAsync(event: GameEvent): Promise<void> {
    const handlers = this.eventHandlers.get(event.type) || [];
    
    // Process handlers concurrently for performance
    await Promise.allSettled(
      handlers.map(handler => handler.handle(event))
    );
  }
}

// Example: Chat system using events
class ChatEventHandler implements EventHandler {
  async handle(event: GameEvent): Promise<void> {
    if (event.type === 'CHAT_MESSAGE') {
      const chatEvent = event as ChatMessageEvent;
      
      // Store message asynchronously
      await this.storeMessage(chatEvent.message);
      
      // Filter and broadcast to room members
      const filteredMessage = await this.filterMessage(chatEvent.message);
      await this.broadcastToRoom(chatEvent.roomId, filteredMessage);
      
      // Update analytics asynchronously
      setImmediate(() => {
        this.updateChatAnalytics(chatEvent);
      });
    }
  }
}
```

---

## 6. Module Integration Guidelines

### 6.1 Module Boundary Definitions

```typescript
// Clear module interfaces
interface GameEngineModule {
  // Command pattern methods
  processCommand(command: GameCommand): Promise<CommandResult>;
  validateCommand(command: GameCommand): Promise<ValidationResult>;
  
  // State machine methods
  getGameState(roomId: string): Promise<GameState>;
  transitionGameState(roomId: string, event: GameEvent): Promise<void>;
}

interface RoomManagementModule {
  // Room lifecycle using state machines
  createRoom(config: RoomConfig): Promise<Room>;
  joinRoom(roomId: string, playerId: string): Promise<JoinResult>;
  
  // Real-time room updates
  updateRoomState(roomId: string, update: RoomUpdate): Promise<void>;
}

interface CommunicationModule {
  // Request-response for real-time features
  sendToPlayer<T>(playerId: string, message: GameMessage): Promise<T>;
  broadcastToRoom(roomId: string, message: GameMessage): Promise<void>;
  
  // Event-driven for non-critical
  publishEvent(event: GameEvent): void;
}
```

### 6.2 Cross-Module Communication Patterns

```typescript
// Service orchestrator that manages different communication patterns
class GameOrchestrator {
  constructor(
    private gameEngine: GameEngineModule,
    private roomManager: RoomManagementModule,
    private communication: CommunicationModule
  ) {}
  
  // Handle player action (Command + Request-Response)
  async handlePlayerAction(action: PlayerAction): Promise<ActionResult> {
    // 1. Create command (Command Pattern)
    const command = MahjongCommandFactory.createCommand(
      action.type, 
      action.params
    );
    
    // 2. Process command synchronously
    const result = await this.gameEngine.processCommand(command);
    
    if (result.success) {
      // 3. Send immediate response (Request-Response)
      await this.communication.sendToPlayer(action.playerId, {
        type: 'ACTION_RESULT',
        success: true,
        gameState: result.newState
      });
      
      // 4. Broadcast to other players
      await this.communication.broadcastToRoom(action.roomId, {
        type: 'GAME_UPDATE',
        state: result.newState,
        lastAction: command
      });
      
      // 5. Publish non-critical events asynchronously
      this.communication.publishEvent({
        type: 'PLAYER_ANALYTICS',
        playerId: action.playerId,
        actionType: action.type,
        timestamp: Date.now()
      });
    }
    
    return result;
  }
}
```

---

## 7. Performance Optimization Guidelines

### 7.1 Response Time Targets

| Operation Type | Target (ms) | Maximum (ms) | Pattern Used |
|----------------|-------------|--------------|--------------|
| Game action processing | <50 | <70 | Command Pattern |
| State transition | <30 | <50 | State Machine |
| Real-time sync | <40 | <60 | Request-Response |
| Score calculation | <100 | <150 | Request-Response |
| Room operations | <200 | <300 | State Machine + Request-Response |

### 7.2 Optimization Strategies

```typescript
// Command processing optimization
class OptimizedCommandProcessor {
  private commandPool = new ObjectPool<GameCommand>();
  private stateCache = new LRUCache<string, GameState>(1000);
  
  async processCommand(command: GameCommand): Promise<CommandResult> {
    // 1. Use object pooling to reduce GC pressure
    const pooledCommand = this.commandPool.acquire();
    
    // 2. Check state cache first
    let gameState = this.stateCache.get(command.roomId);
    if (!gameState) {
      gameState = await this.loadGameState(command.roomId);
      this.stateCache.set(command.roomId, gameState);
    }
    
    // 3. Process command with optimized execution
    const result = await this.executeOptimized(command, gameState);
    
    // 4. Update cache immediately
    if (result.success) {
      this.stateCache.set(command.roomId, result.newState);
    }
    
    this.commandPool.release(pooledCommand);
    return result;
  }
  
  private async executeOptimized(
    command: GameCommand, 
    state: GameState
  ): Promise<CommandResult> {
    // Use optimized execution paths for different command types
    switch (command.type) {
      case 'DRAW_TILE':
        return this.optimizedDrawTile(command as DrawTileCommand, state);
      case 'DISCARD_TILE':
        return this.optimizedDiscardTile(command as DiscardTileCommand, state);
      default:
        return this.standardExecution(command, state);
    }
  }
}
```

---

## 8. Testing Guidelines

### 8.1 Command Pattern Testing

```typescript
describe('Command Pattern Tests', () => {
  it('should validate Taiwan Mahjong commands correctly', async () => {
    const command = new DrawTileCommand('player1', 'room1');
    const gameState = createTestGameState();
    
    const validation = command.validate(gameState);
    expect(validation.isValid).toBe(true);
  });
  
  it('should execute commands deterministically', async () => {
    const command = new DrawTileCommand('player1', 'room1');
    const gameState = createTestGameState();
    
    const result1 = command.execute(gameState);
    const result2 = command.execute(gameState);
    
    // Results should be identical for same input
    expect(result1).toEqual(result2);
  });
  
  it('should enable game replay through command history', async () => {
    const commands = [
      new DrawTileCommand('player1', 'room1'),
      new DiscardTileCommand('player1', 'room1', 'tile1'),
      // ... more commands
    ];
    
    const finalState = await replayCommands(commands);
    expect(finalState.currentPlayer).toBe('player2');
  });
});
```

### 8.2 State Machine Testing

```typescript
describe('State Machine Tests', () => {
  it('should prevent invalid state transitions', async () => {
    const stateMachine = new GameStateMachine();
    
    // Should not allow direct transition from WAITING to PLAYING
    await expect(
      stateMachine.transition(GameEvent.ROUND_COMPLETE, context)
    ).rejects.toThrow('Invalid transition');
  });
  
  it('should handle Taiwan Mahjong game flow correctly', async () => {
    const stateMachine = new GameStateMachine();
    
    await stateMachine.transition(GameEvent.PLAYERS_READY, context);
    expect(stateMachine.currentState).toBe(GameState.DEALING);
    
    await stateMachine.transition(GameEvent.DEALING_COMPLETE, context);
    expect(stateMachine.currentState).toBe(GameState.PLAYING);
  });
});
```

### 8.3 Integration Testing

```typescript
describe('Hybrid Architecture Integration', () => {
  it('should coordinate all patterns correctly', async () => {
    // Test end-to-end flow using all patterns
    const action = {
      type: 'DISCARD_TILE',
      playerId: 'player1',
      roomId: 'room1',
      tileId: 'tile1'
    };
    
    // Should use Command Pattern + Request-Response + State Machine
    const result = await gameOrchestrator.handlePlayerAction(action);
    
    expect(result.success).toBe(true);
    expect(result.responseTime).toBeLessThan(70); // Performance requirement
  });
});
```

---

## 9. Error Handling and Recovery

### 9.1 Command Error Handling

```typescript
class CommandErrorHandler {
  async handleCommandError(
    command: GameCommand, 
    error: CommandError
  ): Promise<ErrorResponse> {
    switch (error.type) {
      case 'VALIDATION_ERROR':
        // Invalid move attempt
        return {
          success: false,
          error: {
            code: 'INVALID_MOVE',
            message: '無效的移動',
            details: error.validationDetails
          },
          canRetry: true
        };
        
      case 'STATE_CONFLICT':
        // Game state changed during command processing
        await this.refreshGameState(command.roomId);
        return {
          success: false,
          error: {
            code: 'STATE_CHANGED',
            message: '遊戲狀態已更新，請重試',
            details: { latestState: await this.getGameState(command.roomId) }
          },
          canRetry: true
        };
        
      case 'NETWORK_ERROR':
        // Network connectivity issues
        return {
          success: false,
          error: {
            code: 'NETWORK_ERROR',
            message: '網路連線問題',
            details: error.networkDetails
          },
          canRetry: true,
          retryAfter: 3000
        };
        
      default:
        // Unexpected errors
        this.logger.error('Unexpected command error:', error);
        return {
          success: false,
          error: {
            code: 'INTERNAL_ERROR', 
            message: '系統錯誤，請稍後再試'
          },
          canRetry: false
        };
    }
  }
}
```

### 9.2 State Recovery Mechanisms

```typescript
class StateRecoveryService {
  async recoverGameState(roomId: string): Promise<GameState> {
    try {
      // 1. Try to recover from command history
      const commands = await this.getCommandHistory(roomId);
      const recoveredState = await this.replayCommands(commands);
      
      // 2. Validate recovered state
      const validation = await this.validateGameState(recoveredState);
      if (validation.isValid) {
        return recoveredState;
      }
      
      // 3. Fallback to last known good state
      const lastGoodState = await this.getLastValidState(roomId);
      if (lastGoodState) {
        this.logger.warn(`Recovered to last good state for room ${roomId}`);
        return lastGoodState;
      }
      
      // 4. Last resort: reset game
      this.logger.error(`Could not recover state for room ${roomId}, resetting`);
      return this.createFreshGameState(roomId);
      
    } catch (error) {
      this.logger.error('State recovery failed:', error);
      throw new Error('Unable to recover game state');
    }
  }
}
```

---

## 10. Monitoring and Observability

### 10.1 Performance Metrics

```typescript
class HybridArchitectureMetrics {
  private metrics = {
    commandProcessingTime: new Histogram('command_processing_time_ms'),
    stateTransitionTime: new Histogram('state_transition_time_ms'),
    requestResponseTime: new Histogram('request_response_time_ms'),
    eventProcessingTime: new Histogram('event_processing_time_ms'),
    
    commandThroughput: new Counter('commands_processed_total'),
    stateTransitions: new Counter('state_transitions_total'),
    errorRate: new Counter('errors_total'),
    
    activeRooms: new Gauge('active_rooms_count'),
    concurrentUsers: new Gauge('concurrent_users_count')
  };
  
  recordCommandProcessing(command: GameCommand, duration: number): void {
    this.metrics.commandProcessingTime
      .labels({ commandType: command.type })
      .observe(duration);
      
    this.metrics.commandThroughput
      .labels({ commandType: command.type })
      .inc();
  }
  
  recordStateTransition(
    from: GameState, 
    to: GameState, 
    duration: number
  ): void {
    this.metrics.stateTransitionTime
      .labels({ from, to })
      .observe(duration);
      
    this.metrics.stateTransitions
      .labels({ from, to })
      .inc();
  }
}
```

### 10.2 Health Checks

```typescript
class ArchitectureHealthCheck {
  async checkSystemHealth(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkCommandProcessor(),
      this.checkStateMachines(),
      this.checkCommunication(),
      this.checkEventSystem()
    ]);
    
    const failed = checks.filter(check => 
      check.status === 'rejected' || 
      (check.status === 'fulfilled' && !check.value.healthy)
    );
    
    return {
      healthy: failed.length === 0,
      components: {
        commandProcessor: checks[0],
        stateMachines: checks[1], 
        communication: checks[2],
        eventSystem: checks[3]
      },
      timestamp: Date.now()
    };
  }
  
  private async checkCommandProcessor(): Promise<ComponentHealth> {
    try {
      const testCommand = new DrawTileCommand('test', 'test');
      const start = Date.now();
      
      // Test command validation
      await testCommand.validate(createTestGameState());
      
      const duration = Date.now() - start;
      return {
        healthy: duration < 50, // Target <50ms
        responseTime: duration,
        details: { targetTime: 50 }
      };
    } catch (error) {
      return {
        healthy: false,
        error: error.message
      };
    }
  }
}
```

---

## 11. Team Coding Standards

### 11.1 Code Organization

```
src/
├── commands/                    # Command Pattern implementations
│   ├── base/
│   │   ├── GameCommand.ts
│   │   └── CommandHandler.ts
│   ├── mahjong/
│   │   ├── DrawTileCommand.ts
│   │   ├── DiscardTileCommand.ts
│   │   └── ...
│   └── index.ts
├── state-machines/              # State Machine implementations  
│   ├── GameStateMachine.ts
│   ├── RoomStateMachine.ts
│   └── states/
├── communication/               # Request-Response patterns
│   ├── RealtimeService.ts
│   ├── WebSocketManager.ts
│   └── protocols/
├── events/                      # Selective Event-Driven
│   ├── EventBus.ts
│   ├── handlers/
│   └── types/
├── modules/                     # Module boundaries
│   ├── game-engine/
│   ├── room-management/
│   ├── user-management/
│   └── communication/
└── shared/                      # Shared utilities
    ├── types/
    ├── validators/
    └── utils/
```

### 11.2 Naming Conventions

```typescript
// Commands: [Action]Command
class DrawTileCommand implements GameCommand { }
class DiscardTileCommand implements GameCommand { }

// State Machines: [Entity]StateMachine  
class GameStateMachine { }
class RoomStateMachine { }

// Request-Response Services: [Domain]Service
class RealtimeService { }
class GameActionService { }

// Event Handlers: [Event]Handler
class ChatMessageHandler { }
class AnalyticsHandler { }

// Module Interfaces: [Module]Module
interface GameEngineModule { }
interface RoomManagementModule { }
```

### 11.3 Documentation Standards

```typescript
/**
 * Taiwan Mahjong Draw Tile Command
 * 
 * Implements the 摸牌 action according to traditional Taiwan Mahjong rules.
 * This command is irreversible once executed (no undo functionality).
 * 
 * @pattern Command Pattern
 * @domain Taiwan Mahjong Game Actions
 * @performance Target: <50ms validation + execution
 */
class DrawTileCommand implements GameCommand {
  /**
   * Validates if the player can draw a tile
   * 
   * @param gameState Current game state
   * @returns Validation result with error details if invalid
   * 
   * @complexity O(1) - constant time validation
   * @side-effects None - pure validation function
   */
  validate(gameState: GameState): ValidationResult {
    // Implementation...
  }
}
```

---

## 12. Migration and Rollout Strategy

### 12.1 Phased Implementation

**Phase 1: Foundation (Month 1)**
- Implement base Command Pattern infrastructure
- Create core State Machine framework
- Set up Request-Response communication layer
- Team training and code review processes

**Phase 2: Game Logic (Months 2-3)**
- Implement all Taiwan Mahjong commands
- Build game flow state machines  
- Integrate command processing with real-time communication
- Performance optimization for <70ms targets

**Phase 3: Full Integration (Months 4-5)**
- Complete module integration
- Add selective event-driven features
- End-to-end testing and validation
- Load testing for 10,000+ concurrent users

**Phase 4: Production Deployment (Months 6-7)**
- Production rollout with monitoring
- Performance tuning and optimization
- Final testing and bug fixes

### 12.2 Risk Mitigation

1. **Gradual Rollout**: Implement one pattern at a time
2. **Fallback Plans**: Maintain ability to revert to simpler patterns
3. **Comprehensive Testing**: Unit, integration, and performance tests
4. **Team Training**: Dedicated training sessions for each pattern
5. **Code Reviews**: Mandatory reviews for all pattern implementations
6. **Performance Monitoring**: Real-time metrics and alerting

---

## 13. Success Criteria

### 13.1 Technical Success Metrics

- ✅ **Response Time**: <70ms average for game actions
- ✅ **System Uptime**: >99.9% availability  
- ✅ **Scalability**: Support 10,000+ concurrent users
- ✅ **State Consistency**: Zero invalid game state transitions
- ✅ **Command Throughput**: >1000 commands/second per instance

### 13.2 Development Success Metrics

- ✅ **Timeline**: Complete implementation in 7 months
- ✅ **Cost Reduction**: 12-15% development cost savings
- ✅ **Team Productivity**: +15% velocity improvement
- ✅ **Bug Reduction**: 30%+ fewer integration bugs
- ✅ **Code Quality**: 90%+ code coverage, clean architecture

### 13.3 Business Success Metrics

- ✅ **Time to Market**: 1 month earlier launch
- ✅ **User Experience**: <100ms response times consistently
- ✅ **System Reliability**: 99.9%+ uptime in production
- ✅ **Maintenance Costs**: 20%+ reduction in operational overhead

---

## Contact and Support

**Architecture Team Lead**: [Contact Information]  
**Technical Documentation**: `/docs/architecture/`  
**Code Examples**: `/examples/hybrid-patterns/`  
**Training Materials**: `/training/hybrid-architecture/`

**Questions or Issues?**
- Create issue in project repository with tag `hybrid-architecture`
- Join #architecture-discussion Slack channel
- Schedule code review session for pattern implementation

---

**Document Maintenance**
- **Next Review**: 2025-09-07
- **Update Frequency**: Monthly during implementation phase
- **Version Control**: All changes tracked in git with detailed commit messages