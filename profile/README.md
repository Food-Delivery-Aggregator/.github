<div align="center">

# AASTU Food Delivery Aggregator System

### Distributed Systems Mini Project – Group 2

**Addis Ababa Science and Technology University (AASTU)**

[![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=flat&logo=nestjs&logoColor=white)](https://nestjs.com/)
[![Express](https://img.shields.io/badge/Express-000000?style=flat&logo=express&logoColor=white)](https://expressjs.com/)
[![Next.js](https://img.shields.io/badge/Next.js-000000?style=flat&logo=next.js&logoColor=white)](https://nextjs.org/)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-FF6600?style=flat&logo=rabbitmq&logoColor=white)](https://www.rabbitmq.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-336791?style=flat&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)

</div>

---

## Project Overview

The **Food Delivery Aggregator System** is a **microservices-based platform** that connects **customers**, **restaurants**, and **delivery agents** in a seamless food ordering and delivery experience.

### Key Features

| Feature | Description |
|---------|-------------|
| **Food Ordering** | Customers browse restaurants, add items to cart, and place orders |
| **Payment Processing** | Integrated with Stripe and Chapa payment gateways |
| **Delivery Tracking** | Real-time order status updates via WebSocket |
| **Notifications** | Event-driven notifications using RabbitMQ |
| **Authentication** | JWT-based auth with email verification |
| **Admin Dashboard** | Platform metrics and user management |

### Distributed Systems Concepts Demonstrated

- **Service Independence**: Each microservice is independently deployable
- **Asynchronous Communication**: Event-driven architecture with RabbitMQ
- **Eventual Consistency**: Services synchronize state through events
- **Scalability**: Kubernetes-ready with horizontal scaling support
- **Fault Tolerance**: Circuit breakers and graceful degradation

---

## Team Members

| Name | Role |
|------|------|
| Miraf Debebe | Developer |
| Mistire Daniel | Developer (**Team Lead**) |
| Nasifay Chala | Developer |
| Natan Addis | Developer |
| Nathnael Keleme | Developer |
| Rediet Birhanu | Developer |

---

# Architecture Deep Dive

This document provides an exhaustive technical analysis of the microservices architecture, covering service communication, authentication, real-time notifications, database design, security patterns, resilience strategies, and infrastructure.

---


## Table of Contents

1.  [System Overview](#1-system-overview)
2.  [How Services Communicate With Each Other](#2-how-services-communicate-with-each-other)
3.  [How the API Gateway Validates Tokens](#3-how-the-api-gateway-validates-tokens)
4.  [Complete Authentication Flow](#4-complete-authentication-flow)
5.  [How the Notification Service Handles Events](#5-how-the-notification-service-handles-events)
6.  [Why and How Socket.io is Used](#6-why-and-how-socketio-is-used)
7.  [RabbitMQ Architecture & Configuration](#7-rabbitmq-architecture--configuration)
8.  [Database Architecture (Database-per-Service)](#8-database-architecture-database-per-service)
9.  [Security Patterns](#9-security-patterns)
10. [Resilience Patterns & Fault Tolerance](#10-resilience-patterns--fault-tolerance)
11. [Frontend Architecture](#11-frontend-architecture)
12. [Infrastructure: Docker & Kubernetes](#12-infrastructure-docker--kubernetes)
13. [Complete Message Flow Diagrams](#13-complete-message-flow-diagrams)

---

## 1. System Overview

The Food Delivery Aggregator is a microservices-based platform consisting of:

| Service               | Technology         | Port  | Purpose                                    |
|-----------------------|--------------------|-------|--------------------------------------------|
| `frontend`            | Next.js (React)    | 3000  | Customer, Restaurant, Driver, Admin UIs    |
| `api-nginx`           | Nginx              | 8080  | Reverse proxy, request routing             |
| `api-gateway`         | NestJS             | 4001  | Swagger aggregation (optional auth)        |
| `auth-service`        | NestJS + Prisma    | 4000  | User registration, login, JWT, email       |
| `order-service`       | Express + Prisma   | 4002  | Restaurants, items, orders, delivery       |
| `payment-service`     | Express + Prisma   | 4003  | Stripe/Chapa payments, webhooks            |
| `notification-service`| NestJS + TypeORM   | 4004  | Event consumption, WebSocket push          |

**Supporting Infrastructure:**
-   **RabbitMQ**: Message broker for async communication
-   **Redis**: Caching/session storage (used by payment-service)
-   **PostgreSQL**: 4 separate database instances (one per service)
-   **MailHog**: Development SMTP server for email testing
-   **Kubernetes (Minikube)**: Orchestration platform

---

## 2. How Services Communicate With Each Other

The system uses a **hybrid communication model** combining synchronous HTTP requests with asynchronous messaging via RabbitMQ.

### 2.1 Synchronous Communication (HTTP via Nginx)

```mermaid
flowchart LR
    subgraph "Client"
        F[Next.js Frontend]
    end

    subgraph "API Layer"
        N["Nginx (:8080)"]
    end

    subgraph "Microservices"
        AUTH["auth-service (:4000)"]
        ORDER["order-service (:4002)"]
        PAY["payment-service (:4003)"]
        NOTIFY["notification-service (:4004)"]
    end

    F -->|"HTTP/HTTPS"| N
    N -->|"/auth/*"| AUTH
    N -->|"/order/*"| ORDER
    N -->|"/payment/*"| PAY
    N -->|"/notification/*"| NOTIFY
    N -->|"/socket.io/*"| NOTIFY
```

**Diagram Explanation:**
1.  **Single Entry Point**: The user's browser (via the Next.js frontend) sends all API requests to one address: `http://localhost:8080` (Nginx).
2.  **Path-Based Routing**: Nginx acts like a traffic cop. It looks at the URL path (e.g., `/auth/login`) and decides which internal service should handle the request.
3.  **Internal Forwarding**: Nginx forwards the request to the correct backend service. The frontend never talks directly to `auth-service:4000`; it only knows about Nginx.
4.  **Why This Matters**: This decouples the frontend from backend service locations. If `auth-service` moves to a different port or IP, only Nginx config changes—not the frontend code.

**How Nginx Routes Requests:**

Nginx uses location-based routing to direct requests to the appropriate backend service. Each service has its own path prefix.

```nginx
# From nginx.conf

location /auth/ {
    proxy_pass http://auth_service;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

location /order/ {
    proxy_pass http://order_service;
    # ... headers
}

location /payment/ {
    rewrite ^/payment/?(.*)$ /$1 break;  # Strips /payment prefix
    proxy_pass http://payment_service;
}

location /socket.io/ {
    proxy_pass http://notification_service;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";  # WebSocket upgrade
}
```

> [!IMPORTANT]
> **CORS Handling**: Nginx handles OPTIONS preflight requests directly, adding the necessary `Access-Control-*` headers. For actual requests, the backend services handle CORS via `app.enableCors()`.

**Key File:** [nginx.conf](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/api-gateway/nginx/nginx.conf)

### 2.2 Asynchronous Communication (RabbitMQ)

For decoupled, event-driven communication, services publish and consume messages through **RabbitMQ**.

```mermaid
flowchart TD
    subgraph "Publishers"
        AUTH_PUB["auth-service - (Topic Exchange)"]
        ORDER_PUB["order-service - (Direct Queue)"]
        PAY_PUB["payment-service - (Direct Queue)"]
    end

    subgraph "RabbitMQ Broker"
        EXCHANGE_AUTH["auth.events - (topic exchange)"]
        QUEUE_NOTIF["notification_queue - (durable queue)"]
        QUEUE_PAY_EVENTS["PAYMENT_EVENTS - (durable queue)"]
    end

    subgraph "Consumers"
        NOTIFY_CON["notification-service"]
        ORDER_CON["order-service"]
        AUDIT_CON["(Future Audit Service)"]
    end

    AUTH_PUB -->|"user.created, user.updated"| EXCHANGE_AUTH
    EXCHANGE_AUTH -->|"bind: user.#"| QUEUE_NOTIF
    EXCHANGE_AUTH -.->|"bind: user.#"| AUDIT_CON

    ORDER_PUB -->|"ORDER_CREATED, DELIVERY_STATUS_UPDATED"| QUEUE_NOTIF
    PAY_PUB -->|"PAYMENT_SUCCESS, PAYMENT_FAILED"| QUEUE_PAY_EVENTS

    QUEUE_NOTIF --> NOTIFY_CON
    QUEUE_PAY_EVENTS --> ORDER_CON
```

**Diagram Explanation:**
1.  **Publishers (Left)**: Services like `auth-service` and `order-service` publish events when something important happens (e.g., a new user registers, an order is created).
2.  **RabbitMQ Broker (Center)**: RabbitMQ receives these events. It uses different patterns:
    *   **Topic Exchange** (`auth.events`): Acts like a mail sorting office. It routes messages based on patterns (e.g., `user.#` matches `user.created`, `user.updated`).
    *   **Direct Queue** (`notification_queue`): A simple mailbox. Messages go directly into the queue.
3.  **Consumers (Right)**: `notification-service` listens to the queue. When a message arrives, it processes it (e.g., sends an email, pushes a WebSocket notification).
4.  **Why This Matters**: Services are **decoupled**. The `auth-service` does not know or care that `notification-service` exists—it just publishes events. If notification-service is down, messages wait in the queue.

**Two Messaging Patterns:**

| Pattern            | Used By           | How It Works                                                                 |
|--------------------|-------------------|------------------------------------------------------------------------------|
| **Topic Exchange** | `auth-service`    | Publishes to `auth.events` exchange with routing keys like `user.created`. Consumers bind with wildcard patterns (`user.#`). |
| **Direct Queue**   | `order-service`, `payment-service` | Publishes directly to a named queue (`notification_queue`, `PAYMENT_EVENTS`). |

**Why Topic Exchange for Auth?**
-   Supports **multiple consumers** with different binding patterns
-   Future services can subscribe to specific user events (e.g., `user.role.updated`) without modifying the publisher

**Why Direct Queue for Orders/Payments?**
-   Simple point-to-point messaging
-   Each message is consumed by exactly one consumer
-   Guaranteed ordering within the queue

**Key Files:**
-   Auth Publisher: [rabbitmq.service.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/auth-service/src/rabbitmq/rabbitmq.service.ts)
-   Order Publisher: [order.service.js](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/order-service/src/core/services/order.service.js)
-   Payment Publisher: [paymentController.js](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/payment-service/src/controllers/paymentController.js)

---

## 3. How the API Gateway Validates Tokens

Token validation is **decentralized**. Each service that needs authentication includes the JWT validation logic and uses the **same shared secret**.

```mermaid
sequenceDiagram
    participant Client
    participant Nginx
    participant Service as "Target Service"

    Client->>Nginx: "GET /order/api/v1/orders<br/>Authorization: Bearer [token]"
    Nginx->>Service: Proxies request with Authorization header
    
    Note right of Service: JwtAuthGuard activated
    Service->>Service: 1. Extract token from Authorization header
    Service->>Service: 2. Verify signature with JWT_SECRET
    Service->>Service: "3. Check expiration (exp claim)"
    Service->>Service: "4. Decode payload { sub, email, role }"
    
    alt Token Valid
        Service->>Client: 200 OK + data
    else Token Invalid/Expired
        Service->>Client: 401 Unauthorized
    end
```

**Diagram Explanation:**
1.  **Request with Token**: The client sends an API request (e.g., `GET /order/api/v1/orders`) with a JWT token in the `Authorization` header.
2.  **Proxy Pass**: Nginx simply forwards the request and headers to the target service (`order-service`).
3.  **Token Validation**: The **JwtAuthGuard** inside `order-service` activates. It extracts the token, verifies its signature using the shared `JWT_SECRET`, checks if it is expired, and decodes the user information.
4.  **Decision**: If the token is valid, the request proceeds and returns data. If the token is invalid or expired, a `401 Unauthorized` error is returned.
5.  **Key Insight**: Token validation is **decentralized**—each service validates tokens independently using the same shared secret. There is no central auth gateway that all requests pass through.

### 3.1 JWT Strategy Implementation

```typescript
// api-gateway/src/auth/jwt.strategy.ts

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: process.env.JWT_SECRET!,  // SHARED SECRET
    });
  }

  async validate(payload: any) {
    // Attach user info to request object
    return { userId: payload.sub, username: payload.username, role: payload.role };
  }
}
```

### 3.2 Token Properties

| Property          | Value           | Purpose                                    |
|-------------------|-----------------|--------------------------------------------|
| Access Token TTL  | 15 minutes      | Short-lived for security                   |
| Refresh Token TTL | 7 days          | Long-lived for seamless re-authentication |
| Algorithm         | HS256 (default) | Symmetric signing with shared secret       |
| Storage           | localStorage    | Frontend stores tokens                     |

> [!WARNING]
> **Security Note**: All services that validate tokens MUST have the same `JWT_SECRET`. If this secret is compromised, all tokens become invalid. Consider rotating secrets periodically and using a secrets manager in production.

**Key Files:**
-   [jwt.strategy.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/api-gateway/src/auth/jwt.strategy.ts)
-   [auth.guard.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/api-gateway/src/auth/auth.guard.ts)

---

## 4. Complete Authentication Flow

### 4.1 User Registration Flow

```mermaid
sequenceDiagram
    participant Frontend
    participant AuthService as "auth-service"
    participant Database as "PostgreSQL (auth_db)"
    participant EmailService as "Email (MailHog)"
    participant RabbitMQ

    Frontend->>AuthService: "POST /auth/signup { email, password, role }"
    AuthService->>AuthService: Hash password with Argon2
    AuthService->>Database: "Create User (emailVerified: false)"
    AuthService->>AuthService: Generate UUID verification token
    AuthService->>Database: Store hashed token in Token table
    AuthService->>EmailService: Send verification email
    AuthService->>RabbitMQ: Publish "user.created" event
    AuthService->>Frontend: "{ message: 'Please verify your email' }"
```

**Diagram Explanation:**
1.  **User Submits Form**: The user fills out the registration form and clicks Sign Up.
2.  **Password Hashing**: The backend hashes the password using Argon2 (so it is never stored in plain text).
3.  **User Created**: A new User record is saved to the database with `emailVerified: false`.
4.  **Token Generation**: A random verification token (UUID) is generated, hashed, and stored in the Token table.
5.  **Email Sent**: The system sends an email to the user containing a verification link with the token.
6.  **Event Published**: The `user.created` event is published to RabbitMQ so other services (like analytics or welcome notifications) can react.
7.  **User Informed**: The frontend shows a message asking the user to check their email.

### 4.2 Email Verification Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant AuthService as "auth-service"
    participant Database as "PostgreSQL"

    User->>Frontend: "Clicks verification link (/verify-email?token=xxx&email=xxx)"
    Frontend->>AuthService: "POST /auth/verify-email { token, email }"
    AuthService->>Database: Find user by email
    AuthService->>Database: Find all EMAIL_VERIFY tokens for user
    AuthService->>AuthService: Verify token hash with Argon2
    
    alt Token Valid
        AuthService->>Database: Update user.emailVerified = true
        AuthService->>Database: Delete all EMAIL_VERIFY tokens
        AuthService->>RabbitMQ: Publish "user.email.verified" event
        AuthService->>Frontend: "{ message: 'Email verified' }"
    else Token Invalid/Expired
        AuthService->>Frontend: 403 Forbidden
    end
```

**Diagram Explanation:**
1.  **User Clicks Link**: The user clicks the verification link in their email (e.g., `/verify-email?token=abc&email=user@example.com`).
2.  **Token Lookup**: The backend finds the user by email and retrieves all `EMAIL_VERIFY` tokens for that user.
3.  **Token Verification**: The backend uses Argon2 to compare the provided token against the stored hashes.
4.  **If Valid**:
    *   The user emailVerified field is set to `true`.
    *   All verification tokens for that user are deleted (to prevent reuse).
    *   A `user.email.verified` event is published to RabbitMQ.
5.  **If Invalid/Expired**: A 403 Forbidden error is returned.

### 4.3 Login Flow with Token Generation

```mermaid
sequenceDiagram
    participant Frontend
    participant AuthService as "auth-service"
    participant Database as "PostgreSQL"

    Frontend->>AuthService: "POST /auth/signin { email, password }"
    AuthService->>Database: Find user by email
    AuthService->>AuthService: Verify password with Argon2
    
    alt Password Correct AND Email Verified
        AuthService->>AuthService: "Sign access_token (15m TTL)"
        AuthService->>AuthService: "Sign refresh_token (7d TTL)"
        AuthService->>Database: Store hashed refresh_token
        AuthService->>Frontend: "{ access_token, refresh_token, user }"
        Frontend->>Frontend: Store in localStorage
    else Invalid
        AuthService->>Frontend: 403 Forbidden
    end
```

**Diagram Explanation:**
1.  **User Submits Credentials**: The user enters their email and password and clicks Login.
2.  **Password Verification**: The backend looks up the user by email and uses Argon2 to verify the password.
3.  **Pre-conditions**: The password must match AND the email must be verified for login to succeed.
4.  **Token Generation**:
    *   An **access_token** (15-minute lifespan) is signed for short-term authentication.
    *   A **refresh_token** (7-day lifespan) is signed for getting new access tokens without re-entering credentials.
    *   The hashed refresh token is stored in the database.
5.  **Frontend Storage**: The frontend stores both tokens in `localStorage` for subsequent API requests.
6.  **If Invalid**: A 403 Forbidden error is returned.

### 4.4 Password Reset Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant AuthService as "auth-service"
    participant EmailService as "Email"

    User->>Frontend: Clicks "Forgot Password"
    Frontend->>AuthService: "POST /auth/forgot-password { email }"
    AuthService->>AuthService: "Generate UUID reset token (15min TTL)"
    AuthService->>Database: Store hashed token
    AuthService->>EmailService: Send reset email

    User->>Frontend: "Clicks reset link, enters new password"
    Frontend->>AuthService: "POST /auth/reset-password { token, newPassword }"
    AuthService->>AuthService: Verify token hash
    AuthService->>AuthService: Hash new password with Argon2
    AuthService->>Database: Update user.password
    AuthService->>Database: Delete all PASSWORD_RESET tokens
    AuthService->>RabbitMQ: Publish "user.password.changed" event
```

**Diagram Explanation:**
1.  **User Requests Reset**: User clicks Forgot Password and enters their email.
2.  **Token Generation**: The backend generates a UUID reset token with a 15-minute TTL.
3.  **Email Sent**: The reset link is emailed to the user.
4.  **User Resets Password**: User clicks the link, enters a new password.
5.  **Validation**: The backend verifies the token hash and hashes the new password.
6.  **Cleanup**: All PASSWORD_RESET tokens for that user are deleted.
7.  **Event Published**: A `user.password.changed` event is published (useful for security alerts).

**Key File:** [auth.service.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/auth-service/src/auth/auth.service.ts)

---

## 5. How the Notification Service Handles Events

The `notification-service` is a **hybrid application** that operates as:
1.  An **HTTP server** for REST APIs (fetch notifications, mark as read)
2.  A **NestJS microservice consumer** listening to RabbitMQ
3.  A **WebSocket server** for real-time push notifications

### 5.1 Event Consumption Architecture

```typescript
// notification-service/src/main.ts

// 1. Create HTTP app
const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('notification');

// 2. Connect as RabbitMQ microservice
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.RMQ,
  options: {
    urls: [process.env.RABBITMQ_URL],
    queue: 'notification_queue',
    queueOptions: { durable: true },
    noAck: true,  // Auto-acknowledge messages
  },
});

// 3. Manually bind queue to topic exchange
const channel = await conn.createChannel();
await channel.assertExchange('auth.events', 'topic', { durable: true });
await channel.assertQueue('notification_queue', { durable: true });
await channel.bindQueue('notification_queue', 'auth.events', 'user.#');

// 4. Start both HTTP and microservice
await app.startAllMicroservices();
await app.listen(4004);
```

### 5.2 Event Routing with @EventPattern

Each consumer uses the `@EventPattern()` decorator to route incoming messages:

```typescript
// notification-service/src/notifications/consumers/auth.consumer.ts

@Controller()
export class AuthConsumer {
  constructor(private readonly notificationsService: NotificationsService) {}

  @EventPattern('user.created')
  async handleUserCreated(@Payload() data: any) {
    const user = data.data;
    await this.notificationsService.createNotification({
      userId: user.userId,
      eventType: 'USER_CREATED',
      message: `Welcome ${user.firstName}! Your account is ready.`,
    });
  }

  @EventPattern('user.email.verified')
  async handleUserEmailVerified(@Payload() data: any) {
    // ... create notification
  }
}
```

### 5.3 Notification Persistence

When a notification is created, it is:
1.  Saved to PostgreSQL (`notify_db`)
2.  Pushed in real-time via WebSocket

```typescript
// notification-service/src/notifications/notifications.service.ts

async createNotification(payload: Partial<Notification>) {
  // 1. Save to database
  const notification = this.repo.create({ ...payload, status: 'SENT' });
  const saved = await this.repo.save(notification);

  // 2. Push via WebSocket
  if (saved.userId) {
    this.gateway.sendNotificationToUser(saved.userId, saved.eventType, saved);
  } else {
    this.gateway.broadcastNotification(saved.eventType, saved);
  }

  return saved;
}
```

**Key Files:**
-   [main.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/notification-service/src/main.ts)
-   [auth.consumer.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/notification-service/src/notifications/consumers/auth.consumer.ts)
-   [order.consumer.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/notification-service/src/notifications/consumers/order.consumer.ts)
-   [notifications.service.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/notification-service/src/notifications/notifications.service.ts)

---

## 6. Why and How Socket.io is Used

### 6.1 Why Use Socket.io?

**Problem**: HTTP is request-response based. The client has to poll the server to get updates.

**Solution**: WebSockets provide a persistent, bidirectional connection. The server can push updates to the client instantly.

| Use Case                     | Without Socket.io          | With Socket.io              |
|------------------------------|----------------------------|-----------------------------|
| Order status changed         | Client polls every 10s     | Server pushes immediately   |
| New order for restaurant     | Owner refreshes manually   | Toast notification appears  |
| Delivery driver picked up    | Customer doesn't know      | Real-time tracking update   |

### 6.2 Socket.io Server (Backend)

The `NotificationsGateway` uses NestJS's WebSocket decorators:

```typescript
// notification-service/src/notifications/notifications.gateway.ts

@WebSocketGateway({
  cors: { origin: '*' },
  namespace: 'notifications',  // Clients connect to /notifications
})
export class NotificationsGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private userSockets: Map<string, string[]> = new Map();  // userId -> socketIds

  handleConnection(client: Socket) {
    const userId = client.handshake.query.userId as string;
    if (userId) {
      client.join(`user_${userId}`);  // Join user-specific room
      this.logger.log(`Client ${client.id} joined room user_${userId}`);
    }
  }

  // Called by NotificationsService when a new notification is created
  sendNotificationToUser(userId: string, eventType: string, payload: any) {
    this.server.to(`user_${userId}`).emit('notification', {
      eventType,
      ...payload,
    });
  }

  broadcastNotification(eventType: string, payload: any) {
    this.server.emit('notification', { eventType, ...payload });
  }
}
```

### 6.3 Socket.io Client (Frontend)

The frontend uses a React Context to manage the socket connection:

```typescript
// frontend/src/context/socket-context.tsx

export function SocketProvider({ children }: { children: React.ReactNode }) {
  const { user } = useAuth();
  const [socket, setSocket] = useState<Socket | null>(null);

  useEffect(() => {
    if (!user) return;

    const socketUrl = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8080';
    
    const newSocket = io(`${socketUrl}/notifications`, {
      query: { userId: user.id },  // Server uses this to join room
      transports: ['websocket', 'polling'],
    });

    newSocket.on('notification', (data: any) => {
      toast(data.eventType || 'New Notification', {
        description: data.message,
      });
    });

    setSocket(newSocket);
    return () => { newSocket.disconnect(); };
  }, [user]);

  return (
    <SocketContext.Provider value={{ socket, isConnected }}>
      {children}
    </SocketContext.Provider>
  );
}
```

### 6.4 Data Flow: Order Created → Real-time Notification

```mermaid
sequenceDiagram
    participant Customer as "Customer (Frontend)"
    participant OrderService as "order-service"
    participant RabbitMQ
    participant NotifService as "notification-service"
    participant WebSocket as "Socket.io Server"
    participant RestaurantOwner as "Restaurant Owner (Frontend)"

    Customer->>OrderService: POST /order/api/v1/orders/create
    OrderService->>OrderService: Create order in DB
    OrderService->>RabbitMQ: "sendToQueue('notification_queue', ORDER_CREATED)"
    OrderService->>Customer: "{ orderId, status: PENDING }"

    RabbitMQ-->>NotifService: "@EventPattern('ORDER_CREATED')"
    NotifService->>NotifService: Save notification to DB
    NotifService->>WebSocket: "gateway.sendNotificationToUser(ownerId, ...)"
    WebSocket-->>RestaurantOwner: "emit('notification', { eventType: 'ORDER_CREATED', ... })"
    RestaurantOwner->>RestaurantOwner: "Toast: 'New order received!'"
```

**Key Files:**
-   [notifications.gateway.ts](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/notification-service/src/notifications/notifications.gateway.ts)
-   [socket-context.tsx](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/frontend/src/context/socket-context.tsx)

---

## 7. RabbitMQ Architecture & Configuration

This is a common point of confusion. Understanding the difference between **Topic Exchanges** and **Direct Queues** is key.

### 7.1 The Two Patterns in Use

| Service          | Messaging Pattern | Exchange/Queue          | Visibility in RabbitMQ UI |
|------------------|-------------------|-------------------------|---------------------------|
| `auth-service`   | Topic Exchange    | `auth.events` (exchange)| **No messages shown**     |
| `order-service`  | Direct Queue      | `notification_queue`    | Messages visible          |
| `payment-service`| Direct Queue      | `PAYMENT_EVENTS`        | Messages visible          |

### 7.2 Why Auth Events Don't Show in Queues

```
                                   ┌─────────────────────────────────┐
                                   │         RabbitMQ Broker         │
                                   │                                 │
auth-service ──publish──▶ [auth.events exchange] ──route──▶ [notification_queue] ──▶ notification-service
                             (topic type)                     (bound with user.#)
                             ┌────────────────┐
                             │ NO STORAGE     │
                             │ Just routing   │
                             └────────────────┘
```

**Key Insight**: An exchange is a **router**, not a **storage**. It doesn't hold messages; it forwards them to bound queues.

### 7.3 Why the Queue Looks Empty

| Scenario                              | Result                                                                 |
|---------------------------------------|------------------------------------------------------------------------|
| `notification-service` is **running** | Messages are consumed in milliseconds. They don't linger.              |
| `notification-service` is **stopped** | Messages wait in `notification_queue` (you'll see them in RabbitMQ UI).|
| `noAck: true` is configured           | Consumer auto-acknowledges, so messages disappear upon receipt.        |

### 7.4 How to See Messages (Debugging)

1.  **Stop** `notification-service`:
    ```bash
    docker stop notification-service
    ```
2.  **Trigger an event** (e.g., register a new user).
3.  **Check RabbitMQ UI**: `http://localhost:15672` → Queues → `notification_queue`
4.  You should now see 1+ messages waiting.
5.  **Start** the service again:
    ```bash
    docker start notification-service
    ```

### 7.5 RabbitMQ vs Kafka: Detailed Comparison

#### Feature Comparison Table

| Feature                  | RabbitMQ                           | Kafka                                |
|--------------------------|------------------------------------|--------------------------------------|
| **Architecture Model**   | Message Broker (push-based)        | Distributed Event Log (pull-based)   |
| **Message Persistence**  | Deleted after acknowledgement      | Retained for configurable period     |
| **Consumer Tracking**    | Broker tracks acknowledgements     | Consumer tracks offset               |
| **Visibility After Consume** | Messages disappear             | Messages remain visible in partition |
| **Ordering Guarantee**   | Per-queue FIFO                     | Per-partition FIFO                   |
| **Replay Capability**    | Not possible                       | Replay from any offset            |
| **Routing Flexibility**  | Exchanges, bindings, patterns   | Topic-based only                  |
| **Latency**              | Very low (sub-millisecond)      | Higher (batching optimized)          |
| **Throughput**           | Moderate (10K-50K msg/sec)         | Very High (millions msg/sec)         |
| **Operational Complexity** | Simple to operate             | Requires ZooKeeper/KRaft          |
| **Memory Footprint**     | Lightweight                     | JVM-based, memory-intensive       |
| **Best For**             | Task queues, RPC, routing          | Event sourcing, streaming, analytics |

---

### 7.6 Why RabbitMQ is the Right Choice for This Project

For this **Food Delivery Aggregator**, RabbitMQ provides several key advantages over Kafka:

#### 1. Lower Operational Complexity

```
RabbitMQ: 1 container (rabbitmq:3-management)
Kafka:    3+ containers (kafka broker, zookeeper/kraft, schema-registry)
```

For a class project or startup MVP, RabbitMQ's simplicity is invaluable. You get a **management UI out-of-the-box** at `http://localhost:15672`.

#### 2. Flexible Routing with Exchanges

RabbitMQ's exchange types enable sophisticated routing:

```mermaid
flowchart LR
    subgraph "Topic Exchange Example"
        PUB["auth-service"] --> EX["auth.events - (topic exchange)"]
        EX -->|"user.created"| Q1["notification_queue"]
        EX -->|"user.role.updated"| Q2["admin_audit_queue"]
        EX -->|"user.*"| Q3["analytics_queue"]
    end
```

**Diagram Explanation:**
1.  **Publisher**: The `auth-service` publishes various user events (like `user.created`, `user.role.updated`) to the `auth.events` topic exchange.
2.  **Exchange Routing**: The topic exchange routes messages based on pattern matching:
    *   `user.created` matches the `notification_queue` (bound with `user.#` pattern).
    *   `user.role.updated` matches the `admin_audit_queue` (for security audits).
    *   `user.*` matches the `analytics_queue` (for all user events).
3.  **Multiple Consumers**: The same message can be delivered to multiple queues if the patterns match—a powerful fan-out capability.
4.  **Why This Matters**: New services can subscribe to existing events without modifying the publisher.

**Kafka doesn't have this.** In Kafka, you'd need separate topics or consumer-side filtering.

#### 3. Request-Reply Pattern (RPC)

RabbitMQ natively supports RPC patterns for synchronous messaging:

```typescript
// Potential future use: Get user details from auth-service
const user = await rabbitMQ.sendAndWait('auth.rpc.get_user', { userId: '123' });
```

Kafka is designed for **fire-and-forget** streaming, not request-reply.

#### 4. Per-Message Acknowledgement

RabbitMQ allows fine-grained control over message acknowledgement:

```typescript
// Manual ack after successful processing
channel.consume('queue', async (msg) => {
  try {
    await processOrder(msg);
    channel.ack(msg);      // Success: remove from queue
  } catch (err) {
    channel.nack(msg, false, true);  // Failure: requeue
  }
});
```

This is perfect for **order processing** where you want messages to retry on failure.

#### 5. Lower Latency for Real-time Notifications

RabbitMQ's push model delivers messages to consumers instantly:

| Scenario                    | RabbitMQ          | Kafka                     |
|-----------------------------|-------------------|---------------------------|
| Order created → notification | **< 5ms**         | 100-500ms (poll interval) |
| Payment success → order update | **< 5ms**       | 100-500ms                 |

For a food delivery app where customers expect **instant status updates**, this matters.

#### 6. Memory Efficiency

RabbitMQ runs efficiently with minimal resources:

| Metric         | RabbitMQ            | Kafka                    |
|----------------|---------------------|---------------------------|
| Base Memory    | ~100-200 MB         | 1-2 GB (JVM heap)         |
| Docker Image   | ~150 MB             | ~500 MB                   |
| CPU Idle       | Minimal             | Constant (log compaction) |

#### When Kafka Would Be Better

Kafka would be the right choice if you needed:
- **Event Replay**: Reprocess all orders from the last month
- **Stream Processing**: Real-time analytics on order trends
- **Massive Scale**: Handling millions of messages per second
- **Event Sourcing**: Rebuilding state from event history

> [!NOTE]
> **Summary**: RabbitMQ is ideal for this project because it provides **low-latency message delivery**, **flexible routing**, **simple operations**, and **reliable message acknowledgement**—all critical for a real-time food delivery system. Kafka's strengths in event streaming and replay aren't needed here.

---

## 8. Database Architecture (Database-per-Service)

Each microservice owns its own database, ensuring loose coupling.

```mermaid
flowchart TB
    subgraph "auth-service"
        AUTH_SVC["auth-service"]
        AUTH_DB[("auth_db - PostgreSQL - :5439")]
    end

    subgraph "order-service"
        ORDER_SVC["order-service"]
        ORDER_DB[("order_db - PostgreSQL - :5440")]
    end

    subgraph "payment-service"
        PAY_SVC["payment-service"]
        PAY_DB[("payment_db - PostgreSQL - :5435")]
    end

    subgraph "notification-service"
        NOTIF_SVC["notification-service"]
        NOTIF_DB[("notify_db - PostgreSQL - :5441")]
    end

    AUTH_SVC --> AUTH_DB
    ORDER_SVC --> ORDER_DB
    PAY_SVC --> PAY_DB
    NOTIF_SVC --> NOTIF_DB
```

**Diagram Explanation:**
1.  **Database-per-Service Pattern**: Each microservice has its own dedicated PostgreSQL database. This ensures:
    *   **Loose Coupling**: Services can evolve their schemas independently.
    *   **Fault Isolation**: A database issue in `payment-service` does not affect `auth-service`.
    *   **Technology Freedom**: Each service could use a different database type if needed.
2.  **Ports**: Each database runs on a different port (5439, 5440, etc.) in Docker/K8s.
3.  **No Cross-Database Joins**: Services CANNOT directly query other services databases. They must communicate via APIs or events.
4.  **Trade-off**: This pattern increases operational complexity (4 databases to manage) but provides better scalability and resilience.

### 8.1 Entity Relationship Diagrams

#### 8.1.1 Auth Database ER Diagram

```mermaid
erDiagram
    USER {
        string id PK "cuid()"
        string email UK
        string password "Argon2 hash"
        string firstName
        string lastName
        Role role "CUSTOMER|RESTAURANT|DELIVERY|ADMIN"
        boolean emailVerified "default: false"
        datetime createdAt
        datetime updatedAt
    }
    
    TOKEN {
        string id PK "cuid()"
        TokenType type "REFRESH|PASSWORD_RESET|EMAIL_VERIFY"
        string tokenHash "Argon2 hash"
        datetime expiresAt
        string userId FK
        datetime createdAt
    }
    
    USER ||--o{ TOKEN : "has many"
```

#### 8.1.2 Order Database ER Diagram

```mermaid
erDiagram
    RESTAURANT {
        uuid id
        string name
        string location
        string ownerId
        datetime createdAt
        datetime updatedAt
    }

    ITEM {
        uuid id
        string name
        string description
        decimal unitPrice
        uuid restaurantId
        datetime createdAt
        datetime updatedAt
    }

    ORDER {
        uuid id
        string userId
        uuid restaurantId
        string customerName
        string customerEmail
        decimal totalPrice
        string status
        string deliveryStatus
        string driverId
        boolean isPaid
        string couponCode
        decimal discount
        datetime createdAt
        datetime updatedAt
    }

    ORDER_ITEM {
        uuid id
        uuid orderId
        uuid itemId
        int quantity
        decimal price
    }

    REVIEW {
        uuid id
        uuid orderId
        string userId
        int rating
        string comment
        datetime createdAt
    }

    COUPON {
        string code
        decimal discount
        boolean isActive
        datetime createdAt
    }

    RESTAURANT ||--o{ ITEM : has
    RESTAURANT ||--o{ ORDER : receives
    ORDER ||--o{ ORDER_ITEM : contains
    ITEM ||--o{ ORDER_ITEM : included_in
    ORDER ||--o| REVIEW : has

```

#### 8.1.3 Payment Database ER Diagram

```mermaid
erDiagram
    PAYMENT {
        uuid id PK
        string orderId "FK to order_db.Order"
        string userId "FK to auth_db.User"
        int amount "in cents"
        string currency "default: ETB"
        string status "PENDING|COMPLETED|FAILED|REFUNDED"
        string gateway "STRIPE|CHAPA"
        string paymentMethod
        json gatewayResponse
        datetime createdAt
        datetime updatedAt
    }
    
    TRANSACTION {
        uuid id PK
        uuid paymentId FK
        string gatewayTransactionId
        string status "PENDING|SUCCESS|FAILURE"
        json response
        datetime createdAt
    }
    
    PAYMENT ||--o{ TRANSACTION : "has many"
```

#### 8.1.4 Notification Database ER Diagram

```mermaid
erDiagram
    NOTIFICATION {
        uuid id PK
        string userId "FK to auth_db.User"
        string eventType "USER_CREATED|ORDER_CREATED|etc."
        string message
        string channel "default: IN_APP"
        string status "default: SENT"
        boolean isRead "default: false"
        datetime createdAt
    }
```

---

### 8.2 Auth Database Schema (Prisma)

```prisma
enum Role {
  CUSTOMER
  RESTAURANT
  DELIVERY
  ADMIN
}

enum TokenType {
  REFRESH
  PASSWORD_RESET
  EMAIL_VERIFY
}

model User {
  id            String   @id @default(cuid())
  email         String   @unique
  password      String   // Argon2 hash
  firstName     String?
  lastName      String?
  role          Role     @default(CUSTOMER)
  emailVerified Boolean  @default(false)
  tokens        Token[]
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}

model Token {
  id        String    @id @default(cuid())
  type      TokenType
  tokenHash String    // Argon2 hash of actual token
  expiresAt DateTime
  userId    String
  user      User      @relation(...)
}
```

**Key Design Decisions:**
-   **Tokens stored as hashes**: Even if DB is compromised, tokens can't be used
-   **Cascade delete**: Deleting a user deletes all their tokens
-   **Role enum**: Type-safe role management

### 8.3 Order Database Schema

```prisma
enum OrderStatus { PENDING, PREPARING, READY, COMPLETED, CANCELLED }
enum DeliveryStatus { PENDING, PICKED_UP, ON_THE_WAY, DELIVERED }

model Restaurant {
  id        String   @id @default(uuid())
  name      String   @unique
  location  String?
  ownerId   String   // References auth_db.User.id
  items     Item[]
  orders    Order[]
}

model Order {
  id             String         @id @default(uuid())
  userId         String         // References auth_db.User.id
  restaurantId   String
  customerName   String?
  customerEmail  String?
  totalPrice     Decimal        @db.Decimal(10, 2)
  status         OrderStatus    @default(PENDING)
  deliveryStatus DeliveryStatus?
  driverId       String?        // References auth_db.User.id
  isPaid         Boolean        @default(false)
  couponCode     String?
  discount       Decimal?
  items          OrderItem[]
  review         Review?
}

model Coupon {
  code      String   @id
  discount  Decimal
  isActive  Boolean  @default(true)
}
```

### 8.4 Payment Database Schema

```prisma
model Payment {
  id               String        @id @default(uuid())
  orderId          String        // References order_db.Order.id
  userId           String        // References auth_db.User.id
  amount           Int           // In cents
  currency         String        @default("ETB")
  status           String        @default("PENDING")
  gateway          String        // "STRIPE" or "CHAPA"
  gatewayResponse  Json?
  transactions     Transaction[]
}

model Transaction {
  id                   String   @id @default(uuid())
  paymentId            String
  gatewayTransactionId String?
  status               String   // PENDING, SUCCESS, FAILURE
  response             Json?
  payment              Payment  @relation(...)
}
```

### 8.5 Notification Database Schema (TypeORM)

```typescript
@Entity('notifications')
export class Notification {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userId: string;

  @Column()
  eventType: string;  // USER_CREATED, ORDER_CREATED, etc.

  @Column({ nullable: true })
  message: string;

  @Column({ default: 'IN_APP' })
  channel: string;  // IN_APP, EMAIL, SMS

  @Column({ default: 'SENT' })
  status: string;

  @Column({ default: false })
  isRead: boolean;

  @CreateDateColumn()
  createdAt: Date;
}
```

**Key Files:**
-   [auth schema](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/auth-service/prisma/schema.prisma)
-   [order schema](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/order-service/prisma/schema.prisma)
-   [payment schema](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/payment-service/prisma/schema.prisma)
-   [notification entity](file:///home/mistire/Projects/class-projects/food-delivery-aggregator/notification-service/src/notifications/entities/notification.entity.ts)

---

## 9. Security Patterns

### 9.1 Password Hashing with Argon2

**Why Argon2?**
-   Winner of the Password Hashing Competition (2015)
-   Memory-hard: Resistant to GPU/ASIC attacks
-   Configurable time/memory cost

```typescript
// auth-service/src/auth/auth.service.ts

import * as argon from 'argon2';

// Hashing a password
const hash = await argon.hash(dto.password);

// Verifying a password
const isValid = await argon.verify(storedHash, providedPassword);
```

### 9.2 Secure Token Storage

**Problem**: If refresh tokens are stored in plaintext, a database breach exposes all user sessions.

**Solution**: Store tokens as Argon2 hashes.

```typescript
// Creating a token
const token = crypto.randomUUID();
const tokenHash = await argon.hash(token);
await this.prisma.token.create({
  data: { type: 'REFRESH', tokenHash, expiresAt: ..., userId }
});

// Validating a token
const tokens = await this.prisma.token.findMany({ where: { userId, type: 'REFRESH' } });
for (const storedToken of tokens) {
  if (await argon.verify(storedToken.tokenHash, providedToken)) {
    return storedToken;  // Valid!
  }
}
```

### 9.3 JWT Token Structure

```json
{
  "sub": "clx1234567890",     // User ID
  "email": "user@example.com",
  "role": "CUSTOMER",
  "iat": 1705850000,          // Issued at
  "exp": 1705850900           // Expires at (15 min for access token)
}
```

---

## 10. Resilience Patterns & Fault Tolerance

### 10.1 Circuit Breaker (order-service)

The order-service uses the `opossum` library to implement the circuit breaker pattern for external calls:

```javascript
// order-service/src/core/utils/resilience.js

import CircuitBreaker from 'opossum';

const options = {
  timeout: 3000,                    // Fail if call takes > 3s
  errorThresholdPercentage: 50,     // Open circuit if 50% fail
  resetTimeout: 30000               // Try again after 30s
};

export function createBreaker(action) {
  const breaker = new CircuitBreaker(action, options);
  
  breaker.on('open', () => console.log('Circuit breaker OPENED'));
  breaker.on('halfOpen', () => console.log('Circuit breaker HALF_OPENED'));
  breaker.on('close', () => console.log('Circuit breaker CLOSED'));
  
  return breaker;
}
```

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open : 50% failures
    Open --> HalfOpen : after 30s
    HalfOpen --> Closed : success
    HalfOpen --> Open : failure
```

**Diagram Explanation:**
1.  **Closed State (Normal)**: The circuit breaker starts closed—requests flow through normally. It monitors the failure rate.
2.  **Open State (Protecting)**: If 50% of requests fail, the circuit opens. ALL subsequent requests immediately fail (without even trying the external service). This protects the system from cascading failures.
3.  **Half-Open State (Testing)**: After 30 seconds, the circuit moves to half-open. It allows ONE test request through:
    *   If it succeeds, the circuit closes again (back to normal).
    *   If it fails, the circuit reopens (back to protecting).
4.  **Why This Matters**: Without a circuit breaker, if an external payment gateway is down, your entire order service would hang waiting for timeouts. With a circuit breaker, requests fail fast, and users see an error immediately instead of a frozen page.

### 10.2 Why Opossum is Only in Order-Service

| Service              | External Dependencies                      | Needs Circuit Breaker? |
|----------------------|--------------------------------------------|------------------------|
| **order-service**    | Payment gateway (simulated)             | **Yes**                |
| **auth-service**     | Only local DB + RabbitMQ                | No                     |
| **payment-service**  | Stripe/Chapa APIs                       | Recommended for Prod   |
| **notification-service** | Only local DB + Socket.io          | No                     |

### 10.3 RabbitMQ Connection Retry

```typescript
// auth-service/src/rabbitmq/rabbitmq.service.ts

private async connect() {
  for (let attempt = 1; attempt <= 10; attempt++) {
    try {
      this.connection = await amqp.connect(rabbitmqUrl);
      this.channel = await this.connection.createConfirmChannel();
      return;  // Success!
    } catch (error) {
      this.logger.warn(`Attempt ${attempt} failed. Retrying in 2s...`);
      await new Promise(r => setTimeout(r, 2000));
    }
  }
  this.logger.error('RabbitMQ connection failed. Running without events.');
}
```

### 10.4 Graceful Degradation

If RabbitMQ is unavailable, the auth-service continues to work:

```typescript
async onModuleInit() {
  await this.connect().catch((error) => {
    this.logger.warn('RabbitMQ unavailable. Events will not be published.');
  });
}

async publish(routingKey: string, message: any): Promise<boolean> {
  if (!this.channel) {
    this.logger.debug(`Event '${routingKey}' not published - no channel.`);
    return false;  // Silently fail
  }
  // ... publish logic
}
```

### 10.5 Resilience Pattern Comparison

| Pattern                | Implementation                  | Purpose                                    |
|------------------------|---------------------------------|--------------------------------------------|
| **Circuit Breaker**    | opossum in order-service        | Fail fast on external service failures     |
| **Retry with Backoff** | RabbitMQ connection in auth     | Handle transient failures during startup   |
| **Graceful Degradation**| RabbitMQ publish in auth       | Continue core operations if optional features fail |
| **Timeout**            | opossum 3s timeout              | Prevent hanging on slow external calls     |

---

## 11. Frontend Architecture

### 11.1 Frontend API Client with Transparent Token Refresh

The frontend uses a custom `ApiClient` class that automatically refreshes expired access tokens:

```typescript
// frontend/src/lib/api.ts

async fetch<T>(endpoint: string, options: RequestInit = {}, isRetry = false): Promise<T> {
  const token = localStorage.getItem('token');
  const refreshToken = localStorage.getItem('refreshToken');

  const response = await fetch(url, {
    headers: { Authorization: `Bearer ${token}` },
    ...options,
  });

  // If 401 and we have a refresh token, try to refresh
  if (response.status === 401 && !isRetry && refreshToken) {
    try {
      const refreshResponse = await fetch('/auth/refresh-token', {
        method: 'POST',
        body: JSON.stringify({ refreshToken })
      });

      if (refreshResponse.ok) {
        const tokens = await refreshResponse.json();
        localStorage.setItem('token', tokens.access_token);
        localStorage.setItem('refreshToken', tokens.refresh_token);
        
        // Retry the original request with new token
        return this.fetch<T>(endpoint, options, true);
      }
    } catch (err) {
      // Refresh failed, logout user
      localStorage.clear();
      window.location.href = '/login';
    }
  }

  return response.json();
}
```

**Flow:**
1.  Request made with access token
2.  If 401, use refresh token to get new tokens
3.  Retry original request
4.  If refresh fails, logout user

---

## 12. Infrastructure: Docker & Kubernetes

### 12.1 Docker vs. Kubernetes: An Architecture Comparison

We evolved the system from a simple Docker Compose setup to a robust Kubernetes cluster.

| Feature | Docker Compose (Old) | Kubernetes (New) | why K8s Wins |
| :--- | :--- | :--- | :--- |
| **Orchestration** | Single-host only. | Multi-host cluster. | **Scale**: Can run on 1000s of servers. |
| **Load Balancing** | Static Nginx container. | **Ingress Controller**. Native, dynamic load balancing (Layer 7). | **Automation**: Automatically discovers new services. |
| **Recovery** | Basic restart. | **Self-Healing**. K8s actively monitors health and kills/replaces "sick" pods. | **Reliability**: Proactive health management. |
| **Scaling** | Manual. | **Autoscaling (HPA)**. Can auto-scale based on CPU/RAM usage. | **Efficiency**: Uses resources only when needed. |
| **Networking** | Internal Docker network. | **Service Discovery**. Stable ClusterIPs and DNS. | **Stability**: Pods can move; IPs stay stable. |

### 12.2 Kubernetes Architecture Deep Dive

```mermaid
graph TD
    subgraph "K8s Cluster (Minikube Node)"
        Ingress[("Ingress Controller\n(Load Balancer)")]
        
        subgraph "Namespaces: default"
            SVC_AUTH[Service: auth-service]
            POD_AUTH[Pod: auth-service]
            
            SVC_ORDER[Service: order-service]
            POD_ORDER[Pod: order-service]
            
            SVC_PAY[Service: payment-service]
            POD_PAY[Pod: payment-service]
            
            SVC_DB[Service: auth-db, order-db...]
            POD_DB[StatefulSet: auth-db-0...]
        end
    end
    
    Client --> Ingress
    Ingress --> SVC_AUTH
    Ingress --> SVC_ORDER
    Ingress --> SVC_PAY
    SVC_AUTH --> SVC_DB
```

**Diagram Explanation:**
1.  **Cluster Boundary**: The large box represents the Kubernetes cluster (Minikube in development). Everything inside runs on the cluster.
2.  **Ingress Controller**: This is the front door of the cluster. It is the only component exposed to the outside world. It receives all incoming HTTP traffic.
3.  **Services**: Each microservice has a Kubernetes `Service` (e.g., `SVC_AUTH`). A Service provides a stable internal IP address and DNS name that does not change even if the underlying Pods restart.
4.  **Pods**: These are the actual running containers (e.g., `POD_AUTH`). If a Pod crashes, Kubernetes automatically creates a new one.
5.  **StatefulSets**: Databases use StatefulSets (instead of Deployments) because they need stable network identities (`auth-db-0`) and persistent storage.
6.  **Traffic Flow**: External client to Ingress to Service to Pod. The client never talks directly to Pods.

**Core Components & Their Role**

1.  **Ingress (`shared/ingress.yaml`)**:
    *   Acts as the unified entry point (Layer 7 Load Balancer).
    *   Terminates endpoints and routes traffic based on URL paths (`/auth`, `/order`, etc.).

2.  **Deployments (`apps/*.yaml`)**:
    *   Manage stateless microservices (`auth`, `order`, `payment`, `notification`, `frontend`).
    *   Handle **Rolling Updates** (zero-downtime deployments).

3.  **StatefulSets (`db/databases.yaml`)**:
    *   Manage stateful applications (PostgreSQL databases, RabbitMQ, Redis).
    *   Provide stable network identities (`auth-db-0`) and stable persistent storage.

4.  **Services (ClusterIP)**:
    *   Provide stable internal IP addresses and DNS names (e.g., `auth-db`, `rabbitmq`).

5.  **ConfigMaps & Secrets**:
    *   **ConfigMap**: Stores non-sensitive configuration.
    *   **Secret**: Stores sensitive data (DB passwords, keys) encoded in Base64.

### 12.3 Kubernetes Fault Tolerance

*   **Self-Healing**: If a container crashes, Kubelet restarts it.
*   **Health Probes**:
    *   **Liveness**: "Is app broken?" -> Restart.
    *   **Readiness**: "Is app initializing?" -> Stop traffic.
*   **Rolling Updates**: K8s spins up new versions and waits for them to be ready before killing old ones.
*   **Persistent Volume Claims (PVC)**: Ensures database data survives pod restarts.

### 12.4 Docker Compose Network Topology (Local Dev)

```mermaid
flowchart TB
    subgraph "External Access"
        BROWSER["Browser"]
    end

    subgraph "Docker Network"
        subgraph "API Layer"
            NGINX["api-nginx - :8080"]
            GW["api-gateway - :4001"]
        end

        subgraph "Services"
            AUTH["auth-service - :4000"]
            ORDER["order-service - :4002"]
            PAY["payment-service - :4003"]
            NOTIF["notification-service - :4004"]
        end

        subgraph "Infrastructure"
            RMQ["rabbitmq"]
            REDIS["redis"]
        end
    end

    BROWSER --> NGINX
    NGINX --> AUTH
    NGINX --> ORDER
    NGINX --> PAY
    NGINX --> NOTIF
    AUTH --> RMQ
```

**Diagram Explanation:**
1.  **External Access**: Users access the application through their browser.
2.  **API Layer**: The browser connects to `api-nginx` on port 8080. Nginx acts as a reverse proxy, routing requests to the correct backend service.
3.  **Services**: Each microservice runs on its own port inside the Docker network. They can communicate with each other via Docker internal DNS (e.g., `auth-service:4000`).
4.  **Infrastructure**: Supporting services like RabbitMQ (message broker) and Redis (caching) are also part of the Docker network. Services connect to them using their container names (e.g., `rabbitmq:5672`).
5.  **Key Insight**: In Docker Compose, all services share a single network (`food-delivery-net`). This is simpler than Kubernetes but less scalable.

---

## 13. Complete Message Flow Diagrams

### 13.1 User Registration → Welcome Notification

```mermaid
sequenceDiagram
    participant FE as "Frontend"
    participant AUTH as "auth-service"
    participant DB as "auth_db"
    participant RMQ as "RabbitMQ"
    participant NOTIF as "notification-service"
    participant WS as "Socket.io"

    FE->>AUTH: POST /auth/signup
    AUTH->>DB: Insert User
    AUTH->>RMQ: "Publish to auth.events (user.created)"
    AUTH->>FE: "{ message: 'Verify email' }"

    RMQ-->>NOTIF: "@EventPattern('user.created')"
    NOTIF->>NOTIF: Save notification to notify_db
    NOTIF->>WS: "sendNotificationToUser(userId)"
    WS-->>FE: "emit('notification', { 'Welcome!' })"
```

**Diagram Explanation:**
1.  **User Signs Up**: The frontend sends a `POST /auth/signup` request to the auth-service.
2.  **User Saved**: The auth-service inserts the new user into `auth_db`.
3.  **Event Published**: The auth-service publishes a `user.created` event to RabbitMQ topic exchange.
4.  **Frontend Response**: The frontend receives a message to verify the email.
5.  **Notification Service Receives Event**: The notification-service (listening with `@EventPattern('user.created')`) picks up the event from the `notification_queue`.
6.  **Notification Saved**: A Welcome notification is saved to `notify_db`.
7.  **Real-time Push**: If the user browser is connected via Socket.io, the notification is pushed instantly.
8.  **User Sees Welcome**: The frontend displays a welcome toast/notification.

### 13.2 Order → Payment → Real-time Update

```mermaid
sequenceDiagram
    participant Customer
    participant ORDER as "order-service"
    participant PAY as "payment-service"
    participant RMQ as "RabbitMQ"
    participant NOTIF as "notification-service"
    participant WS as "Socket.io"
    participant Restaurant

    Customer->>ORDER: Create Order
    ORDER->>ORDER: "Save order (isPaid: false)"
    ORDER->>RMQ: "notification_queue (ORDER_CREATED)"
    ORDER->>Customer: "{ orderId, paymentUrl }"

    RMQ-->>NOTIF: ORDER_CREATED
    NOTIF->>WS: Notify restaurant owner
    WS-->>Restaurant: "New order!"

    Customer->>PAY: Pay for order
    PAY->>PAY: Process payment
    PAY->>RMQ: "PAYMENT_EVENTS (PAYMENT_SUCCESS)"

    RMQ-->>ORDER: PAYMENT_SUCCESS
    ORDER->>ORDER: "Update order (isPaid: true, status: PREPARING)"
    ORDER->>RMQ: "notification_queue (ORDER_STATUS_UPDATED)"

    RMQ-->>NOTIF: ORDER_STATUS_UPDATED
    NOTIF->>WS: Notify customer + restaurant
    WS-->>Customer: "Order is being prepared!"
    WS-->>Restaurant: "Order paid, start cooking!"
```

**Diagram Explanation:**
This diagram shows the complete lifecycle of an order, from creation to payment to real-time updates:

1.  **Order Creation**:
    *   Customer creates an order then `order-service` saves it with `isPaid: false`.
    *   `order-service` publishes `ORDER_CREATED` to RabbitMQ.
    *   Restaurant owner receives a real-time New order! notification via Socket.io.

2.  **Payment Processing**:
    *   Customer pays via the `payment-service` (which talks to Stripe/Chapa).
    *   `payment-service` publishes `PAYMENT_SUCCESS` to RabbitMQ.

3.  **Order Update**:
    *   `order-service` receives `PAYMENT_SUCCESS` and updates the order: `isPaid: true`, `status: PREPARING`.
    *   `order-service` publishes `ORDER_STATUS_UPDATED` to RabbitMQ.

4.  **Real-time Notifications**:
    *   `notification-service` receives `ORDER_STATUS_UPDATED`.
    *   Both the customer (Order is being prepared!) AND the restaurant (Order paid, start cooking!) receive instant Socket.io updates.

**Key Insight**: This entire flow is **asynchronous and event-driven**. The payment-service does not directly call order-service—they communicate via RabbitMQ events, ensuring loose coupling.

---

This comprehensive document covers all major technical aspects of the Food Delivery Aggregator architecture. For any specific deep-dive, refer to the linked source files.

---

## 14. API Reference Summary

### 14.1 Auth Service (`/auth/*`)

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/auth/signup` | POST | No | Register new user |
| `/auth/signin` | POST | No | Login, returns JWT tokens |
| `/auth/logout` | POST | JWT | Invalidate refresh token |
| `/auth/me` | GET | JWT | Get current user info |
| `/auth/refresh-token` | POST | No | Exchange refresh token for new access token |
| `/auth/verify-email` | GET | No | Verify email with token |
| `/auth/forgot-password` | POST | No | Request password reset email |
| `/auth/reset-password` | POST | No | Reset password with token |
| `/auth/change-password` | POST | JWT | Change password (logged in user) |
| `/auth/users` | GET | JWT (Admin) | List all users |
| `/auth/users/stats` | GET | JWT (Admin) | Get user statistics |
| `/auth/users/profile` | GET | JWT | Get own profile |
| `/auth/users/profile` | PATCH | JWT | Update own profile |
| `/auth/users/:id` | GET | JWT (Admin) | Get user by ID |
| `/auth/users/:id` | DELETE | JWT (Admin) | Delete user |
| `/auth/users/:id/role` | PATCH | JWT (Admin) | Update user role |

### 14.2 Order Service (`/order/*`)

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/order/api/v1/restaurants/create` | POST | JWT (Owner) | Create restaurant |
| `/order/api/v1/restaurants/get-all` | GET | No | List all restaurants |
| `/order/api/v1/restaurants/search` | GET | No | Search restaurants |
| `/order/api/v1/restaurants/get-my-own` | GET | JWT (Owner) | Get own restaurant |
| `/order/api/v1/restaurants/:id` | GET | No | Get restaurant by ID |
| `/order/api/v1/restaurants/:id` | PUT | JWT (Owner) | Update restaurant |
| `/order/api/v1/restaurants/:id` | DELETE | JWT (Owner) | Delete restaurant |
| `/order/api/v1/items/create` | POST | JWT (Owner) | Create menu item |
| `/order/api/v1/items/:itemId` | GET | No | Get item by ID |
| `/order/api/v1/items/restaurant/:id` | GET | No | Get items by restaurant |
| `/order/api/v1/items/:itemId` | PUT | JWT (Owner) | Update item |
| `/order/api/v1/items/:itemId` | DELETE | JWT (Owner) | Delete item |
| `/order/api/v1/orders/create` | POST | JWT | Create order |
| `/order/api/v1/orders/get-by-user` | GET | JWT | Get orders by current user |
| `/order/api/v1/orders/available-for-drivers` | GET | JWT (Driver) | Get orders ready for pickup |
| `/order/api/v1/orders/get-by-driver` | GET | JWT (Driver) | Get orders assigned to driver |
| `/order/api/v1/orders/:orderId` | GET | JWT | Get order by ID |
| `/order/api/v1/orders/restaurant/:id` | GET | JWT (Owner) | Get orders by restaurant |
| `/order/api/v1/orders/:orderId/status` | PATCH | JWT (Owner) | Update order status |
| `/order/api/v1/orders/claim/:orderId` | PATCH | JWT (Driver) | Claim order for delivery |
| `/order/api/v1/orders/delivery-status/:orderId` | PATCH | JWT (Driver) | Update delivery status |
| `/order/api/v1/orders/review/:orderId` | POST | JWT | Create review for order |
| `/order/api/v1/orders/metrics/all` | GET | JWT (Admin) | Get platform metrics |
| `/order/api/v1/orders/restaurant/:id/metrics` | GET | JWT (Owner) | Get restaurant metrics |

### 14.3 Payment Service (`/payment/*`)

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/initiate` | POST | JWT | Initiate payment (Stripe/Chapa) |
| `/order/:orderId` | GET | JWT | Get payment by order ID |
| `/:id` | GET | JWT | Get payment by ID |
| `/sandbox/success/:paymentId` | GET | No | Sandbox payment success callback |
| `/webhook/stripe` | POST | No | Stripe webhook handler |
| `/webhook/chapa` | POST | No | Chapa webhook handler |

### 14.4 Notification Service (`/notification/*`)

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/notification` | GET | No | Health check |
| `/notification/api/v1` | GET | No | List all notifications |
| `/notification/api/v1/user/:userId` | GET | JWT | Get notifications for user |
| `/notification/api/v1/:id/read` | PATCH | JWT | Mark notification as read |
| `/notification/api/v1/user/:userId/read-all` | PATCH | JWT | Mark all notifications as read |
| `/notification/api/v1/:id` | DELETE | JWT | Delete notification |
| `/notification/api/v1/user/:userId/unread-count` | GET | JWT | Get unread count |

### 14.5 Health Endpoints (All Services)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Liveness probe (is app running?) |
| `/health/ready` | GET | Readiness probe (is app ready to serve?) |

---

## 15. Environment Variables Reference

### 15.1 Shared Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `JWT_SECRET` | Shared secret for JWT signing | `supersecret` |
| `JWT_REFRESH_SECRET` | Secret for refresh token signing | `supersecretrefresh` |
| `RABBITMQ_URL` | RabbitMQ connection string | `amqp://guest:guest@rabbitmq:5672` |

### 15.2 Auth Service

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:123@auth-db:5432/auth_db` |
| `SMTP_HOST` | SMTP server hostname | `mailhog` |
| `SMTP_PORT` | SMTP server port | `1025` |
| `SMTP_USER` | SMTP username | `test` |
| `SMTP_PASS` | SMTP password | `test` |
| `SMTP_FROM` | Sender email address | `noreply@example.com` |
| `FRONTEND_URL` | Frontend URL for email links | `http://localhost:3000` |

### 15.3 Order Service

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:123@order-db:5432/order` |
| `PORT` | Service port | `4002` |

### 15.4 Payment Service

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:postgres@payment-db:5432/paymentdb` |
| `REDIS_URL` | Redis connection string | `redis://redis:6379` |
| `STRIPE_SECRET_KEY` | Stripe API secret key | `sk_test_...` |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret | `whsec_...` |
| `CHAPA_SECRET_KEY` | Chapa API secret key | `CHASECK_TEST-...` |
| `PAYMENT_MODE` | Payment mode: `stripe`, `chapa`, or `sandbox` | `sandbox` |

### 15.5 Notification Service

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://notify_user:notify_password@db-notif:5432/notify_db` |
| `PORT` | Service port | `4004` |

### 15.6 Frontend

| Variable | Description | Example |
|----------|-------------|---------|
| `NEXT_PUBLIC_API_URL` | Backend API base URL | `http://localhost:8080` |

---

## 16. Error Handling Strategy

### 16.1 Standard Error Response Format

All services return errors in a consistent JSON format:

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "error": "Bad Request",
  "details": [
    { "field": "email", "message": "Invalid email format" }
  ]
}
```

### 16.2 HTTP Status Codes Used

| Status Code | Meaning | When Used |
|-------------|---------|-----------|
| `200` | OK | Successful GET, PATCH |
| `201` | Created | Successful POST (resource created) |
| `400` | Bad Request | Validation errors, malformed request |
| `401` | Unauthorized | Missing or invalid JWT token |
| `403` | Forbidden | Valid token but insufficient permissions |
| `404` | Not Found | Resource does not exist |
| `409` | Conflict | Duplicate resource (e.g., email already exists) |
| `500` | Internal Server Error | Unexpected server error |

### 16.3 RabbitMQ Message Error Handling

| Scenario | Handling |
|----------|----------|
| Consumer throws exception | Message is auto-acknowledged (`noAck: true`), lost |
| Consumer offline | Messages queue up (durable queue) |
| Invalid message format | Consumer logs error, message is discarded |

> [!WARNING]
> **Current Limitation**: Messages are auto-acknowledged. If processing fails, the message is lost. For production, implement manual acknowledgement and dead-letter queues.

### 16.4 Frontend Error Handling

The frontend `ApiClient` handles errors centrally:

```typescript
// If response is not OK
if (!response.ok) {
  if (response.status === 401) {
    // Try refresh token, then retry
  }
  throw new Error(data.message || 'An error occurred');
}
```

---

## 17. Deployment Runbook

### 17.1 Local Development (Docker Compose)

```bash
# 1. Navigate to infrastructure directory
cd infrastructure

# 2. Start all services
docker-compose up -d

# 3. Run database migrations
docker exec -it auth-service npx prisma migrate deploy
docker exec -it order-service npx prisma migrate deploy
docker exec -it payment-service npx prisma migrate deploy

# 4. Access services
# Frontend:     http://localhost:3000
# API Gateway:  http://localhost:8080
# RabbitMQ UI:  http://localhost:15672 (guest/guest)
# MailHog UI:   http://localhost:8025

# 5. View logs
docker-compose logs -f auth-service

# 6. Stop all services
docker-compose down
```

### 17.2 Kubernetes Deployment (Minikube)

```bash
# 1. Start Minikube
minikube start --memory=4096 --cpus=2

# 2. Enable Ingress addon
minikube addons enable ingress

# 3. Navigate to k8s directory
cd infrastructure/k8s

# 4. Apply all manifests
./apply.sh
# OR manually:
kubectl apply -f shared/
kubectl apply -f db/
kubectl apply -f apps/

# 5. Wait for pods to be ready
kubectl get pods -w

# 6. Get Minikube IP
minikube ip

# 7. Access services (add to /etc/hosts)
# <minikube-ip> food-delivery.local

# 8. View logs
kubectl logs -f deployment/auth-service

# 9. Port-forward for debugging
kubectl port-forward svc/auth-service 4000:4000
```

### 17.3 Database Migrations

```bash
# Auth Service (NestJS/Prisma)
docker exec -it auth-service npx prisma migrate deploy
docker exec -it auth-service npx prisma generate

# Order Service (Express/Prisma)
docker exec -it order-service npx prisma migrate deploy

# Payment Service (Express/Prisma)
docker exec -it payment-service npx prisma migrate deploy

# Notification Service (NestJS/TypeORM - auto-syncs)
# No manual migration needed in dev mode
```

---

## 18. Delivery Driver Flow

### 18.1 Driver Lifecycle Sequence Diagram

```mermaid
sequenceDiagram
    participant Driver as "Delivery Driver"
    participant ORDER as "order-service"
    participant RMQ as "RabbitMQ"
    participant NOTIF as "notification-service"
    participant WS as "Socket.io"
    participant Customer

    Driver->>ORDER: GET /orders/available-for-drivers
    ORDER->>Driver: List of orders with status READY

    Driver->>ORDER: PATCH /orders/claim/:orderId
    ORDER->>ORDER: Set driverId, deliveryStatus = PENDING
    ORDER->>RMQ: ORDER_CLAIMED event
    ORDER->>Driver: { success: true }

    RMQ-->>NOTIF: ORDER_CLAIMED
    NOTIF->>WS: Notify customer
    WS-->>Customer: "Driver has claimed your order!"

    Driver->>ORDER: PATCH /orders/delivery-status/:orderId { status: PICKED_UP }
    ORDER->>RMQ: DELIVERY_STATUS_UPDATED
    RMQ-->>NOTIF: DELIVERY_STATUS_UPDATED
    NOTIF->>WS: Notify customer
    WS-->>Customer: "Driver has picked up your order!"

    Driver->>ORDER: PATCH /orders/delivery-status/:orderId { status: ON_THE_WAY }
    ORDER->>RMQ: DELIVERY_STATUS_UPDATED
    WS-->>Customer: "Driver is on the way!"

    Driver->>ORDER: PATCH /orders/delivery-status/:orderId { status: DELIVERED }
    ORDER->>ORDER: Update order status = COMPLETED
    ORDER->>RMQ: ORDER_COMPLETED
    WS-->>Customer: "Order delivered! Please rate your experience."
```

**Diagram Explanation:**
1.  **Browse Available Orders**: Driver views orders that are `READY` for pickup.
2.  **Claim Order**: Driver claims an order, which assigns them as the `driverId`.
3.  **Pickup**: Driver picks up the order and updates status to `PICKED_UP`.
4.  **Transit**: Driver marks order as `ON_THE_WAY`.
5.  **Delivery**: Driver marks order as `DELIVERED`, completing the order lifecycle.
6.  **Real-time Updates**: Customer receives Socket.io notifications at each step.

### 18.2 Delivery Status State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING : Order claimed
    PENDING --> PICKED_UP : Driver picks up
    PICKED_UP --> ON_THE_WAY : Driver starts delivery
    ON_THE_WAY --> DELIVERED : Order delivered
    DELIVERED --> [*]
```

---

## 19. Role-Based Access Control (RBAC) Matrix

| Endpoint Category | CUSTOMER | RESTAURANT | DELIVERY | ADMIN |
|-------------------|----------|------------|----------|-------|
| **Auth** |
| Sign up / Sign in | ✅ | ✅ | ✅ | ✅ |
| View own profile | ✅ | ✅ | ✅ | ✅ |
| Update own profile | ✅ | ✅ | ✅ | ✅ |
| List all users | ❌ | ❌ | ❌ | ✅ |
| Delete user | ❌ | ❌ | ❌ | ✅ |
| Update user role | ❌ | ❌ | ❌ | ✅ |
| **Restaurants** |
| View restaurants | ✅ | ✅ | ✅ | ✅ |
| Create restaurant | ❌ | ✅ | ❌ | ✅ |
| Update own restaurant | ❌ | ✅ | ❌ | ✅ |
| Delete restaurant | ❌ | ✅ (own) | ❌ | ✅ |
| **Menu Items** |
| View menu items | ✅ | ✅ | ✅ | ✅ |
| Create menu item | ❌ | ✅ | ❌ | ✅ |
| Update menu item | ❌ | ✅ (own) | ❌ | ✅ |
| Delete menu item | ❌ | ✅ (own) | ❌ | ✅ |
| **Orders** |
| Create order | ✅ | ❌ | ❌ | ✅ |
| View own orders | ✅ | ✅ | ✅ | ✅ |
| View restaurant orders | ❌ | ✅ (own) | ❌ | ✅ |
| Update order status | ❌ | ✅ (own) | ❌ | ✅ |
| Claim order for delivery | ❌ | ❌ | ✅ | ✅ |
| Update delivery status | ❌ | ❌ | ✅ | ✅ |
| View platform metrics | ❌ | ❌ | ❌ | ✅ |
| **Payments** |
| Initiate payment | ✅ | ❌ | ❌ | ✅ |
| View own payments | ✅ | ✅ | ❌ | ✅ |
| **Notifications** |
| View own notifications | ✅ | ✅ | ✅ | ✅ |
| Mark as read | ✅ | ✅ | ✅ | ✅ |

---

## 20. Known Limitations & Future Improvements

### 20.1 Current Limitations

| Area | Limitation | Impact |
|------|------------|--------|
| **Rate Limiting** | No rate limiting on APIs | Vulnerable to DoS attacks |
| **Distributed Tracing** | No OpenTelemetry/Jaeger integration | Hard to debug cross-service issues |
| **Message Retry** | `noAck: true` means failed messages are lost | Event processing is not guaranteed |
| **Idempotency** | Payment webhooks may not be idempotent | Duplicate payments possible |
| **Caching** | Only payment-service uses Redis | DB load could be reduced |
| **Search** | Basic LIKE queries for restaurant search | No full-text search |
| **File Uploads** | Not implemented | No restaurant images/menus |
| **Mobile Push** | Only WebSocket notifications | No Firebase/APNs integration |

### 20.2 Recommended Improvements

1.  **Add Rate Limiting**
    ```bash
    # Example: express-rate-limit
    npm install express-rate-limit
    ```

2.  **Implement Dead-Letter Queues**
    - Failed messages go to a DLQ for manual inspection
    - Prevents message loss

3.  **Add Distributed Tracing**
    - Install OpenTelemetry SDK
    - Export traces to Jaeger or Zipkin

4.  **Implement Kubernetes HPA**
    ```yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    spec:
      minReplicas: 2
      maxReplicas: 10
      metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              averageUtilization: 70
    ```

5.  **Add API Versioning**
    - Currently mixed (`/api/v1` in some, none in others)
    - Standardize to `/api/v1/*` across all services

---

## 21. Testing Strategy

### 21.1 Current Testing Status

| Service | Unit Tests | Integration Tests | E2E Tests |
|---------|------------|-------------------|-----------|
| `auth-service` | Partial | No | No |
| `order-service` | No | No | No |
| `payment-service` | No | No | No |
| `notification-service` | No | No | No |
| `frontend` | No | No | No |

### 21.2 Recommended Testing Approach

**Unit Tests** (Jest):
```bash
# Run in each service
npm run test
```

**Integration Tests** (Supertest):
```javascript
// Example: auth.integration.test.ts
const response = await request(app)
  .post('/auth/signup')
  .send({ email: 'test@example.com', password: 'password123' });
expect(response.status).toBe(201);
```

**E2E Tests** (Playwright):
```bash
# Run from frontend directory
npx playwright test
```

### 21.3 Test Database Strategy

Use a separate test database or Docker containers:

```yaml
# docker-compose.test.yml
services:
  test-db:
    image: postgres:15
    environment:
      POSTGRES_DB: test_db
```

---

## 22. Glossary of Terms

| Term | Definition |
|------|------------|
| **Access Token** | Short-lived JWT (15 min) used to authenticate API requests |
| **Refresh Token** | Long-lived token (7 days) used to obtain new access tokens |
| **Circuit Breaker** | Pattern that prevents cascading failures by failing fast when a dependency is down |
| **Dead-Letter Queue (DLQ)** | Queue where failed messages are sent for later inspection |
| **Deployment** | Kubernetes resource that manages stateless applications |
| **Exchange** | RabbitMQ component that routes messages to queues based on rules |
| **Graceful Degradation** | System continues to function (with reduced capability) when a component fails |
| **Ingress** | Kubernetes resource that manages external access to services |
| **JWT** | JSON Web Token - a compact, URL-safe means of representing claims |
| **Microservice** | Independently deployable service that does one thing well |
| **Namespace** | Kubernetes resource that isolates resources within a cluster |
| **Pod** | Smallest deployable unit in Kubernetes (one or more containers) |
| **Queue** | RabbitMQ component that stores messages until consumed |
| **Routing Key** | String used by RabbitMQ exchanges to route messages to queues |
| **Service** | Kubernetes resource that provides stable networking for pods |
| **StatefulSet** | Kubernetes resource for managing stateful applications (like databases) |
| **Topic Exchange** | RabbitMQ exchange type that routes based on pattern matching |
| **WebSocket** | Protocol for full-duplex communication over a single TCP connection |

---

## 23. Order Status Reference

### 23.1 Order Status Values

| Status | Description | Set By |
|--------|-------------|--------|
| `PENDING` | Order created, awaiting payment | System |
| `PREPARING` | Payment received, restaurant is cooking | Restaurant Owner |
| `READY` | Food is ready for pickup | Restaurant Owner |
| `COMPLETED` | Order delivered to customer | Driver |
| `CANCELLED` | Order was cancelled | Customer/Owner/Admin |

### 23.2 Delivery Status Values

| Status | Description | Set By |
|--------|-------------|--------|
| `PENDING` | Order claimed, awaiting pickup | Driver (auto on claim) |
| `PICKED_UP` | Driver picked up from restaurant | Driver |
| `ON_THE_WAY` | Driver is en route to customer | Driver |
| `DELIVERED` | Order delivered to customer | Driver |

### 23.3 State Transition Diagram

```mermaid
stateDiagram-v2
    [*] --> PENDING : Order Created
    PENDING --> PREPARING : Payment Success
    PENDING --> CANCELLED : Customer Cancels
    PREPARING --> READY : Food Prepared
    PREPARING --> CANCELLED : Owner Cancels
    READY --> COMPLETED : Delivered
    COMPLETED --> [*]
    CANCELLED --> [*]
```

**Diagram Explanation:**
1.  **PENDING**: Order starts here when customer places it.
2.  **PREPARING**: Moves here after successful payment.
3.  **READY**: Restaurant marks order ready for driver pickup.
4.  **COMPLETED**: Driver marks as delivered.
5.  **CANCELLED**: Can happen from PENDING or PREPARING states.

---

This concludes the comprehensive architecture documentation. For questions or contributions, please refer to the repository README.

