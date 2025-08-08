# Client-Side Network Resiliency Design
## Taiwan Mahjong Online Game Project

**Document Version**: 1.0  
**Date**: 2025-08-14
**Authors**: Sarah Patel (Mobile Architect), David Kim (Frontend Architect)
**Status**: DRAFT

---

## 1. Problem Statement

Mobile and web clients, especially on mobile networks, face unreliable connectivity. The application must gracefully handle network interruptions (disconnections) and seamlessly resume the game state upon reconnection without any data loss or inconsistency. A player's turn should not be lost due to a brief network drop.

This design addresses the client-side mechanisms for achieving this resiliency.

---

## 2. Core Principles

1.  **Server is the Source of Truth**: The client never assumes its state is correct after a reconnection. It always synchronizes with the server.
2.  **Idempotent Commands**: All client-to-server actions (commands) must be idempotent. Sending the same command multiple times should not result in duplicate actions.
3.  **Stateful Connection Protocol**: The WebSocket connection will be enhanced with a simple protocol to manage connection state.
4.  **Immediate Feedback**: The UI must immediately inform the user about their connection status.

---

## 3. Technical Design

The solution involves a combination of client-side state management, a command queue, and a WebSocket heartbeat and handshake protocol.

### 3.1. Connection State Machine

The client will maintain a simple connection state machine:

*   `CONNECTING`: Initial state when the WebSocket is attempting to connect.
*   `CONNECTED`: Connection is active and healthy.
*   `DISCONNECTED`: Connection is lost. The client is aware and will attempt to reconnect.
*   `RECONNECTING`: The client is actively trying to re-establish the connection.
*   `RESYNCING`: The connection is re-established, and the client is waiting for the full game state from the server.

The UI will display clear indicators for `DISCONNECTED`, `RECONNECTING`, and `RESYNCING` states (e.g., a banner or icon).

### 3.2. Heartbeat Mechanism

*   The client will send a `ping` message to the server every **5 seconds**.
*   The server must respond with a `pong` message within **2.5 seconds**.
*   If the client does not receive a `pong` within the timeout, it considers the connection lost and transitions to the `DISCONNECTED` state.

### 3.3. Command Queue & Idempotency

To prevent losing player actions during a network drop, we will implement a client-side command queue.

1.  When a player performs an action (e.g., plays a tile), the command is **not** sent directly.
2.  Instead, it is added to a **persistent queue** in the client's state (e.g., Redux store, persisted to local storage).
3.  Each command is assigned a unique, client-generated UUID (`command_id`).
4.  A separate process sends commands from the queue to the server one by one.
5.  The server, upon receiving a command, includes the `command_id` in its acknowledgment message.
6.  The client only removes the command from the queue upon receiving this specific acknowledgment.

**Idempotency Handling**: The server will track the `command_id`s it has processed for the current game. If it receives a command with a UUID it has already processed, it will not re-execute the action but will re-send the acknowledgment. This prevents duplicate actions if the client re-sends a command after a disconnect.

```typescript
// Client-side pseudocode
function onPlayTile(tile: Tile) {
  const command = {
    command_id: uuidv4(),
    type: 'DAPAI',
    payload: { tile }
  };
  
  dispatch(enqueueCommand(command));
}

// Server-side pseudocode
async function handleCommand(command: ClientCommand) {
  const hasBeenProcessed = await redis.exists(`processed_command:${command.command_id}`);
  if (hasBeenProcessed) {
    // Re-send acknowledgment, but don't re-process
    sendAck(command.command_id);
    return;
  }

  // Process the command...
  await gameService.executeDapai(command.payload.tile);
  
  // Mark as processed and acknowledge
  await redis.set(`processed_command:${command.command_id}`, 1, { EX: 3600 });
  sendAck(command.command_id);
}
```

### 3.4. Reconnection and Resynchronization Flow

1.  When the client enters the `DISCONNECTED` state, it immediately starts a reconnection attempt.
2.  It will use an **exponential backoff** strategy (e.g., retry after 1s, 2s, 4s, 8s...) to avoid overwhelming the server.
3.  Once the WebSocket connection is re-established, the client sends a `RESYNC` message to the server, including its `session_id` and the `game_id`.
4.  The server receives the `RESYNC` message and performs the following:
    a.  Validates the session.
    b.  Sends the **entire current game state** to the client.
    c.  Sends the ID of the **last command it successfully processed** from that client.
5.  The client receives the full game state and overwrites its local version, ensuring consistency.
6.  The client inspects its command queue and removes any commands that the server has already acknowledged.
7.  The client resumes sending any remaining commands from its queue.
8.  Once the state is synced and the queue is clear, the client transitions to the `CONNECTED` state.

---

## 4. UI/UX Considerations

*   **Modal Overlay**: During `DISCONNECTED` and `RECONNECTING` states, a non-intrusive overlay will appear, informing the user of the status and preventing further game actions.
*   **Turn Timer**: The server-side turn timer for the player will be paused for a grace period (e.g., 30 seconds) upon detecting a disconnect, to give the player a chance to reconnect without losing their turn. If the player fails to reconnect within this period, the server may automatically discard a tile for them to keep the game moving.
*   **Optimistic Updates**: For actions with low probability of failure (like playing a valid tile), the UI can show an "optimistic" update immediately, while the command is in the queue. This update should be visually distinct (e.g., slightly transparent) until server confirmation is received.
