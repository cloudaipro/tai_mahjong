# Taiwan Mahjong Game: Modular Monolith to Microservices Migration Strategy

## Overview

This document outlines the detailed migration strategy for transitioning the Taiwan Mahjong online game from a modular monolith architecture to selective microservices based on scaling demands.

## 1. Initial Modular Monolith Architecture Design

### 1.1 Core Module Structure

For the Taiwan Mahjong game, design the initial monolith with these distinct modules:

```
taiwan-mahjong-monolith/
├── modules/
│   ├── user-management/          # User auth, profiles, friends
│   ├── game-engine/             # Core mahjong logic, rules engine
│   ├── room-management/         # Room creation, joining, settings
│   ├── real-time-communication/ # WebSocket handling, messaging
│   ├── scoring-system/          # Taiwan scoring (台數) calculations
│   ├── ai-player/               # AI opponents for practice mode
│   ├── tournament-system/       # Competitions, leaderboards
│   ├── payment-billing/         # In-app purchases, VIP features
│   ├── analytics-reporting/     # Game statistics, user behavior
│   └── notification-system/     # Push notifications, alerts
├── shared/
│   ├── database/               # Database access layer
│   ├── cache/                  # Redis operations
│   ├── events/                 # Internal event bus
│   └── utils/                  # Common utilities
└── api/
    ├── rest/                   # REST API endpoints
    └── websocket/              # WebSocket handlers
```

### 1.2 Module Implementation Guidelines

Each module should follow these principles:

**1. Clear Interfaces:**
```typescript
// Example: Game Engine Module Interface
export interface IGameEngine {
  startGame(roomId: string, players: Player[]): Promise<GameState>;
  processPlayerAction(action: PlayerAction): Promise<GameStateUpdate>;
  validateMove(move: MahjongMove): ValidationResult;
  calculateScore(hand: MahjongHand): ScoreResult;
}
```

**2. Database Schema Separation:**
```sql
-- Each module owns its tables
-- User Management
users, user_profiles, friends, user_settings

-- Game Engine  
games, game_states, game_moves, game_history

-- Room Management
rooms, room_settings, room_participants

-- Scoring System
score_calculations, score_history, ranking_data
```

**3. Hybrid Communication Patterns:**
```typescript
// Hybrid architecture communication system
interface GameEvent {
  type: 'GAME_STARTED' | 'PLAYER_MOVED' | 'GAME_ENDED';
  payload: any;
  timestamp: Date;
  roomId: string;
}

class HybridCommunicationSystem {
  // Command Pattern for game actions
  processCommand(command: GameCommand): Promise<CommandResult>;
  
  // Request-Response for real-time operations
  sendRequest<T>(request: GameRequest): Promise<T>;
  
  // State Machines for game flow management
  transitionState(entity: string, event: StateEvent): Promise<void>;
  
  // Selective Event-Driven for non-critical operations
  publishEvent(event: GameEvent): void;
  subscribe(eventType: string, handler: Function): void;
}
```

## 2. Step-by-Step Migration Process

### Phase 1: Establish Migration Infrastructure (Weeks 1-2)

**1. Set up Service Discovery:**
```typescript
// Service registry for gradual migration
interface ServiceRegistry {
  register(serviceName: string, endpoint: string): void;
  discover(serviceName: string): string[];
  isServiceExternal(serviceName: string): boolean;
}
```

**2. Implement API Gateway:**
```yaml
# API Gateway routing configuration
routes:
  /api/users/**:
    target: ${USER_SERVICE_URL:http://localhost:3000/modules/user-management}
  /api/games/**:
    target: ${GAME_SERVICE_URL:http://localhost:3000/modules/game-engine}
  /ws/rooms/**:
    target: ${ROOM_SERVICE_URL:ws://localhost:3000/modules/room-management}
```

**3. Database Migration Strategy:**
```sql
-- Create separate schemas for each module
CREATE SCHEMA user_management;
CREATE SCHEMA game_engine;
CREATE SCHEMA room_management;

-- Implement cross-schema foreign keys initially
ALTER TABLE game_engine.games 
ADD CONSTRAINT fk_creator 
FOREIGN KEY (creator_id) REFERENCES user_management.users(id);
```

