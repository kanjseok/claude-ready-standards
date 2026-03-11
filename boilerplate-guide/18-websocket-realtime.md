# 18. WebSocket & Real-Time Communication

> Cross-cutting WebSocket and real-time communication guide covering NestJS, FastAPI, and Spring Boot backends
> with React/Next.js and Flutter desktop clients.
> See alongside [03-backend-nestjs](./03-backend-nestjs.md), [14-backend-fastapi](./14-backend-fastapi.md),
> [15-backend-kotlin](./15-backend-kotlin.md), [06-auth-security](./06-auth-security.md),
> [08-flutter-desktop](./08-flutter-desktop.md), and [17-consumer-web-pwa](./17-consumer-web-pwa.md).

---

## 1. Architecture Overview

```text
┌───────────────┐   WebSocket    ┌──────────────────┐    Redis Pub/Sub    ┌──────────────────┐
│  Web Client   │ ◀────────────▶ │  Backend Inst.1  │ ◀────────────────▶ │  Backend Inst.2  │
│  (Socket.IO)  │                │  (WS Gateway)    │                    │  (WS Gateway)    │
└───────────────┘                └────────┬─────────┘                    └────────┬─────────┘
                                          │                                       │
┌───────────────┐                         │                                       │
│ Flutter Client│ ◀───── WebSocket ───────┘                                       │
│ (ws_channel)  │                                                                 │
└───────────────┘                    ┌────▼─────┐                                 │
                                     │  Redis   │ ◀───────────────────────────────┘
                                     │  7.4.x   │
                                     └──────────┘
```

- **Web Clients** connect via Socket.IO (NestJS) or native WebSocket (FastAPI/Spring Boot)
- **Flutter Desktop** connects via native WebSocket (all backends expose a raw WS endpoint)
- **Redis Pub/Sub** synchronizes events across multiple backend instances for horizontal scaling
- WebSocket runs on the **same port** as the HTTP server — no separate port allocation needed

### Protocol Comparison

| Protocol    | Transport             | Backend          | Use Case                          |
| ----------- | --------------------- | ---------------- | --------------------------------- |
| Socket.IO   | WS + HTTP long-poll   | NestJS           | Rich features (rooms, ack, retry) |
| Native WS   | Pure WebSocket        | FastAPI          | Lightweight, async-native         |
| STOMP / WS  | STOMP over WebSocket  | Spring Boot      | Enterprise messaging patterns     |

### WebSocket Port Strategy

WebSocket connections share the existing HTTP server ports. No additional port allocation is required:

| Backend        | HTTP Port        | WebSocket Path        |
| -------------- | ---------------- | --------------------- |
| NestJS         | `BASE_PORT + 10` | `/` (Socket.IO)       |
| FastAPI        | `BASE_PORT + 10` | `/ws`                 |
| Spring Boot    | `BASE_PORT + 50` | `/ws` (STOMP)         |

---

## 2. Event Contract & Shared Types

All WebSocket events share a common contract defined in the `@scope/shared` package. This is the single source of truth for event names and payload schemas.

### Event Names

```typescript
// packages/shared/src/ws/events.ts
export const WS_EVENTS = {
  // Chat
  CHAT_MESSAGE: "chat:message",
  CHAT_TYPING: "chat:typing",
  CHAT_JOIN_ROOM: "chat:join-room",
  CHAT_LEAVE_ROOM: "chat:leave-room",

  // Notifications
  NOTIFICATION_NEW: "notification:new",
  NOTIFICATION_READ: "notification:read",
  NOTIFICATION_BADGE: "notification:badge",

  // Live Data
  LIVE_DATA_UPDATE: "live:data-update",
  LIVE_CURSOR: "live:cursor",
  LIVE_PRESENCE: "live:presence",

  // System
  ERROR: "error",
  PING: "ping",
  PONG: "pong",
} as const;

export type WsEvent = (typeof WS_EVENTS)[keyof typeof WS_EVENTS];
```

### Message Envelope

```typescript
// packages/shared/src/ws/types.ts
import { z } from "zod";

/** Base envelope wrapping all WebSocket messages */
export const wsMessageSchema = z.object({
  event: z.string(),
  data: z.unknown(),
  timestamp: z.string().datetime(),
  requestId: z.string().uuid().optional(),
});

export type WsMessage = z.infer<typeof wsMessageSchema>;

/** Chat message payload */
export const chatMessageSchema = z.object({
  roomId: z.string(),
  content: z.string().min(1).max(5000),
  senderId: z.string(),
  senderName: z.string(),
});

export type ChatMessage = z.infer<typeof chatMessageSchema>;

/** Typing indicator payload */
export const typingSchema = z.object({
  roomId: z.string(),
  userId: z.string(),
  isTyping: z.boolean(),
});

export type TypingIndicator = z.infer<typeof typingSchema>;

/** Notification payload */
export const notificationSchema = z.object({
  id: z.string(),
  type: z.enum(["info", "warning", "success", "error"]),
  title: z.string(),
  body: z.string().optional(),
  actionUrl: z.string().optional(),
});

export type Notification = z.infer<typeof notificationSchema>;

/** Live data update payload */
export const liveDataUpdateSchema = z.object({
  resource: z.string(),
  action: z.enum(["create", "update", "delete"]),
  payload: z.unknown(),
});

export type LiveDataUpdate = z.infer<typeof liveDataUpdateSchema>;

/** Error payload */
export const wsErrorSchema = z.object({
  code: z.string(),
  message: z.string(),
  details: z.unknown().optional(),
});

export type WsError = z.infer<typeof wsErrorSchema>;
```

### Barrel Export

```typescript
// packages/shared/src/ws/index.ts
export * from "./events";
export * from "./types";
```

```typescript
// packages/shared/src/index.ts  (add to existing exports)
export * from "./ws";
```

---

## 3. NestJS Gateway (Socket.IO)

### Additional Dependencies

Add to the existing `package.json` from [03-backend-nestjs](./03-backend-nestjs.md):

```json
{
  "dependencies": {
    "@nestjs/websockets": "^11.0.5",
    "@nestjs/platform-socket.io": "^11.0.5",
    "@socket.io/redis-adapter": "^8.3.0",
    "socket.io": "^4.8.1"
  }
}
```

### Directory Structure Addition

