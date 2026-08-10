# Design a Real-Time Chat System: WebSocket Handlers and Message Queues at the Class Level

## Learning Objectives

- Model the two halves of a chat system separately: the connection layer that owns live sockets and the messaging layer that owns message flow.
- Design the connection registry as the routing table that answers "which socket belongs to which user" under multi-device, multi-instance reality.
- Follow one message from the sender's socket through the broker to the receiver's socket, and know where it gets persisted and where it does not.

## Introduction

A chat system is the case study where the phrase "message queue" stops being a metaphor. A message must travel from one user's screen to another's in milliseconds, over a connection that is alive right now, and the whole design is about the journey. The candidate who tries to model it as a REST call, a request and a response, has missed that chat is push, not pull. The two technical pillars are WebSocket connections, which stay open and carry messages both ways, and a message bus, which carries a message from one chat server to whichever server happens to hold the receiver's connection. Interviewers ask this because it is the clearest case of a distributed system at class level, and because the classic mistake, one server, one map of sockets, fails in the exact way production chat always fails: the user is connected to a different server than the one that received the message.

## Requirements Gathering

Functional requirements:

- A user connects and receives messages for their chats in real time.
- A user sends a message to a chat; all other participants in that chat receive it promptly.
- Messages are persisted and a user can fetch history when they reconnect or load a chat.
- The system shows presence: whether a participant is online, and delivery state (delivered, read).
- A user can be online on multiple devices.

Non-functional requirements:

- Message delivery latency should be dominated by the network, not the server, sub-second end to end.
- A disconnected user must not lose messages; they are delivered on reconnect.

Assumptions to state out loud: one-to-one and small group chats only, no message editing or deletion, no attachments beyond a URL in the payload, ordering is guaranteed per chat per server but not a total order across chats, and presence is online or offline, no "away" or "busy." Cut attachments and cut presence states. The interviewer wants the socket routing and the message flow.

## Identifying Core Entities

The entity list splits cleanly into the connection layer and the message layer, which is the design's thesis in list form.

| Entity | One-line responsibility |
| --- | --- |
| `ChatWebSocketHandler` | The WebSocket endpoint: owns the handshake, per-message dispatch, and disconnect cleanup. |
| `ClientConnection` | One live socket plus its owning user, chat memberships, and device identity. |
| `ConnectionRegistry` | The routing table from user to their live connections. |
| `Message` | The unit of chat: chat ID, sender, timestamp, body, and a client-generated ID. |
| `ChatService` | The domain logic: validate, persist, and publish a message. |
| `MessageBroker` | The bus that routes a message to the server instance holding the receiver's connection. |
| `ChatStore` | The durable message history. |

The two-layer split is the whole design. The connection layer never decides what a message means, and the message layer never opens a socket.

## Class Design

Start with the message, because it is the payload everything else moves. The client-generated ID is the detail that makes delivery honest: it is how the client deduplicates a message it might receive twice, and how the server can accept a resend without duplicating.

```java
public class Message {
    private final String messageId;   // client-generated
    private final String chatId;
    private final String senderId;
    private final long timestampMillis;
    private final String body;

    public Message(String messageId, String chatId, String senderId, long timestampMillis, String body) {
        this.messageId = messageId;
        this.chatId = chatId;
        this.senderId = senderId;
        this.timestampMillis = timestampMillis;
        this.body = body;
    }

    public String getMessageId() { return messageId; }
    public String getChatId() { return chatId; }
    public String getSenderId() { return senderId; }
    public String getBody() { return body; }
}
```

`ClientConnection` wraps one socket with the identity of the user behind it and the chat IDs the user is a member of. The chat membership list is what lets the router decide, locally, whether an incoming message is relevant to this socket.

```java
public class ClientConnection {
    private final WebSocketSession session;
    private final String userId;
    private final String deviceId;
    private final Set<String> chatIds;

    public ClientConnection(WebSocketSession session, String userId, String deviceId, Set<String> chatIds) {
        this.session = session;
        this.userId = userId;
        this.deviceId = deviceId;
        this.chatIds = chatIds;
    }

    public boolean involvedIn(String chatId) { return chatIds.contains(chatId); }
    public String getUserId() { return userId; }
    public WebSocketSession getSession() { return session; }
}
```

