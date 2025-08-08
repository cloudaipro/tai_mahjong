# Technical Specifications: Cryptographic Security and Monitoring Systems

**Document Version**: 1.1  
**Date**: 2025-08-08  
**Status**: Implementation Required  
**Related ADRs**: ADR-002-collaborative-architecture-security-performance, ADR-003-development-methodology-selection  
**Updated**: Integrated with DDD + Clean Architecture (Backend) and Component-based + Redux (Frontend) methodologies

---

## Overview

This document provides detailed technical specifications for implementing cryptographic security measures and comprehensive monitoring systems for the Taiwan Mahjong online game, integrated with the adopted development methodologies:

- **Backend Security Implementation**: DDD + Clean Architecture patterns with security as cross-cutting concern
- **Frontend Security Integration**: Component-based + Redux with secure command processing  
- **Cross-cutting Security**: Type-safe interfaces and end-to-end encryption

These specifications build upon collaborative architecture review recommendations and methodology selections.

## 1. Cryptographic Security Specifications

### 1.1 Digital Signature System

**Algorithm**: ECDSA (Elliptic Curve Digital Signature Algorithm)  
**Curve**: P-384 (secp384r1)  
**Hash Function**: SHA-384  
**Key Size**: 384-bit private key, 768-bit public key

#### Implementation Requirements with DDD + Clean Architecture:

```typescript
// Domain Layer - Security Value Objects
class CommandSignature {
  constructor(
    private readonly signature: Uint8Array,
    private readonly algorithm: 'ECDSA-P384',
    private readonly timestamp: number
  ) {}
  
  isValid(publicKey: CryptoKey, payload: string): Promise<boolean> {
    return webcrypto.subtle.verify(
      { name: 'ECDSA', hash: 'SHA-384' },
      publicKey,
      this.signature,
      new TextEncoder().encode(payload)
    );
  }
}

// Domain Layer - Secure Command Entity
class SecureGameCommand {
  constructor(
    private readonly gameCommand: GameCommand,
    private readonly playerId: PlayerId,
    private readonly nonce: Nonce,
    private readonly signature: CommandSignature
  ) {}
  
  // Domain method for signature validation
  async validateSignature(verificationService: ISignatureVerificationService): Promise<ValidationResult> {
    const payload = this.createPayload();
    const isValid = await this.signature.isValid(
      await verificationService.getPublicKey(this.playerId),
      payload
    );
    
    return isValid ? 
      ValidationResult.success() : 
      ValidationResult.failure('Invalid signature');
  }
  
  private createPayload(): string {
    return JSON.stringify({
      command: this.gameCommand.serialize(),
      playerId: this.playerId.value,
      nonce: this.nonce.value,
      timestamp: this.signature.timestamp
    });
  }
}

// Application Layer - Security Service
@Injectable()
class SecureCommandService {
  constructor(
    private readonly signatureService: ISignatureVerificationService,
    private readonly nonceService: INonceValidationService
  ) {}
  
  async validateSecureCommand(secureCommand: SecureGameCommand): Promise<SecurityValidationResult> {
    // Validate signature
    const signatureValidation = await secureCommand.validateSignature(this.signatureService);
    if (!signatureValidation.isSuccess()) {
      return SecurityValidationResult.failure('Invalid signature');
    }
    
    // Validate nonce (prevent replay attacks)
    const nonceValidation = await this.nonceService.validateNonce(
      secureCommand.getNonce()
    );
    if (!nonceValidation.isSuccess()) {
      return SecurityValidationResult.failure('Replay attack detected');
    }
    
    return SecurityValidationResult.success();
  }
}

// Infrastructure Layer - Cryptographic Configuration
interface CryptographicConfig {
  algorithm: {
    name: 'ECDSA';
    namedCurve: 'P-384';
    hash: 'SHA-384';
  };
  keyUsage: ['sign', 'verify'];
  extractable: boolean;
  keyRotationPeriod: number; // 90 days in milliseconds
}
```

#### Frontend Security Integration with Component-based + Redux:

```typescript
// Redux Middleware for Command Security
const securityMiddleware: Middleware = (store) => (next) => (action) => {
  if (isGameAction(action)) {
    // Sign command before sending to backend
    const secureCommand = signGameCommand(action, getCurrentPlayerId());
    
    // Send secure command via WebSocket
    webSocketService.sendSecureCommand(secureCommand);
    
    // Don't process action locally until server confirms
    return;
  }
  
  return next(action);
};

// React Component for Secure Command Processing
const SecureGameActionsComponent: React.FC = () => {
  const dispatch = useAppDispatch();
  const [signingInProgress, setSigningInProgress] = useState(false);
  
  const executeSecureCommand = useCallback(async (gameAction: GameAction) => {
    setSigningInProgress(true);
    
    try {
      // Create secure command with signature
      const secureCommand = await createSecureCommand(gameAction);
      
      // Dispatch through security middleware
      dispatch(secureCommand);
      
    } catch (error) {
      // Handle security errors in UI
      dispatch(showSecurityError('Command signing failed'));
    } finally {
      setSigningInProgress(false);
    }
  }, [dispatch]);
  
  return (
    <ActionButtonsComponent 
      onAction={executeSecureCommand}
      disabled={signingInProgress}
    />
  );
};

// Type-safe Command Signing Interface
interface SecureCommandCreator {
  signMopaiCommand(playerId: string, gameId: string): Promise<SecureCommand>;
  signDapaiCommand(playerId: string, gameId: string, tile: Tile): Promise<SecureCommand>;
  signHupaiCommand(playerId: string, gameId: string): Promise<SecureCommand>;
}
```

#### Key Management:
- **Key Generation**: Server-side HSM (Hardware Security Module) preferred
- **Key Storage**: Encrypted at rest with AES-256-GCM
- **Key Rotation**: Every 90 days, with 7-day overlap period
- **Key Distribution**: Secure key exchange using ECDH
- **Frontend Key Handling**: Client-side key caching with secure storage (Web Crypto API)

### 1.2 Anti-Replay Protection

**Mechanism**: Time-based tokens with cryptographic nonces  
**Window**: 30-second acceptance window  
**Nonce Size**: 128-bit random values  
**Storage**: Redis with automatic expiration

#### Nonce Management:
```typescript
interface NonceValidation {
  nonce: Uint8Array;      // 16 bytes random
  timestamp: number;      // Unix timestamp in ms
  playerId: string;       // Player identifier
  commandId: string;      // Unique command identifier
  expiration: number;     // Auto-expire in 60 seconds
}
```

### 1.3 Encryption Standards

**Transport Layer**: TLS 1.3 minimum  
**WebSocket Encryption**: WSS (WebSocket Secure) mandatory  
**Data at Rest**: AES-256-GCM with unique DEK (Data Encryption Keys)  
**Key Derivation**: PBKDF2 with 100,000 iterations, SHA-256

#### Encryption Configuration:
```typescript
interface EncryptionConfig {
  transport: {
    minVersion: 'TLSv1.3';
    cipherSuites: [
      'TLS_AES_256_GCM_SHA384',
      'TLS_CHACHA20_POLY1305_SHA256'
    ];
    certificateValidation: true;
  };
  
  dataAtRest: {
    algorithm: 'AES-256-GCM';
    keyDerivation: 'PBKDF2';
    iterations: 100000;
    saltLength: 32;
  };
  
  webSocket: {
    protocol: 'wss';
    compression: true;
    maxFrameSize: 16384;
  };
}
```

## 2. Monitoring and Observability Specifications

### 2.1 Performance Monitoring

**Primary Metrics**:
- Game Action Response Time: <70ms (95th percentile)
- Command Processing Latency: <25ms average
- State Machine Transitions: <35ms average
- Database Query Time: <15ms (95th percentile)
- Cache Hit Rate: >95%

#### Monitoring Implementation:
```typescript
interface PerformanceMetrics {
  responseTime: {
    target: 70;           // milliseconds
    warning: 50;          // milliseconds
    critical: 100;        // milliseconds
    measurement: '95th percentile';
  };
  
  commandProcessing: {
    target: 25;           // milliseconds
    buckets: [5, 10, 15, 25, 35, 50, 75, 100];
    labels: ['command_type', 'validation_result'];
  };
  
  cachePerformance: {
    hitRateTarget: 0.95;  // 95%
    levels: ['local', 'redis', 'database'];
    alertThreshold: 0.90; // Alert if below 90%
  };
}
```

### 2.2 Security Monitoring

**Security Metrics**:
- Command Signature Validation Rate: 100%
- Replay Attack Detection: Real-time
- Anti-Cheat Anomaly Score Tracking: Continuous
- Failed Authentication Attempts: Rate-limited logging