```text
apps/backend/src/
├── gateway/
│   ├── gateway.module.ts            # WebSocket module
│   ├── events.gateway.ts            # Main WebSocket gateway
│   ├── ws-auth.middleware.ts         # JWT handshake middleware
│   └── ws-exception.filter.ts       # WebSocket exception filter
├── modules/
│   ├── chat/
│   │   ├── chat.gateway.ts          # Chat-specific gateway
│   │   └── chat.service.ts
│   └── notifications/
│       ├── notifications.gateway.ts # Notification-specific gateway
│       └── notifications.service.ts
```

### Main Gateway

```typescript
// src/gateway/events.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from "@nestjs/websockets";
import { Logger, UseFilters, UsePipes, ValidationPipe } from "@nestjs/common";
import { Server, Socket } from "socket.io";
import { WS_EVENTS } from "@scope/shared";
import { WsExceptionFilter } from "./ws-exception.filter";

@WebSocketGateway({
  cors: {
    origin: process.env.FRONTEND_URL ?? "http://localhost:15310",
    credentials: true,
  },
  transports: ["websocket", "polling"],
})
@UseFilters(new WsExceptionFilter())
@UsePipes(new ValidationPipe({ whitelist: true, transform: true }))
export class EventsGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server!: Server;

  private readonly logger = new Logger(EventsGateway.name);

  handleConnection(client: Socket) {
    const userId = client.data.userId as string;
    this.logger.log(`Client connected: ${client.id} (user: ${userId})`);

    // Join user-specific room for targeted notifications
    client.join(`user:${userId}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
  }

  @SubscribeMessage(WS_EVENTS.PING)
  handlePing(@ConnectedSocket() client: Socket): void {
    client.emit(WS_EVENTS.PONG, { timestamp: new Date().toISOString() });
  }

  /** Send a notification to a specific user across all their connections */
  sendToUser(userId: string, event: string, data: unknown): void {
    this.server.to(`user:${userId}`).emit(event, data);
  }

  /** Broadcast to all connected clients */
  broadcast(event: string, data: unknown): void {
    this.server.emit(event, data);
  }
}
```

### WebSocket Exception Filter

```typescript
// src/gateway/ws-exception.filter.ts
import { Catch, ArgumentsHost } from "@nestjs/common";
import { BaseWsExceptionFilter, WsException } from "@nestjs/websockets";
import { Socket } from "socket.io";
import { WS_EVENTS } from "@scope/shared";

@Catch()
export class WsExceptionFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const client = host.switchToWs().getClient<Socket>();

    const error =
      exception instanceof WsException
        ? { code: "WS_ERROR", message: exception.message }
        : { code: "INTERNAL_ERROR", message: "Internal server error" };

    client.emit(WS_EVENTS.ERROR, {
      ...error,
      timestamp: new Date().toISOString(),
    });
  }
}
```

### Gateway Module

```typescript
// src/gateway/gateway.module.ts
import { Module } from "@nestjs/common";
import { EventsGateway } from "./events.gateway";
import { WsAuthMiddleware } from "./ws-auth.middleware";
import { AuthModule } from "../modules/auth/auth.module";

@Module({
  imports: [AuthModule],
  providers: [EventsGateway, WsAuthMiddleware],
  exports: [EventsGateway],
})
export class GatewayModule {}
```

### AppModule Integration

Add `GatewayModule` to the existing `AppModule` from [03-backend-nestjs](./03-backend-nestjs.md):

```typescript
// src/app.module.ts — add to imports array
import { GatewayModule } from "./gateway/gateway.module";

@Module({
  imports: [
    // ... existing imports (ConfigModule, ThrottlerModule, PrismaModule, etc.)
    GatewayModule,
  ],
})
export class AppModule {}
```

---

## 4. FastAPI WebSocket Endpoints

### Additional Dependencies

Add to the existing `pyproject.toml` from [14-backend-fastapi](./14-backend-fastapi.md):

```toml
[project]
dependencies = [
    # ... existing dependencies
    "websockets>=14.2",
]
```

> **Note**: FastAPI includes native WebSocket support. The `websockets` package provides the underlying ASGI WebSocket implementation.

### Directory Structure Addition

```text
apps/backend-py/app/
├── ws/
│   ├── __init__.py
│   ├── manager.py                 # Connection lifecycle manager
│   ├── router.py                  # WebSocket endpoint
│   └── auth.py                    # WebSocket token validation
├── modules/
│   ├── chat/
│   │   └── ws_handler.py          # Chat message handler
│   └── notifications/
│       └── ws_handler.py          # Notification handler
```

### Connection Manager

```python
# app/ws/manager.py
from __future__ import annotations

import asyncio
import json
import logging
from dataclasses import dataclass, field
from fastapi import WebSocket

logger = logging.getLogger(__name__)


@dataclass
class ConnectedClient:
    websocket: WebSocket
    user_id: str
    rooms: set[str] = field(default_factory=set)


class ConnectionManager:
    def __init__(self) -> None:
        self._connections: dict[str, list[ConnectedClient]] = {}
        self._lock = asyncio.Lock()

    async def connect(self, websocket: WebSocket, user_id: str) -> ConnectedClient:
        await websocket.accept()
        client = ConnectedClient(websocket=websocket, user_id=user_id)
        async with self._lock:
            self._connections.setdefault(user_id, []).append(client)
        logger.info("Client connected: user=%s", user_id)
        return client

    async def disconnect(self, client: ConnectedClient) -> None:
        async with self._lock:
            clients = self._connections.get(client.user_id, [])
            if client in clients:
                clients.remove(client)
            if not clients:
                self._connections.pop(client.user_id, None)
        logger.info("Client disconnected: user=%s", client.user_id)

    async def send_to_user(self, user_id: str, event: str, data: dict) -> None:
        message = json.dumps({"event": event, "data": data})
        for client in self._connections.get(user_id, []):
            try:
                await client.websocket.send_text(message)
            except Exception:
                logger.warning("Failed to send to user=%s", user_id)

    async def broadcast(self, event: str, data: dict) -> None:
        message = json.dumps({"event": event, "data": data})
        for clients in self._connections.values():
            for client in clients:
                try:
                    await client.websocket.send_text(message)
                except Exception:
                    pass

    async def send_to_room(self, room: str, event: str, data: dict) -> None:
        message = json.dumps({"event": event, "data": data})
        for clients in self._connections.values():
            for client in clients:
                if room in client.rooms:
                    try:
                        await client.websocket.send_text(message)
                    except Exception:
                        pass


manager = ConnectionManager()
```

### WebSocket Endpoint

```python
# app/ws/router.py
import json
import logging
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

from app.ws.auth import verify_ws_token
from app.ws.manager import manager