`ConnectionRegistry` is the routing table. On a single server it maps user to a list of connections, one per device. The detail that matters for the interview is that this registry is local to one server instance, and the broker is what ties the registries together. The candidate who puts all sockets in one global map has drawn the single-server chat that falls over behind a load balancer.

```java
public class ConnectionRegistry {
    private final Map<String, List<ClientConnection>> connections = new ConcurrentHashMap<>();

    public void register(ClientConnection connection) {
        connections.computeIfAbsent(connection.getUserId(), k -> new CopyOnWriteArrayList<>())
                .add(connection);
    }

    public void unregister(ClientConnection connection) {
        List<ClientConnection> list = connections.get(connection.getUserId());
        if (list != null) {
            list.remove(connection);
        }
    }

    public List<ClientConnection> connectionsFor(String userId) {
        return connections.getOrDefault(userId, List.of());
    }
}
```

`ChatWebSocketHandler` is the connection lifecycle, the only place the socket API appears. On message, it routes the JSON payload to the chat service; on close, it cleans up the registry. It never touches the store and it never decides routing.

```java
public class ChatWebSocketHandler {
    private final ConnectionRegistry registry;
    private final ChatService chatService;

    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = session.getAttributes().get("userId").toString();
        String deviceId = session.getAttributes().get("deviceId").toString();
        Set<String> chats = chatService.loadChatIds(userId);
        registry.register(new ClientConnection(session, userId, deviceId, chats));
    }

    public void handleMessage(WebSocketSession session, String payload) {
        Message message = jsonToMessage(payload);
        chatService.receiveMessage(message);
    }

    public void afterConnectionClosed(WebSocketSession session) {
        ClientConnection connection = findConnection(session);
        registry.unregister(connection);
    }
}
```

`ChatService.receiveMessage` is the message flow in miniature: validate the sender is a member, persist the message so history survives, then publish to the broker so live delivery can happen. Persist and publish are both part of one logical operation, which is why real chat systems use a message queue that doubles as the history store.

```java
public class ChatService {
    private final ChatStore store;
    private final MessageBroker broker;
    private final Set<String> dedupeIds = ConcurrentHashMap.newKeySet();

    public void receiveMessage(Message message) {
        if (!dedupeIds.add(message.getMessageId())) {
            return; // duplicate send, ignore
        }
        store.save(message);
        broker.publish(message);
    }

    public void deliver(ClientConnection connection, Message message) {
        connection.getSession().sendMessage(textMessage(message));
    }
}
```

`MessageBroker` is where the distributed routing lives. On a single instance it can be a simple fan-out to the local registry. Across instances it is a topic per chat: every instance subscribes to the chats its users are in, and a message published to a chat topic reaches every instance holding a member. The router then delivers to the sockets whose membership matches, which is why `ClientConnection` carries its chat IDs.

```java
public class MessageBroker {
    private final ConnectionRegistry registry;

    public void publish(Message message) {
        // in production: publish to a chat topic on Kafka/Redis pub-sub,
        // every instance subscribed to that topic receives it here.
        for (ClientConnection connection : registry.connectionsForAll()) {
            if (connection.involvedIn(message.getChatId())
                    && !connection.getUserId().equals(message.getSenderId())) {
                registry.deliver(connection, message);
            }
        }
    }
}
```

