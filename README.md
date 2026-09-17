<div align="center">
  
  <!-- Replace the src with your actual project banner image or GIF -->
  <img src="https://via.placeholder.com/1000x250/000000/FFFFFF/?text=REZERVO+%7C+High-Concurrency+Booking" alt="REZERVO Banner" width="100%" />

  <br />
  <br />

  # 🎟️ REZERVO
  **High-Concurrency Event Booking & Payment Platform**

  [![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)]()
  [![Node.js](https://img.shields.io/badge/Node.js-43853D?style=for-the-badge&logo=node.js&logoColor=white)]()
  [![Fastify](https://img.shields.io/badge/Fastify-000000?style=for-the-badge&logo=fastify&logoColor=white)]()
  [![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)]()
  [![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)]()
  [![Docker](https://img.shields.io/badge/Docker-2CA5E0?style=for-the-badge&logo=docker&logoColor=white)]()

  > **Engineering Philosophy:** Don't just make the happy path work. Build the system so the important failure modes are difficult or impossible to produce. Prioritize engineering depth and system reliability over feature breadth.

</div>

<br />

## 🎯 The Core Problem: The Seat-Booking Race Condition

REZERVO is a production-oriented platform built to solve one specific, high-stakes engineering challenge: **The Race Condition**.

When hundreds of users attempt to book the exact same seat at the exact same millisecond, the system must definitively process only one winner without double-booking.

```mermaid
journey
    title 100 Concurrent Requests for Seat H-4
    section The Race
      100 Users Request Seat H-4: 7:00:00.000: User
      PostgreSQL Row Lock Engaged: 7:00:00.050: Database
    section The Resolution
      One Successful Booking Hold: 7:00:00.100: System
      99 Requests Rejected Gracefully: 7:00:00.120: System

🔒 The Core InvariantONE EVENT + ONE SEAT = AT MOST ONE CONFIRMED BOOKINGThis strict invariant is enforced at the lowest level—the database—as well as heavily guarded through application-level logic and unique constraints.🏗️ System ArchitectureREZERVO leverages a robust, decoupled architecture designed for high throughput and reliable background processing.Code snippetgraph TD
    A[🧑‍💻 Client / Browser] -->|HTTPS| B(⚛️ React Web App)
    B -->|REST API| C{⚙️ Node + Fastify API}
    
    C -->|Reads/Writes & Row Locks| D[(🐘 PostgreSQL)]
    C -->|Caching & Queues| E[(🟥 Redis)]
    
    E -->|Processes Expiries| F[👷 Worker Process]
    
    G[💳 Payment Provider] -->|Webhooks| C
    
    classDef frontend fill:#61DAFB,stroke:#000,stroke-width:2px,color:#000;
    classDef backend fill:#339933,stroke:#000,stroke-width:2px,color:#fff;
    classDef database fill:#336791,stroke:#000,stroke-width:2px,color:#fff;
    classDef cache fill:#DC382D,stroke:#000,stroke-width:2px,color:#fff;
    
    class B frontend;
    class C,F backend;
    class D database;
    class E cache;
🗄️ Database Schema & Data ModelThe PostgreSQL database acts as the single source of truth. It relies on strict foreign keys, indexing, and UNIQUE constraints to maintain data integrity.Code snippeterDiagram
    USERS ||--o{ BOOKINGS : places
    EVENTS ||--o{ SEATS : contains
    SEATS ||--o{ HOLDS : has
    SEATS ||--o{ BOOKINGS : reserves
    BOOKINGS ||--o| PAYMENTS : requires
    
    USERS {
        uuid id PK
        string email
        string password_hash
    }
    EVENTS {
        uuid id PK
        string title
        datetime scheduled_at
    }
    SEATS {
        uuid id PK
        uuid event_id FK
        string seat_number
        boolean is_available
    }
    HOLDS {
        uuid id PK
        uuid seat_id FK
        uuid user_id FK
        datetime expires_at
    }
    BOOKINGS {
        uuid id PK
        uuid seat_id FK
        uuid user_id FK
        string status
    }
🔐 Concurrency & Reliability Strategy1. Database-Level Locking (Pessimistic Locking)The booking system utilizes PostgreSQL transactions and row-level locking (SELECT ... FOR UPDATE) when competing requests attempt to operate on the same seat.SQLBEGIN;

-- Lock the specific row to prevent dirty reads and race conditions
SELECT *
FROM seats
WHERE id = $1
FOR UPDATE;

-- Re-check current availability
-- Create hold / booking

COMMIT;
2. Idempotent Payments & Outbox PatternPayment operations utilize strict idempotency keys. If network latency causes a client to retry a payment request:Request #1 → Processed normally.Request #2 → Database catches idempotency key, returns Request #1's result.Request #3 → Ignored, returns existing result.Payment webhooks are heavily deduplicated using the Transactional Outbox Pattern to ensure replayed webhook events do not trigger duplicate state changes.🚀 Getting StartedPrerequisitesDocker & Docker ComposeNode.js v18+npm or pnpmLocal InstallationClone the repositoryBashgit clone [https://github.com/yourusername/REZERVO.git](https://github.com/yourusername/REZERVO.git)
cd REZERVO
Environment VariablesBashcp .env.example .env
# Add your payment gateway sandbox keys and database credentials
Spin up Infrastructure (PostgreSQL & Redis)Bashdocker-compose up -d
Install Dependencies & Run MigrationsBashnpm install
npm run db:migrate
Start the Application ServicesBashnpm run dev
The API will be available at http://localhost:3000 and the Web UI at http://localhost:5173.🧪 Testing the FailuresREZERVO deliberately engineers and tests the failures the system is meant to prevent.To run the concurrency stress tests (simulating 100 users attempting to book the same seat):Bashnpm run test:concurrency
ScenarioTriggerExpected System BehaviorConcurrent Booking100 requests targeting 1 seat simultaneously1 successful response, 99 rejected responses.Payment RetryThe exact same payment request sent 3 times1 payment effect, 1 booking, 0 duplicates.Hold ExpirationUser abandons checkoutRedis/Worker detects expiry, seat returns to AVAILABLE.Webhook ReplaySame webhook payload received 3 times1 processing effect, 2 duplicate events safely ignored.📁 Repository StructurePlaintextREZERVO/
├── apps/
│   ├── web/              # React frontend (Vite, Tailwind)
│   ├── api/              # Fastify backend API (Node.js)
│   └── worker/           # Background job processor
├── packages/
│   ├── shared/           # Shared types (Zod schemas, interfaces)
│   └── config/           # ESLint, TS configs
├── infra/
│   ├── docker/           # Container configurations
│   └── migrations/       # SQL schema migrations
├── tests/
│   ├── integration/      # API tests
│   ├── concurrency/      # Race condition stress tests
│   └── load/             # Throughput testing
└── docs/                 # Architectural Decision Records (ADRs)
👨‍💻 AuthorMohammed Rehan AhamedB.Tech — Computer Science & EngineeringPassionate about backend architecture, data structures, and building resilient systems that solve complex computational problems gracefully.