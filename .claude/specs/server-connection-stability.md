---
status: active
priority: high
area: server
---

# Server Connection: Stability and Auto-Reconnect

## Problem

The WebSocket/connection to the backend server drops randomly and doesn't recover. The client loses its data feed and stops showing new strikes. The user has to manually refresh the page.

## Goal

1. **Detect disconnection quickly** -- monitor the connection health with lightweight heartbeats, not just relying on TCP timeout detection which can take minutes.
2. **Reconnect automatically** -- on disconnect, immediately attempt reconnection with exponential backoff.
3. **Make the connection more stable** -- investigate why it drops in the first place. Could be server-side timeout, keep-alive misconfiguration, or error handling gaps.

## Design

- **Heartbeat**: Client sends a ping every 15-30 seconds. If no pong within 5 seconds, consider the connection dead and begin reconnect.
- **Reconnect**: Exponential backoff starting at 1s, max 30s. Reset backoff on successful reconnect.
- **UI indicator**: Show a subtle connection status indicator so the user knows when data is stale.
- **Server side**: Ensure the server handles client disconnects gracefully and doesn't accumulate dead connections.

## Acceptance Criteria

- [ ] Client detects server disconnect within 10 seconds
- [ ] Auto-reconnect initiates immediately after detection
- [ ] Reconnect succeeds and data flow resumes without page refresh
- [ ] Connection survives server restarts (client reconnects)
- [ ] No connection-related memory leaks on either side
- [ ] Status indicator shows connection state to user