### Phase 2: Extract High-Traffic Modules (Weeks 3-8)

**Priority Order Based on Hybrid Architecture and Taiwan Mahjong Characteristics:**

1. **Real-time Communication Service** (Week 3-4) - Request-Response pattern extraction
2. **Game Engine Service** (Week 5-6) - Command Pattern + State Machine extraction
3. **Room Management Service** (Week 7-8) - State Machine pattern extraction

#### 2.1 Real-time Communication Service Extraction

**Why First:** Handles WebSocket connections, most resource-intensive, needs horizontal scaling. Uses Request-Response pattern for low-latency communication.

```typescript
// Extract WebSocket handling to separate service with Request-Response pattern
class RealtimeCommunicationService {
  private wsServer: WebSocketServer;
  private roomConnections: Map<string, Set<WebSocket>>;
  private requestHandlers: Map<string, RequestHandler>;
  
  // Request-Response pattern for immediate player actions
  async handlePlayerRequest<T>(request: GameRequest): Promise<GameResponse<T>> {
    const startTime = Date.now();
    
    try {
      // Process request synchronously for low latency
      const handler = this.requestHandlers.get(request.type);
      if (!handler) {
        throw new Error(`No handler for request type: ${request.type}`);
      }
      
      const result = await handler.handle(request);
      const responseTime = Date.now() - startTime;
      
      // Target <70ms response time
      if (responseTime > 70) {
        this.logger.warn(`Slow response: ${responseTime}ms for ${request.type}`);
      }
      
      return {
        success: true,
        data: result,
        responseTime,
        requestId: request.id
      };
    } catch (error) {
      return {
        success: false,
        error: error.message,
        responseTime: Date.now() - startTime,
        requestId: request.id
      };
    }
  }
  
  async broadcastToRoom(roomId: string, message: any): Promise<void> {
    const connections = this.roomConnections.get(roomId);
    if (connections) {
      const payload = JSON.stringify(message);
      connections.forEach(ws => ws.send(payload));
    }
  }
  
  async handlePlayerMove(roomId: string, playerId: string, move: any): Promise<void> {
    // Use Request-Response for immediate validation
    const validationResponse = await this.handlePlayerRequest({
      type: 'VALIDATE_MOVE',
      payload: { roomId, playerId, move },
      id: generateRequestId(),
      timestamp: Date.now()
    });
    
    if (validationResponse.success) {
      await this.broadcastToRoom(roomId, {
        type: 'PLAYER_MOVED',
        playerId,
        move,
        timestamp: new Date(),
        responseTime: validationResponse.responseTime
      });
    }
  }
}
```

**Migration Steps:**
```bash
# 1. Deploy new real-time service
docker deploy realtime-communication-service

# 2. Update API Gateway to route WebSocket traffic
# 3. Migrate existing WebSocket connections gradually
# 4. Remove WebSocket handling from monolith
```

#### 2.2 Game Engine Service Extraction

**Why Second:** Core game logic, complex Taiwan Mahjong rules, needs dedicated resources. Uses Command Pattern + State Machine for deterministic execution.

