# Smox Lab — Product Requirements Document (PRD)

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** 2026-05-03  
**Owner:** Smox Lab Product Team

---

## Table of Contents

1. [Overview](#overview)
2. [Goals & Success Metrics](#goals--success-metrics)
3. [User Personas](#user-personas)
4. [Feature Requirements](#feature-requirements)
   - [Kiosk Experience](#1-kiosk-experience)
   - [AR Try-On](#2-ar-try-on)
   - [Design Canvas](#3-design-canvas)
   - [Order & Print Pipeline](#4-order--print-pipeline)
   - [Gaming Zone](#5-gaming-zone)
   - [Web Platform](#6-web-platform)
   - [Influencer & Creator System](#7-influencer--creator-system)
   - [Admin Dashboard](#8-admin-dashboard)
5. [Non-Functional Requirements](#non-functional-requirements)
6. [Constraints & Assumptions](#constraints--assumptions)
7. [Out of Scope (v1)](#out-of-scope-v1)

---

## Overview

Smox Lab is an immersive retail platform that lets customers design, preview, and receive custom-printed apparel — all within a physical store, in real time. It fuses touchscreen kiosks, augmented reality try-on, AI-assisted design generation, instant in-store printing, and an entertainment layer to create a shopping experience that is memorable, social, and deeply interactive.

### Problem Statement

Print-on-demand today is:
- **Disconnected** — entirely online, with no physical or emotional engagement.
- **Slow** — delivery takes days or weeks.
- **Template-limited** — little room for genuine creative ownership.
- **Boring** — there is no experience, just a transaction.

### Solution

Smox Lab moves this entire workflow into a physical space and wraps it in an experience layer:  
design → preview (AR) → confirm → print → pickup — all within a single store visit.

---

## Goals & Success Metrics

| Goal | KPI | Target (6-month) |
|---|---|---|
| Drive in-store conversion | % of kiosk sessions leading to a confirmed order | ≥ 60% |
| Reduce friction in design | Avg. time from kiosk start to order confirmation | ≤ 8 minutes |
| Increase AR engagement | % of orders that use the AR try-on step | ≥ 75% |
| Fast fulfillment | Avg. time from order confirmation to pickup | ≤ 12 minutes |
| Creator monetization | Active influencer accounts publishing designs | 50+ at launch |
| Platform expansion | SaaS pilot stores signed | 3 within 12 months |

---

## User Personas

### 1. The Walk-In Customer (Primary)
- Age: 16–35
- Goal: Walk in, design something unique, leave wearing (or carrying) it
- Pain points: Waiting, limited options, no creative input
- Needs: Fast, intuitive design tools, instant gratification

### 2. The Fashion Creator / Influencer
- Age: 18–40
- Goal: Publish original designs and earn commissions when others order them
- Pain points: No platform for monetizing wearable designs at retail
- Needs: Creator dashboard, design upload, analytics, payout system

### 3. The Store Operator
- Goal: Monitor orders, manage printer queues, track sales
- Needs: Real-time admin dashboard, printer status, inventory alerts

### 4. The Brand / Retailer (SaaS Customer)
- Goal: License Smox Lab technology for their own stores
- Needs: White-label kiosk software, API access, reporting

---

## Feature Requirements

### 1. Kiosk Experience

**Priority:** P0 (Must Have)

The kiosk is the primary customer-facing entry point. It runs on a large touchscreen display (minimum 32") in-store.

#### 1.1 Product Selection
- Customer selects product type: T-shirt, Hoodie, Cap, Tote Bag, etc.
- Each product displays: available colors, sizes, fabric/material info, base price.
- 3D model preview of product rotates on-screen using **Three.js**.

#### 1.2 Size & Color Configurator
- Interactive size guide with body measurement reference.
- Color palette with fabric swatch previews.
- Real-time 3D model update when color/size is changed.

#### 1.3 Session Management
- Each kiosk session is anonymous by default.
- Optional: Customer can log in / create an account for design saving and loyalty points.
- Session timeout after 3 minutes of inactivity → auto-reset with data wipe.

#### 1.4 Accessibility
- Text size toggle (standard / large).
- High-contrast mode.
- Multi-language support (Arabic, French, English at minimum).

---

### 2. AR Try-On

**Priority:** P0 (Must Have)

#### 2.1 Camera Integration
- Kiosk includes a built-in front-facing camera (minimum 1080p).
- Real-time body detection using pose estimation (TensorFlow.js / MediaPipe).
- Customer body outline is detected and used to overlay the product.

#### 2.2 AR Overlay
- Selected product is rendered on the customer's body using Three.js and a WebGL AR layer.
- Customer can:
  - Move the design left/right/up/down on the garment.
  - Scale the design up or down.
  - Toggle between front/back view.
- Overlay runs at ≥ 24 fps on the kiosk hardware.

#### 2.3 Design Positioning Sync
- Changes made in the AR view are synced back to the design canvas.
- Final design placement confirmed before order submission.

#### 2.4 Photo / GIF Capture
- Customer can capture a still photo or short GIF of themselves in the AR try-on.
- Share via QR code to their phone (optional).
- Photo is stored for 24h then auto-deleted unless the customer opts in to saving.

---

### 3. Design Canvas

**Priority:** P0 (Must Have)

#### 3.1 Free-Hand Drawing
- Full touch-based drawing canvas.
- Tools: brush, pen, eraser, shapes, fill bucket.
- Undo/Redo (up to 30 steps).
- Layer support (up to 5 layers).

#### 3.2 Text Tool
- Custom text input with font selector (10+ fonts at launch).
- Text color, size, bold/italic/underline, outline.
- Text curved/arched along a path.

#### 3.3 Image Upload
- Customer can upload an image from a USB drive or QR-linked phone transfer.
- Supported formats: PNG, JPEG, SVG.
- AI auto-background-removal on upload.

#### 3.4 AI Design Generation (Premium)
- Customer enters a text prompt: _"a futuristic city in neon colors"_
- System calls the **OpenAI DALL·E / GPT-4o Vision** API or Stable Diffusion endpoint.
- Returns 3–4 design options displayed on-screen.
- Customer selects and places the AI-generated image onto the canvas.
- Each AI generation costs a credit (bundled with premium tier or sold separately).

#### 3.5 Template Library
- 50+ base templates at launch (abstract, typography, illustration styles).
- Templates are editable — they serve as starting points, not endpoints.
- Influencer-published designs appear here (with creator attribution).

#### 3.6 Design Validation
- Before submission, validate:
  - Minimum DPI: 150 dpi equivalent for print area.
  - No copyrighted logos or faces detected (AI moderation via OpenAI moderation API).
  - Print area bounds respected.

---

### 4. Order & Print Pipeline

**Priority:** P0 (Must Have)

#### 4.1 Order Confirmation Flow
- Summary screen: product, size, color, design preview, and estimated price.
- Customer confirms the order — **no payment at the kiosk**.
- On confirmation → order is created in the system → design is sent to the print queue → ticket is issued.

#### 4.2 Print Queue
- Orders are queued via **Redis** with priority (FIFO by default).
- WebSocket push to printer module with job details (design file + product specs).
- Estimated wait time displayed on the ticket and on the kiosk screen.

#### 4.3 Printer Integration
- Abstraction layer for DTG (Direct-to-Garment) printer communication over IoT/WebSocket.
- Support multiple printer brands via adapter pattern.
- Status events: `QUEUED → PRINTING → DONE → ERROR`.
- On error: operator notified via admin dashboard + customer re-queued automatically.

#### 4.4 Pickup Ticket
- Printed physical ticket issued by the kiosk with:
  - Order ID / ticket number, estimated ready time, design thumbnail, total price.
- Optional: QR code version sent to customer's phone.
- Customer uses this ticket at the cashier counter to pay and collect their item.
- Customer can track production status on the lounge display screen while waiting.

#### 4.5 Cashier Checkout (Physical Counter)
- When the order status is `READY`, the customer proceeds to the cashier.
- Cashier scans or manually enters the ticket number to pull up the order.
- Payment is handled at the cashier counter (cash, card, mobile payment — standard POS).
- Cashier marks the order as `PAID → PICKED_UP` in the admin dashboard.
- Item is handed to the customer.

---

### 5. Gaming Zone (Physical Lounge)

**Priority:** P1 (Should Have)**

The Gaming Zone is a **dedicated physical area inside the Smox Lab store** — a lounge equipped with gaming stations (consoles, arcade machines, or PC setups) where customers wait while their order is being printed. It is not a digital feature of the kiosk; it is a designed part of the store layout and customer experience.

#### 5.1 Purpose
- Eliminate the perception of waiting — turn fulfillment time (8–15 min) into entertainment.
- Make the store social and shareable — a reason to stay, post, and return.
- Differentiate Smox Lab from any standard retail or print shop.

#### 5.2 Hardware & Setup (Store Operations Scope)
- Dedicated lounge area with seating, screens, and gaming stations.
- Recommended setups: gaming consoles (PS5 / Xbox), arcade cabinets, or PC gaming stations.
- Branded environment — Smox Lab aesthetic, LED lighting, display screens.
- Area is separate from the kiosk zone but within the same store floor.

#### 5.3 Software Integration (Smox Lab Platform Scope)
- After order confirmation, the kiosk ticket directs the customer to the Gaming Zone.
- A **display screen inside the Gaming Zone** shows a live order status board (all in-progress orders, anonymized by ticket number).
- When an order status changes to `READY`, the customer's ticket number is highlighted on the status board with a visual + audio alert.
- The status board is a dedicated Next.js page, served on a screen mounted in the lounge, connected to the WebSocket order feed.

#### 5.4 Order Status Display (Software Requirement)
- URL: `/store/queue-display` — a fullscreen, TV-optimized view.
- Displays: active ticket numbers, estimated time remaining per order, status (`PRINTING` / `READY FOR PICKUP`).
- Auto-refreshes via WebSocket — no manual reload needed.
- Plays a sound + animates the ticket number when status becomes `READY`.
- No customer PII displayed — ticket code only.

---

### 6. Web Platform

**Priority:** P1 (Should Have)**

#### 6.1 Customer Account
- Sign up / login via email or social (Google, Apple).
- View past orders and re-order.
- Save designs to a personal library.
- Loyalty points dashboard.

#### 6.2 Pre-Design from Home
- Customer can design from home on the web version.
- On arrival at store, scan QR code → design loads on kiosk.
- Reduces kiosk session time.

#### 6.3 Design Marketplace
- Browse and purchase designs published by creators.
- Filter by style, popularity, category.
- Purchase unlocks the design for a single-use print at the kiosk.

---

### 7. Influencer & Creator System

**Priority:** P2 (Nice to Have — Phase 2)**

#### 7.1 Creator Accounts
- Separate creator onboarding flow.
- Portfolio page with published designs.
- Verification badge for top creators.

#### 7.2 Design Publishing
- Creator uploads a design with tags, description, pricing.
- Design is reviewed (automated AI moderation + manual review queue).
- On approval, design is listed in the marketplace.

#### 7.3 Commission Model
- Creator earns a % of each sale of their design (configurable per creator, default 15%).
- Monthly payout via bank transfer or digital wallet.
- Creator dashboard: design performance, earnings, impressions.

---

### 9. Prayer Time Pause (Morocco)

**Priority:** P0 (Must Have — Morocco market)**

Smox Lab stores operating in Morocco observe the five daily Islamic prayer times. When a prayer time is approaching, the system gracefully pauses all customer-facing screens and the print queue, displays a respectful notice, and automatically resumes afterward.

#### 9.1 Prayer Times Coverage

The five daily prayers handled by the system:

| Prayer | Arabic | Approx. Window |
|---|---|---|
| Fajr | الفجر | Pre-dawn |
| Dhuhr | الظهر | Midday |
| Asr | العصر | Afternoon |
| Maghrib | المغرب | Sunset |
| Isha | العشاء | Night |

> Fajr typically falls outside store operating hours and may be excluded from active pause logic depending on store opening times.

#### 9.2 Pre-Prayer Warning (5 minutes before)

- All kiosk screens display a **non-blocking overlay banner**:
  > _"صلاة الظهر بعد 5 دقائق — Prayer in 5 minutes. Please complete your order or save your design."_
- The banner includes a countdown timer.
- Customers currently in a session can finish their order normally during this window.
- New sessions are blocked from starting in the last 2 minutes before prayer.

#### 9.3 Prayer Pause (at prayer time)

- All kiosk screens transition to a **full-screen Prayer Pause screen**:
  - Respectful design: Islamic geometric pattern or plain branded screen.
  - Message in Arabic and French: _"توقف للصلاة — Pause pour la prière"_
  - Estimated resume time displayed.
- The print queue is **paused** — no new jobs dispatched to printers.
- Jobs already in progress on a printer are allowed to **complete** before the printer is considered paused.
- The queue display screen in the Gaming Lounge also switches to the Prayer Pause view.
- Admin dashboard shows a `PRAYER_PAUSE` system status badge.

#### 9.4 Resume

- **Automatic resume:** System resumes automatically after a configurable prayer duration (default: 30 minutes for Dhuhr/Asr/Isha, 20 minutes for Maghrib).
- **Manual resume:** Store operator can resume early from the admin dashboard.
- On resume:
  - Kiosk screens return to the idle/welcome state.
  - Print queue resumes from where it paused.
  - Customers are not required to re-enter their session data (it is preserved in Redis during the pause).

#### 9.5 Configuration (Per Store)

Store operators can configure the following in the admin dashboard:

| Setting | Default | Description |
|---|---|---|
| City / coordinates | Casablanca | Used to calculate accurate local prayer times |
| Calculation method | Moroccan Ministry of Habous | Recognized official Moroccan standard |
| Prayer duration (per prayer) | 30 min | How long the pause lasts before auto-resume |
| Prayers to pause | Dhuhr, Asr, Maghrib, Isha | Fajr excluded by default |
| Warning lead time | 5 minutes | How early the pre-prayer banner appears |
| Manual override | Enabled | Operator can pause/resume manually at any time |

#### 9.6 Edge Cases

- **Order paid but not yet printed when pause starts:** Order is preserved in queue; resumes printing after prayer. Customer ticket shows updated estimated time.
- **Customer mid-confirmation when pause starts:** The order confirmation is not interrupted — the ticket is issued, and the print job is queued for post-prayer production.
- **Prayer time falls outside store hours:** No action taken; system skips that prayer event.
- **Offline / API failure for prayer times:** System falls back to the last-cached prayer schedule. If no cache exists, operator receives an alert and must enable pause manually.

---

### 8. Admin Dashboard

**Priority:** P0 (Must Have)**

#### 8.1 Order Management
- Real-time order list with status filters.
- Drill-down into individual order: design, customer (if logged in), printer assigned, timestamps.

#### 8.2 Printer Management
- Live printer status board (online, printing, error, offline).
- Manual queue management: pause, skip, reassign.
- Maintenance mode per printer.

#### 8.3 Analytics
- Daily/weekly/monthly: orders, revenue, avg. order value.
- Top products, top designs, peak hours.
- AR session usage rate, AI generation usage.

#### 8.5 Prayer Time Management
- View today's prayer schedule for the store's configured city.
- Manual pause / resume button (override auto-schedule).
- Configure prayer duration and which prayers trigger a pause.
- View pause history log (prayer, start time, resume time, orders affected).
- Upload / manage template library.
- Manage product catalog (prices, colors, sizes).
- Creator design approval queue.

---

## Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Performance** | Kiosk UI renders at ≥ 60 fps; AR at ≥ 24 fps |
| **Reliability** | System uptime ≥ 99.5% during store hours |
| **Latency** | AI design generation response ≤ 6 seconds |
| **Scalability** | Backend supports 10 concurrent kiosks per store, 100 stores via SaaS |
| **Security** | PCI-DSS compliance handled by cashier POS system (outside kiosk scope); GDPR compliance for EU; no PII in logs |
| **Offline resilience** | Kiosk functions in limited mode if internet drops (no AI, no AR cloud sync) |
| **Data retention** | AR photos deleted after 24h unless opt-in; design files retained 90 days |
| **Accessibility** | WCAG 2.1 AA on web platform |

---

## Constraints & Assumptions

- Kiosk hardware is a fixed configuration managed by Smox Lab (not BYOD).
- All in-store printers are DTG-capable and can receive print jobs over LAN/WebSocket.
- Initial markets: Morocco, France, UAE (multilingual from day one).
- AI generation credits are consumed per call to the OpenAI API — cost must be factored into premium pricing.
- Stable Diffusion self-hosted is a Phase 2 option for cost optimization at scale.

---

## Out of Scope (v1)

- Mobile app (iOS / Android) — web-responsive is sufficient for v1.
- Delivery / shipping integration — in-store pickup only.
- Multi-store inventory sync — each store operates independently in v1.
- NFT or blockchain integration.
- Voice UI / voice commands.