# ADR-006: Client-Side Network Resiliency Architecture

**Status**: Accepted  
**Date**: 2025-08-08  
**Decision Makers**: Sarah Patel (Mobile Architect) + David Kim (Frontend Architect)  
**Related**: ADR-001 (Hybrid Architecture), Mobile Performance Requirements

## Context

The collaborative architecture review identified network instability as a primary concern for mobile users. Taiwan's gaming audience frequently plays on mobile networks with variable connectivity. The review team conducted analysis showing:

### Network Challenges:
- **4G/5G Instability**: Frequent brief disconnections during mobile gaming
- **WiFi Handoffs**: Connection drops during location changes
- **Tunnels/Elevators**: Temporary signal loss in urban environments
- **Network Congestion**: Reduced bandwidth during peak usage

### User Impact:
- Lost game commands during network interruptions
- Game state desynchronization
- Frustrating reconnection experiences
- Potential for rage-quitting and abuse

## Decision

We will implement a **comprehensive client-side network resiliency architecture** featuring:

### 1. Client-Side Command Queue
- Persistent command storage with unique UUIDs
- Idempotent command processing
- Automatic retry mechanisms
- Local state optimistic updates

### 2. WebSocket Heartbeat Protocol
- 5-second ping intervals
- 2.5-second timeout detection
- Proactive connection monitoring
- Early disconnection detection

### 3. Anti-Abuse Mechanisms
- Frequent disconnection tracking
- Grace period management (30 seconds default)
- Progressive penalties for abuse
- Game integrity protection

### 4. State Resynchronization Protocol
- Full game state recovery on reconnection
- Command queue reconciliation
- Conflict resolution strategies
- Seamless user experience restoration

## Architecture Design

```typescript
// Client-Side Command Queue Implementation
class ResilientCommandQueue {
  private commandQueue: CommandWithId[] = [];
  private disconnectCount = 0;
  private readonly MAX_DISCONNECTS = 3;
  private connectionState: ConnectionState = 'CONNECTING';
  
  async enqueueCommand(command: GameCommand): Promise<void> {
    const commandWithId = {
      ...command,
      id: uuidv4(),
      timestamp: Date.now(),
      retryCount: 0
    };
    
    // Add to persistent queue
    this.commandQueue.push(commandWithId);
    await this.persistQueue();
    
    // Immediate optimistic update
    this.applyOptimisticUpdate(command);
    
    // Process if connected
    if (this.connectionState === 'CONNECTED') {
      await this.processQueue();
    }
  }
  
  async handleReconnection(): Promise<void> {
    this.disconnectCount++;
    this.connectionState = 'RESYNCING';
    
    // Anti-abuse mechanism
    if (this.disconnectCount > this.MAX_DISCONNECTS) {
      await this.sendMessage({
        type: 'FREQUENT_DISCONNECT_PENALTY',
        message: 'Grace period revoked due to frequent disconnections'
      });
    }
    
    // Resync game state
    await this.resyncGameState();
    
    // Process queued commands
    await this.processQueue();
    
    this.connectionState = 'CONNECTED';
  }
  
  private async processQueue(): Promise<void> {
    for (const command of this.commandQueue) {
      try {
        const result = await this.sendCommand(command);
        if (result.acknowledged) {
          this.removeFromQueue(command.id);
        }
      } catch (error) {
        command.retryCount++;
        if (command.retryCount > 3) {
          this.moveToFailedQueue(command);
        }
      }
    }
  }
}

// WebSocket Heartbeat Manager
class HeartbeatManager {
  private pingInterval: NodeJS.Timeout;
  private pongTimeout: NodeJS.Timeout;
  private readonly PING_INTERVAL = 5000; // 5 seconds
  private readonly PONG_TIMEOUT = 2500;  // 2.5 seconds
  
  startHeartbeat(): void {
    this.pingInterval = setInterval(() => {
      this.sendPing();
      
      this.pongTimeout = setTimeout(() => {
        // No pong received - connection lost
        this.onConnectionLost();
      }, this.PONG_TIMEOUT);
      
    }, this.PING_INTERVAL);
  }
  
  onPongReceived(): void {
    clearTimeout(this.pongTimeout);
  }
  
  private onConnectionLost(): void {
    this.connectionState = 'DISCONNECTED';
    this.startReconnectionProcess();
  }
}

// Reconnection Strategy
class ReconnectionManager {
  private reconnectAttempt = 0;
  private readonly MAX_ATTEMPTS = 10;
  private readonly BASE_DELAY = 1000; // 1 second
  
  async startReconnection(): Promise<void> {
    while (this.reconnectAttempt < this.MAX_ATTEMPTS) {
      try {
        await this.attemptConnection();
        this.onReconnectionSuccess();
        return;
      } catch (error) {
        this.reconnectAttempt++;
        const delay = this.calculateBackoffDelay();
        await this.sleep(delay);
      }
    }
    
    this.onReconnectionFailure();
  }
  
  private calculateBackoffDelay(): number {
    // Exponential backoff: 1s, 2s, 4s, 8s, 16s, max 30s
    return Math.min(
      this.BASE_DELAY * Math.pow(2, this.reconnectAttempt),
      30000
    );
  }
}
```

