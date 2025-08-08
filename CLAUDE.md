# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Taiwan Mahjong online game development project that has completed comprehensive architecture planning and is ready for implementation. The goal is to create an authentic 16-tile Taiwan Mahjong game with web and mobile versions that faithfully follows traditional Taiwan Mahjong rules.

**Current Phase**: Implementation-ready with complete architecture and MVP plan
**Status**: Ready for development team assembly and Phase 1 kickoff

## Key Architecture Context

**Full Product Architecture** (8-month timeline):
- **Multi-platform target**: Web (React.js + TypeScript), Mobile (React Native)
- **Backend**: Node.js hybrid modular monolith with DDD + Clean Architecture
- **Database**: PostgreSQL + Redis with advanced caching strategies
- **Real-time**: WebSocket with custom protocol and network resiliency
- **Security**: ECDSA cryptographic security with 12-layer protection
- **Deployment**: Docker + Kubernetes with multi-region support

**MVP Architecture** (16-week timeline) - **CURRENT FOCUS**:
- **Platform**: Web-first with Progressive Web App (PWA) for mobile
- **Backend**: Node.js + Express with simplified modular monolith (3 core modules)
- **Database**: PostgreSQL + Redis with basic caching
- **Real-time**: Socket.io with heartbeat protocol and command queue
- **Security**: JWT + HTTPS + basic rate limiting
- **Deployment**: Docker + AWS single region

## Important Documentation Files

### Core Documentation
- `docs/台灣麻將線上遊戲PRD.md` - Product Requirements Document (updated with recalibrated targets)
- `docs/detailed-implementation-roadmap.md` - Complete 8-month implementation plan with integrated architecture review findings

### Architecture Documentation  
- `docs/architecture-recalibration-2025-08-08.md` - Comprehensive architecture recalibration report
- `docs/architecture-recalibration-validation-report.md` - Validation against original objectives
- `docs/mvp-collaborative-architecture-session-2025-08-08.md` - 8-specialist MVP planning session
- `docs/mvp-technical-specifications.md` - Complete 16-week MVP implementation blueprint

### Architecture Decision Records (ADRs) - 13 Total
- `.architecture/decisions/ADR-INDEX.md` - Complete ADR index and governance

**Foundation Series (ADR-001)**:
- `ADR-001-hybrid-architecture-decision.md` - Core hybrid architecture approach
- `ADR-001-modular-monolith-architecture.md` - Modular monolith structure with DDD
- `ADR-001-technology-stack-selection.md` - React/React Native + Node.js + PostgreSQL/Redis

**Security & Performance Series (ADR-002)**:
- `ADR-002-collaborative-architecture-security-performance.md` - Integrated security-performance framework
- `ADR-002-comprehensive-security-architecture.md` - ECDSA cryptographic security (full product)
- `ADR-002-real-time-multiplayer-architecture.md` - WebSocket-based real-time communication

**Development & Optimization Series (ADR-003)**:
- `ADR-003-development-methodology-selection.md` - DDD + Clean Architecture + Component-based + Redux
- `ADR-003-performance-optimization-strategy.md` - Multi-layer caching with <70ms targets

**Recalibration Series (ADR-004-008) - Post-Review Enhancements**:
- `ADR-004-architectural-governance-framework.md` - Automated fitness functions for CI/CD
- `ADR-005-performance-targets-recalibration.md` - Evidence-based mobile 45-50fps targets
- `ADR-006-network-resiliency-architecture.md` - Client-side command queue + heartbeat protocol
- `ADR-007-4-layer-testing-framework.md` - Comprehensive Taiwan Mahjong rule validation
- `ADR-008-mvp-architecture-decisions.md` - MVP scope and 16-week technical decisions

### Reference Materials
- `docs/from_gemini/` - Collaborative architecture review outputs from 5-specialist review
- `reference/` - Comprehensive Taiwan Mahjong rules documentation (11 files in Chinese and English)
- `reference/FromGaaS/` - System analysis screenshots for architectural reference
- `prompt.txt` - Original project brief and requirements

## Reference Materials Structure

The `reference/` directory contains extensive Taiwan Mahjong documentation:
- Traditional 16-tile rules and gameplay mechanics
- Taiwan-specific scoring (台數) systems  
- Special rules and edge cases
- Visual guides and examples
- These files are essential for understanding authentic Taiwan Mahjong requirements

## MCP Server Configuration

Five MCP servers are configured in Claude Code system:
1. **context7**: For retrieving up-to-date library documentation and code examples
2. **graphiti-memory**: Knowledge graph memory service with Neo4j backend
3. **firecrawl-mcp**: Advanced web scraping and content extraction
4. **puppeteer**: Browser automation and interaction
5. **sequential-thinking**: Dynamic problem-solving through structured thought processes

## Development Workflow