logger = logging.getLogger(__name__)
router = APIRouter()


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket, token: str | None = None):
    # Authenticate via query parameter: ws://host/ws?token=<jwt>
    user = await verify_ws_token(token)
    if user is None:
        await websocket.close(code=4001, reason="Unauthorized")
        return

    client = await manager.connect(websocket, user.id)
    try:
        while True:
            raw = await websocket.receive_text()
            try:
                message = json.loads(raw)
            except json.JSONDecodeError:
                await websocket.send_text(
                    json.dumps({"event": "error", "data": {"message": "Invalid JSON"}})
                )
                continue

            event = message.get("event", "")
            data = message.get("data", {})

            # Route events to handlers
            if event == "chat:message":
                await handle_chat_message(client, data)
            elif event == "chat:join-room":
                room_id = data.get("roomId", "")
                client.rooms.add(room_id)
            elif event == "chat:leave-room":
                room_id = data.get("roomId", "")
                client.rooms.discard(room_id)
            elif event == "ping":
                await websocket.send_text(
                    json.dumps({"event": "pong", "data": {"timestamp": "..."}})
                )
    except WebSocketDisconnect:
        await manager.disconnect(client)


async def handle_chat_message(client, data: dict) -> None:
    room_id = data.get("roomId", "")
    if room_id not in client.rooms:
        return
    await manager.send_to_room(room_id, "chat:message", {
        **data,
        "senderId": client.user_id,
    })
```

### Register WebSocket Router

Add to the existing `main.py` from [14-backend-fastapi](./14-backend-fastapi.md):

```python
# app/main.py — add after existing routers
from app.ws.router import router as ws_router

app.include_router(ws_router)
```

### Concept Comparison: FastAPI vs NestJS

| Concept              | NestJS (Socket.IO)                   | FastAPI (Native WS)                       |
| -------------------- | ------------------------------------ | ----------------------------------------- |
| Gateway / Endpoint   | `@WebSocketGateway()`                | `@router.websocket("/ws")`                |
| Event handling       | `@SubscribeMessage("event")`         | Manual `if event == "..."` dispatch       |
| Connection lifecycle | `OnGatewayConnection` interface      | `manager.connect()` / `manager.disconnect()` |
| Room support         | Built-in `client.join("room")`       | Manual `client.rooms.add("room")`         |
| Broadcasting         | `server.emit()` / `server.to()`      | `manager.broadcast()` / `manager.send_to_room()` |
| Error handling       | `WsExceptionFilter`                  | try/except with JSON error response       |

---

## 5. Spring Boot STOMP WebSocket

### Additional Dependencies

Add to the existing `build.gradle.kts` from [15-backend-kotlin](./15-backend-kotlin.md):

```kotlin
dependencies {
    // ... existing dependencies

    // WebSocket + STOMP
    implementation("org.springframework.boot:spring-boot-starter-websocket")
}
```

> **Note**: Spring Boot BOM manages the version. No explicit version needed.

### Directory Structure Addition

```text
apps/backend-kt/src/main/kotlin/com/example/backend/
├── common/
│   └── config/
│       └── WebSocketConfig.kt       # STOMP broker configuration
├── modules/
│   ├── chat/
│   │   ├── ChatController.kt        # STOMP message controller
│   │   └── ChatService.kt
│   └── notifications/
│       ├── NotificationController.kt
│       └── NotificationService.kt
├── ws/
│   ├── StompAuthInterceptor.kt      # CONNECT frame auth
│   └── WsEventListener.kt           # Connection lifecycle events
```

### WebSocket Configuration

```kotlin
// src/main/kotlin/com/example/backend/common/config/WebSocketConfig.kt
package com.example.backend.common.config

import com.example.backend.ws.StompAuthInterceptor
import org.springframework.context.annotation.Configuration
import org.springframework.messaging.simp.config.ChannelRegistration
import org.springframework.messaging.simp.config.MessageBrokerRegistry
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker
import org.springframework.web.socket.config.annotation.StompEndpointRegistry
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer

@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig(
    private val stompAuthInterceptor: StompAuthInterceptor,
    private val appProperties: AppProperties,
) : WebSocketMessageBrokerConfigurer {

    override fun configureMessageBroker(registry: MessageBrokerRegistry) {
        // Clients subscribe to /topic/* (broadcast) and /queue/* (user-specific)
        registry.enableSimpleBroker("/topic", "/queue")
        // Clients send messages to /app/*
        registry.setApplicationDestinationPrefixes("/app")
        // User-specific destinations: /user/{userId}/queue/*
        registry.setUserDestinationPrefix("/user")
    }

    override fun registerStompEndpoints(registry: StompEndpointRegistry) {
        registry
            .addEndpoint("/ws")
            .setAllowedOrigins(*appProperties.cors.getOriginList().toTypedArray())
    }

    override fun configureClientInboundChannel(registration: ChannelRegistration) {
        registration.interceptors(stompAuthInterceptor)
    }
}
```

### STOMP Message Controller

```kotlin
// src/main/kotlin/com/example/backend/modules/chat/ChatController.kt
package com.example.backend.modules.chat

import org.springframework.messaging.handler.annotation.DestinationVariable
import org.springframework.messaging.handler.annotation.MessageMapping
import org.springframework.messaging.handler.annotation.SendTo
import org.springframework.messaging.simp.SimpMessagingTemplate
import org.springframework.stereotype.Controller
import java.time.ZoneOffset
import java.time.ZonedDateTime
import java.time.format.DateTimeFormatter

data class ChatMessageRequest(
    val content: String,
    val senderId: String,
    val senderName: String,
)

data class ChatMessageResponse(
    val roomId: String,
    val content: String,
    val senderId: String,
    val senderName: String,
    val timestamp: String,
)

@Controller
class ChatController(
    private val messagingTemplate: SimpMessagingTemplate,
) {

    @MessageMapping("/chat/{roomId}")
    @SendTo("/topic/chat/{roomId}")
    fun handleMessage(
        @DestinationVariable roomId: String,
        message: ChatMessageRequest,
    ): ChatMessageResponse {
        return ChatMessageResponse(
            roomId = roomId,
            content = message.content,
            senderId = message.senderId,
            senderName = message.senderName,
            timestamp = ZonedDateTime.now(ZoneOffset.UTC).format(DateTimeFormatter.ISO_INSTANT),
        )
    }
}
```

### Sending Notifications to Specific Users

```kotlin
// src/main/kotlin/com/example/backend/modules/notifications/NotificationService.kt
package com.example.backend.modules.notifications