#### Security Event Configuration:
```typescript
interface SecurityMonitoring {
  signatureValidation: {
    successRate: 1.0;     // 100% required
    alertOnFailure: true;
    escalationThreshold: 0.99; // Alert if below 99%
  };
  
  replayAttackDetection: {
    detectionWindow: 30000;    // 30 seconds
    maxAttempts: 3;           // Per player per minute
    blockDuration: 300000;    // 5 minutes
  };
  
  antiCheatScoring: {
    suspiciousThreshold: 0.8;  // Flag for review
    blockingThreshold: 0.95;   // Immediate action
    mlModelVersion: 'v1.0';
    updateFrequency: 'weekly';
  };
}
```

### 2.3 System Health Monitoring

**Infrastructure Metrics**:
- CPU Usage: <70% average
- Memory Usage: <80% average  
- Disk I/O: <1000 IOPS average
- Network Latency: <50ms between services

#### Health Check Specifications:
```typescript
interface HealthMonitoring {
  endpoints: [
    '/health/live',         // Liveness probe
    '/health/ready',        // Readiness probe
    '/health/startup'       // Startup probe
  ];
  
  checks: {
    database: {
      timeout: 5000;        // 5 seconds
      query: 'SELECT 1';
      interval: 30000;      // 30 seconds
    };
    
    redis: {
      timeout: 2000;        // 2 seconds  
      command: 'PING';
      interval: 15000;      // 15 seconds
    };
    
    webSocket: {
      connectionCount: true;
      averageLatency: true;
      failedConnections: true;
    };
  };
}
```

## 3. Taiwan Mahjong Specific Monitoring

### 3.1 Game Integrity Monitoring

**Game Rule Validation**:
- Taiwan Mahjong Rule Compliance: 100%
- Score Calculation Accuracy: 100%
- Tile Distribution Fairness: Statistical validation
- Game State Consistency: Cross-player validation

#### Game Integrity Configuration:
```typescript
interface GameIntegrityMonitoring {
  ruleCompliance: {
    taiwanMahjongRules: {
      handSize: 16;              // Max tiles per player
      flowerTiles: 8;            // Total flower tiles
      scoringSystem: 'taiwan';   // Taiwan scoring (台數)
    };
    
    validation: {
      serverSide: true;          // Mandatory server validation
      crossValidation: true;     // Multi-player state check
      historicalVerification: true; // Command history integrity
    };
  };
  
  fairnessMetrics: {
    tileDistribution: {
      chiSquareTest: true;       // Statistical randomness
      playerWinRates: true;      // Balanced win distribution  
      shufflingEntropy: true;    // Cryptographic randomness
    };
    
    scoringValidation: {
      calculationAccuracy: 1.0;  // 100% accuracy required
      consistencyCheck: true;    // Same input = same output
      auditTrail: true;         // Full calculation logging
    };
  };
}
```

### 3.2 Player Behavior Analytics

**Behavioral Patterns**:
- Action Timing Analysis
- Decision Pattern Recognition  
- Win Rate Statistical Analysis
- Social Interaction Monitoring

#### Behavioral Analytics:
```typescript
interface BehaviorAnalytics {
  playerMetrics: {
    actionTiming: {
      averageDecisionTime: 'milliseconds';
      patternRecognition: 'ml_enabled';
      outlierDetection: true;
    };
    
    gameplayPatterns: {
      preferredActions: ['摸牌', '打牌', '碰牌', '槓牌', '胡牌'];
      winningStrategies: 'pattern_analysis';
      riskTolerance: 'statistical_model';
    };
  };
  
  socialMetrics: {
    chatBehavior: 'content_analysis';
    friendInteractions: 'network_analysis';
    reportingPatterns: 'abuse_detection';
  };
}
```

## 4. Alert and Notification System

### 4.1 Alert Severity Levels

**Critical Alerts** (Immediate Response Required):
- Security breach detected
- System unavailable (>5 minutes)
- Database connection failure
- Authentication system compromise

**Warning Alerts** (Response within 1 hour):
- Performance degradation (>70ms response time)
- Cache hit rate below 90%
- Unusual player behavior patterns
- Resource utilization above 80%

**Info Alerts** (Response within 24 hours):
- Player reports and feedback
- System optimization opportunities
- Usage pattern changes
- Scheduled maintenance reminders

#### Alert Configuration:
```typescript
interface AlertingSystem {
  channels: {
    critical: ['pagerduty', 'slack_critical', 'email'];
    warning: ['slack_monitoring', 'email'];
    info: ['slack_general', 'dashboard'];
  };
  
  escalation: {
    noResponseTime: {
      critical: 300000;    // 5 minutes
      warning: 3600000;    // 1 hour  
    };
    
    escalationPath: [
      'on_call_engineer',
      'senior_engineer',
      'engineering_manager',
      'cto'
    ];
  };
}
```