```typescript
// Hybrid game engine service with Command Pattern and State Machines
class GameEngineService {
  private commandProcessor: CommandProcessor;
  private gameStateMachine: GameStateMachine;
  private commandHistory: Map<string, GameCommand[]>;
  
  async processPlayerAction(gameId: string, action: PlayerAction): Promise<GameStateUpdate> {
    // Create command from action (Command Pattern)
    const command = this.createCommand(action);
    
    // Load current game state
    const gameState = await this.gameRepository.getGameState(gameId);
    
    // Validate command
    const validation = command.validate(gameState);
    if (!validation.isValid) {
      throw new Error(`Invalid move: ${validation.errorCode}`);
    }
    
    // Execute command deterministically
    const updatedState = command.execute(gameState);
    
    // Update state machine if needed
    if (this.shouldTransitionState(command, updatedState)) {
      const stateEvent = this.mapCommandToStateEvent(command);
      await this.gameStateMachine.transition(stateEvent, {
        gameId,
        gameState: updatedState
      });
    }
    
    // Store command in history for replay capability
    if (!this.commandHistory.has(gameId)) {
      this.commandHistory.set(gameId, []);
    }
    this.commandHistory.get(gameId)!.push(command);
    
    // Calculate scoring if game ended (Request-Response for immediate results)
    if (updatedState.gameEnded) {
      updatedState.finalScores = await this.calculateScoresSync(
        updatedState.playerHands
      );
    }
    
    // Persist updated state
    await this.gameRepository.saveGameState(gameId, updatedState);
    
    return {
      gameState: updatedState,
      command: command,
      stateTransition: this.gameStateMachine.getCurrentState(),
      timestamp: Date.now()
    };
  }
  
  private createCommand(action: PlayerAction): GameCommand {
    switch (action.type) {
      case 'DRAW_TILE':
        return new DrawTileCommand(action.playerId, action.roomId);
      case 'DISCARD_TILE':
        return new DiscardTileCommand(action.playerId, action.roomId, action.tileId);
      case 'PONG':
        return new PongCommand(action.playerId, action.roomId, action.targetTile);
      // ... other Taiwan Mahjong commands
      default:
        throw new Error(`Unknown action type: ${action.type}`);
    }
  }
  
  // Game replay capability using command history
  async replayGame(gameId: string, upToSequence?: number): Promise<GameState> {
    const commands = this.commandHistory.get(gameId) || [];
    const replayCommands = upToSequence 
      ? commands.slice(0, upToSequence) 
      : commands;
    
    let gameState = await this.getInitialGameState(gameId);
    
    for (const command of replayCommands) {
      gameState = command.execute(gameState);
    }
    
    return gameState;
  }
}
```

#### 2.3 Room Management Service Extraction

**Why Third:** Manages game sessions, needs to scale with concurrent rooms. Uses State Machine pattern for room lifecycle management.

```typescript
class RoomManagementService {
  private roomStateMachine: RoomStateMachine;
  private eventBus: SelectiveEventSystem;
  
  async createRoom(settings: RoomSettings, creatorId: string): Promise<Room> {
    const room = new Room({
      id: generateId(),
      settings,
      creatorId,
      status: RoomState.CREATED // State Machine initial state
    });
    
    // Initialize state machine for this room
    await this.roomStateMachine.initialize(room.id, RoomState.CREATED);
    
    await this.roomRepository.save(room);
    
    // Transition to WAITING state (State Machine)
    await this.roomStateMachine.transition(room.id, RoomEvent.ROOM_SETUP_COMPLETE);
    
    // Use selective event-driven for non-critical notifications
    this.eventBus.publishEvent({
      type: 'ROOM_CREATED',
      payload: { roomId: room.id, settings },
      timestamp: Date.now(),
      roomId: room.id
    });
    
    return room;
  }
  
  async joinRoom(roomId: string, playerId: string): Promise<JoinRoomResult> {
    const currentState = await this.roomStateMachine.getCurrentState(roomId);
    
    // Validate state transition
    if (currentState !== RoomState.WAITING_FOR_PLAYERS) {
      throw new Error(`Cannot join room in state: ${currentState}`);
    }
    
    const room = await this.roomRepository.findById(roomId);
    if (!room) {
      throw new Error('Room not found');
    }
    
    // Add player to room
    room.addPlayer(playerId);
    await this.roomRepository.save(room);
    
    // Check if room is full and transition state
    if (room.isFull()) {
      await this.roomStateMachine.transition(roomId, RoomEvent.ROOM_FULL);
      
      // Notify game engine to start game (Request-Response for immediate response)
      const gameStartResult = await this.gameEngineService.startGame({
        roomId,
        players: room.players,
        settings: room.settings
      });
      
      return {
        success: true,
        room,
        gameStarted: true,
        gameId: gameStartResult.gameId
      };
    }
    
    return {
      success: true,
      room,
      gameStarted: false
    };
  }
  
  async handleRoomEvent(roomId: string, event: RoomEvent): Promise<void> {
    try {
      await this.roomStateMachine.transition(roomId, event);
      
      // Handle state-specific actions
      const newState = await this.roomStateMachine.getCurrentState(roomId);
      await this.handleStateChange(roomId, newState);
      
    } catch (error) {
      this.logger.error(`Room state transition failed: ${roomId}`, error);
      throw error;
    }
  }
  
  private async handleStateChange(roomId: string, newState: RoomState): Promise<void> {
    switch (newState) {
      case RoomState.PLAYING:
        // Room entered playing state
        this.eventBus.publishEvent({
          type: 'ROOM_GAME_STARTED',
          payload: { roomId },
          timestamp: Date.now(),
          roomId
        });
        break;
        
      case RoomState.FINISHED:
        // Game finished, calculate final results
        await this.finalizeRoom(roomId);
        break;
    }
  }
}
```