import org.springframework.messaging.simp.SimpMessagingTemplate
import org.springframework.stereotype.Service

data class NotificationPayload(
    val id: String,
    val type: String,
    val title: String,
    val body: String? = null,
    val actionUrl: String? = null,
)

@Service
class NotificationService(
    private val messagingTemplate: SimpMessagingTemplate,
) {

    fun sendToUser(userId: String, notification: NotificationPayload) {
        messagingTemplate.convertAndSendToUser(
            userId,
            "/queue/notifications",
            notification,
        )
    }

    fun broadcast(notification: NotificationPayload) {
        messagingTemplate.convertAndSend("/topic/notifications", notification)
    }
}
```

### Concept Comparison: Spring Boot vs NestJS

| Concept              | NestJS (Socket.IO)               | Spring Boot (STOMP)                             |
| -------------------- | -------------------------------- | ----------------------------------------------- |
| Gateway / Endpoint   | `@WebSocketGateway()`            | `@EnableWebSocketMessageBroker`                 |
| Event handling       | `@SubscribeMessage("event")`     | `@MessageMapping("/destination")`               |
| Broadcasting         | `server.emit()`                  | `messagingTemplate.convertAndSend("/topic/...")`|
| User targeting       | `server.to("user:id")`           | `messagingTemplate.convertAndSendToUser()`      |
| Room / topic         | `client.join("room")`            | Subscribe to `/topic/chat/{roomId}`             |
| Auth middleware       | Socket.IO middleware             | `ChannelInterceptor` on CONNECT                 |

---

## 6. Authentication & Authorization

WebSocket authentication happens during the initial handshake, not on every message. All backends verify the JWT token from [06-auth-security](./06-auth-security.md) before accepting the connection.

### NestJS: Socket.IO Handshake Middleware

```typescript
// src/gateway/ws-auth.middleware.ts
import { Injectable, Logger } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { Socket } from "socket.io";
import type { TokenPayload } from "@scope/shared";

@Injectable()
export class WsAuthMiddleware {
  private readonly logger = new Logger(WsAuthMiddleware.name);

  constructor(private readonly jwtService: JwtService) {}

  /**
   * Apply as Socket.IO middleware in the gateway's afterInit().
   * Token is passed via: io({ auth: { token: "Bearer xxx" } })
   */
  createMiddleware() {
    return (socket: Socket, next: (err?: Error) => void) => {
      try {
        const authHeader =
          (socket.handshake.auth?.token as string) ??
          socket.handshake.headers.authorization ??
          "";

        const token = authHeader.replace("Bearer ", "");
        if (!token) {
          return next(new Error("Authentication required"));
        }

        const payload = this.jwtService.verify<TokenPayload>(token);
        socket.data.userId = payload.sub;
        socket.data.email = payload.email;
        socket.data.role = payload.role;
        next();
      } catch {
        this.logger.warn(`WS auth failed: ${socket.id}`);
        next(new Error("Invalid or expired token"));
      }
    };
  }
}
```

Apply the middleware in the gateway:

```typescript
// src/gateway/events.gateway.ts — add afterInit
import { WsAuthMiddleware } from "./ws-auth.middleware";

@WebSocketGateway({ /* ... */ })
export class EventsGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  constructor(private readonly wsAuth: WsAuthMiddleware) {}

  afterInit(server: Server) {
    server.use(this.wsAuth.createMiddleware());
  }

  // ... handleConnection, handleDisconnect, etc.
}
```

### FastAPI: Query Parameter Token Validation

```python
# app/ws/auth.py
import logging
import jwt
from app.core.config import settings
from app.models.user import User
from sqlalchemy.ext.asyncio import AsyncSession

logger = logging.getLogger(__name__)


async def verify_ws_token(token: str | None) -> User | None:
    """Verify JWT from WebSocket query parameter: ws://host/ws?token=<jwt>"""
    if not token:
        return None

    try:
        payload = jwt.decode(token, settings.JWT_PUBLIC_KEY, algorithms=["RS256"])
        user_id = payload.get("sub")
        if not user_id:
            return None

        # In production, fetch user from DB to verify active status
        # For now, return a minimal user object from token claims
        return User(id=user_id, email=payload.get("email", ""))
    except jwt.InvalidTokenError:
        logger.warning("WS auth failed: invalid token")
        return None
```

### Spring Boot: STOMP CONNECT Interceptor

```kotlin
// src/main/kotlin/com/example/backend/ws/StompAuthInterceptor.kt
package com.example.backend.ws

import com.example.backend.modules.auth.JwtProvider
import org.springframework.messaging.Message
import org.springframework.messaging.MessageChannel
import org.springframework.messaging.simp.stomp.StompCommand
import org.springframework.messaging.simp.stomp.StompHeaderAccessor
import org.springframework.messaging.support.ChannelInterceptor
import org.springframework.messaging.support.MessageHeaderAccessor
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.stereotype.Component

@Component
class StompAuthInterceptor(
    private val jwtProvider: JwtProvider,
) : ChannelInterceptor {

    override fun preSend(message: Message<*>, channel: MessageChannel): Message<*>? {
        val accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor::class.java)
            ?: return message

        if (accessor.command == StompCommand.CONNECT) {
            val authHeader = accessor.getFirstNativeHeader("Authorization") ?: ""
            val token = authHeader.removePrefix("Bearer ").trim()

            if (token.isNotEmpty() && jwtProvider.validateToken(token)) {
                val authentication = jwtProvider.getAuthentication(token)
                accessor.user = authentication
            } else {
                throw IllegalArgumentException("Invalid or missing authentication token")
            }
        }

        return message
    }
}
```

### Authentication Flow Summary

```text
1. Client obtains JWT via REST API (POST /auth/google) → see 06-auth-security
2. Client initiates WebSocket connection with the token:
   - Socket.IO:  io({ auth: { token: "Bearer <jwt>" } })
   - Native WS:  new WebSocket("ws://host/ws?token=<jwt>")
   - STOMP:      client.connect({ Authorization: "Bearer <jwt>" })
