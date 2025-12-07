# Real-time Patterns

## Quick Decision Matrix

| If you need... | Use | Avoid |
|----------------|-----|-------|
| Bi-directional, low latency (chat, games) | WebSocket | HTTP polling |
| Server-to-client updates only | SSE | WebSocket (overkill) |
| iOS background notifications | APNs | WebSocket (won't work in background) |
| Occasional updates, simple setup | HTTP polling | Over-engineering |
| Scalable pub/sub | Redis Pub/Sub + WebSocket | Direct WebSocket |

---

## WebSocket

| Aspect | Details |
|--------|---------|
| **Best for** | Chat, live collaboration, gaming, real-time dashboards |
| **Pros** | Bi-directional, low latency, persistent connection, full-duplex |
| **Cons** | Connection management complexity, stateful, scaling requires infrastructure |
| **When to use** | Live chat, multiplayer features, collaborative editing, live feeds |
| **When to avoid** | Simple notifications (use APNs), server-only updates (use SSE) |
| **Works well with** | Socket.io, ws (Node), Redis Pub/Sub for scaling |
| **iOS considerations** | URLSessionWebSocketTask (native), Starscream library, disconnects when app backgrounds |

### WebSocket Architecture
```
                    ┌─────────────┐
                    │   Redis     │
                    │   Pub/Sub   │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   WS Server 1 │  │   WS Server 2 │  │   WS Server 3 │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
    Clients            Clients            Clients
```

### WebSocket Best Practices
- Implement heartbeat/ping-pong for connection health
- Handle reconnection with exponential backoff
- Use Redis Pub/Sub for multi-server scaling
- Authenticate on connection (token in query or first message)
- Room/channel pattern for group messaging

---

## Server-Sent Events (SSE)

| Aspect | Details |
|--------|---------|
| **Best for** | Live feeds, notifications, stock tickers, server-to-client only |
| **Pros** | Simple, HTTP-based, auto-reconnect built-in, works through proxies |
| **Cons** | Uni-directional (server to client only), limited browser connections |
| **When to use** | News feeds, live scores, notification streams, dashboard updates |
| **When to avoid** | Bi-directional needs (chat), iOS background (use APNs) |
| **Works well with** | Standard HTTP servers, EventSource API |
| **iOS considerations** | No native support, need third-party library, limited use case |

### SSE vs WebSocket

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| Direction | Server → Client | Bi-directional |
| Protocol | HTTP | WS (upgrade from HTTP) |
| Reconnect | Automatic | Manual implementation |
| Browser support | Good | Excellent |
| iOS native | No | Yes (iOS 13+) |

---

## Apple Push Notifications (APNs)

| Aspect | Details |
|--------|---------|
| **Best for** | iOS alerts, background app refresh, silent updates |
| **Pros** | Works when app closed, battery efficient, native iOS integration |
| **Cons** | Not guaranteed delivery, requires Apple setup, payload size limits |
| **When to use** | User alerts, background data sync, wake-up triggers |
| **When to avoid** | Real-time chat (use WebSocket), time-critical guaranteed delivery |
| **Works well with** | WebSocket (hybrid approach), background refresh |
| **iOS considerations** | Required for any notification when app not active |

### APNs Notification Types

| Type | Use Case | User Sees | App State |
|------|----------|-----------|-----------|
| Alert | User-facing notifications | Yes | Any |
| Silent | Background data refresh | No | Background |
| Background | Content update trigger | Optional | Background |

### APNs Backend Setup
```
1. Apple Developer account → Create APNs key (.p8)
2. Backend stores → Key ID, Team ID, .p8 file
3. Send notification → POST to api.push.apple.com
4. Use HTTP/2 for connection efficiency
5. Handle feedback for invalid tokens
```

### APNs Payload Structure
```json
{
  "aps": {
    "alert": {
      "title": "New Message",
      "body": "John: Hey, are you there?"
    },
    "badge": 5,
    "sound": "default",
    "content-available": 1
  },
  "custom_data": {
    "conversation_id": "123"
  }
}
```

---

## HTTP Polling

| Aspect | Details |
|--------|---------|
| **Best for** | Simple updates, low-frequency data, fallback |
| **Pros** | Simple to implement, stateless, works everywhere |
| **Cons** | Latency, wasted requests, server load, battery drain |
| **When to use** | Low-frequency updates, fallback when WebSocket unavailable |
| **When to avoid** | Real-time requirements, mobile (battery), high frequency |
| **Works well with** | Any HTTP server |
| **iOS considerations** | Avoid for real-time, use for occasional sync only |

### Polling Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| Short polling | Fixed interval requests | Simple, low-frequency |
| Long polling | Server holds until data or timeout | Near real-time without WebSocket |
| Exponential backoff | Increasing intervals on no change | Efficient polling |

---

## Hybrid Real-time Architecture (Recommended for iOS)

Best approach combines multiple patterns:

```
┌─────────────────────────────────────────────────┐
│                    iOS App                       │
├─────────────────────────────────────────────────┤
│  Active (Foreground)  │  Background/Closed      │
│  ───────────────────  │  ──────────────────     │
│  WebSocket            │  APNs                   │
│  - Chat messages      │  - New message alerts   │
│  - Live updates       │  - Silent refresh       │
│  - Presence           │  - Badge updates        │
└─────────────────────────────────────────────────┘
```

### Hybrid Flow
1. App foreground → Connect WebSocket for live updates
2. App backgrounds → Disconnect WebSocket, rely on APNs
3. APNs silent push → Trigger background fetch
4. App opens → Reconnect WebSocket, sync any missed data

---

## Scaling Real-time

| Scale | Solution |
|-------|----------|
| Single server | Direct WebSocket |
| Multiple servers | Redis Pub/Sub + WebSocket |
| Large scale | Dedicated service (Pusher, Ably) or custom with Redis Cluster |

### Redis Pub/Sub Pattern
```
1. Client A sends message → WS Server 1
2. Server 1 publishes to Redis channel
3. All WS servers subscribed receive message
4. Each server broadcasts to connected clients
```

---

## Real-time Checklist

- [ ] Determine update frequency and direction
- [ ] Choose pattern (WebSocket, SSE, APNs, polling)
- [ ] Plan for iOS background state (APNs required)
- [ ] Implement reconnection with backoff
- [ ] Set up Redis Pub/Sub if multiple servers
- [ ] Handle authentication on connection
- [ ] Plan offline message queue/sync
- [ ] Configure APNs for push notifications
- [ ] Test battery impact on iOS