## 3. Identifying Extraction Candidates

### 3.1 Metrics-Based Decision Framework

**Real-time Monitoring:**
```typescript
interface ServiceMetrics {
  requestsPerSecond: number;
  averageResponseTime: number;
  memoryUsage: number;
  cpuUsage: number;
  errorRate: number;
  activeConnections: number; // For WebSocket services
}

class MigrationDecisionEngine {
  shouldExtractService(moduleName: string, metrics: ServiceMetrics): boolean {
    const thresholds = {
      requestsPerSecond: 1000,     // High traffic
      averageResponseTime: 500,    // Slow responses
      memoryUsage: 0.7,           // 70% memory usage
      cpuUsage: 0.6,              // 60% CPU usage
      errorRate: 0.05             // 5% error rate
    };
    
    return metrics.requestsPerSecond > thresholds.requestsPerSecond ||
           metrics.averageResponseTime > thresholds.averageResponseTime ||
           metrics.memoryUsage > thresholds.memoryUsage ||
           metrics.cpuUsage > thresholds.cpuUsage ||
           metrics.errorRate > thresholds.errorRate;
  }
}
```

### 3.2 Taiwan Mahjong Specific Indicators

**Extract When These Conditions Are Met:**

1. **Real-time Communication:** >1000 concurrent WebSocket connections
2. **Game Engine:** >100 concurrent games running
3. **AI Player:** AI computation affecting main game performance
4. **Tournament System:** Tournament events causing traffic spikes
5. **Scoring System:** Complex Taiwan scoring calculations causing delays

## 4. Technical Implementation Details

### 4.1 Service Communication Patterns

```typescript
// Synchronous communication for critical game operations
class GameEngineClient {
  async validateMove(move: MahjongMove): Promise<ValidationResult> {
    try {
      const response = await this.httpClient.post('/api/validate-move', {
        move,
        timeout: 100 // Fast validation required
      });
      return response.data;
    } catch (error) {
      // Fallback to local validation for reliability
      return this.localValidator.validate(move);
    }
  }
}

// Asynchronous communication for non-critical operations
class NotificationService {
  async sendGameResult(gameResult: GameResult): Promise<void> {
    await this.messageQueue.publish('game-results', gameResult, {
      retry: 3,
      delay: 1000
    });
  }
}
```

### 4.2 Database Decomposition Strategy

**Phase 1: Schema Separation**
```sql
-- Separate databases while maintaining referential integrity
CREATE DATABASE taiwan_mahjong_users;
CREATE DATABASE taiwan_mahjong_games;
CREATE DATABASE taiwan_mahjong_rooms;

-- Use distributed foreign keys
-- games.creator_id references users.id (maintained by application)
```

