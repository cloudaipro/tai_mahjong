# Technical Specifications: Cryptographic Security and Monitoring Systems

**Document Version**: 1.0  
**Date**: 2025-08-08  
**Status**: Implementation Required  
**Related ADR**: ADR-002-collaborative-architecture-security-performance

---

## Overview

This document provides detailed technical specifications for implementing cryptographic security measures and comprehensive monitoring systems for the Taiwan Mahjong online game, as mandated by the collaborative architecture review recommendations.

## 1. Cryptographic Security Specifications

### 1.1 Digital Signature System

**Algorithm**: ECDSA (Elliptic Curve Digital Signature Algorithm)  
**Curve**: P-384 (secp384r1)  
**Hash Function**: SHA-384  
**Key Size**: 384-bit private key, 768-bit public key

#### Implementation Requirements:

```typescript
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

interface SecureCommand {
  payload: {
    command: GameCommand;
    playerId: string;
    timestamp: number;
    nonce: number[];
    roomId: string;
  };
  signature: number[];
  version: string;
}
```

#### Key Management:
- **Key Generation**: Server-side HSM (Hardware Security Module) preferred
- **Key Storage**: Encrypted at rest with AES-256-GCM
- **Key Rotation**: Every 90 days, with 7-day overlap period
- **Key Distribution**: Secure key exchange using ECDH

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

**Performance Requirements**:
- ✅ <70ms average response time
- ✅ >99.9% system availability
- ✅ >95% cache hit rate
- ✅ 10,000+ concurrent user support

**Monitoring Requirements**:
- ✅ Real-time threat detection
- ✅ Comprehensive audit logging
- ✅ Automated alerting system
- ✅ Interactive dashboards

---

**Document Status**: Approved for Implementation  
**Review Schedule**: Monthly technical review  
**Contact**: Security Team, DevOps Team, Development Team  
**Last Updated**: 2025-08-08