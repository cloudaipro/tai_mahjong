# GEMINI.md - tai_mahjong

## Project Overview

This is a Taiwan Mahjong online game development project currently in the planning and documentation phase. The goal is to create an authentic 16-tile Taiwan Mahjong game with web and mobile versions (Android, iOS) that faithfully follows traditional Taiwan Mahjong rules.

**Current Phase**: Documentation and planning - no implementation code exists yet.

## Key Architecture Context

- **Multi-platform target**: Web (React.js + TypeScript), Mobile (React Native/Flutter)
- **Backend**: Node.js modular monolith architecture
- **Database**: PostgreSQL + Redis  
- **Real-time**: WebSocket for multiplayer functionality
- **Deployment**: Docker + Kubernetes

## Important Documentation Files

- `docs/台灣麻將線上遊戲PRD.md` - Complete Product Requirements Document with technical specifications
- `prompt.txt` - Original project brief and requirements
- `reference/` - Comprehensive Taiwan Mahjong rules documentation (11 files in Chinese and English)
- `reference/FromGaaS/` - System analysis screenshots for architectural reference

## Reference Materials Structure

The `reference/` directory contains extensive Taiwan Mahjong documentation:
- Traditional 16-tile rules and gameplay mechanics
- Taiwan-specific scoring (台數) systems  
- Special rules and edge cases
- Visual guides and examples
- These files are essential for understanding authentic Taiwan Mahjong requirements

## MCP Server Configuration

Two MCP servers are configured in `.mcp.json`:

1. **context7**: For retrieving up-to-date library documentation
2. **graphiti-memory**: Knowledge graph memory service with Neo4j backend
   - Connection: `bolt://localhost:7687` 
   - Credentials: neo4j/demodemo

## Development Workflow

This project uses a PRP (Product Requirements Process) workflow:
- Templates located in `PRPs/templates/prp_base.md`
- Emphasizes context-rich documentation and validation loops
- Follow all rules specified in CLAUDE.md (this file)

## Project Structure Understanding

```
tai_mahjong/
├── docs/                    # Product documentation (PRD)
├── reference/              # Taiwan Mahjong rules reference materials  
│   ├── FromGaaS/          # System architecture screenshots
│   └── *.md               # Rule documentation (Chinese/English)
├── PRPs/                  # Product Requirements Process templates
├── prompt.txt             # Original project requirements
└── README.md              # Project overview in Chinese
```

## Development Guidelines

When working on this project:

1. **Understand Taiwan Mahjong Rules**: Always reference the comprehensive rule documentation in `reference/` before implementing game logic
2. **Follow Authentic Rules**: The game must be 100% faithful to traditional Taiwan 16-tile Mahjong rules
3. **Multi-platform Considerations**: Design for web and mobile from the start
4. **Documentation First**: This project emphasizes thorough documentation before implementation
5. **Cultural Authenticity**: Preserve Taiwan Mahjong terminology and cultural elements

## Current Development Status

- ✅ Game rules analysis and documentation complete
- ✅ Product Requirements Document (PRD) complete  
- ⏳ Ready for technical architecture and implementation planning
- ⏳ No build/test commands available yet (pre-implementation phase)

## Important Notes

- Project documentation is primarily in Traditional Chinese
- Reference materials include both Chinese and English versions
- Focus on traditional Taiwan Mahjong rules, not other regional variants
- Multiplayer real-time functionality is a core requirement