Diagram: one message, from the sender's socket on server A to the receiver's socket on server B. The registry is per-instance; the broker is the bridge; the membership filter is the router.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 430" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah9" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="920" height="430" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">One message, sender to receiver</text>

  <rect x="20" y="120" width="290" height="280" rx="10" fill="#f8fafc" stroke="#94a3b8" stroke-dasharray="6 4"/>
  <text x="35" y="142" font-size="13" font-weight="bold" fill="#64748b">Server A</text>
  <rect x="610" y="120" width="290" height="280" rx="10" fill="#f8fafc" stroke="#94a3b8" stroke-dasharray="6 4"/>
  <text x="625" y="142" font-size="13" font-weight="bold" fill="#64748b">Server B</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah9)">
    <line x1="150" y1="192" x2="166" y2="192"/>
    <line x1="225" y1="214" x2="225" y2="256"/>
    <line x1="170" y1="260" x2="95" y2="336"/>
    <line x1="260" y1="288" x2="356" y2="288"/>
    <line x1="510" y1="262" x2="621" y2="247"/>
    <line x1="755" y1="245" x2="801" y2="240"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="40" y="170" width="110" height="44" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="95" y="191" text-anchor="middle" font-weight="bold" fill="#1e3a8a">socket</text>
    <text x="95" y="206" text-anchor="middle" font-size="12" fill="#1e40af">user U1</text>
    <rect x="170" y="170" width="110" height="44" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="225" y="191" text-anchor="middle" font-weight="bold" fill="#334155">ChatWebSocket</text>
    <text x="225" y="206" text-anchor="middle" font-size="12" fill="#64748b">Handler</text>
    <rect x="80" y="260" width="180" height="56" rx="8" fill="#fffbeb" stroke="#f59e0b"/>
    <text x="170" y="283" text-anchor="middle" font-weight="bold" fill="#92400e">ChatService</text>
    <text x="170" y="301" text-anchor="middle" font-size="12" fill="#b45309">dedupe → save → publish</text>
    <rect x="40" y="340" width="110" height="44" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="95" y="361" text-anchor="middle" font-weight="bold" fill="#14532d">ChatStore</text>
    <text x="95" y="376" text-anchor="middle" font-size="12" fill="#15803d">persist history</text>

    <rect x="360" y="240" width="150" height="70" rx="8" fill="#f5f3ff" stroke="#a78bfa"/>
    <text x="435" y="263" text-anchor="middle" font-weight="bold" fill="#5b21b6">MessageBroker</text>
    <text x="435" y="281" text-anchor="middle" font-size="12" fill="#6d28d9">topic per chat</text>
    <text x="435" y="299" text-anchor="middle" font-size="12" fill="#6d28d9">routes to U2's server</text>

    <rect x="620" y="220" width="135" height="70" rx="8" fill="#e0e7ff" stroke="#6366f1"/>
    <text x="688" y="243" text-anchor="middle" font-weight="bold" fill="#3730a3">ConnectionRegistry</text>
    <text x="688" y="261" text-anchor="middle" font-size="12" fill="#4338ca">instance-local</text>
    <text x="688" y="277" text-anchor="middle" font-size="12" fill="#4338ca">U2 → [socket]</text>
    <rect x="805" y="215" width="85" height="44" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="848" y="236" text-anchor="middle" font-weight="bold" fill="#1e3a8a">socket</text>
    <text x="848" y="251" text-anchor="middle" font-size="12" fill="#1e40af">user U2</text>
  </g>

  <g font-size="12" fill="#475569">
    <text x="300" y="280">publish</text>
    <text x="175" y="330">save</text>
    <text x="560" y="238">delivered</text>
    <text x="780" y="232">deliver</text>
  </g>