**Phase 2: Event Sourcing for Game State**
```typescript
// Event sourcing for game state management
interface GameEvent {
  id: string;
  gameId: string;
  type: 'GAME_STARTED' | 'TILE_DRAWN' | 'TILE_DISCARDED' | 'HAND_DECLARED';
  payload: any;
  timestamp: Date;
  version: number;
}

class GameEventStore {
  async appendEvent(event: GameEvent): Promise<void> {
    await this.eventRepository.save(event);
    await this.eventBus.publish('game-event', event);
  }
  
  async getGameState(gameId: string): Promise<GameState> {
    const events = await this.eventRepository.getEventsByGameId(gameId);
    return this.gameStateProjector.project(events);
  }
}
```

## 5. Data Consistency During Migration

### 5.1 Distributed Transaction Pattern

```typescript
// Saga pattern for distributed transactions
class GameStartSaga {
  async startGame(gameRequest: StartGameRequest): Promise<void> {
    const sagaId = generateId();
    
    try {
      // Step 1: Reserve room
      await this.roomService.reserveRoom(gameRequest.roomId, sagaId);
      
      // Step 2: Initialize game state
      const gameId = await this.gameService.createGame(gameRequest, sagaId);
      
      // Step 3: Notify players
      await this.notificationService.notifyPlayers(
        gameRequest.playerIds, 
        { gameId, roomId: gameRequest.roomId }
      );
      
      // Complete saga
      await this.sagaRepository.completeSaga(sagaId);
      
    } catch (error) {
      // Compensating transactions
      await this.compensate(sagaId, error);
    }
  }
}
```

### 5.2 Event-Driven Consistency

```typescript
// Eventually consistent updates via events
class ScoreUpdateHandler {
  async handleGameEnded(event: GameEndedEvent): Promise<void> {
    // Update player statistics
    await this.userService.updatePlayerStats(event.playerScores);
    
    // Update leaderboards
    await this.tournamentService.updateRankings(event.playerScores);
    
    // Send notifications
    await this.notificationService.sendGameResults(event);
  }
}
```

## 6. Real-time Gaming Considerations

### 6.1 WebSocket Connection Management

```typescript
class ConnectionManager {
  private connections: Map<string, WebSocket> = new Map();
  private roomMemberships: Map<string, Set<string>> = new Map();
  
  async handleReconnection(userId: string, newSocket: WebSocket): Promise<void> {
    // Close old connection
    const oldSocket = this.connections.get(userId);
    if (oldSocket) {
      oldSocket.close();
    }
    
    // Register new connection
    this.connections.set(userId, newSocket);
    
    // Restore room memberships
    const userRooms = await this.getUserActiveRooms(userId);
    for (const roomId of userRooms) {
      await this.joinRoom(userId, roomId);
      
      // Send current game state
      const gameState = await this.gameService.getCurrentState(roomId);
      newSocket.send(JSON.stringify({
        type: 'GAME_STATE_SYNC',
        payload: gameState
      }));
    }
  }
}
```

### 6.2 State Synchronization

```typescript
class GameStateSynchronizer {
  async synchronizeGameState(roomId: string): Promise<void> {
    const gameState = await this.gameService.getLatestState(roomId);
    const stateHash = this.calculateStateHash(gameState);
    
    // Send state hash to all clients
    await this.realtimeService.broadcastToRoom(roomId, {
      type: 'STATE_HASH_CHECK',
      hash: stateHash
    });
    
    // Clients report their state hash
    // Resync clients with different hash
  }
  
  private calculateStateHash(gameState: GameState): string {
    return crypto.createHash('md5')
      .update(JSON.stringify(gameState))
      .digest('hex');
  }
}
```

## 7. Risk Mitigation Strategies

### 7.1 Gradual Migration with Feature Flags

```typescript
// Feature flag system for gradual migration
class MigrationFeatureFlags {
  private flags: Map<string, boolean> = new Map([
    ['use-external-realtime-service', false],
    ['use-external-game-engine', false],
    ['use-external-room-service', false]
  ]);
  
  async getUserService(): Promise<UserService> {
    if (await this.isEnabled('use-external-user-service')) {
      return this.externalUserService;
    }
    return this.monolithUserService;
  }
}
```

