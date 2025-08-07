# ADR-002: Real-time Multiplayer Architecture for Taiwan Mahjong

## Status
Proposed

## Context
Taiwan Mahjong is fundamentally a 4-player real-time game where every action (drawing, discarding, claiming tiles) must be synchronized across all players with minimal latency. The game has complex state transitions and requires:

- Sub-100ms response time for game actions
- Reliable state synchronization across 4 players
- Handling network disruptions and reconnections
- Prevention of cheating through client-side manipulation
- Support for 1,000+ concurrent game rooms per server

The architecture must handle Taiwan Mahjong specific requirements:
- Complex tile claiming priority rules (碰 > 吃, 胡 > everything)
- Automatic flower tile (花牌) replacement
- Real-time scoring calculations
- Turn-based gameplay with time limits

## Decision Drivers
- **Low Latency**: Game actions must feel instantaneous
- **State Consistency**: All players must see identical game state
- **Fault Tolerance**: Handle disconnections gracefully
- **Cheat Prevention**: Server-authoritative game logic
- **Scalability**: Support thousands of concurrent rooms
- **Taiwan Mahjong Rules**: Complex claiming and priority systems

## Decision

### Architecture Pattern: Event-Driven Server-Authoritative Model

**Core Components:**

1. **Game Room Manager**
   - Manages room lifecycle and player connections
   - Handles room creation, joining, and cleanup
   - Maintains room state and player sessions

2. **Game State Engine**
   - Server-authoritative game state management
   - Validates all player actions against Taiwan Mahjong rules
   - Handles complex claiming priority resolution
   - Manages automatic flower replacement sequences

3. **Real-time Communication Layer**
   - Socket.IO for WebSocket connections with fallback
   - Event-driven message passing between client and server
   - Client prediction with server reconciliation

4. **Session Management**
   - Redis-based session storage for reconnection
   - Player authentication and room permissions
   - Persistent game state during disconnections

### Communication Protocol

**Client-to-Server Events:**
```typescript
// Player Actions
'game:draw_tile' -> server validates and broadcasts
'game:discard_tile' -> server validates and broadcasts  
'game:claim_tile' -> server validates priority and resolves
'game:declare_ready' -> server validates hand and broadcasts
'game:declare_win' -> server validates win condition

// Room Management  
'room:join' -> server adds player to room
'room:leave' -> server removes player and handles replacement
```

**Server-to-Client Events:**
```typescript
// Game State Updates
'game:state_update' -> full game state sync
'game:tile_drawn' -> tile drawn by player
'game:tile_discarded' -> tile discarded to table
'game:tile_claimed' -> tile claimed by player
'game:game_end' -> game completed with scoring

// System Events
'room:player_joined' -> new player notification
'room:player_disconnected' -> disconnection notification
'system:reconnection_data' -> full state for reconnection
```

### State Synchronization Strategy

1. **Authoritative Server State**
   - All game logic executed on server
   - Client sends intentions, server validates and executes
   - Server broadcasts confirmed actions to all clients

2. **Client Prediction with Rollback**
   - Client predicts own actions for immediate feedback
   - Server confirmation reconciles any prediction errors
   - Rollback mechanism for invalid predictions

3. **Differential State Updates**
   - Send only changed state data to minimize bandwidth
   - Full state sync on reconnection or desync detection
   - Periodic state checksums for consistency validation

### Fault Tolerance Mechanisms

1. **Disconnection Handling**
   - 30-second grace period for reconnection
   - Game pause when player disconnects
   - AI replacement for permanently disconnected players

2. **Network Resilience**
   - Automatic reconnection with exponential backoff
   - Message queuing during temporary disconnections
   - State restoration from Redis on reconnection

3. **Error Recovery**
   - Invalid action handling with user feedback
   - State desynchronization detection and recovery
   - Graceful degradation for partial network failures

## Consequences

### Positive
- **Responsive Gameplay**: Client prediction provides immediate feedback
- **Fair Play**: Server authority prevents cheating
- **Reliable**: Handles network issues gracefully
- **Scalable**: Stateless server design with Redis state storage
- **Authentic**: Proper Taiwan Mahjong rule implementation

