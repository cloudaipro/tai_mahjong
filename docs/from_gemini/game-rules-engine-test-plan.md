# Game Rules Engine: Test Plan
## Taiwan Mahjong Online Game Project

**Document Version**: 1.0  
**Date**: 2025-08-20
**Author**: Lisa Wang (Game Systems Architect)
**Status**: DRAFT

---

## 1. Introduction

The absolute integrity of the Taiwan Mahjong rules engine is paramount to the success of this project. Any deviation from authentic, traditional 16-tile Taiwan Mahjong rules will result in a loss of player trust. This document outlines the comprehensive test plan to ensure 100% rule accuracy and fairness.

Our testing strategy will be multi-layered, combining automated unit and integration tests, scenario-based testing, and manual validation by Mahjong experts.

---

## 2. Testing Scope

### In Scope:
*   All aspects of the core game logic and rules engine.
*   Game setup: Wall building, dealing, seating.
*   Player actions: Drawing (`摸牌`), discarding (`打牌`), Chow (`吃`), Pung (`碰`), Kong (`槓`).
*   Winning conditions (`胡牌`) and scoring (`台數`).
*   Special rules and edge cases.
*   Game flow: Turn management, round progression, handling of draws (`流局`).

### Out of Scope:
*   UI/UX testing (covered by the frontend test plan).
*   Network stability testing.
*   Performance and load testing.

---

## 3. Test Layers

### 3.1. Layer 1: Unit Tests

**Goal**: To test individual functions and components of the rules engine in isolation.
**Framework**: Jest (or similar TypeScript-compatible test runner).
**Coverage Target**: >95% for the entire domain logic module.

**Key Areas for Unit Tests**:
*   **Tile Validation**: Is a tile valid? Is it a suit, honor, or flower tile?
*   **Hand Analysis**: Functions that identify Pungs, Chows, Kongs, pairs in a hand.
*   **Winning Hand (`胡牌`) Validation**: Does a given hand meet the basic requirements for a winning hand (e.g., 5 sets and a pair)?
*   **Scoring (`台數`) Calculation**: Test individual scoring patterns in isolation (e.g., a hand with "All Pungs" (`碰碰胡`) should correctly calculate the points for that pattern).
*   **Shuffle Fairness**: Statistical analysis of the shuffling algorithm to ensure randomness and prevent biases.

### 3.2. Layer 2: Integration Tests

**Goal**: To test how different parts of the rules engine work together.

**Key Areas for Integration Tests**:
*   **Action Validation**: Can a player legally claim a discard to form a Pung? Can they form a Chow? Does the engine correctly prevent illegal claims?
*   **State Transitions**: Does drawing a tile correctly transition the game state? Does declaring a win end the round properly?
*   **Priority Checks**: Test the correct priority of actions (Win > Kong > Pung > Chow). For example, if one player discards a tile that allows another player to Pung and a third player to win, the win must take precedence.

### 3.3. Layer 3: Scenario-Based Testing (End-to-End)

**Goal**: To test the entire game flow from start to finish using predefined game scenarios. These tests will use a "scripted" game where all player actions are predetermined to force the game into specific, testable states.

**Scenario Definition Format**:
Each scenario will be a JSON file defining the initial wall state and a sequence of player actions.

```json
{
  "scenarioName": "Test 'Robbing the Kong' (搶槓)",
  "initialWall": ["W1", "W2", ...], // A specific, non-random wall
  "actions": [
    { "player": 1, "action": "DISCARD", "tile": "E" },
    { "player": 2, "action": "PUNG", "tile": "E" },
    // ... more actions to set up the scenario
    { "player": 2, "action": "ADD_KONG", "tile": "E" }, // Player 2 tries to upgrade Pung to Kong
    { "player": 3, "action": "DECLARE_WIN", "tile": "E" } // Player 3 should be able to win off this tile
  ],
  "expectedOutcome": {
    "winner": 3,
    "winningTile": "E",
    "score": {
      "player3": { "total": 9, "patterns": ["Robbing the Kong", "Common Hand"] }
    }
  }
}
```

**Key Scenarios to Test**:
*   **Standard Win**: A simple, common win.
*   **Self-Drawn Win (`自摸`)**.
*   **All special scoring hands**: "Great Winds", "All Honors", "Thirteen Orphans", etc.
*   **Robbing the Kong (`搶槓`)**.
*   **Last Tile Win (`海底撈月`)**.
*   **Drawn Game (`流局`)**: Test all conditions that lead to a drawn game.
*   **Illegal Win (`詐胡`)**: Test that the system correctly identifies and penalizes an attempt to declare a win with an invalid hand.
*   **"Ready Hand" (`聽牌`) states**.
*   **"Sacred Discard" (`過水`) rules**: Test that a player is correctly prevented from winning on a discard if they previously passed on a valid win.

### 3.4. Layer 4: Manual & Expert Validation

**Goal**: To catch nuances that automated tests might miss and to get a final seal of approval on authenticity.

*   **Internal QA**: A dedicated QA team will play the game manually, following test scripts but also performing exploratory testing.
*   **Expert Review**: We will hire or consult with recognized Taiwan Mahjong masters. They will be given access to the game and asked to play, specifically trying to "break" the rules or find inconsistencies. Their feedback will be treated as high-priority bugs.
*   **Community Beta**: A closed beta with experienced players from the Taiwan Mahjong community will be conducted to gather feedback on rule interpretations and game feel.

---

## 4. Test Execution and Reporting

*   All automated tests (Layers 1-3) will be executed in the CI/CD pipeline on every commit. A failed test will block the build.
*   A dashboard will track test coverage and pass/fail rates over time.
*   Manual and expert validation results will be logged in the project's issue tracking system.
