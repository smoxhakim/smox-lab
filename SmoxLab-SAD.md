# Smox Lab — System Architecture Document (SAD)

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** 2026-05-03  
**Owner:** Smox Lab Engineering Team

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [System Diagram](#system-diagram)
3. [Frontend Architecture](#frontend-architecture)
   - [Kiosk App](#kiosk-app-nextjs--react)
   - [Web Platform](#web-platform-nextjs--react)
4. [Backend Architecture](#backend-architecture)
   - [NestJS Microservices](#nestjs-microservices)
   - [API Gateway](#api-gateway)
   - [Service Breakdown](#service-breakdown)
5. [Real-Time Layer](#real-time-layer)
6. [AI Integration Layer](#ai-integration-layer)
7. [Database Design](#database-design)
8. [IoT & Printer System](#iot--printer-system)
9. [Infrastructure & Deployment](#infrastructure--deployment)
10. [Security Architecture](#security-architecture)
11. [Data Flow Diagrams](#data-flow-diagrams)

---

## Architecture Overview

Smox Lab uses a **microservices-based** backend exposed through a unified **API Gateway**, paired with two distinct **Next.js** frontends — one for the in-store kiosk and one for the public web platform. Real-time communication between services and devices is handled by **WebSockets** via **Socket.io** in NestJS. A dedicated **AI Service** acts as a proxy/orchestrator to external AI providers. The **print pipeline** is treated as a hardware integration layer with its own queue and IoT adapter.

### Key Design Principles

| Principle | Implementation |
|---|---|
| Decoupled services | Each domain is a standalone NestJS module/microservice |
| Real-time by default | WebSocket gateways on order, printer, and kiosk services |
| Offline resilience | Kiosk works in degraded mode if cloud is unreachable |
| Horizontal scalability | Stateless services behind load balancer; Redis for shared state |
| Extensibility | IoT printer layer uses an adapter pattern for multi-vendor support |

---

## System Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                                   │
│                                                                        │
│  ┌─────────────────────┐        ┌──────────────────────────────────┐  │
│  │   Kiosk App         │        │   Web Platform                   │  │
│  │   (Next.js + React) │        │   (Next.js + React)              │  │
│  │   Three.js (3D/AR)  │        │   Design Marketplace             │  │
│  │   TF.js / MediaPipe │        │   Creator Dashboard              │  │
│  └────────┬────────────┘        └───────────────┬──────────────────┘  │
└───────────┼─────────────────────────────────────┼────────────────────┘
            │  HTTPS / WSS                         │  HTTPS
            ▼                                      ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY (NestJS)                           │
│         Auth  │  Rate Limiting  │  Routing  │  Load Balancing         │
└──────┬────────┬────────┬────────┬───────────┬──────────────┬─────────┘
       │        │        │        │           │              │
       ▼        ▼        ▼        ▼           ▼              ▼
  ┌────────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐  ┌────────────┐
  │ Auth   │ │Order │ │Design│ │ User │ │  AI    │  │  Admin     │
  │Service │ │Svc   │ │Svc   │ │Svc   │ │Service │  │  Service   │
  └────────┘ └──┬───┘ └──┬───┘ └──────┘ └───┬────┘  └────────────┘
                │         │                   │
                ▼         ▼                   ▼
         ┌──────────┐ ┌────────┐     ┌──────────────┐
         │  Redis   │ │  S3 /  │     │  OpenAI API  │
         │  Queue   │ │  R2    │     │  Stable Diff │
         └────┬─────┘ │(Assets)│     └──────────────┘
              │        └────────┘
              ▼
   ┌──────────────────────┐
   │  Printer Service     │
   │  (IoT / WebSocket)   │
   └──────────────────────┘
              │
              ▼
   ┌──────────────────────┐
   │  DTG Printers (LAN)  │
   └──────────────────────┘

         ┌───────────────┐
         │  PostgreSQL   │  ← Primary DB (all services)
         └───────────────┘
         ┌───────────────┐
         │    Redis      │  ← Queue + Cache + PubSub
         └───────────────┘
```

---

## Frontend Architecture

### Kiosk App (Next.js + React)

The kiosk is a locked-down, full-screen browser application designed for a touchscreen environment.

#### Tech Stack
| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router) |
| UI | React 19 + Tailwind CSS |
| 3D / AR | Three.js + @react-three/fiber + @react-three/drei |
| Pose Estimation | TensorFlow.js (`@tensorflow-models/pose-detection`) or MediaPipe Holistic |
| Canvas / Drawing | Konva.js or Fabric.js |
| Real-time | Socket.io client |
| State management | Zustand |
| Animations | Framer Motion |

#### Why TensorFlow.js vs MediaPipe
| | TensorFlow.js | MediaPipe (Recommended) |
|---|---|---|
| Performance | Good, heavier bundle | Lighter WASM, faster |
| Pose accuracy | High | Higher, Google-maintained |
| Kiosk fit | Works | Better for real-time kiosk |
| Recommendation | Use for fallback/offline | **Primary choice** |

> **Recommendation:** Use **MediaPipe Holistic** for pose/body detection. Fall back to TF.js PoseDetection if MediaPipe is unavailable. Both are client-side — no round-trip to server needed for AR overlay.

#### Kiosk App Route Structure
```
app/
├── (kiosk)/
│   ├── page.tsx              → Idle / Welcome screen
│   ├── products/
│   │   └── [id]/page.tsx     → Product selection
│   ├── configure/page.tsx    → Size / color picker
│   ├── design/page.tsx       → Design canvas
│   ├── ar-preview/page.tsx   → AR Try-On
│   ├── order/page.tsx        → Order summary + confirmation (no payment)
│   ├── confirmation/page.tsx → Ticket + directions to physical Gaming Lounge
│   ├── queue-display/page.tsx → Fullscreen order status board (mounted on TV in Gaming Lounge)
│   └── prayer-pause/page.tsx → Full-screen prayer pause overlay (shown on all kiosks + lounge TV)
├── api/                      → Next.js API routes (thin proxy)
└── _components/
    ├── ThreeCanvas/          → Three.js product viewer
    ├── AROverlay/            → MediaPipe + WebGL overlay
    ├── DesignCanvas/         → Fabric.js or Konva canvas
    └── KioskShell/           → Full-screen kiosk wrapper
```

#### Offline Resilience Strategy
- Core kiosk UI and product catalog are cached via **Service Worker** (PWA mode).
- AI design generation: disabled when offline, replaced with template-only mode.
- AR: fully client-side, works offline.
- Orders: queued locally with IndexedDB, synced when connection restores.

---

### Web Platform (Next.js + React)

The web platform handles customer accounts, the design marketplace, and pre-design from home.

#### Route Structure
```
app/
├── page.tsx                 → Landing / marketing
├── auth/
│   ├── login/page.tsx
│   └── signup/page.tsx
├── design/page.tsx          → Design canvas (web version)
├── marketplace/
│   ├── page.tsx             → Browse designs
│   └── [designId]/page.tsx → Design detail + purchase
├── account/
│   ├── page.tsx             → Dashboard
│   ├── orders/page.tsx
│   └── designs/page.tsx
├── creator/
│   ├── page.tsx             → Creator dashboard
│   ├── publish/page.tsx     → Upload design
│   └── earnings/page.tsx
└── admin/
    ├── page.tsx             → Order overview
    ├── printers/page.tsx    → Printer management
    └── analytics/page.tsx  → Store analytics
```

---

## Backend Architecture

### NestJS Microservices

The backend is organized as **NestJS modules** in a monorepo (using **Turborepo** or **Nx**). Each module can be deployed as an independent microservice behind the API Gateway.

```
apps/
├── gateway/         → API Gateway (NestJS)
├── auth-service/    → Authentication + JWT
├── user-service/    → User accounts + profiles
├── order-service/   → Orders, ticket generation, status tracking
├── design-service/  → Design CRUD + asset storage
├── print-service/   → Printer queue + IoT integration
├── ai-service/      → AI proxy (OpenAI / Stable Diffusion)
├── admin-service/   → Dashboard APIs + analytics
└── prayer-service/  → Prayer time scheduler + store pause logic (Morocco)

libs/
├── common/          → Shared DTOs, guards, decorators
├── database/        → Prisma ORM setup + shared schema
└── events/          → Shared event types (for WebSocket / event bus)
```

### API Gateway

- Built with **NestJS + @nestjs/platform-fastify** (faster than Express for high-throughput kiosks).
- Handles:
  - JWT validation (via `AuthGuard`)
  - Rate limiting (`@nestjs/throttler`)
  - Request routing to downstream microservices
  - WebSocket proxy / upgrade

### Service Breakdown

#### Auth Service
| Responsibility | Detail |
|---|---|
| Registration / login | Email + password, social OAuth (Google, Apple) |
| JWT issuance | Access token (15min) + Refresh token (7 days) |
| Kiosk sessions | Anonymous session token, no account required |
| Roles | `CUSTOMER`, `CREATOR`, `STORE_OPERATOR`, `ADMIN` |

#### User Service
| Responsibility | Detail |
|---|---|
| Profile management | Name, preferences, saved designs |
| Loyalty points | Earn on purchase, redeem at kiosk |
| Saved designs | Link to Design Service assets |

#### Order Service
| Responsibility | Detail |
|---|---|
| Order creation | Validates design + product config, creates order on customer confirmation |
| Status machine | `CONFIRMED → QUEUED → PRINTING → READY → PAID → PICKED_UP` |
| WebSocket emit | Broadcasts status changes to kiosk, lounge display, and admin dashboard |
| Ticket generation | PDF/QR ticket via `pdf-lib` or `pdfkit` — printed at kiosk on confirmation |
| Cashier lookup | REST endpoint for cashier to fetch order by ticket number/QR scan |
| Cashier checkout | Cashier marks order `PAID → PICKED_UP` via admin dashboard / cashier terminal |

#### Design Service
| Responsibility | Detail |
|---|---|
| Design CRUD | Save / load canvas state (JSON) |
| Asset upload | Images stored in S3-compatible storage (AWS S3 / Cloudflare R2) |
| Print file generation | Converts canvas JSON → high-res PNG/PDF for printer |
| DPI validation | Ensures print-ready quality before queue |
| Moderation | Calls OpenAI Moderation API on design content |

#### Print Service
| Responsibility | Detail |
|---|---|
| Queue management | Redis Bull queue — one queue per printer |
| Job dispatch | Pushes print job payload to printer via WebSocket / LAN |
| Status tracking | Listens for printer events, updates Order Service |
| Error handling | Retry logic (3 attempts), operator alert on failure |
| Printer registry | Maps printer IDs to IP addresses + capabilities |

#### AI Service
| Responsibility | Detail |
|---|---|
| Design generation | Proxies to OpenAI DALL·E 3 or Stable Diffusion |
| Background removal | Calls `remove.bg` API or self-hosted `rembg` |
| Prompt safety | Pre-filters user prompts before sending to AI |
| Credit system | Validates user has credits before generating |
| Response caching | Caches identical prompts for 24h via Redis |

#### Admin Service
| Responsibility | Detail |
|---|---|
| Order management | Aggregate view across all orders |
| Printer dashboard | Real-time printer status via WebSocket subscription |
| Analytics | Pre-aggregated metrics served from materialized views |
| CMS | Product catalog, template library management |
| Prayer time management | View schedule, manual pause/resume, configure duration per prayer |

#### Prayer Service

| Responsibility | Detail |
|---|---|
| Prayer time fetching | Calls [AlAdhan API](https://aladhan.com/prayer-times-api) daily at midnight to fetch next day's times |
| Schedule caching | Stores daily prayer schedule in Redis (`prayer:schedule:{storeId}:{date}`) |
| Cron scheduler | `@nestjs/schedule` — schedules warning job (T-5min) and pause job (T±0) per prayer |
| Pause orchestration | Emits `store:prayer:warning` and `store:prayer:pause` events via WebSocket |
| Print queue gate | Signals Print Service to halt queue dispatch during pause window |
| Auto-resume | Schedules resume job after configured prayer duration |
| Manual override | REST endpoint for operator to pause/resume from admin dashboard |
| Offline fallback | If API unreachable, uses last cached schedule; alerts operator if no cache exists |

---

## Real-Time Layer

All real-time communication uses **Socket.io** inside NestJS `@WebSocketGateway`.

### Namespaces

| Namespace | Consumers | Events |
|---|---|---|
| `/kiosk` | Kiosk app | `order:status`, `printer:ready`, `session:timeout`, `store:prayer:warning`, `store:prayer:pause`, `store:prayer:resume` |
| `/admin` | Admin dashboard | `order:new`, `printer:status`, `queue:update`, `store:prayer:pause`, `store:prayer:resume` |
| `/queue-display` | Physical Gaming Lounge status screen (TV/monitor in-store) | `order:ready`, `queue:update`, `store:prayer:pause`, `store:prayer:resume` |

> **Note:** The Gaming Zone is a **physical lounge area** in the store with real gaming hardware (consoles, arcade machines). It is not a digital feature. The `/queue-display` namespace drives a fullscreen status board mounted in the lounge so customers know when their order is ready — no app or interaction needed, just a live TV screen.

### Event Flow — Order Lifecycle

```
[Kiosk confirms order]
        │
        ▼
Order Service → emits order:created → Redis PubSub
        │
        ├──► Admin namespace: order:new
        │
        └──► Print Service subscribes → job pushed to Bull queue
                │
                ▼
        Printer receives job → status: PRINTING
                │
                ├──► Print Service → Order Service: status update
                ├──► Kiosk namespace: order:status { status: "PRINTING" }
                ├──► Admin namespace: printer:status
                │
                ▼
        Print complete → status: READY
                │
                ├──► Kiosk namespace: order:status { status: "READY" }
                └──► Queue-display namespace: order:ready → ticket number highlighted on lounge TV screen
```

---

## AI Integration Layer

### Provider Strategy

| Use Case | Provider (v1) | Alternative (v2) |
|---|---|---|
| Design generation from prompt | OpenAI DALL·E 3 | Self-hosted Stable Diffusion XL |
| Design enhancement / style | GPT-4o with image input | Replicate API |
| Background removal | `rembg` (self-hosted Python) | remove.bg API |
| Content moderation | OpenAI Moderation API | AWS Rekognition |
| Text style suggestions | GPT-4o-mini | Local Llama 3 |

### AI Service Flow

```
[User enters prompt on kiosk]
        │
        ▼
AI Service receives request
        │
        ├── 1. Validate user credits (call User Service)
        ├── 2. Sanitize + enhance prompt (internal prompt processor)
        ├── 3. Check Redis cache for identical prompt
        │       └── Cache hit → return cached result immediately
        │
        ├── 4. Send to OpenAI DALL·E 3 API
        │       └── Receive 3 image URLs
        │
        ├── 5. Run background removal on each image (rembg)
        │
        ├── 6. Store results in S3 + cache in Redis (24h TTL)
        │
        └── 7. Return image URLs to client
```

### Prompt Enhancement (Internal)

Before sending to DALL·E, the AI Service appends a standardized suffix to improve print quality:

```
User prompt: "a futuristic city in neon colors"

Enhanced prompt sent to API:
"a futuristic city in neon colors, vector art style, 
high contrast, white background, print-ready, 
no gradients, flat illustration, centered composition, 
t-shirt graphic design"
```

This prompt engineering is managed in the **AI Prompt Architecture document**.

---

## Prayer Time System (Morocco)

### Overview

The Prayer Service is a dedicated NestJS module that fetches daily prayer schedules, schedules cron jobs for each prayer event, and orchestrates a system-wide pause and resume across all kiosks, the print queue, and the lounge display screen.

### External API: AlAdhan

Prayer times are sourced from the **[AlAdhan Prayer Times API](https://aladhan.com/prayer-times-api)** — a free, reliable REST API that supports the official **Moroccan Ministry of Habous** calculation method.

```
GET https://api.aladhan.com/v1/timingsByCity/{date}
    ?city=Casablanca
    &country=Morocco
    &method=21          ← method 21 = Moroccan Ministry of Habous
```

Example response (relevant fields):
```json
{
  "data": {
    "timings": {
      "Fajr":    "05:12",
      "Dhuhr":   "13:04",
      "Asr":     "16:38",
      "Maghrib": "19:47",
      "Isha":    "21:12"
    },
    "date": {
      "gregorian": { "date": "03-05-2026" }
    }
  }
}
```

### Scheduler Architecture

```
[Midnight Cron — daily]
        │
        ▼
Prayer Service fetches tomorrow's times from AlAdhan API
        │
        ├── Cache in Redis: prayer:schedule:{storeId}:{YYYY-MM-DD}  (TTL: 48h)
        │
        └── For each active prayer (Dhuhr, Asr, Maghrib, Isha):
                │
                ├── Schedule WARNING job  → at (prayerTime - 5 minutes)
                └── Schedule PAUSE job    → at (prayerTime)
                        └── Schedule RESUME job → at (prayerTime + configuredDuration)
```

NestJS implementation uses `@nestjs/schedule` with dynamic cron expressions:

```typescript
// apps/prayer-service/src/prayer.scheduler.ts

@Injectable()
export class PrayerScheduler {

  constructor(
    private readonly schedulerRegistry: SchedulerRegistry,
    private readonly prayerGateway: PrayerGateway,
    private readonly printService: PrintServiceClient,
    private readonly redis: Redis,
    private readonly config: PrayerConfigService,
  ) {}

  async schedulePrayersForDate(storeId: string, date: string): Promise<void> {
    const schedule = await this.fetchOrCachedSchedule(storeId, date);
    const activePrayers = this.config.getActivePrayers(storeId);
    // e.g. ['Dhuhr', 'Asr', 'Maghrib', 'Isha']

    for (const prayer of activePrayers) {
      const prayerTime = parse(schedule.timings[prayer], 'HH:mm', new Date());
      const warningTime = subMinutes(prayerTime, 5);
      const resumeTime  = addMinutes(prayerTime, this.config.getDuration(storeId, prayer));

      this.addDynamicTimeout(`${storeId}:${prayer}:warning`, warningTime, () =>
        this.onPrayerWarning(storeId, prayer)
      );

      this.addDynamicTimeout(`${storeId}:${prayer}:pause`, prayerTime, () =>
        this.onPrayerPause(storeId, prayer)
      );

      this.addDynamicTimeout(`${storeId}:${prayer}:resume`, resumeTime, () =>
        this.onPrayerResume(storeId, prayer)
      );
    }
  }

  private async onPrayerWarning(storeId: string, prayer: string): Promise<void> {
    this.prayerGateway.emitToStore(storeId, 'store:prayer:warning', { prayer, minutesLeft: 5 });
  }

  private async onPrayerPause(storeId: string, prayer: string): Promise<void> {
    // 1. Block new kiosk sessions
    await this.redis.set(`store:${storeId}:paused`, 'true', 'EX', 7200);

    // 2. Pause print queue (Print Service stops dispatching new jobs)
    await this.printService.pauseQueue(storeId);

    // 3. Broadcast pause to all screens
    this.prayerGateway.emitToStore(storeId, 'store:prayer:pause', { prayer });

    // 4. Log pause event
    await this.logPauseEvent(storeId, prayer, 'PAUSED');
  }

  private async onPrayerResume(storeId: string, prayer: string): Promise<void> {
    await this.redis.del(`store:${storeId}:paused`);
    await this.printService.resumeQueue(storeId);
    this.prayerGateway.emitToStore(storeId, 'store:prayer:resume', { prayer });
    await this.logPauseEvent(storeId, prayer, 'RESUMED');
  }
}
```

### Prayer Pause State in Redis

| Key | Type | TTL | Value |
|---|---|---|---|
| `prayer:schedule:{storeId}:{date}` | Hash | 48h | Full timings object from AlAdhan |
| `store:{storeId}:paused` | String | 2h | `"true"` — acts as a gate for new sessions |

### Kiosk Behavior on WebSocket Events

```typescript
// apps/kiosk/src/_components/KioskShell/PrayerLayer.tsx

useEffect(() => {
  socket.on('store:prayer:warning', ({ prayer, minutesLeft }) => {
    showWarningBanner(prayer, minutesLeft);   // Non-blocking banner overlay
    blockNewSessions();                        // Prevent new session starts
  });

  socket.on('store:prayer:pause', ({ prayer }) => {
    saveCurrentSessionToRedis();              // Preserve in-progress session
    showPrayerPauseScreen(prayer);            // Full-screen pause UI
  });

  socket.on('store:prayer:resume', () => {
    hidePrayerPauseScreen();
    restoreSessionIfExists();                 // Reload saved session state
    goToIdleScreen();
  });
}, [socket]);
```

### Prayer Pause Screen (Kiosk UI)

The full-screen pause overlay is a branded, culturally respectful screen:

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│            ✦  Smox Lab  ✦                            │
│                                                      │
│         ┌────────────────────────────┐               │
│         │  وقت الصلاة                │               │
│         │  Prayer Time               │               │
│         │                            │               │
│         │  صلاة الظهر                │               │
│         │  Dhuhr Prayer              │               │
│         │                            │               │
│         │  نعود بعد ٣٠ دقيقة         │               │
│         │  Back in ~30 minutes       │               │
│         └────────────────────────────┘               │
│                                                      │
│   [Islamic geometric pattern background — subtle]    │
│                                                      │
└──────────────────────────────────────────────────────┘
```

- Background: dark branded color with a subtle geometric SVG pattern.
- Text: Arabic (primary, right-aligned) + French/English (secondary).
- No countdown timer on the pause screen — avoids pressure.
- Screen dims to 30% brightness after 2 minutes to reduce energy use.

### Prayer Schedule Prisma Schema

```prisma
model PrayerConfig {
  id               String   @id @default(uuid())
  storeId          String   @unique
  city             String   @default("Casablanca")
  country          String   @default("Morocco")
  calculationMethod Int     @default(21)  // AlAdhan method 21 = Moroccan Habous
  activePrayers    String[] @default(["Dhuhr", "Asr", "Maghrib", "Isha"])
  warningMinutes   Int      @default(5)
  durations        Json     // { "Dhuhr": 30, "Asr": 30, "Maghrib": 20, "Isha": 30 }
  updatedAt        DateTime @updatedAt
}

model PrayerPauseLog {
  id        String   @id @default(uuid())
  storeId   String
  prayer    String
  pausedAt  DateTime
  resumedAt DateTime?
  resumedBy String?  // "AUTO" | "OPERATOR:{userId}"
  ordersAffected Int @default(0)
}
```

### Offline Fallback

```
AlAdhan API unreachable at midnight
        │
        ├── Check Redis: prayer:schedule:{storeId}:{yesterday}
        │       └── Found → extrapolate times (±1 min from yesterday is safe)
        │               └── Log WARNING to admin dashboard
        │
        └── No cache at all
                └── Alert admin: "Prayer times unavailable — manual pause required"
                └── Manual pause button made prominent in admin dashboard
```

---

## Database Design

### ORM: Prisma

All services share a single PostgreSQL instance via **Prisma ORM** with a shared schema in the `libs/database` package.

### Core Schema

```prisma
model User {
  id            String   @id @default(uuid())
  email         String   @unique
  role          Role     @default(CUSTOMER)
  profile       Profile?
  orders        Order[]
  designs       Design[]
  loyaltyPoints Int      @default(0)
  createdAt     DateTime @default(now())
}

model Product {
  id          String   @id @default(uuid())
  name        String
  type        String   // "tshirt" | "hoodie" | "cap"
  colors      Json     // ["#FFFFFF", "#000000", ...]
  sizes       String[] // ["XS", "S", "M", "L", "XL"]
  basePrice   Float
  model3dUrl  String   // Three.js GLTF model
  printAreas  Json     // { front: {x,y,w,h}, back: {x,y,w,h} }
  active      Boolean  @default(true)
}

model Design {
  id           String       @id @default(uuid())
  userId       String?
  canvasJson   Json         // Fabric.js / Konva serialized state
  previewUrl   String       // S3 thumbnail
  printFileUrl String?      // High-res print file
  isPublished  Boolean      @default(false)
  isTemplate   Boolean      @default(false)
  creatorId    String?
  orders       Order[]
  tags         String[]
  createdAt    DateTime     @default(now())
}

model Order {
  id           String      @id @default(uuid())
  userId       String?
  kioskId      String
  productId    String
  designId     String
  size         String
  color        String
  status       OrderStatus @default(CONFIRMED)
  totalPrice   Float
  ticketCode   String      @unique
  cashierId    String?     // Operator who processed cashier checkout
  paidAt       DateTime?   // Set when cashier marks PAID
  pickedUpAt   DateTime?
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt

  product      Product     @relation(fields: [productId], references: [id])
  design       Design      @relation(fields: [designId], references: [id])
  user         User?       @relation(fields: [userId], references: [id])
}

enum OrderStatus {
  CONFIRMED   // Customer confirmed at kiosk — ticket issued, no payment yet
  QUEUED      // Job waiting in print queue
  PRINTING    // Currently being printed
  READY       // Print complete — ready for pickup
  PAID        // Customer paid at cashier counter
  PICKED_UP   // Item collected — order closed
  CANCELLED
}

model Printer {
  id        String        @id @default(uuid())
  storeId   String
  name      String
  ipAddress String
  status    PrinterStatus @default(OFFLINE)
  model     String
  lastSeen  DateTime?
}

enum PrinterStatus {
  ONLINE
  PRINTING
  ERROR
  OFFLINE
  MAINTENANCE
}
```

### Redis Usage

| Key Pattern | Type | TTL | Purpose |
|---|---|---|---|
| `session:{kioskId}` | Hash | 30min | Kiosk session state |
| `order:queue:{printerId}` | List | — | Bull print queue |
| `ai:cache:{promptHash}` | String | 24h | Cached AI generation |
| `order:status:{orderId}` | String | 2h | Fast status lookup |
| `printer:status:{id}` | Hash | — | Live printer state |

---

## IoT & Printer System

### Printer Communication Protocol

```
NestJS Print Service
        │
        │  WebSocket (LAN)
        ▼
Printer Adapter Layer
        │
        ├── DTG Printer Agent (Node.js daemon on printer PC)
        │       └── Receives job payload
        │       └── Converts to printer-native format
        │       └── Sends status events back
        │
        └── Direct TCP/IP (for printers with built-in network stack)
```

### Print Job Payload

```typescript
interface PrintJob {
  jobId: string;
  orderId: string;
  printFileUrl: string;   // S3 URL to high-res PNG
  product: {
    type: string;         // "tshirt"
    color: string;        // "#FFFFFF"
    size: string;         // "M"
  };
  printArea: {
    position: "FRONT" | "BACK";
    x: number;
    y: number;
    width: number;
    height: number;
  };
  priority: number;
  createdAt: string;
}
```

### Printer Status Events (Inbound)

```typescript
type PrinterEvent =
  | { type: "JOB_STARTED";   jobId: string }
  | { type: "JOB_PROGRESS";  jobId: string; percent: number }
  | { type: "JOB_COMPLETE";  jobId: string }
  | { type: "JOB_ERROR";     jobId: string; reason: string }
  | { type: "PRINTER_ONLINE" }
  | { type: "PRINTER_OFFLINE" }
```

---

## Infrastructure & Deployment

### Cloud Provider

**Primary:** AWS (or equivalent — Azure / GCP are viable alternatives)

### Services Used

| AWS Service | Purpose |
|---|---|
| ECS Fargate | NestJS microservices containers |
| RDS (PostgreSQL) | Primary database |
| ElastiCache (Redis) | Queue + cache |
| S3 | Asset storage (designs, print files, 3D models) |
| CloudFront | CDN for assets + Next.js static output |
| ALB | Load balancer for API Gateway |
| Route 53 | DNS |
| ECR | Container registry |
| CloudWatch | Logging + monitoring |

### CI/CD Pipeline

```
GitHub (monorepo)
    │
    ├── PR → GitHub Actions: lint, test, build
    │
    └── Merge to main:
            ├── Docker build + push to ECR
            ├── ECS service update (rolling deploy)
            └── Next.js build → S3 + CloudFront invalidation
```

### Environments

| Environment | Purpose |
|---|---|
| `development` | Local dev with Docker Compose |
| `staging` | Full stack on AWS, linked to test printers |
| `production` | Live store(s) |

### Docker Compose (Local Dev)

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: driplab
      POSTGRES_USER: driplab
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine

  gateway:
    build: ./apps/gateway
    ports: ["3000:3000"]
    depends_on: [postgres, redis]

  order-service:
    build: ./apps/order-service
    depends_on: [postgres, redis]

  ai-service:
    build: ./apps/ai-service
    environment:
      OPENAI_API_KEY: ${OPENAI_API_KEY}

  kiosk:
    build: ./apps/kiosk
    ports: ["3001:3001"]

  web:
    build: ./apps/web
    ports: ["3002:3002"]
```

---

## Security Architecture

### Authentication

- **JWT (RS256)** — asymmetric keys; public key shared with all services.
- Access tokens: 15-minute TTL.
- Refresh tokens: 7-day TTL, stored in HTTP-only cookie.
- Kiosk sessions: anonymous JWT with `role: KIOSK`, scoped to order/design actions only.

### Data Security

- All API traffic over **HTTPS/WSS** (TLS 1.3).
- S3 assets: private by default, served via **signed URLs** (1-hour expiry).
- Database: encrypted at rest (AWS RDS encryption enabled).
- PII (email): never logged; masked in all log outputs.
- Payment: handled entirely by the **cashier's POS system** — Smox Lab kiosk and backend never process or store payment data.

### Input Validation

- All API inputs validated with **class-validator + class-transformer** (NestJS pipes).
- Design uploads: file type whitelisting (PNG, JPEG, SVG only), virus scan via ClamAV before storage.
- AI prompts: pre-screened by OpenAI Moderation API before dispatch.

### Rate Limiting

| Endpoint | Limit |
|---|---|
| `POST /ai/generate` | 5 requests / minute / session |
| `POST /orders` | 10 requests / minute / kiosk |
| `POST /auth/login` | 5 requests / minute / IP |

---

## Data Flow Diagrams

### Flow 1: Kiosk Session → Order → Print → Cashier

```
1.  Customer approaches kiosk
2.  Kiosk: GET /products → display product list
3.  Customer selects product + size + color
4.  Customer designs on canvas
    └── (optional) POST /ai/generate → returns design options
5.  Customer enters AR preview
    └── MediaPipe: client-side only, no server call
6.  Customer confirms design
7.  POST /designs → save canvas JSON → returns designId
8.  Order summary screen → customer reviews product + price → taps CONFIRM
    └── No payment at kiosk
9.  POST /orders { productId, designId, size, color }
    └── Order Service: creates order, status = CONFIRMED
    └── Design Service: generates high-res print file (async)
    └── Redis: push job to print queue
10. Kiosk: prints physical ticket (ticket number, design thumbnail, price, est. time)
    └── WebSocket: order:status → kiosk shows QUEUED confirmation screen
11. Kiosk directs customer to the physical Gaming Lounge to wait
12. Print Service: picks job from queue
    └── Sends PrintJob payload to printer via WebSocket
    └── Order status → PRINTING
13. Printer events: PRINTING → COMPLETE
14. Order Service: status = READY
15. WebSocket (queue-display namespace): order:ready emitted
    └── Gaming Lounge TV highlights the ticket number with visual + audio alert
    └── Customer sees their number → walks to pickup counter
16. Customer presents ticket at cashier counter
    └── Cashier scans QR or enters ticket number
    └── Cashier terminal calls GET /orders/ticket/{ticketCode}
    └── Order details + price displayed on cashier screen
17. Customer pays at cashier POS (cash / card / mobile — standard POS, not Smox Lab)
18. Cashier marks order PAID → PICKED_UP in admin dashboard
    └── Item handed to customer — order closed
```

### Flow 2: AI Design Generation

```
1. User types prompt on kiosk design canvas
2. Kiosk → POST /ai/generate { prompt, style, userId }
3. AI Service:
   a. Check credits → OK
   b. Sanitize prompt
   c. Check Redis cache → miss
   d. Enhance prompt (append print-ready suffix)
   e. POST OpenAI DALL·E 3 API
   f. Receive 3 image URLs
   g. Download + run rembg background removal
   h. Upload to S3
   i. Cache result in Redis (24h)
   j. Deduct 1 credit from user
4. Return { images: [url1, url2, url3] }
5. Kiosk: display 3 options on canvas
6. User selects one → placed on canvas as layer
```