### 4.2 Dashboard and Visualization

**Real-time Dashboards**:
1. **Operational Dashboard**: System health, response times, user counts
2. **Security Dashboard**: Threat detection, authentication stats, anomalies  
3. **Game Analytics Dashboard**: Player behavior, game statistics, Taiwan Mahjong metrics
4. **Business Dashboard**: Revenue metrics, user engagement, retention rates

#### Dashboard Specifications:
```typescript
interface DashboardConfig {
  operational: {
    refreshRate: 5000;        // 5 seconds
    metrics: [
      'response_time',
      'concurrent_users', 
      'system_health',
      'error_rates'
    ];
    alertIntegration: true;
  };
  
  security: {
    refreshRate: 1000;        // 1 second
    metrics: [
      'threat_detection',
      'failed_logins',
      'cheat_detection',
      'encryption_status'
    ];
    confidentialityLevel: 'restricted';
  };
  
  gameAnalytics: {
    refreshRate: 30000;       // 30 seconds
    metrics: [
      'game_completion_rate',
      'taiwan_scoring_stats',
      'player_engagement',
      'rule_violation_rate'
    ];
    historicalData: true;
  };
}
```

## 5. Data Retention and Compliance

### 5.1 Data Classification

**Highly Sensitive Data** (7-year retention):
- Player personal information
- Payment information
- Authentication credentials
- Security audit logs

**Game Data** (3-year retention):
- Game session recordings
- Player statistics
- Taiwan Mahjong game logs
- Performance metrics

**Temporary Data** (30-day retention):
- Session tokens
- Cache data
- Debug logs
- Monitoring metrics

### 5.2 GDPR Compliance

**Data Subject Rights**:
- Right to access personal data
- Right to data portability
- Right to erasure ("right to be forgotten")
- Right to rectification

#### GDPR Implementation:
```typescript
interface GDPRCompliance {
  dataMapping: {
    personalData: ['email', 'name', 'ip_address', 'device_id'];
    gameData: ['statistics', 'preferences', 'friends_list'];
    technicalData: ['session_logs', 'error_logs'];
  };
  
  retention: {
    activeUser: '3_years';
    inactiveUser: '30_days_after_last_login';
    deletedUser: 'immediate_anonymization';
  };
  
  dataExport: {
    format: 'json';
    delivery: 'secure_download_link';
    expiration: '7_days';
  };
}
```

## 6. Disaster Recovery and Business Continuity

### 6.1 Backup Strategy

**Database Backups**:
- Full backup: Daily at 02:00 UTC
- Incremental backup: Every 6 hours
- Point-in-time recovery: 15-minute granularity
- Cross-region replication: Real-time

**Application Backups**:
- Code repository: Git with multiple mirrors
- Configuration: Encrypted backup daily
- Secrets: HSM with geographic distribution

### 6.2 Recovery Objectives

**Recovery Time Objective (RTO)**: 15 minutes  
**Recovery Point Objective (RPO)**: 5 minutes  
**Maximum Tolerable Downtime**: 1 hour per month

#### Disaster Recovery Configuration:
```typescript
interface DisasterRecovery {
  objectives: {
    rto: 900;                 // 15 minutes in seconds
    rpo: 300;                 // 5 minutes in seconds
    maximumDowntime: 3600;    // 1 hour per month
  };
  
  procedures: {
    automaticFailover: true;
    manualOverride: true;
    rollbackCapability: true;
    testingSchedule: 'monthly';
  };
  
  infrastructure: {
    primaryRegion: 'us-west-2';
    backupRegions: ['us-east-1', 'eu-west-1'];
    dataReplication: 'synchronous';
    loadBalancing: 'multi_region';
  };
}
```

## 7. Implementation Timeline

### Phase 1: Cryptographic Foundation (Weeks 1-4)
- Implement ECDSA signature system
- Deploy anti-replay protection
- Set up basic security monitoring

### Phase 2: Advanced Monitoring (Weeks 5-8)  
- Deploy performance monitoring stack
- Implement security event correlation
- Create real-time dashboards

### Phase 3: Game-Specific Features (Weeks 9-12)
- Taiwan Mahjong integrity monitoring
- Player behavior analytics
- Advanced threat detection

### Phase 4: Compliance and Testing (Weeks 13-16)
- GDPR compliance implementation
- Disaster recovery testing
- Security audit and penetration testing

## 8. Success Criteria

**Security Requirements**:
- ✅ 100% command signature validation
- ✅ Zero successful replay attacks
- ✅ <0.1% false positive rate in cheat detection
- ✅ Full GDPR compliance