### Negative
- **Complexity**: Event-driven architecture requires careful state management
- **Resource Usage**: Server needs to maintain game state for all rooms
- **Latency Sensitivity**: Poor networks will impact user experience
- **Development Time**: Complex synchronization logic takes time to implement correctly

### Neutral
- **Testing Requirements**: Need comprehensive integration tests for all scenarios
- **Monitoring Needs**: Requires detailed logging and metrics for troubleshooting

## Technical Implementation

### Game Room State Structure
```typescript
interface GameRoom {
  id: string;
  players: [Player, Player, Player, Player];
  gameState: MahjongGameState;
  currentPlayer: number;
  phase: 'waiting' | 'dealing' | 'playing' | 'claiming' | 'finished';
  deck: MahjongTile[];
  discardPile: MahjongTile[];
  claimQueue: ClaimAction[];
}

interface MahjongGameState {
  round: number;
  dealer: number;
  wind: 'east' | 'south' | 'west' | 'north';
  tilesRemaining: number;
  specialFlags: {
    consecutiveDealer: number;
    flowersRevealed: number[];
  };
}
```

### Claiming Priority Resolution
```typescript
class ClaimResolver {
  resolveClaims(claims: ClaimAction[]): ClaimAction | null {
    // Taiwan Mahjong priority: 胡 > 碰/槓 > 吃
    const winClaims = claims.filter(c => c.type === 'win');
    if (winClaims.length > 0) return winClaims[0];
    
    const pongKongClaims = claims.filter(c => c.type === 'pong' || c.type === 'kong');
    if (pongKongClaims.length > 0) return pongKongClaims[0];
    
    const chowClaims = claims.filter(c => c.type === 'chow');
    if (chowClaims.length > 0) return chowClaims[0];
    
    return null;
  }
}
```

## Alternatives Considered

### Alternative 1: Peer-to-Peer Architecture
- **Pros**: Lower server costs, reduced latency
- **Cons**: Vulnerable to cheating, NAT traversal issues, complex synchronization

### Alternative 2: Traditional HTTP Polling
- **Pros**: Simpler to implement, works with all networks
- **Cons**: Higher latency, inefficient bandwidth usage, poor real-time experience

### Alternative 3: WebRTC Data Channels
- **Pros**: Lowest possible latency, direct peer communication
- **Cons**: Complex implementation, firewall issues, no server authority

### Alternative 4: Custom TCP Protocol
- **Pros**: Maximum performance, full control
- **Cons**: No web browser support, increased development complexity

## Implementation Phases

### Phase 1: Core Infrastructure (Weeks 1-2)
- Set up Socket.IO server with room management
- Implement basic authentication and session handling
- Create Redis integration for state persistence

### Phase 2: Game Logic Integration (Weeks 3-4)
- Integrate Taiwan Mahjong rules engine
- Implement server-authoritative action validation
- Add claiming priority resolution system

### Phase 3: Client Synchronization (Weeks 5-6)
- Implement client prediction and rollback
- Add differential state updates
- Create reconnection and state restoration

### Phase 4: Fault Tolerance (Weeks 7-8)
- Add comprehensive error handling
- Implement disconnection management
- Create monitoring and logging systems

## Validation Criteria

### Performance Requirements
- [ ] Game action response time < 100ms (95th percentile)
- [ ] Room capacity: 1,000+ concurrent rooms per server
- [ ] Reconnection time < 5 seconds
- [ ] State consistency: 99.99% across all scenarios

### Functional Requirements
- [ ] All Taiwan Mahjong claiming rules correctly implemented
- [ ] Automatic flower tile replacement works correctly
- [ ] Complex multi-player claiming scenarios handled properly
- [ ] AI player replacement functions seamlessly

### Reliability Requirements
- [ ] Graceful handling of all network failure scenarios
- [ ] No game state corruption under any circumstances
- [ ] Complete game recovery after server restart
- [ ] Comprehensive logging for debugging

## References
- Taiwan Mahjong Rules Documentation (`reference/` directory)
- PRD Section 7: Multiplayer & Social Features
- PRD Section 9.1: Performance Requirements
- Socket.IO Documentation and Best Practices

---
**Decision Date**: August 7, 2025  
**Review Date**: September 7, 2025  
**Dependencies**: ADR-001 (Technology Stack Selection)  
**Status**: Awaiting technical validation and team review