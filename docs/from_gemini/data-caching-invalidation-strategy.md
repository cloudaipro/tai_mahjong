# Data Caching & Invalidation Strategy
## Taiwan Mahjong Online Game Project

**Document Version**: 1.0  
**Date**: 2025-08-14
**Author**: James Thompson (Data Architect)
**Status**: DRAFT

---

## 1. Overview

This document details the data caching and invalidation strategy for the Taiwan Mahjong Online Game. The primary goal is to meet the stringent performance requirement of <70ms response time for all critical game actions. A multi-level caching architecture will be employed to minimize database load and reduce latency.

The core of our strategy is to keep active game state in a high-speed in-memory cache (Redis), while using the PostgreSQL database as the persistent source of truth.

---

## 2. Caching Architecture

We will implement a two-level caching system (L1/L2) in front of our L3 persistent database.

*   **L1 Cache (Application Memory - Future Optimization)**: Not in initial scope. Could be added later for ultra-low latency on frequently accessed, read-only data (e.g., game rules).
*   **L2 Cache (Distributed Cache - Redis)**: The primary cache for all active game state, user sessions, and frequently accessed read data.
*   **L3 Storage (Persistent Database - PostgreSQL)**: The ultimate source of truth for all data.

### Redis Data Structures

We will leverage Redis's rich data structures to efficiently store game data:

*   **Active Game State**: `Redis Hashes`. A single hash per game (`game:{game_id}`) will store the entire game state object. This allows for atomic updates of the game state and efficient retrieval of the full object.
    *   `Key`: `game:c1b2a3d4-e5f6-7890-1234-567890abcdef`
    *   `Value`: A JSON-serialized string of the `Game` aggregate.
*   **Player Sessions**: `Redis Hashes`.
    *   `Key`: `session:{session_id}`
    *   `Value`: `userId`, `gameId`, `lastSeen`, etc.
*   **Leaderboards**: `Redis Sorted Sets`.
    *   `Key`: `leaderboard:monthly:elo`
    *   `Score`: Player's ELO rating
    *   `Value`: `player_id`
*   **Game Presence**: `Redis Sets`. To track which players are in which game room.
    *   `Key`: `room:{game_id}:players`
    *   `Value`: A set of `player_id`s.

---

## 3. Cache Read/Write Pattern: Cache-Aside

We will use the **Cache-Aside** pattern for reading data. This pattern promotes resilience, as the application can still function (albeit slower) if the cache fails.

### Read Flow:

1.  The application requests data (e.g., a game state).
2.  It first attempts to retrieve the data from the **L2 Cache (Redis)**.
3.  **Cache Hit**: If the data is in Redis, it is returned directly to the application.
4.  **Cache Miss**: If the data is not in Redis, the application retrieves it from the **L3 Storage (PostgreSQL)**.
5.  The application then **populates the L2 Cache (Redis)** with the data retrieved from the database.
6.  The data is returned to the application.

```typescript
// Pseudocode for Cache-Aside Read
async function getGame(gameId: string): Promise<Game> {
  const cachedGame = await redis.get(`game:${gameId}`);
  if (cachedGame) {
    return JSON.parse(cachedGame); // Cache Hit
  }

  // Cache Miss
  const dbGame = await postgres.query('SELECT * FROM games WHERE game_id = $1', [gameId]);
  if (dbGame) {
    // Populate cache before returning
    await redis.set(`game:${gameId}`, JSON.stringify(dbGame), { EX: 3600 }); // Set TTL (e.g., 1 hour)
    return dbGame;
  }
  
  return null;
}
```

---

## 4. Cache Invalidation Strategy

Proper cache invalidation is the most critical part of this strategy. Incorrect invalidation leads to stale data and inconsistent game states. We will use a **Write-Through**-like approach for invalidation.

### Write/Update Flow:

For active games, writes are the most important operation. A player making a move must result in an immediate and consistent state change for all players.

1.  A command to modify the game state is received by the application (e.g., `executeDapai` - play a tile).
2.  The application service loads the `Game` aggregate from the **L2 Cache (Redis)**.
3.  The command is executed against the domain model in memory, modifying the game state.
4.  The application then does the following **in a single transaction or atomic operation**:
    a.  Persists the new game state to the **L3 Database (PostgreSQL)**.
    b.  Writes the updated game state back to the **L2 Cache (Redis)**.
    c.  (If using event sourcing) Persists the command to the `game_commands` table in PostgreSQL.
5.  Only after the database and cache are successfully updated is the success response sent back to the player.

This ensures that the cache is always consistent with the database. If the cache write fails, we can either retry or evict the cache key, forcing a cache miss on the next read.

### Cache Eviction Policies:

*   **Time-To-Live (TTL)**: All cache keys will have a TTL. For active games, this might be 1-2 hours. For user sessions, 24 hours. This acts as a fallback to prevent stale data from living forever in the cache.
*   **Least Recently Used (LRU)**: Redis will be configured with an LRU eviction policy. If the cache runs out of memory, it will automatically evict the least recently used keys. This is a standard safety measure.
*   **Explicit Invalidation**: When a game ends, the application will explicitly delete the active game state key (`game:{game_id}`) from Redis to free up memory.

---

## 5. Handling Cache Failures

*   **Redis Unavailability**: If the Redis cluster is down, the application will fall back to reading directly from PostgreSQL. Performance will be degraded, but the system will remain available. Write operations may fail or be queued depending on the desired behavior in this scenario. An alert will be triggered immediately.
*   **Write Failures**: If a write to Redis fails after a successful database write, the system will evict the corresponding cache key. This ensures that the next read will be a cache miss, fetching the correct, updated data from PostgreSQL and repopulating the cache.