3. Server validates JWT during handshake / CONNECT frame
4. On success: connection accepted, userId stored in session
5. On failure: connection rejected with error code
6. Token refresh: client disconnects, refreshes via REST, reconnects
```

---

## 7. Scaling with Redis Pub/Sub

When running multiple backend instances behind a load balancer, Redis Pub/Sub ensures events reach all connected clients regardless of which instance they are connected to.

```text
┌──────────┐  emit   ┌──────────┐  publish  ┌─────────┐  subscribe  ┌──────────┐  emit   ┌──────────┐
│ Client A │ ──────▶ │ Inst. 1  │ ────────▶ │  Redis  │ ──────────▶ │ Inst. 2  │ ──────▶ │ Client B │
└──────────┘         └──────────┘           └─────────┘             └──────────┘         └──────────┘
```

### NestJS: Socket.IO Redis Adapter

```typescript
// src/gateway/gateway.module.ts
import { Module, OnModuleInit } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { createAdapter } from "@socket.io/redis-adapter";
import { createClient } from "redis";
import { EventsGateway } from "./events.gateway";
import { WsAuthMiddleware } from "./ws-auth.middleware";
import { AuthModule } from "../modules/auth/auth.module";

@Module({
  imports: [AuthModule],
  providers: [EventsGateway, WsAuthMiddleware],
  exports: [EventsGateway],
})
export class GatewayModule implements OnModuleInit {
  constructor(
    private readonly gateway: EventsGateway,
    private readonly configService: ConfigService,
  ) {}

  async onModuleInit() {
    const redisUrl =
      this.configService.get<string>("redis.url") ?? "redis://localhost:15350";

    const pubClient = createClient({ url: redisUrl });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.gateway.server.adapter(createAdapter(pubClient, subClient));
  }
}
```

### FastAPI: Redis Pub/Sub

```python
# app/ws/redis_pubsub.py
import asyncio
import json
import logging

import redis.asyncio as aioredis

from app.core.config import settings
from app.ws.manager import manager

logger = logging.getLogger(__name__)


class RedisPubSub:
    def __init__(self) -> None:
        self._redis: aioredis.Redis | None = None
        self._pubsub: aioredis.client.PubSub | None = None
        self._channel = "ws:broadcast"

    async def connect(self) -> None:
        self._redis = aioredis.from_url(settings.REDIS_URL)
        self._pubsub = self._redis.pubsub()
        await self._pubsub.subscribe(self._channel)

    async def publish(self, event: str, data: dict) -> None:
        if self._redis:
            message = json.dumps({"event": event, "data": data})
            await self._redis.publish(self._channel, message)

    async def listen(self) -> None:
        """Background task: relay Redis messages to local WebSocket clients."""
        if not self._pubsub:
            return
        async for msg in self._pubsub.listen():
            if msg["type"] == "message":
                try:
                    payload = json.loads(msg["data"])
                    await manager.broadcast(payload["event"], payload["data"])
                except Exception:
                    logger.exception("Redis listener error")

    async def close(self) -> None:
        if self._pubsub:
            await self._pubsub.unsubscribe(self._channel)
        if self._redis:
            await self._redis.close()


redis_pubsub = RedisPubSub()
```

Register in the FastAPI lifespan:

```python
# app/main.py — update lifespan
from app.ws.redis_pubsub import redis_pubsub

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_pg()
    await init_mongo()
    await redis_pubsub.connect()
    asyncio.create_task(redis_pubsub.listen())
    yield
    await redis_pubsub.close()
```

### Spring Boot: Redis MessageListener

Add `application.yml` Redis configuration (already present from [15-backend-kotlin](./15-backend-kotlin.md)):

```kotlin
// src/main/kotlin/com/example/backend/common/config/RedisMessageConfig.kt
package com.example.backend.common.config

import com.fasterxml.jackson.databind.ObjectMapper
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.redis.connection.RedisConnectionFactory
import org.springframework.data.redis.listener.ChannelTopic
import org.springframework.data.redis.listener.RedisMessageListenerContainer
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter
import org.springframework.messaging.simp.SimpMessagingTemplate

@Configuration
class RedisMessageConfig {

    @Bean
    fun topic(): ChannelTopic = ChannelTopic("ws:broadcast")

    @Bean
    fun listenerAdapter(
        messagingTemplate: SimpMessagingTemplate,
        objectMapper: ObjectMapper,
    ): MessageListenerAdapter {
        val delegate = RedisWsRelay(messagingTemplate, objectMapper)
        return MessageListenerAdapter(delegate, "onMessage")
    }

    @Bean
    fun redisListenerContainer(
        connectionFactory: RedisConnectionFactory,
        listenerAdapter: MessageListenerAdapter,
        topic: ChannelTopic,
    ): RedisMessageListenerContainer {
        val container = RedisMessageListenerContainer()
        container.setConnectionFactory(connectionFactory)
        container.addMessageListener(listenerAdapter, topic)
        return container
    }
}

class RedisWsRelay(
    private val messagingTemplate: SimpMessagingTemplate,
    private val objectMapper: ObjectMapper,
) {
    fun onMessage(message: String) {
        val node = objectMapper.readTree(message)
        val event = node.get("event")?.asText() ?: return
        val data = node.get("data")
        messagingTemplate.convertAndSend("/topic/$event", data)
    }
}
```

---

## 8. Client Integration

### 8.1 React / Next.js (Socket.IO Client)

Add to the existing `package.json` from [17-consumer-web-pwa](./17-consumer-web-pwa.md) or [07-admin-nextjs](./07-admin-nextjs.md):

```json
{
  "dependencies": {
    "socket.io-client": "^4.8.1"
  }
}
```

#### useSocket Hook

```typescript
// src/hooks/use-socket.ts
"use client";

import { useEffect, useRef, useCallback, useState } from "react";
import { io, type Socket } from "socket.io-client";

type ConnectionStatus = "connected" | "disconnected" | "reconnecting";

interface UseSocketOptions {
  readonly url?: string;
  readonly enabled?: boolean;
}

const SOCKET_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:15320";

export function useSocket({ url = SOCKET_URL, enabled = true }: UseSocketOptions = {}) {
  const socketRef = useRef<Socket | null>(null);
  const [status, setStatus] = useState<ConnectionStatus>("disconnected");

  useEffect(() => {
    if (!enabled) return;

    const token = localStorage.getItem("access_token");
    if (!token) return;

    const socket = io(url, {
      auth: { token: `Bearer ${token}` },
      transports: ["websocket", "polling"],
      reconnection: true,
      reconnectionAttempts: 10,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 30000,
    });

    socket.on("connect", () => setStatus("connected"));
    socket.on("disconnect", () => setStatus("disconnected"));
    socket.on("reconnect_attempt", () => setStatus("reconnecting"));

    socketRef.current = socket;

    return () => {
      socket.disconnect();
      socketRef.current = null;
      setStatus("disconnected");
    };
  }, [url, enabled]);

  const emit = useCallback((event: string, data?: unknown) => {
    socketRef.current?.emit(event, data);
  }, []);

  const on = useCallback((event: string, handler: (data: unknown) => void) => {
    socketRef.current?.on(event, handler);
    return () => {
      socketRef.current?.off(event, handler);
    };
  }, []);

  return { socket: socketRef.current, status, emit, on };
}
```

#### TanStack Query Cache Invalidation

Invalidate query caches when receiving real-time data updates. Integrates with the query keys from [17-consumer-web-pwa](./17-consumer-web-pwa.md):

```typescript
// src/hooks/use-realtime-sync.ts
"use client";