</svg>
```

The connection between the local registry and the remote topic is the interview's key insight: the registry is instance-local, the broker connects the instances, and the membership filter on each connection is what turns "every server got the message" into "exactly the right sockets delivered it."

## Design Patterns Used

The pattern here is the Observer pattern, and for once it is the honest fit: the receiver's connections are observers of a chat topic, and the broker notifies them on publication. Naming it is correct and shows you recognize the shape. The connection handler is a Facade over the socket API, and the registry is a plain routing structure. The one thing to resist is a Command pattern wrapping each message type, and an Actor model, which is a real production architecture but a detour the interviewer did not ask for. The topic-per-chat fan-out is the structural idea that matters, and it is not a GoF pattern; it is pub-sub, and if the interviewer wants to dig into the broker's internals, that is the next chapter's problem.

## Handling Edge Cases / Concurrency

The edges are delivery, not data integrity. The disconnected receiver: the message is persisted before it is published, so a receiver who is offline gets it from the store on reconnect, when the client fetches history since its last message ID. The message is never dependent on the socket being open at send time.

The duplicate message: a client retries a send because the acknowledgment was lost, and the server receives the same message twice. The client-generated ID plus the dedupe set is the guard, the same shape as the order idempotency from the e-commerce chapter. The receiver may also receive the same message twice if the broker delivers and the reconnect history overlaps, and the client deduplicates by message ID on its side too.

The concurrent close: a connection closes while a message is being delivered to it, which throws mid-send. The handler must catch, drop, and rely on the persisted history for the recovery, which is exactly why persistence comes before delivery in the flow. The multi-instance edge: the registry is per-instance, so the broker's topic subscription must be per-chat, not per-instance-wide, or every server delivers every message to every connection and filters, which works but wastes the network.

## Common Mistakes

The most common mistake is the global socket map. A `Map<String, WebSocketSession>` in a static holder, with the implicit assumption that all users connect to one server. The interviewer asks "what happens with two servers behind a load balancer" and the map does not exist on the server that got the message. The instance-local registry plus the broker is not a distributed refinement, it is the correct shape from the start.

The second mistake is skipping persistence. The candidate designs a beautiful push system where a message exists only on the wire. The receiver is offline for ten minutes and the message is gone, which is a chat product that cannot be used. Persistence is not a bolt-on, it is what makes delivery best-effort safe.

The third mistake is delivery order assumed. The candidate says "messages are delivered in order" without qualifying it per chat and per instance. A broker with multiple partitions can reorder across a chat, and the honest statement is ordering per chat within one server, with a sequence number per chat in the store for the client to sort by. Unqualified ordering claims are the mark of a candidate who has not seen a real message bus.

## Interview Perspective

A weak answer is a single `ChatServer` class with a socket map and a `broadcast(message)` that loops all sockets. The interviewer asks "the receiver is on a different server" and the candidate has no second server, "the receiver is offline" and the message vanishes, "the receiver is on two devices" and the map holds one connection per user. Every follow-up is a hole.

A strong answer says "the connection layer and the message layer are separate, the registry is instance-local, the broker is the bridge, the membership filter on each connection is what routes precisely, and persistence before publish is what makes offline delivery a history fetch instead of a miracle." Follow-ups to expect: "what if the user is on two devices" (the registry holds a list per user, both sockets deliver), "how do you do read receipts" (a separate receipt message type on the same path, which the membership filter already routes), "how do you scale the broker" (partition the chat topics, accept per-chat ordering as the guarantee). The strongest candidates volunteer the client-generated message ID and the reconnect history fetch without prompting.

## Knowledge Check

1. A user is connected to server A. A second user sends a message to their shared chat, and the message arrives at server B. Trace the path from the sender's socket to the receiver's socket, naming each component it passes through.
2. The receiver is offline when a message is sent, then reconnects. Explain what makes the message appear on their screen, and why the message's survival depends on a component other than the delivery path.
3. The client's acknowledgment is lost and the client resends the same message. Walk through what the server sees, which data prevents a duplicate, and what the receiver does if a duplicate still arrives.

## Key Takeaways

- Two layers, always: the connection layer owns sockets, the message layer owns flow. No overlap.
- The registry is per-instance. The broker is the bridge between instances, and the membership filter is the router.
- Persist before publish. Offline delivery is a history fetch, not a special case.
- The client-generated message ID is the dedupe key on both ends.
- Ordering is per chat, per instance, and the store carries sequence numbers so clients can sort.

## What's Next

The chat system used a broker to fan messages out to interested connections. The pub-sub system removes the connections and the domain entirely and asks the question the chat system quietly depended on: what is inside the broker, and how does it store, route, and replay messages for thousands of subscribers per topic?

---

This article explains how to design a real-time chat by splitting the per-instance socket registry from the message layer a broker bridges. Its point of view is that a global socket map fails behind every load balancer, and persisting before publishing enables offline delivery.