**Performance Requirements** (Post-Architecture Recalibration):
- ✅ <70ms backend response time (95th percentile)
- ✅ 60fps web rendering with Performance Budget framework
- ✅ 45-50fps mobile rendering with Low-Fidelity Mode fallback
- ✅ <2 second network reconnection recovery
- ✅ >99.9% system availability
- ✅ >95% cache hit rate with proactive refresh
- ✅ 10,000+ concurrent user support

**Monitoring Requirements**:
- ✅ Real-time threat detection
- ✅ Comprehensive audit logging
- ✅ Automated alerting system
- ✅ Interactive dashboards

## 9. Architecture Recalibration Enhancements (2025-08-08)

Based on the collaborative 5-specialist architecture review, the following enhancements have been integrated:

### 9.1 Enhanced Caching Strategy

**Proactive Cache Refresh Implementation**:
```typescript
class ProactiveCacheService {
  private readonly REFRESH_THRESHOLD = 0.75; // Refresh at 75% TTL
  
  async getWithProactiveRefresh<T>(
    key: string, 
    fetcher: () => Promise<T>,
    ttl: number
  ): Promise<T> {
    const cached = await this.redis.get(key);
    const remainingTtl = await this.redis.ttl(key);
    
    if (cached) {
      // Proactive refresh to prevent thundering herd
      if (remainingTtl > 0 && remainingTtl < (ttl * this.REFRESH_THRESHOLD)) {
        this.scheduleBackgroundRefresh(key, fetcher, ttl);
      }
      return JSON.parse(cached);
    }
    
    // Cache miss - fetch and cache
    const fresh = await fetcher();
    await this.redis.set(key, JSON.stringify(fresh), { EX: ttl });
    return fresh;
  }
}
```

### 9.2 Network Resiliency Framework

**Client-Side Command Queue with Anti-Abuse**:
- Idempotent command processing with UUIDs
- WebSocket heartbeat protocol (5s ping, 2.5s timeout)
- Progressive disconnection penalties (3+ loses grace period)
- Exponential backoff reconnection (1s, 2s, 4s, 8s, max 30s)

### 9.3 Architectural Governance Automation

**Fitness Functions in CI/CD Pipeline**:
```typescript
// Automated Architecture Compliance Tests
describe('Architecture Governance', () => {
  test('Domain layer dependency isolation', () => {
    const violations = archUnit.classes()
      .that().resideInAPackage('domain.*')
      .should().onlyDependOnClassesThat()
      .resideInAnyPackage('domain.*', 'shared.*');
    
    expect(violations.evaluate()).toHaveLength(0);
  });
});
```

### 9.4 4-Layer Testing Framework

**Comprehensive Taiwan Mahjong Rule Validation**:
1. **Unit Tests**: >95% coverage of domain logic
2. **Integration Tests**: Cross-module interaction validation
3. **JSON Scenario Tests**: Edge case scenarios (過水, 詐胡, 搶槓)
4. **Expert Validation**: Taiwan Mahjong master approval

### 9.5 Performance Budget Framework

**Mobile 3D Rendering Optimization**:
```typescript
class MobilePerformanceBudget {
  private readonly TARGET_FPS = 45;
  private readonly MAX_POLYGONS = 5000;
  private readonly MAX_DRAW_CALLS = 50;
  
  monitorAndAdjust(): void {
    if (this.currentFPS < this.TARGET_FPS) {
      this.enableLowFidelityMode();
    }
  }
  
  private enableLowFidelityMode(): void {
    this.switchTo2DMode();
    this.notifyUser('Optimized for better performance');
  }
}
```

### 9.6 Implementation Impact

| Enhancement Area | Implementation Week | Owner | Impact |
|------------------|-------------------|-------|---------|
| **Architectural Governance** | Week 3-4 | System Architect | Prevents architectural drift |
| **Caching Enhancement** | Week 3-4 | Data Architect | Eliminates thundering herd |
| **Network Resiliency** | Week 3-4, 9-10 | Mobile + Frontend | <2s reconnection |
| **Testing Framework** | Week 5-6+ | Game Systems | 100% rule accuracy |
| **Performance Budget** | Week 9-10, 21-22 | Frontend + Mobile | 45-50fps mobile |

---

**Document Status**: Approved for Implementation (Recalibrated)  
**Review Schedule**: Bi-weekly architecture validation + Monthly technical review  
**Contact**: Architecture Review Board, Security Team, DevOps Team, Development Team  
**Last Updated**: 2025-08-08 (Post-Collaborative Review)