import { useEffect } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { useSocket } from "./use-socket";
import { WS_EVENTS, type LiveDataUpdate } from "@scope/shared";
import { queryKeys } from "@/lib/query-keys";

export function useRealtimeSync() {
  const { on, status } = useSocket();
  const queryClient = useQueryClient();

  useEffect(() => {
    if (status !== "connected") return;

    const unsubscribe = on(WS_EVENTS.LIVE_DATA_UPDATE, (raw) => {
      const update = raw as LiveDataUpdate;

      // Invalidate the relevant query cache based on the resource type
      if (update.resource === "notifications") {
        queryClient.invalidateQueries({ queryKey: queryKeys.notifications.all });
      } else if (update.resource === "results") {
        queryClient.invalidateQueries({ queryKey: queryKeys.results.all });
      }
    });

    return unsubscribe;
  }, [on, status, queryClient]);
}
```

#### Zustand Notification Store

```typescript
// src/stores/notification-store.ts
import { create } from "zustand";

interface NotificationState {
  readonly unreadCount: number;
  readonly increment: () => void;
  readonly reset: () => void;
  readonly setCount: (count: number) => void;
}

export const useNotificationStore = create<NotificationState>()((set) => ({
  unreadCount: 0,
  increment: () => set((state) => ({ unreadCount: state.unreadCount + 1 })),
  reset: () => set({ unreadCount: 0 }),
  setCount: (count) => set({ unreadCount: count }),
}));
```

### 8.2 Flutter Desktop (web_socket_channel)

Add to the existing `pubspec.yaml` from [08-flutter-desktop](./08-flutter-desktop.md):

```yaml
dependencies:
  # ... existing dependencies
  web_socket_channel: ^3.0.2
```

#### WebSocket Provider (Riverpod)

```dart
// lib/core/ws/ws_provider.dart
import 'dart:async';
import 'dart:convert';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:web_socket_channel/web_socket_channel.dart';

part 'ws_provider.g.dart';

enum WsStatus { connected, disconnected, reconnecting }

@Riverpod(keepAlive: true)
class WsConnection extends _$WsConnection {
  WebSocketChannel? _channel;
  Timer? _reconnectTimer;
  int _reconnectAttempts = 0;
  static const _maxReconnectAttempts = 10;

  final _messageController = StreamController<Map<String, dynamic>>.broadcast();
  Stream<Map<String, dynamic>> get messages => _messageController.stream;

  @override
  WsStatus build() => WsStatus.disconnected;

  Future<void> connect(String url, String token) async {
    _reconnectAttempts = 0;
    await _doConnect(url, token);
  }

  Future<void> _doConnect(String url, String token) async {
    try {
      final uri = Uri.parse('$url?token=$token');
      _channel = WebSocketChannel.connect(uri);
      await _channel!.ready;
      state = WsStatus.connected;
      _reconnectAttempts = 0;

      _channel!.stream.listen(
        (data) {
          final decoded = jsonDecode(data as String) as Map<String, dynamic>;
          _messageController.add(decoded);
        },
        onDone: () => _handleDisconnect(url, token),
        onError: (_) => _handleDisconnect(url, token),
      );
    } catch (_) {
      _handleDisconnect(url, token);
    }
  }

  void _handleDisconnect(String url, String token) {
    state = WsStatus.disconnected;
    if (_reconnectAttempts < _maxReconnectAttempts) {
      state = WsStatus.reconnecting;
      final delay = Duration(
        seconds: (1 << _reconnectAttempts).clamp(1, 30),
      );
      _reconnectAttempts++;
      _reconnectTimer = Timer(delay, () => _doConnect(url, token));
    }
  }

  void send(String event, Map<String, dynamic> data) {
    _channel?.sink.add(jsonEncode({'event': event, 'data': data}));
  }

  void disconnect() {
    _reconnectTimer?.cancel();
    _channel?.sink.close();
    state = WsStatus.disconnected;
  }
}
```

#### Usage in a Feature Provider

```dart
// lib/features/notifications/providers/notification_ws_provider.dart
import 'dart:async';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../../core/ws/ws_provider.dart';

part 'notification_ws_provider.g.dart';

@riverpod
class NotificationListener extends _$NotificationListener {
  StreamSubscription? _subscription;

  @override
  int build() {
    final ws = ref.watch(wsConnectionProvider.notifier);

    _subscription = ws.messages
        .where((msg) => msg['event'] == 'notification:new')
        .listen((msg) {
      state = state + 1; // increment unread count
    });

    ref.onDispose(() => _subscription?.cancel());
    return 0;
  }
}
```

---

## 9. Use Case Patterns

### 9.1 Chat / Messaging

```text
Client A                    Server                    Client B
   │                          │                          │
   │── chat:join-room ──────▶│                          │
   │                          │◀── chat:join-room ──────│
   │                          │                          │
   │── chat:message ────────▶│── chat:message ────────▶│
   │                          │                          │
   │── chat:typing ─────────▶│── chat:typing ─────────▶│
   │                          │                          │
   │── chat:leave-room ─────▶│                          │
```

**Key events:**

| Event             | Direction      | Payload                                    |
| ----------------- | -------------- | ------------------------------------------ |
| `chat:join-room`  | Client → Server | `{ roomId }`                              |
| `chat:leave-room` | Client → Server | `{ roomId }`                              |
| `chat:message`    | Bidirectional   | `{ roomId, content, senderId, senderName }`|
| `chat:typing`     | Bidirectional   | `{ roomId, userId, isTyping }`             |

**Implementation notes:**

- Messages are broadcast to the room only (not all connected clients)
- Typing indicators use a debounce (500ms) on the client side
- Message history is fetched via REST API on room join, not via WebSocket

### 9.2 Real-Time Notifications

```text
Server Event (e.g., new comment)
   │
   ▼
Backend Service ── notification:new ──▶ User's WebSocket connections
                                         │
                                         ▼
                                       Client updates badge count
                                       Client shows toast notification
