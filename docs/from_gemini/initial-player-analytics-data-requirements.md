# Initial Player Analytics: Data Requirements
## Taiwan Mahjong Online Game Project

**Document Version**: 1.0  
**Date**: 2025-08-21
**Authors**: James Thompson (Data Architect), Lisa Wang (Game Systems Architect)
**Status**: DRAFT

---

## 1. Introduction

This document defines the initial set of data points and the corresponding schema required for player analytics. The goal is to gather essential data from day one to understand player behavior, balance the game, and inform future feature development.

The approach is to start with a focused set of key metrics and expand over time. The data will be collected via an event-driven pipeline, separate from the main game's transactional database, to avoid impacting performance.

---

## 2. Goals of Initial Analytics

*   **Player Engagement**: How often and for how long do people play?
*   **Game Balance**: Are any scoring patterns or rules being exploited? Is the game fair?
*   **Player Progression**: How quickly do new players learn? What are the common sticking points?
*   **Feature Usage**: Which game modes and features are most popular?

---

## 3. Analytics Event-Driven Architecture

1.  The game server's application logic will emit analytics events (e.g., `GameFinished`, `PlayerRegistered`).
2.  These events will be published to a dedicated, lightweight message queue (e.g., AWS SQS, or a simple in-memory queue to start).
3.  A separate "Analytics Collector" service will consume events from this queue.
4.  The collector service will then transform and load this data into a dedicated analytics data store (e.g., a separate PostgreSQL database or a data warehouse like BigQuery/Redshift).

This decouples analytics from the core game loop, ensuring game performance is never affected by analytics load.

---

## 4. Key Events and Data Points

The following events will be captured.

### Event: `PlayerRegistered`
*   `player_id`: UUID
*   `registration_timestamp`: ISO 8601
*   `country_code`: String (derived from IP)

### Event: `GameStarted`
*   `game_id`: UUID
*   `start_timestamp`: ISO 8601
*   `game_mode`: String (e.g., "Ranked", "Casual", "AI_Practice")
*   `player_ids`: Array[UUID]

### Event: `GameFinished`
*   `game_id`: UUID
*   `end_timestamp`: ISO 8601
*   `duration_seconds`: Integer
*   `winning_player_id`: UUID (or null if a draw)
*   `winning_tile`: String
*   `is_self_drawn`: Boolean
*   `final_scores`: Object (map of `player_id` to their final score change)
*   `winning_hand_patterns`: Array[String] (list of scoring patterns, e.g., ["All Pungs", "Mixed Suit"])

### Event: `TurnCompleted`
*   `game_id`: UUID
*   `player_id`: UUID
*   `turn_number`: Integer
*   `duration_seconds`: Integer
*   `action_taken`: String (e.g., "DISCARD", "PUNG", "KONG")

### Event: `PlayerDisconnected`
*   `game_id`: UUID
*   `player_id`: UUID
*   `disconnect_timestamp`: ISO 8601
*   `turn_number_at_disconnect`: Integer

---

## 5. Initial Analytics Database Schema

This schema will reside in a separate database from the main transactional game database.

**Table: `dim_players`** (Dimension table for players)
```sql
CREATE TABLE dim_players (
    player_id UUID PRIMARY KEY,
    registration_date DATE,
    country_code VARCHAR(2)
);
```

**Table: `fact_games`** (Fact table for completed games)
```sql
CREATE TABLE fact_games (
    game_id UUID PRIMARY KEY,
    start_time TIMESTAMP WITH TIME ZONE,
    end_time TIMESTAMP WITH TIME ZONE,
    duration_seconds INTEGER,
    game_mode VARCHAR(20),
    winning_player_id UUID REFERENCES dim_players(player_id),
    is_self_drawn BOOLEAN,
    winning_hand_patterns TEXT[] -- Array of strings
);
```

**Table: `fact_game_players`** (Bridge table for players in games)
```sql
CREATE TABLE fact_game_players (
    game_id UUID REFERENCES fact_games(game_id),
    player_id UUID REFERENCES dim_players(player_id),
    score_change INTEGER,
    PRIMARY KEY (game_id, player_id)
);
```

**Table: `fact_turns`** (Fact table for individual turns, can be aggregated)
```sql
CREATE TABLE fact_turns (
    turn_id SERIAL PRIMARY KEY,
    game_id UUID REFERENCES fact_games(game_id),
    player_id UUID REFERENCES dim_players(player_id),
    turn_number INTEGER,
    duration_seconds INTEGER,
    action_taken VARCHAR(20)
);
```

---

## 6. Key Performance Indicators (KPIs) to Track

Using the schema above, we can answer our initial questions:

*   **Daily Active Users (DAU)**: `COUNT(DISTINCT player_id) FROM fact_game_players WHERE game_start_time >= NOW() - '1 day'::interval`
*   **Average Game Duration**: `AVG(duration_seconds) FROM fact_games`
*   **Most Common Winning Patterns**: `SELECT unnest(winning_hand_patterns) as pattern, COUNT(*) FROM fact_games GROUP BY pattern ORDER BY COUNT(*) DESC`
*   **Player Retention Rate**: (Requires more complex cohort analysis)
*   **Average Turn Time**: `AVG(duration_seconds) FROM fact_turns`

This initial setup provides a strong foundation for our analytics capabilities without over-engineering the solution at this early stage.