## Implementation Strategy

### Phase 1: Foundation (Week 3-4)
- Command queue infrastructure
- Basic WebSocket heartbeat
- Connection state management

### Phase 2: Enhancement (Week 9-10)
- Optimistic updates integration
- UI connection indicators
- Anti-abuse mechanisms

### Phase 3: Mobile Integration (Week 21-22)
- Mobile-specific optimizations
- Battery-aware reconnection
- Native network change detection

## Benefits

### User Experience
- **Seamless Gameplay**: Network drops don't interrupt game flow
- **No Lost Actions**: All player commands are preserved and executed
- **Quick Recovery**: <2 second reconnection time
- **Transparent Operation**: Users barely notice brief disconnections

### Technical Benefits
- **Reduced Server Load**: Intelligent retry mechanisms prevent spam
- **Game Integrity**: Anti-abuse features prevent exploitation
- **Scalability**: Handles network instability without degradation
- **Mobile Optimized**: Designed for cellular network challenges

### Business Impact
- **User Retention**: Frustrating disconnections don't cause player churn
- **Competitive Advantage**: Superior network handling vs. competitors
- **Market Expansion**: Playable in areas with poor connectivity

## Consequences

### Positive
- **Improved User Satisfaction**: Smooth gameplay despite network issues
- **Broader Device/Network Support**: Works well on poor connections
- **Reduced Support Tickets**: Fewer network-related user complaints
- **Fair Play Protection**: Anti-abuse prevents disconnection exploitation

### Negative
- **Implementation Complexity**: Sophisticated state management required
- **Storage Overhead**: Command queue requires local persistence
- **Battery Impact**: Additional network monitoring and processing

### Risks
- **Queue Overflow**: Long disconnections could fill command queue
- **State Conflicts**: Complex resynchronization edge cases
- **False Penalties**: Anti-abuse system might flag legitimate issues

## Success Metrics

- **Recovery Time**: 95% of reconnections complete within 2 seconds
- **Command Preservation**: 99.9% of user commands successfully executed
- **User Satisfaction**: <5% network-related complaints in user feedback
- **Abuse Prevention**: <1% of games affected by disconnection abuse

## Testing Strategy

1. **Network Simulation**: Test various disconnection scenarios
2. **Mobile Device Testing**: Real-world cellular network validation
3. **Load Testing**: Queue performance under high command volume
4. **Edge Case Testing**: Long disconnections, rapid reconnections, etc.

## Compliance

- **Owner**: Mobile Architect (Sarah Patel) + Frontend Architect (David Kim)
- **Implementation Timeline**: Week 3-4, 9-10, 21-22
- **Success Criteria**: <2 second reconnection, 99.9% command preservation
- **Review Schedule**: Weekly network resilience testing

---

**Status**: Active Implementation  
**Approved By**: Architecture Review Board ✅  
**Priority**: High (Critical for mobile market)