```

**Key events:**

| Event                | Direction      | Payload                                |
| -------------------- | -------------- | -------------------------------------- |
| `notification:new`   | Server → Client | `{ id, type, title, body, actionUrl }` |
| `notification:read`  | Client → Server | `{ id }`                               |
| `notification:badge` | Server → Client | `{ unreadCount }`                      |

**Implementation notes:**

- Server pushes to user-specific rooms (`user:{userId}`)
- Client acknowledges read status via WebSocket, server updates DB
- Badge count synced on connect and on each new notification

### 9.3 Live Data Synchronization

```text
DB Change (e.g., record updated)
   │
   ▼
Backend Service ── live:data-update ──▶ All subscribed clients
                                         │
                                         ▼
                                       Client invalidates TanStack Query cache
                                       UI re-fetches and re-renders
```

**Key events:**

| Event              | Direction      | Payload                                |
| ------------------ | -------------- | -------------------------------------- |
| `live:data-update` | Server → Client | `{ resource, action, payload }`       |
| `live:presence`    | Bidirectional   | `{ userId, status, lastSeen }`        |
| `live:cursor`      | Bidirectional   | `{ userId, x, y, pageId }`           |

**Implementation notes:**

- Data updates trigger cache invalidation rather than pushing full payloads
- Client re-fetches via TanStack Query after invalidation (eventual consistency)
- Presence updates sent every 30 seconds as heartbeats

---

## 10. Error Handling & Reconnection

### Error Envelope

All backends emit errors in a consistent format:

```typescript
// Error payload (matches wsErrorSchema from §2)
{
  "event": "error",
  "data": {
    "code": "AUTH_EXPIRED",       // Machine-readable error code
    "message": "Token has expired", // Human-readable message
    "details": null               // Optional additional context
  }
}
```

### Common Error Codes

| Code              | Description                        | Client Action            |
| ----------------- | ---------------------------------- | ------------------------ |
| `AUTH_REQUIRED`   | No token provided                  | Redirect to login        |
| `AUTH_EXPIRED`    | JWT token expired                  | Refresh token, reconnect |
| `AUTH_INVALID`    | Invalid JWT token                  | Redirect to login        |
| `RATE_LIMITED`    | Too many messages                  | Back off, retry later    |
| `ROOM_NOT_FOUND`  | Requested room does not exist     | Show error in UI         |
| `INTERNAL_ERROR`  | Unexpected server error            | Retry with backoff       |

### Client Reconnection Strategy

Socket.IO handles reconnection automatically. For native WebSocket clients, implement exponential backoff:

```typescript
// Reconnection configuration (Socket.IO — already built in)
const socket = io(url, {
  reconnection: true,
  reconnectionAttempts: 10,       // Max attempts before giving up
  reconnectionDelay: 1000,        // Initial delay (ms)
  reconnectionDelayMax: 30000,    // Max delay cap (ms)
  randomizationFactor: 0.5,      // Jitter factor
});
```

### Connection State Machine

```text
         connect()
  ┌────────────────────┐
  ▼                    │
DISCONNECTED ──────▶ CONNECTING ──────▶ CONNECTED
  ▲                                        │
  │         connection lost                │
  │  ┌──────────────────┐                 │
  │  ▼                  │                 │
  RECONNECTING ◀────────┘─────────────────┘
       │
       │ max attempts exceeded
       ▼
  DISCONNECTED (permanent)
```

### Heartbeat Configuration

| Parameter        | Socket.IO Default | Recommendation |
| ---------------- | ----------------- | -------------- |
| `pingInterval`   | 25000 ms          | 25000 ms       |
| `pingTimeout`    | 20000 ms          | 20000 ms       |
| Total tolerance  | 45000 ms          | 45 seconds     |

---

## 11. Infrastructure & Monitoring

### Docker Compose Addition

Add to the existing `docker-compose.yml` from [09-docker-infrastructure](./09-docker-infrastructure.md). WebSocket runs on the same container as the HTTP server — no separate service needed. The key change is enabling sticky sessions for Socket.IO when using a load balancer:

```yaml
# docker-compose.prod.yml — WebSocket-aware configuration
services:
  backend:
    # ... existing config
    environment:
      - WS_ENABLED=true
    deploy:
      replicas: 2

  # Nginx reverse proxy with sticky sessions for Socket.IO
  nginx:
    image: nginx:alpine
    ports:
      - "${BASE_PORT}:80"
    volumes:
      - ./infra/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend
```

### Nginx Configuration for WebSocket

```nginx
# infra/nginx/nginx.conf
upstream backend {
    ip_hash;  # Sticky sessions for Socket.IO
    server backend:15320;
}

server {
    listen 80;

    # HTTP API
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket (Socket.IO)
    location /socket.io/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    # WebSocket (Native WS for FastAPI / Spring Boot)
    location /ws {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

### Health Check Endpoint

Add a WebSocket-aware health indicator to existing health endpoints:

```typescript
// NestJS — src/gateway/events.gateway.ts — add method
getConnectionCount(): number {
  return this.server?.engine?.clientsCount ?? 0;
}
```

```typescript
// Health controller addition
@Get("ws")
@ApiOperation({ summary: "WebSocket health check" })
wsHealth() {
  return {
    status: "ok",
    connections: this.eventsGateway.getConnectionCount(),
    timestamp: new Date().toISOString(),
  };
}
```

### Environment Variables

| Variable              | Description                          | Default                         |
| --------------------- | ------------------------------------ | ------------------------------- |
| `WS_ENABLED`          | Enable/disable WebSocket gateway     | `true`                          |
| `WS_CORS_ORIGIN`      | Allowed WebSocket origins           | Same as `FRONTEND_URL`          |
| `REDIS_URL`           | Redis connection for Pub/Sub adapter | `redis://localhost:15350`       |

### Monitoring Metrics

Track these metrics via application logs or an APM tool:

| Metric                     | Description                              |
| -------------------------- | ---------------------------------------- |
| `ws.connections.active`    | Current number of active connections     |
| `ws.connections.total`     | Total connections since startup          |
| `ws.messages.in`           | Inbound messages per second              |
| `ws.messages.out`          | Outbound messages per second             |
| `ws.errors`                | WebSocket error count                    |
| `ws.rooms.active`          | Number of active rooms                   |

---

## 12. Testing Patterns

### NestJS: Socket.IO Gateway Testing

```typescript
// src/gateway/__tests__/events.gateway.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import { INestApplication } from "@nestjs/common";
import { io, Socket as ClientSocket } from "socket.io-client";
import { AppModule } from "../../app.module";

describe("EventsGateway", () => {
  let app: INestApplication;
  let clientSocket: ClientSocket;

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    await app.listen(0); // Random port
    const port = app.getHttpServer().address().port;

    clientSocket = io(`http://localhost:${port}`, {
      auth: { token: "Bearer test-jwt-token" },
      transports: ["websocket"],
    });

    await new Promise<void>((resolve) => clientSocket.on("connect", resolve));
  });

  afterAll(async () => {
    clientSocket.disconnect();
    await app.close();
  });

  it("should respond to ping with pong", (done) => {
    clientSocket.on("pong", (data) => {
      expect(data).toHaveProperty("timestamp");
      done();
    });
    clientSocket.emit("ping");
  });
});
```

### FastAPI: WebSocket Endpoint Testing

```python
# tests/test_ws.py
import pytest
from httpx import ASGITransport, AsyncClient
from starlette.testclient import TestClient
from app.main import app