This project uses a PRP (Product Requirements Process) workflow:
- Templates located in `PRPs/templates/prp_base.md`
- Emphasizes context-rich documentation and validation loops
- Follow all rules specified in CLAUDE.md (this file)

## Project Structure Understanding

```
tai_mahjong/
├── docs/                           # Complete project documentation
│   ├── 台灣麻將線上遊戲PRD.md         # Product Requirements Document
│   ├── detailed-implementation-roadmap.md  # 8-month implementation plan
│   ├── architecture-recalibration-*    # Architecture review outcomes
│   ├── mvp-*                          # MVP planning and specifications
│   ├── from_gemini/                   # 5-specialist review outputs
│   └── technical-specifications-*.md   # Technical specs and monitoring
├── .architecture/                      # Architectural Decision Records
│   └── decisions/
│       ├── ADR-INDEX.md               # ADR governance and index
│       └── adrs/                      # Individual ADRs (004-008 current)
├── reference/                         # Taiwan Mahjong rules reference
│   ├── FromGaaS/                      # System architecture screenshots
│   └── *.md                          # Rule documentation (Chinese/English)
├── PRPs/                             # Product Requirements Process templates
├── prompt.txt                        # Original project requirements
└── README.md                         # Project overview in Chinese
```

## Development Guidelines

### Core Principles
1. **Taiwan Mahjong Rule Authenticity**: 100% faithful to traditional Taiwan 16-tile Mahjong rules (non-negotiable)
2. **Architecture Compliance**: Follow ADR decisions and architectural boundaries enforced by fitness functions
3. **MVP-First Approach**: Implement MVP scope before expanding to full product features
4. **Expert Validation**: All game logic must pass 4-layer testing including Taiwan Mahjong expert approval
5. **Cultural Authenticity**: Preserve Taiwan Mahjong terminology and cultural elements

### Implementation Guidelines
- **Reference Documentation**: Always check `reference/` for Taiwan Mahjong rules before implementing game logic
- **ADR Compliance**: Review relevant ADRs in `.architecture/decisions/adrs/` before making architectural changes
- **Performance Targets**: MVP targets are <100ms backend response, 60fps web, 30fps mobile web
- **Testing Requirements**: >95% test coverage for game logic, comprehensive scenario testing for edge cases
- **Security Standards**: Follow JWT + HTTPS security framework for MVP, prepare for future ECDSA enhancement

## Current Development Status

### Architecture & Planning Phase (COMPLETE)
- ✅ Game rules analysis and documentation complete
- ✅ Product Requirements Document (PRD) complete and updated  
- ✅ Collaborative architecture review with 5 specialists complete
- ✅ Architecture recalibration with realistic performance targets complete
- ✅ MVP planning session with 8 specialists complete
- ✅ 13 Architectural Decision Records (ADRs) documented (8 foundation + 5 post-review)
- ✅ Complete technical specifications for both MVP and full product
- ✅ Implementation roadmap with 16-week MVP and 8-month full product timelines

### Ready for Implementation
- ✅ **MVP Development Ready**: 16-week timeline, 8-10 developer team, complete technical specs
- ✅ **Architecture Validated**: Unanimous approval from 8-specialist collaborative review
- ✅ **Documentation Complete**: All planning, architecture, and technical documentation finished
- 🚀 **Next Step**: Stakeholder approval → Team assembly → Phase 1 implementation kickoff

### Build/Test Commands (Post-Implementation)
```bash
# MVP will use:
npm install          # Install dependencies
npm run build       # Build application  
npm run test        # Run test suite (>95% coverage for game logic)
npm run dev         # Development server
npm run lint        # Code linting and formatting
docker-compose up   # Local development environment
```

## Key Project Characteristics

### Cultural & Technical Requirements
- **Language**: Project documentation is primarily in Traditional Chinese
- **Rule Authenticity**: Focus on traditional Taiwan Mahjong rules, not other regional variants  
- **Expert Validation**: Taiwan Mahjong masters will validate rule implementation
- **Multi-language Reference**: Reference materials include both Chinese and English versions

### Core Technical Features
- **Real-time Multiplayer**: 4-player simultaneous gameplay is core requirement
- **Network Resilience**: Client-side command queue handles mobile network instability
- **Cross-platform**: Web-first with PWA mobile support for MVP, React Native for full product
- **Performance**: <100ms response times with 60fps web rendering

### Architecture Characteristics  
- **Modular Monolith**: Clean boundaries with automated fitness function enforcement
- **Event Sourcing**: Complete game replay capability via command logging
- **Hybrid Timeline**: 16-week MVP for market validation → 8-month full product
- **Collaborative Design**: Architecture validated by 8 specialist architects

## Graphiti Memory Integration

This project's complete architecture sessions, decisions, and technical details are stored in the Graphiti knowledge graph under the `taiwan_mahjong_project` group for future reference and context retrieval.