# AI Component Alternative Analysis

**Date:** 2025-08-09

**Participants:** System Architect, Game Systems Architect

**Issue:** The current project plan includes an AI component for single-player mode, practice/learning features, and potentially other advanced functionalities. However, using a complex AI (like a neural network-based one) introduces significant uncertainty, development complexity, and testing difficulties. This could jeopardize project timelines and budget.

**Objective:** Explore and evaluate a more deterministic alternative to the AI component that still fulfills the core requirements for practice, learning, and core gameplay.

---

## Proposed Alternative: Rule-Based Expert System

Instead of a machine learning-based AI, we propose implementing a **Rule-Based Expert System** to control NPC (non-player character) opponents and provide player guidance.

### How it Works:

A rule-based system operates on a set of predefined rules that encode the logic of a domain expert—in this case, an expert Mahjong player. The system would evaluate the game state (tiles in hand, discarded tiles, other players' moves) and make decisions based on a prioritized list of rules.

**Example Rules:**
*   **Priority 1 (Winning):** If I can declare "Hu" (win), do it.
*   **Priority 2 (Pung/Kang):** If an opponent discards a tile that allows me to form a "Pung" (triplet) or "Kang" (quad), evaluate the strategic value. If it improves my hand significantly, claim the discard.
*   **Priority 3 (Defense):** If an opponent is in a "Ready" state (聽牌), avoid discarding tiles they might need. Prioritize discarding "safe" tiles.
*   **Priority 4 (Hand Improvement):** Discard tiles that are least likely to help form a winning hand (e.g., isolated honor tiles early in the game).

### Difficulty Levels:

This system naturally supports different difficulty levels by adjusting the complexity and depth of the rule set:
*   **Easy:** Uses a basic set of rules, makes obvious moves, doesn't employ complex defensive strategies.
*   **Medium:** Adds more nuanced rules for hand evaluation and basic defensive play.
*   **Hard:** Utilizes a comprehensive rulebook, including advanced strategic concepts, probability analysis (e.g., tracking discarded tiles to infer opponents' hands), and sophisticated defensive maneuvers.

---

## Discussion Points for Architects

### For the System Architect:

1.  **Integration:** How does a rule-based system fit into the overall modular monolith architecture? It could be a self-contained module (`GameLogic.NPCPlayer`).
2.  **Performance:** What are the performance implications? A rule-based engine is generally lightweight and fast, executing within the main game loop without needing separate, resource-intensive processing.
3.  **Scalability:** While this system doesn't "learn" on its own, the rule set can be expanded. How do we design the system to make adding or modifying rules easy without impacting performance? (e.g., loading rules from a configuration file or a database).

### For the Game Systems Architect:

1.  **Gameplay Authenticity:** Can a rule-based system accurately simulate human-like Mahjong play? How can we design the rules to avoid robotic, predictable behavior? (e.g., introducing weighted randomness for decisions with similar strategic value).
2.  **Feature Fulfillment:** How does this approach satisfy the "practice and learning" requirements?
    *   **Practice:** Players can compete against predictable opponents at various skill levels.
    *   **Learning:** The same rule engine can be used to power a "Hint" system, suggesting the optimal move to a human player based on its logic.
3.  **Testability:** A deterministic, rule-based system is highly testable. We can create specific game scenarios and assert that the NPC makes the expected move, which is a major advantage over non-deterministic AI. What would a testing strategy for this module look like?

---

## Next Steps

1.  Discuss the feasibility and trade-offs of this approach.
2.  Define the initial rule set required for an MVP.
3.  Outline the architecture for the rule engine module.

---
## Collaborative Analysis (2025-08-09)

### System Architect Analysis

This approach is highly favorable from a system-level perspective.

1.  **Integration:** A rule-based engine can be implemented as a self-contained, stateless module (e.g., `NpcDecisionEngine`) within our modular monolith. Its interface would be simple: it accepts the current game state (player's hand, discard history, etc.) and returns a specific action (e.g., `DISCARD TileX`, `CALL Pung`). This clean separation minimizes architectural coupling.
2.  **Performance:** The performance overhead is negligible. Unlike a machine-learning model requiring significant computational resources, a rule-based system executes simple conditional logic. We can ensure low latency by setting a hard limit on rule-chain depth to prevent complex, cascading evaluations.
3.  **Scalability & Maintainability:** The key is to externalize the rules. I recommend storing the rule sets in a human-readable format like YAML. This allows game designers to define and tweak NPC difficulty levels and "personalities" without requiring new code deployments, which is a major operational advantage. These rule files can be loaded into memory on application startup.

### Game Systems Architect Analysis

This is a practical and robust solution for the game's core needs.

1.  **Gameplay Authenticity:** A purely deterministic system can feel robotic. We can mitigate this by introducing **weighted probabilities** and **NPC profiles**. For example, a "defensive" profile might have a higher weight on rules for breaking up its own hand to avoid feeding a winning player, while an "aggressive" profile would prioritize completing its own hand at all costs. This adds a layer of unpredictability and makes the NPCs feel more distinct.
2.  **Feature Fulfillment:** This approach is ideal for the required features. The "Hint" system for players can be powered by the very same engine—we simply run the engine on the human player's hand and suggest the highest-priority move. The tiered difficulty levels (Easy, Medium, Hard) are directly implemented by loading different, progressively more complex rule sets.
3.  **Testability:** This is the most significant advantage. We can create a suite of unit tests for specific game scenarios. For example: `given_game_state_A_when_tile_B_is_discarded_then_NPC_should_call_Pung`. This makes development predictable and debugging straightforward, which is nearly impossible with a non-deterministic AI. It dramatically reduces project risk.