def test_websocket_unauthorized():
    """Connection without token should be rejected."""
    client = TestClient(app)
    with pytest.raises(Exception):
        with client.websocket_connect("/ws"):
            pass


def test_websocket_ping_pong():
    """Authenticated connection should respond to ping."""
    client = TestClient(app)
    with client.websocket_connect("/ws?token=valid-test-jwt") as ws:
        ws.send_json({"event": "ping", "data": {}})
        response = ws.receive_json()
        assert response["event"] == "pong"
```

### Spring Boot: STOMP Testing

```kotlin
// src/test/kotlin/com/example/backend/ws/ChatControllerTest.kt
package com.example.backend.ws

import com.example.backend.modules.chat.ChatMessageRequest
import com.example.backend.modules.chat.ChatMessageResponse
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.server.LocalServerPort
import org.springframework.messaging.converter.MappingJackson2MessageConverter
import org.springframework.messaging.simp.stomp.StompFrameHandler
import org.springframework.messaging.simp.stomp.StompHeaders
import org.springframework.messaging.simp.stomp.StompSession
import org.springframework.messaging.simp.stomp.StompSessionHandlerAdapter
import org.springframework.web.socket.client.standard.StandardWebSocketClient
import org.springframework.web.socket.messaging.WebSocketStompClient
import java.lang.reflect.Type
import java.util.concurrent.CompletableFuture
import java.util.concurrent.TimeUnit
import kotlin.test.assertEquals

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ChatControllerTest {

    @LocalServerPort
    private var port: Int = 0

    @Test
    fun `should broadcast chat message to room subscribers`() {
        val client = WebSocketStompClient(StandardWebSocketClient())
        client.messageConverter = MappingJackson2MessageConverter()

        val url = "ws://localhost:$port/ws"
        val session: StompSession = client
            .connectAsync(url, object : StompSessionHandlerAdapter() {})
            .get(5, TimeUnit.SECONDS)

        val future = CompletableFuture<ChatMessageResponse>()

        session.subscribe("/topic/chat/room-1", object : StompFrameHandler {
            override fun getPayloadType(headers: StompHeaders): Type =
                ChatMessageResponse::class.java

            override fun handleFrame(headers: StompHeaders, payload: Any?) {
                future.complete(payload as ChatMessageResponse)
            }
        })

        session.send("/app/chat/room-1", ChatMessageRequest(
            content = "Hello!",
            senderId = "user-1",
            senderName = "Test User",
        ))

        val response = future.get(5, TimeUnit.SECONDS)
        assertEquals("Hello!", response.content)
        assertEquals("room-1", response.roomId)
    }
}
```

### Client: Vitest with Mocked Socket

```typescript
// src/hooks/__tests__/use-socket.test.ts
import { renderHook, act } from "@testing-library/react";
import { vi, describe, it, expect, beforeEach } from "vitest";

// Mock socket.io-client
vi.mock("socket.io-client", () => {
  const handlers: Record<string, Function> = {};
  const mockSocket = {
    on: vi.fn((event: string, handler: Function) => {
      handlers[event] = handler;
    }),
    off: vi.fn(),
    emit: vi.fn(),
    disconnect: vi.fn(),
    connected: true,
    // Helper to simulate server events
    _trigger: (event: string, data: unknown) => handlers[event]?.(data),
  };
  return {
    io: vi.fn(() => mockSocket),
    _mockSocket: mockSocket,
  };
});

import { useSocket } from "../use-socket";
import { _mockSocket } from "socket.io-client";

describe("useSocket", () => {
  beforeEach(() => {
    localStorage.setItem("access_token", "test-token");
  });

  it("should connect and track status", () => {
    const { result } = renderHook(() => useSocket());
    expect(result.current.status).toBeDefined();
  });

  it("should emit events", () => {
    const { result } = renderHook(() => useSocket());
    act(() => {
      result.current.emit("chat:message", { content: "Hello" });
    });
    expect(_mockSocket.emit).toHaveBeenCalledWith("chat:message", { content: "Hello" });
  });
});
```

---

## Verification

```bash
# NestJS — Verify WebSocket gateway starts
pnpm --filter @scope/backend run dev
# → Check logs: "EventsGateway initialized"

# FastAPI — Verify WebSocket endpoint
cd apps/backend-py && uv run uvicorn app.main:app --reload --port 15320
# → Test: websocat ws://localhost:15320/ws?token=<jwt>

# Spring Boot — Verify STOMP endpoint
cd apps/backend-kt && ./gradlew bootRun
# → Test: Connect to ws://localhost:15360/ws with a STOMP client

# Redis Pub/Sub — Verify adapter works
redis-cli -p 15350 ping   # → PONG

# Client — Verify Socket.IO connection (browser DevTools)
# Network → WS → Verify handshake and message frames

# Run tests
pnpm --filter @scope/backend test
cd apps/backend-py && uv run pytest tests/test_ws.py
cd apps/backend-kt && ./gradlew test --tests "*ChatControllerTest"
```

### Checklist

- [ ] WebSocket gateway starts without errors alongside the HTTP server
- [ ] Client can connect with valid JWT and receives `pong` on `ping`
- [ ] Client without JWT is rejected at handshake
- [ ] Chat messages broadcast to room members only
- [ ] Notifications reach the targeted user
- [ ] Redis Pub/Sub relays events across multiple backend instances
- [ ] Reconnection works after server restart
- [ ] Nginx proxies WebSocket connections correctly
- [ ] All test suites pass