### 7.2 Circuit Breaker Pattern

```typescript
class CircuitBreaker {
  private failureCount: number = 0;
  private lastFailureTime: Date | null = null;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  
  async execute<T>(operation: () => Promise<T>, fallback: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (this.shouldAttemptReset()) {
        this.state = 'HALF_OPEN';
      } else {
        return await fallback();
      }
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      return await fallback();
    }
  }
}
```

### 7.3 Health Checks and Monitoring

```typescript
class ServiceHealthChecker {
  async checkGameEngineHealth(): Promise<HealthStatus> {
    try {
      const startTime = Date.now();
      await this.gameEngineService.ping();
      const responseTime = Date.now() - startTime;
      
      return {
        status: 'healthy',
        responseTime,
        timestamp: new Date(),
        checks: {
          database: await this.checkDatabaseConnection(),
          redis: await this.checkRedisConnection(),
          memory: this.getMemoryUsage()
        }
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        error: error.message,
        timestamp: new Date()
      };
    }
  }
}
```

## 8. Migration Decision Triggers

### 8.1 Performance Metrics Thresholds

Extract services when hitting these thresholds:

| Module | Trigger Condition |
|--------|------------------|
| Real-time Communication | >1000 concurrent WebSocket connections |
| Game Engine | >100 concurrent games running |
| AI Player | AI computation affecting main game performance |
| Tournament System | Tournament events causing traffic spikes |
| Scoring System | Complex Taiwan scoring calculations causing delays >500ms |

### 8.2 Resource Usage Indicators

```typescript
interface ExtractionThresholds {
  requestsPerSecond: 1000;     // High traffic
  averageResponseTime: 500;    // Slow responses (ms)
  memoryUsage: 0.7;           // 70% memory usage
  cpuUsage: 0.6;              // 60% CPU usage
  errorRate: 0.05;            // 5% error rate
}
```

## 9. Implementation Timeline

### Months 1-2: Foundation
- Set up modular monolith structure
- Implement event bus and service interfaces
- Create monitoring and metrics collection
- Set up CI/CD for gradual deployment

### Months 3-4: First Extraction
- Extract Real-time Communication service
- Implement WebSocket connection management
- Set up service discovery and API gateway
- Monitor performance and stability

### Months 5-6: Core Game Logic
- Extract Game Engine service
- Implement distributed state management
- Set up event sourcing for game states
- Performance testing with load simulation

### Months 7-8: Scaling Preparation
- Extract additional services based on metrics
- Implement advanced monitoring and alerting
- Set up auto-scaling for high-traffic services
- Disaster recovery procedures

## 10. Taiwan Mahjong Specific Considerations

### 10.1 Game State Consistency
- Critical for Taiwan Mahjong rule enforcement
- Use event sourcing to maintain audit trail
- Implement state validation at service boundaries

### 10.2 Real-time Performance
- Taiwan Mahjong requires low-latency moves
- Keep game engine and real-time communication co-located initially
- Extract only when scaling absolutely requires it

### 10.3 Cultural Authenticity
- Preserve Taiwan Mahjong terminology in all services
- Maintain traditional scoring (台數) calculation accuracy
- Ensure consistent rule interpretation across distributed services

## 11. Success Metrics

### 11.1 Technical Metrics
- Service response time <100ms for game actions
- WebSocket message latency <50ms
- 99.9% uptime for game services
- Zero data inconsistency in game states

### 11.2 Business Metrics
- Faster feature development velocity
- Reduced operational complexity
- Lower infrastructure costs initially
- Smooth user experience during scaling

## Conclusion

This migration strategy provides a systematic approach to evolving the Taiwan Mahjong game from a modular monolith to microservices. The key principle is to migrate based on actual performance metrics and business needs rather than following a predetermined schedule.

Start with the modular monolith to accelerate initial development, then extract services only when scaling demands it. This approach ensures you maintain the real-time gaming experience and cultural authenticity that are critical for Taiwan Mahjong while providing the flexibility to scale as your player base grows.