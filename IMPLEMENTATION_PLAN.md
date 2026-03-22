# Session Relay - Implementation Plan

## Executive Summary

SessionRelay is scheduling and booking software for recording studios that coordinates rooms, engineers, holds, quotes, and client bookings in one workflow.

This document outlines the implementation plan for a web-based scheduling system for audio studios. The system will facilitate session bookings between clients and audio engineers, manage room availability, and provide intelligent recommendations based on engineer specialties and session requirements.

## Table of Contents

1. [System Overview](#system-overview)
2. [Core Features](#core-features)
3. [Technical Architecture](#technical-architecture)
4. [Database Schema](#database-schema)
5. [Business Logic & Rules](#business-logic--rules)
6. [Technology Stack](#technology-stack)
7. [User Flows](#user-flows)
8. [Implementation Phases](#implementation-phases)
9. [API Design](#api-design)
10. [Email Notification System](#email-notification-system)
11. [Future Extensibility](#future-extensibility)
12. [Next Steps](#next-steps)

---

## System Overview

### Current Studio Setup
- **3 Full-time Engineers**
- **2 Rooms (expanding to 3)**:
  - **Room A**: Large tracking room + control room; suited for bands, full tracking, overdubs, mixing
  - **Room B**: Small tracking room + control room; suited for overdubs, vocals, audiobooks, single instruments
  - **Room C** (Future): Control room only; suited for mixing, editing, mastering; not available for live tracking sessions

### Key Challenges Being Solved
1. Coordinating availability across multiple engineers and rooms
2. Matching client needs with appropriate engineer expertise and room type
3. Reducing administrative overhead in scheduling
4. Providing clients with real-time availability
5. Maintaining quote-based confirmation while enabling self-service booking requests

### Booking Workflow Model

**Quote-Based Confirmation System**:
The system uses a state-driven booking lifecycle to ensure availability is protected while giving engineers time to quote and clients time to accept:

1. **Client Request** → System creates soft hold (prevents double-booking)
2. **Engineer Quotes** → Pricing calculated from room + engineer rates with optional adjustments
3. **Client Accepts** → Soft hold becomes confirmed booking
4. **TTL-Based Expiration** → Requests/quotes expire automatically if not acted upon, releasing holds

**Booking Lead Time Constraints**:
- **Minimum lead time**: 3 days (configurable) - ensures workflow can complete
- **Maximum lead time**: 180 days (configurable) - aligns with realistic studio horizons
- **Admin override**: Studio staff can bypass constraints for manual/urgent bookings

**State Lifecycle**:
- `REQUESTED` (72h TTL) → Engineer sends quote, requests info, or declines
- `AWAITING_CLIENT_INFO` (48h TTL) → Client provides requested details
- `QUOTED` (48h TTL) → Client accepts or quote expires
- `BOOKED` → Confirmed and locked
- `EXPIRED` / `CANCELLED` / `DECLINED` → Hold released, time slot available

**Key Benefits**:
- No indefinite holds blocking availability
- Clear responsibility at each stage (engineer vs client)
- Automatic expiration prevents stale requests
- Flexible pricing (standard rates + optional adjustments)

---

## Core Features

### 1. Client Booking Portal

#### Booking Limits & Abuse Prevention

To prevent calendar abuse, the system may limit active standalone `REQUESTED` holds per client while allowing one recurring request batch within the booking horizon.

Recurring requests should count as a single request batch for abuse-prevention purposes, while still creating individual booking records for lifecycle handling.

- **Guest Booking**: No account required, email-based
- **Real-time Availability**: View available time slots across rooms and engineers
  - **Booking Lead Time Constraints**:
    - Minimum lead time: 3 days (configurable) - clients cannot book sessions starting within N days
    - Maximum lead time: 180 days (~6 months, configurable) - clients cannot book beyond N days
    - Availability window: `[today + minimum_lead_time_days, today + maximum_lead_time_days]`
    - Rationale: Ensures booking workflow (request → quote → acceptance) can complete without blocking near-term availability
    - **⚠️ Server-side Enforcement**: Lead time constraints MUST be validated in backend API (`POST /api/bookings`), not just in the UI. Prevents bypassing via direct API calls or browser manipulation.
- **Session Details Capture**:
  - Band/Artist/Client name
  - Session type (Audiobook, Tracking, Mixing, Mastering, Overdubs)
  - Duration (configurable minimum session duration, set by studio staff/admins)
  - Contact info (email required, phone optional)
  - Additional notes
- **Smart Recommendations**: System suggests optimal engineer/room combinations based on:
  - Session type
  - Engineer preferences and specialties
  - Room capabilities
  - Availability
- **Booking Validation Rules**: The UI validates selected engineer/room/session combinations in real time and prevents submission when the selection violates room capabilities, engineer rules, or studio booking policies
  - **Eligibility Constraints** (must block booking):
    - Room C selected for band/live tracking sessions
    - Engineer unavailable at selected time
    - Room already booked
    - Session violates room capabilities
  - **Preference Scoring** (warn but allow, or rank lower):
    - Room is not ideal but still technically valid
    - Engineer prefers not to do that session type
    - Suboptimal time slot for engineer
- **Recurring Bookings**: Schedule repeated sessions (e.g., every Monday at 2 PM for 3 hours)

### 2. Engineer Dashboard
- **Schedule Management**:
  - View personal calendar
  - Block out OOO (Out of Office) time
  - Set working hours/availability
- **Profile Management**:
  - Define specialties (Audiobook, Tracking, Mixing, Mastering, Overdubs)
  - Prioritize preferred session types
  - Set hourly rates by service type (tracking, mixing, audiobook editing, mastering, etc.)
- **Booking & Quoting Workflow**:
  - **State-Based Lifecycle Management**:
    - `REQUESTED`: Client submits request → soft hold created → engineer notified
    - `AWAITING_CLIENT_INFO` (optional): Engineer requests more details → client must respond
    - `QUOTED`: Engineer sends quote → client must accept/decline
    - `BOOKED`: Client accepts quote → time slot fully locked
    - `EXPIRED`: Request/quote timed out → hold released
    - `CANCELLED`/`DECLINED`: Explicit rejection → hold released
    - `COMPLETED`/`FINAL_INVOICED`: Post-session states
  - **Quote as First-Class Concept (V1)**:
    - **No upfront pricing** - all bookings require engineer-generated quotes
    - Quote includes:
      - `quoted_amount`: Total price for the session
      - `quote_sent_at`: Timestamp when quote was generated
      - `expires_at`: When the quote expires (typically 24-48 hours)
      - Breakdown: Room rate + Engineer rate + Manual adjustments
    - Quote calculation: `(room_rate + engineer_rate) × hours + manual_adjustment`
    - Booking only confirmed after client accepts quote (status → BOOKED)
  - **State-Based Expiration (TTL)**:
    - `REQUESTED`: 72 hours (configurable) → engineer responsible
      - Daily reminders to engineer
      - Day 3: notify studio owner
      - No action → request expires, hold released
    - `AWAITING_CLIENT_INFO`: 24-48 hours (configurable) → client responsible
      - No response → request expires, hold released
    - `QUOTED`: 24-48 hours (configurable) → client responsible
      - No acceptance → quote expires, hold released
  - **Soft Hold Behavior**:
    - Prevents other bookings for the time slot
    - Temporary and tied to booking state
    - Never indefinite
    - Automatically released on expiration or cancellation

### 3. Room Management
- **Room Profiles**:
  - Room capabilities and equipment
  - Pricing tier
  - Suitable session types
- **Availability Calendar**: Visual representation of room bookings
- **Auto-assignment Logic**:
  - Audiobooks → always Room B
  - Bands/live tracking → Room A only (Room C not available for live tracking)
  - Mixing/mastering/editing → Room A or Room C
  - Overdubs/single instruments → Room B preferred, Room A if unavailable

### 4. Client Database
- **Client Records**:
  - Contact information (email, phone)
  - Booking history
  - Project notes
  - Previous engineers worked with
- **Search and Filter**: Quick lookup by name, email, or project

### 5. Pricing System
- **Two-Component Pricing Model**:
  - **Room Pricing**:
    - Set by owner/admin
    - Can vary by room (Room A, B, or C may have different rates)
    - Can optionally vary by session type (e.g., tracking vs. mixing)
  - **Engineer Pricing**:
    - Set by individual engineer
    - Can vary by service type (tracking, mixing, audiobook editing, mastering, etc.)
  - Combined calculation for total session cost: (Room Rate + Engineer Rate) × Hours
- **Pricing Model (V1)**:
  - **All sessions require quotes** - no upfront pricing displayed to clients
  - Client submits booking request without seeing cost
  - Engineer reviews and generates custom quote
  - Client receives quote via email and can accept/decline
  - **Future Enhancement idea**: Configurable pricing visibility modes (exact, range, hidden, quote-required) based on session type

#### Pricing Visibility Lifecycle (V1)

**Before Quote is Sent** (Status: REQUESTED or AWAITING_CLIENT_INFO):
- ❌ **No pricing visible** to client anywhere
- Public APIs (`/api/engineers`, `/api/rooms`, `/api/availability`) return **no pricing fields**
- Booking request flow shows no cost estimates
- Booking request confirmation email: "Your engineer will send you a quote within 72 hours"

**When Quote is Sent** (Status: QUOTED):
- ✅ **Pricing visible** in quote email sent to client
- Quote email includes:
  - `quotedAmount`: Total price (e.g., $350.00)
  - Breakdown: Room rate ($100/hr × 2hr) + Engineer rate ($75/hr × 2hr) + Adjustment (-$50 repeat client discount)
  - Expiration: "This quote expires in 48 hours"
  - Call-to-action: "Accept Quote" button

**After Quote Accepted** (Status: BOOKED):
- ✅ **Accepted quote amount visible** in:
  - **Confirmation email** (client & engineer): "Total: $350.00"
  - **Booking status page** (`/api/public/bookings/status/:confirmationCode`): Shows `quotedAmount` and `quoteAcceptedAt`
  - **Engineer dashboard**: Full quote breakdown visible
  - **Admin/Owner views**: Full quote breakdown + payment tracking

**Engineer/Admin Pricing Visibility** (All States):
- ✅ Engineers and admins always see full pricing details:
  - Base room rates and engineer rates (in admin settings)
  - Quote calculations and adjustments (when sending quotes)
  - Historical quote amounts (in booking history)

---

## Technical Architecture

### Architecture Pattern: Three-Tier Application

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT LAYER                          │
│  ┌──────────────────┐         ┌──────────────────┐     │
│  │  Client Portal   │         │ Engineer Dashboard│     │
│  │   (React SPA)    │         │    (React SPA)    │     │
│  └──────────────────┘         └──────────────────┘     │
└─────────────────────────────────────────────────────────┘
                            │
                            │ REST API / GraphQL
                            ▼
┌─────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                      │
│  ┌────────────────────────────────────────────────┐    │
│  │         Backend API (Node.js/Express)          │    │
│  │  - Business Logic                              │    │
│  │  - Authentication & Authorization              │    │
│  │  - Booking Engine                              │    │
│  │  - Recommendation Algorithm                    │    │
│  │  - Email Service                               │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                     DATA LAYER                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  PostgreSQL  │  │    Redis     │  │  File Store  │ │
│  │  (Primary DB)│  │   (Cache)    │  │  (Optional)  │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Frontend (React)
- **Client Portal**:
  - Booking form with real-time availability
  - Calendar view
  - Booking confirmation and status tracking
- **Engineer Dashboard**:
  - Personal schedule view (showing REQUESTED, QUOTED, BOOKED states)
  - Pending requests queue (requires quote generation)
  - Quote management (view sent quotes, track acceptances)
  - Profile/preferences management (including service-based pricing)
  - OOO management
- **Admin Panel** (Future):
  - Studio overview and analytics
  - Room management (add/edit rooms, set room pricing by session type)
  - Studio configuration settings:
    - Minimum session duration
    - Booking lead time constraints:
      * Minimum lead time (days) - default: 3
      * Maximum lead time (days) - default: 180
      * Admin override for manual bookings
    - State expiration TTL settings:
      * REQUESTED timeout (hours) - default: 72
      * AWAITING_CLIENT_INFO timeout (hours) - default: 48
      * QUOTED timeout (hours) - default: 48
    - Business hours and booking policies
    - **V1 Note**: Pricing visibility config NOT included in V1 admin panel (always quote-required)
  - Engineer management (view all engineers, pricing oversight)
  - Manual booking creation (bypasses lead time constraints)

#### 2. Backend API (Node.js/Express or Python/FastAPI)
- **Authentication Service**: JWT-based auth for engineers/admin
- **Booking Service**: Core scheduling logic, conflict detection, and soft hold management
- **Quote Engine**:
  - Calculate pricing from room rates + engineer service rates
  - Support manual adjustments (discounts, overrides)
  - Generate quote with expiration
- **State Machine Service**: Manage booking lifecycle state transitions
- **Expiration Service**: Background job to handle TTL-based expiration and release holds
- **Recommendation Engine**: Match clients with engineers/rooms
- **Notification Service**: Email notifications for requests, quotes, acceptances, expirations, escalations
- **Calendar Service**: Availability calculation (respecting soft holds), recurring event handling, lead time validation

#### 3. Database (PostgreSQL)
- Relational data model for entities and relationships
- Support for complex queries (availability, conflicts, recommendations)
- Transaction support for booking operations

---

## Database Schema

### Entity Relationship Diagram

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Clients   │       │  Bookings   │       │  Engineers  │
│─────────────│       │─────────────│       │─────────────│
│ id (PK)     │───┐   │ id (PK)     │   ┌───│ id (PK)     │
│ email       │   └──<│ client_id   │   │   │ name        │
│ phone       │       │ engineer_id │>──┘   │ email       │
│ name        │       │ room_id     │>──┐   │ created_at  │
│ created_at  │       │ session_type│   │   └─────────────┘
└─────────────┘       │ start_time  │   │          │
                      │ end_time    │   │          │
                      │ status      │   │   ┌──────▼──────────┐
                      │ total_hours │   │   │EngineerSpecialty│
                      │ notes       │   │   │─────────────────│
                      │ created_at  │   │   │ id (PK)         │
                      └─────────────┘   │   │ engineer_id (FK)│
                             │          │   │ session_type    │
                             │          │   │ priority        │
                      ┌──────▼──────┐   │   └─────────────────┘
                      │  Rooms      │   │
                      │─────────────│   │   ┌─────────────────┐
                      │ id (PK)     │<──┘   │ EngineerOOO     │
                      │ name        │       │─────────────────│
                      │ type        │       │ id (PK)         │
                      │ capacity    │       │ engineer_id (FK)│
                      │ description │       │ start_time      │
                      └─────────────┘       │ end_time        │
                             │              │ reason          │
                             │              └─────────────────┘
                      ┌──────▼──────────┐
                      │RoomCapabilities │   ┌─────────────────┐
                      │─────────────────│   │RecurringBookings│
                      │ id (PK)         │   │─────────────────│
                      │ room_id (FK)    │   │ id (PK)         │
                      │ session_type    │   │ booking_id (FK) │
                      │ is_preferred    │   │ recurrence_rule │
                      │ is_allowed      │   │ end_date        │
                      └─────────────────┘   └─────────────────┘
                             │
                      ┌──────▼──────────┐   ┌─────────────────┐
                      │  RoomPricing    │   │ Escalations     │
                      │─────────────────│   │─────────────────│
                      │ id (PK)         │   │ id (PK)         │
                      │ room_id (FK)    │   │ booking_id (FK) │
                      │ session_type    │   │ escalated_at    │
                      │ hourly_rate     │   │ resolved_at     │
                      └─────────────────┘   └─────────────────┘

                      ┌─────────────────┐
                      │EngineerPricing  │
                      │─────────────────│
                      │ id (PK)         │
                      │ engineer_id (FK)│
                      │ service_type    │
                      │ hourly_rate     │
                      └─────────────────┘
```

### Table Definitions

#### `engineers`
```sql
CREATE TABLE engineers (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(50),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `engineer_pricing`
```sql
CREATE TABLE engineer_pricing (
  id SERIAL PRIMARY KEY,
  engineer_id INTEGER REFERENCES engineers(id) ON DELETE CASCADE,
  service_type VARCHAR(50) NOT NULL, -- 'tracking', 'mixing', 'mastering', 'audiobook_editing', etc.
  hourly_rate DECIMAL(10,2) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(engineer_id, service_type)
);
```

#### `engineer_specialties`
```sql
CREATE TABLE engineer_specialties (
  id SERIAL PRIMARY KEY,
  engineer_id INTEGER REFERENCES engineers(id) ON DELETE CASCADE,
  session_type VARCHAR(50) NOT NULL,
  priority INTEGER DEFAULT 0, -- Higher = more preferred
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(engineer_id, session_type)
);
```

#### `engineer_ooo`
```sql
CREATE TABLE engineer_ooo (
  id SERIAL PRIMARY KEY,
  engineer_id INTEGER REFERENCES engineers(id) ON DELETE CASCADE,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP NOT NULL,
  reason VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CHECK (end_time > start_time)
);
```

#### `rooms`
```sql
CREATE TABLE rooms (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL UNIQUE,
  type VARCHAR(50), -- 'tracking', 'control', 'mixing_only'
  capacity INTEGER,
  description TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `room_pricing`
```sql
CREATE TABLE room_pricing (
  id SERIAL PRIMARY KEY,
  room_id INTEGER REFERENCES rooms(id) ON DELETE CASCADE,
  session_type VARCHAR(50), -- NULL means default/base rate for all session types
  hourly_rate DECIMAL(10,2) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(room_id, session_type)
);
```

#### `room_capabilities`
```sql
CREATE TABLE room_capabilities (
  id SERIAL PRIMARY KEY,
  room_id INTEGER REFERENCES rooms(id) ON DELETE CASCADE,
  session_type VARCHAR(50) NOT NULL,
  is_allowed BOOLEAN DEFAULT true, -- false means this room cannot be used for this session type
  is_preferred BOOLEAN DEFAULT false, -- true means this room is ideal for this session type
  UNIQUE(room_id, session_type)
);
```

#### `clients`
```sql
CREATE TABLE clients (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL,
  phone VARCHAR(50),
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `bookings`
```sql
CREATE TABLE bookings (
  id SERIAL PRIMARY KEY,
  client_id INTEGER REFERENCES clients(id),
  engineer_id INTEGER REFERENCES engineers(id),
  room_id INTEGER REFERENCES rooms(id),
  session_type VARCHAR(50) NOT NULL,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP NOT NULL,
  total_hours DECIMAL(5,2) NOT NULL,

  -- State-based lifecycle
  status VARCHAR(50) NOT NULL DEFAULT 'REQUESTED',

  -- Valid states: REQUESTED, AWAITING_CLIENT_INFO, QUOTED, BOOKED, EXPIRED, CANCELLED, DECLINED, COMPLETED, FINAL_INVOICED
  CHECK (status IN (
    'REQUESTED',
    'AWAITING_CLIENT_INFO',
    'QUOTED',
    'BOOKED',
    'EXPIRED',
    'CANCELLED',
    'DECLINED',
    'COMPLETED',
    'FINAL_INVOICED'
  )),

  -- State transitions (with idempotency guards - see Business Logic Rules below):
  -- REQUESTED → QUOTED | AWAITING_CLIENT_INFO | EXPIRED | DECLINED (engineer declines)
  -- AWAITING_CLIENT_INFO → REQUESTED | EXPIRED
  -- QUOTED → BOOKED | EXPIRED (no client response) | DECLINED (client explicitly rejects)
  -- BOOKED → COMPLETED | CANCELLED
  -- COMPLETED → FINAL_INVOICED

  -- Expiration management (strict lifecycle ownership - see Business Logic Rules)
  expires_at TIMESTAMP, -- When current state expires (REQUESTED: 72h, AWAITING_CLIENT_INFO: 48h, QUOTED: 48h)
  -- ⚠️ RULE: expires_at can ONLY be set/changed during state transitions via application logic. No manual updates allowed.

  -- Quote fields
  -- V1 Note: Quote is a first-class workflow concept (not a separate table in V1)
  quoted_amount DECIMAL(10,2), -- Total quoted price (only set when status = QUOTED or BOOKED)
  room_rate DECIMAL(10,2), -- Room hourly rate at time of quote
  engineer_rate DECIMAL(10,2), -- Engineer hourly rate at time of quote
  manual_adjustment DECIMAL(10,2) DEFAULT 0, -- Discount or override amount (+/-)
  quote_sent_at TIMESTAMP, -- When engineer sent the quote
  quote_accepted_at TIMESTAMP, -- When client accepted the quote (status → BOOKED)

  -- Rescheduling & Cancellation tracking (V1 - lightweight policy)
  -- V1 Note: Reschedule = new booking request + link to original. Do not modify BOOKED records in place.
  rescheduled_from_booking_id INTEGER REFERENCES bookings(id), -- This booking replaces a previous one
  rescheduled_to_booking_id INTEGER REFERENCES bookings(id), -- This booking was replaced by a new one
  cancelled_at TIMESTAMP, -- When booking was cancelled
  cancelled_by VARCHAR(50), -- 'CLIENT' | 'ENGINEER' | 'ADMIN' | 'SYSTEM' (e.g., OOO conflict)
  CHECK (cancelled_by IS NULL OR cancelled_by IN (
    'CLIENT',
    'ENGINEER',
    'ADMIN',
    'SYSTEM'
  )),

  -- Structured reason tracking (use codes + optional freeform text)
  reason_code VARCHAR(100), -- Standardized reason code (see Reason Codes section below)
  CHECK (reason_code IS NULL OR reason_code IN (
    'DECLINED_DUE_TO_ENGINEER_OOO',
    'DECLINED_BY_ENGINEER_UNAVAILABLE',
    'DECLINED_BY_ENGINEER_OTHER',
    'DECLINED_BY_CLIENT_QUOTE',
    'CANCELLED_BY_CLIENT',
    'CANCELLED_BY_ENGINEER',
    'CANCELLED_BY_ADMIN',
    'CANCELLED_DUE_TO_RESCHEDULE',
    'CANCELLED_DUE_TO_ENGINEER_OOO'
  )),
  reason_text TEXT, -- Optional freeform explanation (user-provided or auto-generated)

  -- Examples:
  -- DECLINED_DUE_TO_ENGINEER_OOO, DECLINED_BY_ENGINEER_UNAVAILABLE, DECLINED_BY_CLIENT_QUOTE
  -- CANCELLED_BY_CLIENT, CANCELLED_BY_ENGINEER, CANCELLED_DUE_TO_RESCHEDULE

  -- Future enhancement fields (not enforced in V1):
  -- cancellation_window_hours (e.g., 48h/24h), deposit_amount, late_fee

  -- Additional info
  notes TEXT,
  client_notes TEXT, -- Notes from client when requesting
  engineer_notes TEXT, -- Notes from engineer (internal or to client)

  -- Confirmation code (secure, high-entropy, non-sequential - see Security section)
  confirmation_code VARCHAR(20) UNIQUE NOT NULL, -- Public-facing ID for client lookups (e.g., 'BK-A7X9-K4M2')

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CHECK (end_time > start_time)
);
```

#### `recurring_bookings` (Future Enhancement - Not Used in V1)
**V1 Note**: Recurring bookings are created as independent `bookings` records. No special linking or grouping.

**Future Schema**:
```sql
CREATE TABLE recurring_bookings (
  id SERIAL PRIMARY KEY,
  recurring_group_id UUID NOT NULL, -- Groups related bookings together
  parent_booking_id INTEGER REFERENCES bookings(id) ON DELETE CASCADE,
  recurrence_rule VARCHAR(255) NOT NULL, -- e.g., 'WEEKLY:MONDAY', 'WEEKLY:MONDAY,WEDNESDAY'
  end_date DATE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**V1 Note**: Alternative time slots (when engineer declines) are sent via email only - not stored in database in V1. Future enhancement: Add `alternative_slots` table linked to bookings.

#### `escalations`
```sql
CREATE TABLE escalations (
  id SERIAL PRIMARY KEY,
  booking_id INTEGER REFERENCES bookings(id) ON DELETE CASCADE,
  engineer_id INTEGER REFERENCES engineers(id),
  escalated_to_email VARCHAR(255) NOT NULL,
  escalated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  resolved BOOLEAN DEFAULT false,
  resolved_at TIMESTAMP
);
```

#### `studio_settings`
```sql
CREATE TABLE studio_settings (
  id SERIAL PRIMARY KEY,
  setting_key VARCHAR(100) NOT NULL UNIQUE,
  setting_value TEXT,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example settings:
-- INSERT INTO studio_settings (setting_key, setting_value, description) VALUES
-- ('minimum_session_duration_hours', '2', 'Minimum session duration in hours'),
-- ('business_hours_start', '09:00', 'Studio opening time'),
-- ('business_hours_end', '21:00', 'Studio closing time'),
-- ('minimum_lead_time_days', '3', 'Minimum days between booking request and session start'),
-- ('maximum_lead_time_days', '180', 'Maximum days into future that clients can book'),
-- ('ttl_requested_hours', '72', 'Hours before REQUESTED bookings expire'),
-- ('ttl_awaiting_client_info_hours', '48', 'Hours before AWAITING_CLIENT_INFO bookings expire'),
-- ('ttl_quoted_hours', '48', 'Hours before QUOTED bookings expire'),
-- ('allow_admin_override_lead_time', 'true', 'Allow admins to bypass lead time constraints for manual bookings');
```

#### `pricing_visibility_rules` (Future Enhancement - Not Used in V1)
**V1 Note**: All session types use `quote_required` mode. No configurable pricing visibility.

**Future Schema** (for configurable visibility modes):
```sql
CREATE TABLE pricing_visibility_rules (
  id SERIAL PRIMARY KEY,
  session_type VARCHAR(50) NOT NULL,
  project_classification VARCHAR(50), -- NULL means applies to all project types
  visibility_mode VARCHAR(50) NOT NULL, -- 'exact', 'range', 'hidden', 'quote_required'
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(session_type, project_classification)
);

-- Future example rules (not used in V1):
-- INSERT INTO pricing_visibility_rules (session_type, visibility_mode) VALUES
-- ('Audiobook', 'exact'),
-- ('Tracking', 'range'),
-- ('Mixing', 'exact'),
-- ('Mastering', 'quote_required');
```

---

## Business Logic & Rules

### Availability Blocking Rules

**V1 Simplified Approach**: No state priority order - all active bookings block equally

**Core Principle**: A time slot is available ONLY if BOTH the engineer AND the room are free

**Availability = Engineer Available ∧ Room Available**

A time slot is **unavailable** (blocked from new bookings) if ANY existing booking for that **engineer OR room** (independently) overlaps with the requested time and is in one of these states:

**Blocking States**:
- `REQUESTED` - Soft hold active, awaiting engineer action
- `AWAITING_CLIENT_INFO` - Soft hold active, awaiting client response
- `QUOTED` - Soft hold active, awaiting client acceptance
- `BOOKED` - Confirmed booking, hard lock

**Examples**:
```
Scenario 1: Engineer busy, room free
- Existing: Engineer A + Room 1, 2pm-4pm, status=BOOKED
- New Request: Engineer A + Room 2, 2pm-4pm
- Result: ❌ BLOCKED (Engineer A not available during 2pm-4pm)

Scenario 2: Room busy, engineer free
- Existing: Engineer A + Room 1, 2pm-4pm, status=BOOKED
- New Request: Engineer B + Room 1, 2pm-4pm
- Result: ❌ BLOCKED (Room 1 not available during 2pm-4pm)

Scenario 3: Both free
- Existing: Engineer A + Room 1, 2pm-4pm, status=BOOKED
- New Request: Engineer B + Room 2, 2pm-4pm
- Result: ✅ AVAILABLE (different engineer AND different room)
```

**All states block equally in V1**: If any overlapping booking exists for the same engineer OR room + time in any of the above states, reject the new request. No priority ordering needed.

A time slot is **available** (released for new bookings) when a booking transitions to:

- `EXPIRED` - Request/quote timed out, hold automatically released
- `CANCELLED` - Client cancelled, hold manually released
- `DECLINED` - Engineer declined, hold manually released

**Implementation Notes**:
- Availability check queries must check BOTH engineer AND room independently
- Filter out bookings with status IN ('EXPIRED', 'CANCELLED', 'DECLINED', 'COMPLETED', 'FINAL_INVOICED')
- Background expiration job transitions `REQUESTED`, `AWAITING_CLIENT_INFO`, `QUOTED` → `EXPIRED` when `expires_at` < NOW()
- Immediate release on `DECLINED` or `CANCELLED` actions

### Concurrency & Double Booking Prevention

**Strategy**: First valid request wins

**Implementation**:
1. **Transactional Check**: When creating a booking request, use database transaction with row-level locking
2. **Query for Conflicts** (check engineer OR room independently):
   ```sql
   -- Check if EITHER the engineer OR the room is already booked/held
   SELECT id FROM bookings
   WHERE status IN ('REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED', 'BOOKED')
     AND (
       engineer_id = ?  -- Same engineer (regardless of room)
       OR room_id = ?   -- Same room (regardless of engineer)
     )
     AND (start_time, end_time) OVERLAPS (?, ?)
   FOR UPDATE;
   ```

   **Alternative explicit query** (same logic, more readable):
   ```sql
   -- Check engineer availability
   SELECT id FROM bookings
   WHERE engineer_id = ?
     AND status IN ('REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED', 'BOOKED')
     AND (start_time, end_time) OVERLAPS (?, ?)
   FOR UPDATE;

   -- Check room availability
   SELECT id FROM bookings
   WHERE room_id = ?
     AND status IN ('REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED', 'BOOKED')
     AND (start_time, end_time) OVERLAPS (?, ?)
   FOR UPDATE;

   -- If either query returns results → conflict exists
   ```

3. **Conflict Resolution**:
   - If engineer conflict exists → return error "Engineer not available during requested time"
   - If room conflict exists → return error "Room not available during requested time"
   - If no conflicts → insert booking with status='REQUESTED'
4. **Race Condition Handling**: Database-level unique constraint or serializable transaction isolation

**Edge Cases**:
- Two simultaneous requests for same slot: First transaction commits wins, second fails with conflict error
- Quote acceptance while another request pending: Transaction ensures only one succeeds

### State Transition Guards & Idempotency

**Requirement**: All state transitions must validate current state before allowing transition (prevents invalid operations)

**Implementation Pattern**:
```javascript
// Example: Accept Quote
function acceptQuote(bookingId, confirmationCode) {
  const booking = getBookingByConfirmationCode(confirmationCode);

  // Idempotency guard
  if (booking.status === 'BOOKED') {
    return { success: true, message: "Quote already accepted" };
  }

  // State guard
  if (booking.status !== 'QUOTED') {
    throw new Error(`Cannot accept quote. Current status: ${booking.status}. Expected: QUOTED`);
  }

  // Check expiration
  if (booking.expires_at && new Date() > new Date(booking.expires_at)) {
    throw new Error("Quote has expired");
  }

  // Proceed with transition
  updateBooking(bookingId, {
    status: 'BOOKED',
    quote_accepted_at: NOW(),
    expires_at: null // Clear expiration
  });
}
```

**Required Guards by Endpoint**:

| Endpoint | Current Status Must Be | Idempotency Behavior |
|----------|----------------------|---------------------|
| `POST /api/bookings/:id/quote` | REQUESTED or AWAITING_CLIENT_INFO | If already QUOTED, return existing quote |
| `POST /api/public/bookings/accept-quote` | QUOTED | If already BOOKED, return success |
| `POST /api/public/bookings/decline-quote` | QUOTED | If already DECLINED/EXPIRED, return success |
| `POST /api/bookings/:id/request-info` | REQUESTED | If already AWAITING_CLIENT_INFO, return existing request |
| `POST /api/bookings/:id/decline` | REQUESTED or AWAITING_CLIENT_INFO or QUOTED | If already DECLINED, return success |
| `POST /api/public/bookings/cancel` | BOOKED | If already CANCELLED, return success |
| `POST /api/public/bookings/request-reschedule` | BOOKED | Check if reschedule already pending |

**Benefits**:
- Prevents double-processing (e.g., accepting quote twice)
- Protects against race conditions
- Provides clear error messages for invalid operations
- Safe retries for network failures

### expires_at Ownership & Lifecycle Rules

**Core Principle**: `expires_at` is **lifecycle-driven only** - no manual updates outside state transitions

**Rules**:

1. **Set expires_at ONLY during these state transitions**:
   ```javascript
   // New booking request
   REQUESTED: expires_at = NOW() + 72 hours (configurable)

   // Engineer requests more info
   AWAITING_CLIENT_INFO: expires_at = NOW() + 48 hours (configurable)

   // Engineer sends quote
   QUOTED: expires_at = NOW() + 48 hours (configurable)
   ```

2. **Clear expires_at when transitioning to terminal or active states**:
   ```javascript
   BOOKED: expires_at = null (no longer expires)
   COMPLETED: expires_at = null
   CANCELLED: expires_at = null
   DECLINED: expires_at = null
   FINAL_INVOICED: expires_at = null
   ```

3. **Never allow direct UPDATE of expires_at**:
   ```sql
   -- ❌ INVALID - bypasses lifecycle logic
   UPDATE bookings SET expires_at = '2024-04-01' WHERE id = 101;

   -- ✅ VALID - via state transition
   UPDATE bookings SET status = 'QUOTED', expires_at = NOW() + INTERVAL '48 hours'
   WHERE id = 101 AND status IN ('REQUESTED', 'AWAITING_CLIENT_INFO');
   ```

4. **Background expiration job** (runs every 5-10 minutes):
   ```javascript
   function expireStaleBookings() {
     // Transition expired bookings to EXPIRED status
     db.execute(`
       UPDATE bookings
       SET status = 'EXPIRED',
           updated_at = NOW()
       WHERE status IN ('REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED')
         AND expires_at IS NOT NULL
         AND expires_at < NOW()
     `);
   }
   ```

5. **TTL Extension Limits** (Prevent indefinite soft holds):
   ```javascript
   // Track how many times TTL has been extended for a booking
   // Example: REQUESTED (72h) → AWAITING_CLIENT_INFO (48h) → QUOTED (48h) = 3 extensions

   **V1 Constraint**: Maximum total hold time = 14 days from initial request

   // Check before allowing state transition that extends TTL
   function canExtendTTL(booking) {
     const ageInDays = (NOW() - booking.created_at) / (1000 * 60 * 60 * 24);
     const maxHoldDays = 14; // Configurable via system_settings

     if (ageInDays >= maxHoldDays) {
       return {
         allowed: false,
         reason: "Booking has exceeded maximum hold period (14 days). Please create a new booking request."
       };
     }

     return { allowed: true };
   }

   // Example validation in request-info endpoint
   POST /api/bookings/:id/request-info
   {
     // 1. Check age of booking
     if (!canExtendTTL(booking)) {
       return 400: "Cannot request info - booking too old. Auto-expiring and releasing slot."
     }

     // 2. If within limits, allow extension
     UPDATE bookings
     SET status = 'AWAITING_CLIENT_INFO',
         expires_at = NOW() + 48 hours
     WHERE id = :id AND status = 'REQUESTED';
   }
   ```

   **Rationale**:
   - Prevents clients/engineers from holding slots indefinitely via repeated info requests
   - Forces fresh availability checks after 14 days (market conditions may change)
   - Reasonable window for quote iteration while preventing abuse

   **Alternative Approach** (count transitions instead of days):
   - Maximum 3 TTL extensions per booking (e.g., REQUESTED → AWAITING_CLIENT_INFO → REQUESTED → QUOTED)
   - Simpler to track but less flexible

**Database Enforcement** (Optional but Recommended):
```sql
-- Prevent direct expires_at updates via trigger
CREATE TRIGGER enforce_expires_at_lifecycle
BEFORE UPDATE ON bookings
FOR EACH ROW
WHEN (
  NEW.expires_at IS DISTINCT FROM OLD.expires_at
  AND NEW.status = OLD.status  -- expires_at changed without status change
)
EXECUTE FUNCTION reject_expires_at_update();
```

### Confirmation Code Generation

**Security Requirements**:
- ✅ High-entropy (non-guessable)
- ✅ Non-sequential (cannot predict next code)
- ✅ NOT derived from booking ID (prevents enumeration attacks)
- ✅ Collision-resistant (unique constraint in database)

**Recommended Implementation**:

```javascript
function generateConfirmationCode() {
  // Format: BK-XXXX-YYYY (13 characters total)
  // XXXX: 4 random alphanumeric chars (uppercase, no ambiguous chars)
  // YYYY: 4 random alphanumeric chars

  const chars = '23456789ABCDEFGHJKLMNPQRSTUVWXYZ'; // 32 chars (exclude 0,O,1,I for clarity)
  const randomString = (length) => {
    return Array.from(crypto.getRandomValues(new Uint8Array(length)))
      .map(x => chars[x % chars.length])
      .join('');
  };

  return `BK-${randomString(4)}-${randomString(4)}`;
  // Example: BK-A7X9-K4M2
}

// Ensure uniqueness with retry logic
async function createConfirmationCode(maxRetries = 5) {
  for (let i = 0; i < maxRetries; i++) {
    const code = generateConfirmationCode();
    const exists = await db.query('SELECT 1 FROM bookings WHERE confirmation_code = ?', [code]);
    if (!exists) return code;
  }
  throw new Error('Failed to generate unique confirmation code');
}
```

**Entropy Calculation**:
- Character set: 32 characters (no ambiguous chars)
- Code length: 8 characters (XXXX-YYYY)
- Total combinations: 32^8 = 1,099,511,627,776 (over 1 trillion)
- Collision probability: Extremely low even with millions of bookings

**Alternative Implementation** (UUIDs):
```javascript
// Using UUID v4 (128-bit random)
import { v4 as uuidv4 } from 'uuid';

function generateConfirmationCode() {
  // Format: BK-{first 8 chars of UUID}
  return `BK-${uuidv4().substring(0, 8).toUpperCase()}`;
  // Example: BK-A7F3D2E1
}
```

**Security Considerations**:
- ❌ Do NOT use: `BK-${booking.id}` (guessable)
- ❌ Do NOT use: Sequential counters (predictable)
- ❌ Do NOT use: Timestamp-based codes (low entropy)
- ✅ DO use: Cryptographically secure random generators

### Standardized Reason Codes

**Purpose**: Enable analytics, automated reporting, and consistent handling of declines/cancellations

**V1 Reason Codes**:

#### Decline Reasons (status: DECLINED)

**Note**: Use `DECLINED` (explicit action taken) status only for explicit rejections. Timeouts use `EXPIRED` (no action taken) status (no reason code needed).

| Reason Code | Description | Set By | When Used |
|------------|-------------|--------|-----------|
| `DECLINED_DUE_TO_ENGINEER_OOO` | Engineer is out of office during requested time | System (auto) | OOO period added that conflicts with REQUESTED/QUOTED |
| `DECLINED_BY_ENGINEER_UNAVAILABLE` | Engineer manually declined due to unavailability | Engineer | Engineer clicks "Decline" with custom reason |
| `DECLINED_BY_ENGINEER_OTHER` | Engineer declined for other reasons (see reason_text) | Engineer | Catch-all for freeform declines |
| `DECLINED_BY_CLIENT_QUOTE` | Client explicitly rejected quote | Client | Client clicks "Decline Quote" link in email |

**EXPIRED Status** (separate from DECLINED):
- No reason code needed - the status itself indicates timeout
- Set automatically by background job when `expires_at < NOW()`
- Applies to: REQUESTED, AWAITING_CLIENT_INFO, QUOTED states

#### Cancellation Reasons (status: CANCELLED)

| Reason Code | Description | Set By | When Used |
|------------|-------------|--------|-----------|
| `CANCELLED_BY_CLIENT` | Client cancelled after booking confirmed | Client | Client clicks "Cancel Booking" |
| `CANCELLED_BY_ENGINEER` | Engineer cancelled confirmed booking | Engineer | Engineer cancels from dashboard |
| `CANCELLED_BY_ADMIN` | Admin manually cancelled booking | Admin | Override/emergency cancellation |
| `CANCELLED_DUE_TO_RESCHEDULE` | Booking cancelled because new reschedule quote accepted | System (auto) | Reschedule flow completes successfully |
| `CANCELLED_DUE_TO_ENGINEER_OOO` | Engineer added OOO and manually cancelled BOOKED session | Engineer/System | OOO conflicts with confirmed booking (manual action required) |

**Database Schema Update**:
```sql
-- Already added in bookings table:
reason_code VARCHAR(100),     -- One of the codes above
reason_text TEXT              -- Optional freeform explanation
```

**API Usage Example**:
```javascript
// Engineer declines booking
POST /api/bookings/101/decline
{
  "reason_code": "DECLINED_BY_ENGINEER_UNAVAILABLE",
  "reason_text": "I have a personal conflict that day and cannot accommodate this session."
}

// System auto-declines due to OOO
// (internal - not exposed via API)
updateBooking(101, {
  status: 'DECLINED',
  reason_code: 'DECLINED_DUE_TO_ENGINEER_OOO',
  reason_text: `Engineer ${engineerName} will be out of office during this time.`,
  cancelled_by: 'SYSTEM'
});
```

**Benefits**:
- ✅ Structured data for analytics (e.g., "30% of declines are due to OOO")
- ✅ Automated email templates based on reason code
- ✅ Future automation (e.g., suggest alternative engineer if OOO)
- ✅ Consistent client-facing messaging

### Recurring Bookings

**V1 Implementation**: Treat as independent bookings

**Behavior**:
- When client creates recurring booking (e.g., "Every Monday 2-5 PM for 4 weeks")
- System creates **separate booking records** with status='REQUESTED' for each occurrence
- Each booking:
  - Has its own `id`, `confirmation_code`, `expires_at`, lifecycle
  - Must be individually quoted by engineer
  - Can expire/be declined independently
  - No atomic "accept all" or "decline all"

**Lead Time Constraint for Recurring Bookings**:
```javascript
// Only create occurrences that fall within maximum_lead_time_days
const maxLeadTimeDays = getConfigValue('maximum_lead_time_days'); // default: 180
const bookingHorizon = new Date();
bookingHorizon.setDate(bookingHorizon.getDate() + maxLeadTimeDays);

const validOccurrences = recurringDates.filter(date => {
  return date <= bookingHorizon;
});

// Example: If recurring request is "Every Monday for 12 months"
// but max_lead_time_days = 180 days (~6 months)
// Only create ~26 Monday occurrences (6 months worth), not all 52
```

**Behavior**:
- Do NOT generate occurrences beyond `maximum_lead_time_days`
- Client must submit new recurring request once booking horizon moves forward
- Prevents creating too many speculative bookings far in future
- Aligns with business constraint: "clients cannot book beyond N days"

**Rationale**:
- Simplifies V1 implementation (no special recurring logic)
- Allows flexible pricing per session (engineer can adjust quotes individually)
- Avoids complex rollback scenarios ("decline one but accept others")
- Respects maximum booking horizon constraint

**Future Enhancement**:
- Add `recurring_group_id` to link related bookings
- UI option to "quote all at once" or "decline all"
- Atomic operations on booking groups
- Auto-generation of future occurrences as booking horizon advances

### Security: Magic Links & Signed Tokens

**Requirement**: All client-facing action links (in emails) must be signed and time-bound

**Problem**: Unsafe magic links expose security risks
```
❌ UNSAFE: https://studio.com/bookings/accept-quote?id=101
   - Exposes internal booking ID (enumeration attack)
   - No authentication (anyone with link can accept quote)
   - No expiration (link valid forever)
   - Vulnerable to replay attacks
```

**Solution**: Use signed tokens with embedded confirmation code

**Recommended Implementation**:

```javascript
// Generate signed token for magic link
function generateSignedToken(bookingId, confirmationCode, action, expiresInHours = 72) {
  const payload = {
    bookingId,           // Internal ID (not exposed in URL)
    confirmationCode,    // Public-facing code
    action,              // 'accept-quote' | 'decline-quote' | 'view-status' | 'cancel-booking'
    exp: Math.floor(Date.now() / 1000) + (expiresInHours * 3600)
  };

  // Sign with secret key (use JWT or HMAC)
  const token = jwt.sign(payload, process.env.MAGIC_LINK_SECRET, { algorithm: 'HS256' });
  return token;
}

// Verify signed token
function verifyMagicLink(token) {
  try {
    const payload = jwt.verify(token, process.env.MAGIC_LINK_SECRET);

    // Check expiration
    if (payload.exp < Math.floor(Date.now() / 1000)) {
      throw new Error('Link has expired');
    }

    return payload;
  } catch (err) {
    throw new Error('Invalid or expired link');
  }
}
```

**Secure Magic Link Examples**:

```javascript
// ✅ SAFE: Accept quote link (in email)
const token = generateSignedToken(booking.id, booking.confirmation_code, 'accept-quote', 48);
const link = `https://studio.com/bookings/accept-quote?token=${token}`;
// Example: https://studio.com/bookings/accept-quote?token=eyJhbGciOiJIUzI1NiIs...

// ✅ SAFE: Decline quote link
const declineToken = generateSignedToken(booking.id, booking.confirmation_code, 'decline-quote', 48);
const declineLink = `https://studio.com/bookings/decline-quote?token=${declineToken}`;

// ✅ SAFE: View booking status (public, but signed)
const statusToken = generateSignedToken(booking.id, booking.confirmation_code, 'view-status', 720); // 30 days
const statusLink = `https://studio.com/bookings/status?token=${statusToken}`;

// ✅ SAFE: Cancel booking
const cancelToken = generateSignedToken(booking.id, booking.confirmation_code, 'cancel-booking', 720);
const cancelLink = `https://studio.com/bookings/cancel?token=${cancelToken}`;
```

**API Endpoint Pattern**:

```javascript
// Accept quote via magic link
app.post('/api/public/bookings/accept-quote', async (req, res) => {
  const { token } = req.body;

  // Verify and decode token
  const payload = verifyMagicLink(token);

  // Validate action
  if (payload.action !== 'accept-quote') {
    return res.status(403).json({ error: 'Invalid action for this token' });
  }

  // Fetch booking by confirmation code (NOT id)
  const booking = await db.getBookingByConfirmationCode(payload.confirmationCode);

  if (!booking) {
    return res.status(404).json({ error: 'Booking not found' });
  }

  // Double-check confirmation code matches (prevent token replay)
  if (booking.confirmation_code !== payload.confirmationCode) {
    return res.status(403).json({ error: 'Confirmation code mismatch' });
  }

  // Idempotency guard
  if (booking.status === 'BOOKED') {
    return res.json({ success: true, message: 'Quote already accepted' });
  }

  // State guard
  if (booking.status !== 'QUOTED') {
    return res.status(400).json({ error: `Cannot accept quote. Current status: ${booking.status}` });
  }

  // Proceed with transition
  await acceptQuote(booking.id, payload.confirmationCode);

  return res.json({ success: true, message: 'Quote accepted! Booking confirmed.' });
});
```

**Security Properties**:
- ✅ **Signed**: Cannot forge or tamper with tokens
- ✅ **Time-bound**: Tokens expire after configured period
- ✅ **Action-scoped**: Token only valid for specific action
- ✅ **No internal IDs in URL**: Only signed token visible
- ✅ **Replay-resistant**: Token includes confirmation code validation
- ✅ **Idempotent**: Safe to click link multiple times

**Token Expiration Recommendations**:

| Link Type | Expires In | Rationale |
|-----------|-----------|-----------|
| Accept Quote | 48 hours | Matches quote expiration TTL |
| Decline Quote | 48 hours | Matches quote expiration TTL |
| View Status | 30 days | Long-lived for convenience |
| Cancel Booking | 30 days | Valid until session date |
| Request More Info | 48 hours | Matches info request TTL |

**Alternative: One-Time-Use Tokens** (Future Enhancement):
```javascript
// Store token usage in database
CREATE TABLE magic_link_tokens (
  token_hash VARCHAR(64) PRIMARY KEY,
  booking_id INTEGER REFERENCES bookings(id),
  action VARCHAR(50),
  used_at TIMESTAMP,
  expires_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

// Mark token as used after first use (prevents replay)
function useToken(token) {
  const hash = sha256(token);
  const result = db.execute(
    'UPDATE magic_link_tokens SET used_at = NOW() WHERE token_hash = ? AND used_at IS NULL',
    [hash]
  );
  if (result.rowCount === 0) {
    throw new Error('Token already used or invalid');
  }
}
```

**V1 Simplification**: Use stateless JWT tokens with expiration. One-time-use tracking is optional.

---

## Technology Stack

### Recommended Stack (Option A - JavaScript/TypeScript Full Stack)

#### Frontend
- **Framework**: React 18+ with TypeScript
- **State Management**: React Query (server state) + Zustand (client state)
- **UI Components**:
  - shadcn/ui or Chakra UI (component library)
  - react-big-calendar or FullCalendar (calendar views)
- **Form Handling**: React Hook Form + Zod (validation)
- **HTTP Client**: Axios or Fetch API
- **Build Tool**: Vite
- **Styling**: Tailwind CSS

#### Backend
- **Runtime**: Node.js 18+
- **Framework**: Express.js or Fastify
- **Language**: TypeScript
- **ORM**: Prisma or TypeORM
- **Authentication**: JWT with jsonwebtoken
- **Email**: Nodemailer with SMTP
- **Validation**: Zod or Joi
- **Scheduling**: node-cron or Bull (for background jobs)
- **Date/Time**: date-fns or Day.js

#### Database
- **Primary**: PostgreSQL 14+
- **Caching**: Redis (optional for MVP, recommended for production)

#### DevOps & Tools
- **Development**: Docker Compose (for local DB)
- **API Testing**: Postman or Insomnia
- **Version Control**: Git
- **Linting**: ESLint + Prettier

### Alternative Stack (Option B - Python Backend)

#### Backend
- **Framework**: FastAPI (Python 3.10+)
- **ORM**: SQLAlchemy with Alembic (migrations)
- **Authentication**: python-jose (JWT)
- **Email**: FastAPI-Mail or python-smtplib
- **Validation**: Pydantic (built into FastAPI)
- **Background Tasks**: Celery + Redis
- **Date/Time**: python-dateutil

*Frontend remains the same as Option A*

### Why These Choices?

1. **React**: Industry standard, large ecosystem, great for complex UIs like calendars
2. **TypeScript**: Type safety reduces bugs, especially important for scheduling logic
3. **PostgreSQL**: Robust relational DB, excellent for complex queries and data integrity
4. **Express/FastAPI**: Lightweight, fast, easy to extend for future integrations
5. **Prisma/SQLAlchemy**: Type-safe database access, easy migrations

---

## User Flows

### Flow 1: Client Requests a Booking

```
1. Client visits booking portal
2. Selects session type from dropdown (Audiobook, Tracking, etc.)
3. System displays:
   - Recommended engineers (based on specialty match)
   - Available time slots within booking window:
     * Earliest: today + minimum_lead_time_days (default: 3 days)
     * Latest: today + maximum_lead_time_days (default: 180 days)
   - **No pricing displayed** (V1: all bookings require quotes)
4. Client fills out form:
   - Name/Band name
   - Email (required)
   - Phone (optional)
   - Preferred date/time (within lead time window)
   - Duration (minimum hours configurable by studio)
   - Notes
5. Client can:
   a) Select recommended engineer/room combo, OR
   b) Manually choose engineer and room
6. System validates selection in real-time:
   - **Eligibility Constraints** (blocks submission):
     * Room C selected for live tracking → blocked
     * Engineer unavailable → blocked
     * Room already booked → blocked
   - **Preference Warnings** (warns but allows):
     * "Room B is preferred for audiobook sessions"
     * "This engineer typically handles mixing sessions"
7. For recurring bookings:
   - Client checks "Recurring Session" box
   - Selects frequency (weekly, bi-weekly, etc.)
   - Sets end date
8. System shows booking summary (no pricing - quote will be sent by engineer)
9. Client submits request
10. System:
    - Creates client record (or updates if email exists)
    - Creates booking with status "REQUESTED"
    - Creates soft hold (prevents other bookings for this time slot)
    - Sets expires_at = now + ttl_requested_hours (default: 72 hours)
    - Sends confirmation email to client
    - Sends notification email to engineer
11. Client receives confirmation:
    "Your booking request has been sent to [Engineer Name].
     You'll receive a quote within 72 hours. This time slot is temporarily held for you."
```

### Flow 2: Engineer Sends Quote

```
1. Engineer receives email: "New booking request from [Client Name]"
   - Contains booking details, client info, requested time, session type
   - "Send Quote" and "Decline" action buttons (magic links)
2. Engineer clicks "Send Quote" link
3. Browser opens to quote generation page showing:
   - Booking details (time, duration, room, session type)
   - Auto-calculated pricing:
     * Room rate: $X/hour (from room_pricing for this room + session type)
     * Engineer rate: $Y/hour (from engineer_pricing for this service type)
     * Subtotal: (X + Y) × hours
   - Optional manual adjustment field (discount or override)
   - Quote expiration window (default: 48 hours)
   - Notes field (visible to client)
4. Engineer can:
   a) Accept calculated pricing, OR
   b) Apply manual adjustment (e.g., -$50 discount for repeat client)
5. Engineer submits quote
6. System:
   - Updates booking status to "QUOTED"
   - Records: quoted_amount, room_rate, engineer_rate, manual_adjustment
   - Sets quote_sent_at = now
   - Sets expires_at = now + ttl_quoted_hours (default: 48 hours)
   - Sends quote email to client with:
     * Pricing breakdown
     * Total amount
     * Expiration date/time
     * "Accept Quote" button
7. Client receives quote email
```

### Flow 3: Client Accepts Quote

```
1. Client receives quote email
2. Email shows:
   - Session details
   - Pricing breakdown (room + engineer + adjustments)
   - Total amount
   - Expiration: "This quote expires in 48 hours"
   - "Accept Quote" button
3. Client clicks "Accept Quote"
4. Browser opens to acceptance confirmation page
5. Client confirms acceptance
6. System:
   - Updates booking status to "BOOKED"
   - Sets quote_accepted_at = now
   - Converts soft hold to hard lock
   - Clears expires_at (no longer temporary)
   - Sends confirmation emails to:
     * Client: "Your session is confirmed!"
     * Engineer: "Quote accepted by [Client Name]"
   - Blocks time slot permanently on calendars
7. Booking is now confirmed and locked
```

### Flow 3a: Engineer Requests More Info

```
1. Engineer receives booking request email
2. Engineer needs clarification before quoting
3. Engineer clicks "Request Info" link
4. Opens to form:
   - Text area for questions (required)
   - Example: "Can you provide more details about the session goals?"
5. Engineer submits
6. System:
   - Updates booking status to "AWAITING_CLIENT_INFO"
   - Sets expires_at = now + ttl_awaiting_client_info_hours (default: 48 hours)
   - Sends email to client with:
     * Engineer's questions
     * Reply field
     * "Submit Response" button
7. Client responds via email or web form
8. System:
   - Updates booking status back to "REQUESTED"
   - Resets expires_at = now + ttl_requested_hours
   - Notifies engineer that client has responded
9. Engineer can now send quote
```

### Flow 3b: Engineer Declines Booking

```
1. Engineer clicks "Decline" link in email
2. Opens to decline form:
   - Text area for reason (required)
   - Option to suggest up to 3 alternative time slots
3. For each alternative:
   - Engineer picks date, start time, end time
   - System validates availability
4. Engineer submits
5. System:
   - Updates booking status to "DECLINED"
   - Releases soft hold (time slot becomes available)
   - Records decline reason and alternative suggestions
   - Sends email to client with:
     * Reason for decline
     * Alternative time slots with "Request This Slot" buttons
6. Client can click alternative to create new booking request
```

### Flow 4: Automated Expiration and Escalation

```
Background job runs every hour to handle expiration and escalation:

**Expiration Check**:
1. Find all bookings where:
   - expires_at <= now
   - status IN ('REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED')
2. For each expired booking:
   - Update status to 'EXPIRED'
   - Release soft hold (time slot becomes available)
   - Send notification email:
     * To client: "Your booking request has expired. Please submit a new request if still interested."
     * To engineer (if applicable): "Booking request expired without action."

**Escalation Check** (before expiration):
1. Find all bookings where:
   - status = 'REQUESTED'
   - expires_at is within 24 hours
   - engineer hasn't taken action
2. Send reminder email to engineer:
   - "Urgent: Booking request expires in 24 hours"
   - Include quick action links (Send Quote, Decline, Request Info)

**Owner Escalation** (72 hours):
1. Find all bookings where:
   - status = 'REQUESTED'
   - created_at >= 72 hours ago (or approaching expiration)
   - No engineer action yet
2. Send escalation email to studio owner:
   - "Engineer [Name] has not responded to booking request #[ID]"
   - Full booking details
   - Actions: "Send Quote on Behalf", "Decline", "Extend Deadline"
3. Owner can:
   - Take action on behalf of engineer
   - Extend expiration deadline
   - Contact engineer directly
```

### Flow 5: Engineer Sets OOO Time

```
1. Engineer logs into dashboard
2. Navigates to "My Availability"
3. Clicks "Block Out Time"
4. Fills form:
   - Start date/time
   - End date/time
   - Reason (optional, e.g., "Vacation", "Medical leave")
5. System validates and detects conflicts with existing bookings in that time range
6. System shows conflict resolution UI:

   **If OOO overlaps REQUESTED, AWAITING_CLIENT_INFO, or QUOTED bookings:**
   ✅ Auto-decline allowed (these are soft holds, not confirmed)
   - Shows list of affected requests/quotes
   - Engineer clicks "Confirm OOO" → System auto-declines with email:
     "Sorry, [Engineer Name] will be out of office during this time."
   - Suggested alternative slots are offered to clients
   - Soft holds are released immediately

   **If OOO overlaps BOOKED bookings:**
   ❌ Auto-decline NOT allowed (these are confirmed bookings)
   - Shows list of confirmed bookings
   - Engineer must choose:
     a) Manually cancel/reschedule each BOOKED session (follow cancellation flow), OR
     b) Adjust OOO dates to avoid conflicts

7. Once saved, OOO period blocks availability in booking system
8. Future booking requests during OOO periods are automatically blocked (no soft hold created)

**V1 Note**: Engineers can add OOO retroactively (even if they forgot earlier). System handles conflicts gracefully.
```

### Flow 6: Rescheduling & Cancellation (V1 Scope)

#### Rescheduling a BOOKED Session

```
**V1 Approach**: Reschedule = New booking request + linkage to original (do NOT modify BOOKED records in place)

**Constraint**: Only ONE active reschedule request allowed per booking at a time
- If booking #101 already has an active reschedule (booking #202 in REQUESTED/AWAITING_CLIENT_INFO/QUOTED status)
- Client cannot initiate another reschedule until #202 is resolved (BOOKED, DECLINED, EXPIRED, or CANCELLED)
- Prevents: "reschedule the reschedule" loops and complex chains
- Validation: Check if `rescheduled_to_booking_id` exists AND that booking's status is active

1. Client or Engineer initiates reschedule:
   - Client: Clicks "Request Reschedule" link in booking confirmation email
   - Engineer: Clicks "Reschedule" button in dashboard for existing booking
   - **Validation**: System checks if active reschedule already exists (reject if true)

2. Client fills out new booking request form:
   - Pre-filled with: client info, session type, duration
   - Must select: new date/time (subject to lead time constraints and availability)
   - Required field: "Rescheduling from booking #BK-101-ABC"

3. New booking created with status: REQUESTED
   - Field `rescheduled_from_booking_id` = original booking ID
   - Creates new soft hold for new date/time

4. Original booking updated:
   - Field `rescheduled_to_booking_id` = new booking ID
   - Status remains BOOKED (not cancelled yet)
   - Flag: "Reschedule pending approval"

5. Engineer reviews new request and sends quote (follow normal quote workflow)

6. Once new quote is accepted (status → BOOKED):
   - Original booking updated:
     * Status → CANCELLED
     * `cancelled_at` = timestamp
     * `cancelled_by` = 'CLIENT' or 'ENGINEER'
     * `reason_text` = "Rescheduled to booking #BK-202-XYZ"
   - New booking becomes primary
   - Client receives confirmation: "Your session has been rescheduled to [new date/time]"

7. If new quote is declined or expires:
   - Original booking remains BOOKED (unchanged)
   - New booking → DECLINED or EXPIRED
   - Client can retry rescheduling or keep original booking

**OOO / Rescheduling Interaction**:
- When engineer adds OOO that conflicts with BOOKED sessions:
  * System suggests initiating reschedule flow for each affected booking
  * Engineer can click "Suggest Reschedule" → auto-emails client with alternative dates
  * Follows same workflow as above (new booking request + linkage)

  **NOTE: When a rescheduled booking is confirmed and the original booking is cancelled, those updates should occur transactionally.**
```

#### Cancellation Flow (V1 - Lightweight Policy)

```
**V1 Scope**: Track cancellation events. No automated fees or deposits in V1.

1. Cancellation can be initiated by:
   - Client: Clicks "Cancel Booking" in confirmation email
   - Engineer: Clicks "Cancel" in dashboard
   - Admin: Manual override
   - System: OOO conflict auto-cancellation (only for REQUESTED/QUOTED)

2. System confirms cancellation intent:
   - Shows cancellation policy reminder (if any)
   - Asks for optional reason

3. Booking updated:
   - Status → CANCELLED
   - `cancelled_at` = timestamp
   - `cancelled_by` = 'CLIENT' | 'ENGINEER' | 'ADMIN' | 'SYSTEM'
   - `reason_text` = user-provided or auto-generated (e.g., "Engineer OOO")

4. Notifications sent:
   - Email to client: "Your booking #BK-101-ABC has been cancelled"
   - Email to engineer: "Booking #BK-101-ABC cancelled by [CLIENT/ENGINEER/ADMIN]"
   - Soft hold released (if applicable)

**V1 Cancellation Policy** (Lightweight):
- No cancellation windows enforced (e.g., 48h/24h)
- No deposits or late fees
- Treat reschedule as cancellation from a policy perspective
- Track `cancelled_at` for future analytics/reporting

**Future Enhancement Fields** (noted in schema, not enforced in V1):
- `cancellation_window_hours` - time before session when cancellation fee applies
- `deposit_amount` - upfront payment to reduce no-shows
- `late_fee` - charge for cancellations within window
```

---

## Implementation Phases

### Phase 1: MVP (Weeks 1-4)

**Goal**: Core booking functionality with quote-based confirmation and soft hold management

#### Week 1: Project Setup & Database
- [ ] Initialize Git repository
- [ ] Set up development environment (Node.js, PostgreSQL, Docker)
- [ ] Create database schema
- [ ] Run initial migrations
- [ ] Seed database with:
  - 3 engineers (with test emails)
  - 2 rooms (A and B)
  - Sample room capabilities
  - Sample engineer specialties

#### Week 2: Backend API - Core Endpoints
- [ ] Set up Express/FastAPI project structure
- [ ] Implement authentication for engineers (JWT)
- [ ] Create API endpoints:
  - `GET /api/availability` - Get available time slots (respecting soft holds + lead time)
  - `POST /api/bookings` - Create booking request (status: REQUESTED, creates soft hold)
  - `GET /api/bookings/:id` - Get booking details (engineer/admin only)
  - `GET /api/public/bookings/status/:confirmationCode` - Client lookup using confirmation code (no internal ID exposed)
  - `POST /api/bookings/:id/quote` - Generate and send quote
  - `POST /api/public/bookings/accept-quote` - Client accepts quote (QUOTED → BOOKED)
  - `POST /api/public/bookings/decline-quote` - Client declines quote (QUOTED → DECLINED)
  - `POST /api/bookings/:id/request-info` - Engineer requests more client info
  - `POST /api/bookings/:id/decline` - Engineer declines with optional alternatives
  - `POST /api/public/bookings/cancel` - Cancel booking (V1 - lightweight policy)
  - `POST /api/public/bookings/request-reschedule` - Reschedule booking (creates new linked request)
  - `GET /api/engineers` - List engineers with specialties (no pricing in V1)
  - `GET /api/rooms` - List rooms with capabilities (no pricing in V1)
- [ ] Implement booking validation logic:
  - Lead time constraints (min/max) - **SERVER-SIDE ENFORCEMENT REQUIRED**
  - Room capabilities eligibility checks
  - Engineer availability checks
- [ ] Implement soft hold management and conflict detection
- [ ] Create quote calculation engine (room rate + engineer rate + adjustments)

#### Week 3: Frontend - Client Portal
- [ ] Set up React project with Vite
- [ ] Create components:
  - Booking form with lead time validation
  - Session type selector
  - Date/time picker (restricted to booking window: today+3 to today+180)
  - Engineer/room selector with real-time eligibility validation
  - Booking summary (no pricing shown - quote required)
- [ ] Implement real-time availability check (respecting soft holds)
- [ ] Show recommended engineer/room combinations
- [ ] Display eligibility constraints (blocking) vs preference warnings
- [ ] Build booking confirmation page (soft hold created)
- [ ] Build quote acceptance page (for client email links)

#### Week 4: Engineer Dashboard & Email
- [ ] Build engineer login page
- [ ] Create engineer dashboard:
  - Personal calendar view (showing REQUESTED, QUOTED, BOOKED states)
  - Pending requests list (REQUESTED status)
  - Quote generation form with:
    * Auto-calculated pricing (room + engineer rates)
    * Manual adjustment field
    * Quote expiration setting
  - Request info form
  - Decline form with alternatives
- [ ] Set up email service (Nodemailer/SendGrid)
- [ ] Implement email templates:
  - Booking request to engineer (with "Send Quote", "Request Info", "Decline" links)
  - Quote sent to client (with pricing breakdown and "Accept Quote" link)
  - Quote accepted confirmation (to engineer and client)
  - Request expired notification (to client and engineer)
  - Decline with alternatives (to client)
- [ ] Implement background expiration job (check every hour)
- [ ] Test end-to-end booking lifecycle
- [ ] Bug fixes and refinements

**MVP Deliverables**:
- ✅ Clients can request sessions (with lead time constraints)
- ✅ Soft holds prevent double-booking
- ✅ Engineers receive booking request notifications
- ✅ Engineers can generate quotes with pricing breakdown
- ✅ Clients can accept quotes to confirm bookings
- ✅ State-based lifecycle (REQUESTED → QUOTED → BOOKED)
- ✅ TTL-based expiration with automatic hold release
- ✅ Availability checking (respecting soft holds and lead time)
- ✅ Conflict detection and eligibility validation

---

### Phase 2: Enhanced Features (Weeks 5-6)

#### Week 5: Recurring Bookings & OOO
- [ ] Implement recurring booking logic
- [ ] Create recurring booking UI
- [ ] Build OOO management:
  - Engineer OOO form
  - OOO calendar view
  - Conflict detection with existing bookings
- [ ] Update availability algorithm to account for OOO

#### Week 6: Client Management & Pricing
- [ ] Build client database features:
  - Client detail page
  - Booking history view
  - Notes and preferences
- [ ] Implement pricing calculation:
  - Room pricing by room and optional session type
  - Engineer pricing by service type
  - Combined calculation: (Room Rate + Engineer Rate) × Hours
  - Display pricing according to session type and project classification
  - Store pricing breakdown in booking record
- [ ] Add search/filter for clients
- [ ] Auto-escalation background job (3-day rule)

**Phase 2 Deliverables**:
- ✅ Recurring bookings
- ✅ Engineer OOO management
- ✅ Client history tracking
- ✅ Automated pricing
- ✅ Email escalation

---

### Phase 3: Polish & Optimization (Week 7-8)

#### Week 7: UI/UX Improvements
- [ ] Enhance calendar views (FullCalendar integration)
- [ ] Add loading states and error handling
- [ ] Improve mobile responsiveness
- [ ] Add confirmation dialogs for critical actions
- [ ] Implement toast notifications
- [ ] Add booking status badges

#### Week 8: Testing & Documentation
- [ ] Write unit tests for critical functions
- [ ] Integration tests for booking flow
- [ ] Create user documentation:
  - Client booking guide
  - Engineer dashboard guide
- [ ] Performance testing
- [ ] Security audit (input validation, SQL injection prevention)
- [ ] Prepare for deployment

**Phase 3 Deliverables**:
- ✅ Polished UI/UX
- ✅ Comprehensive testing
- ✅ Documentation
- ✅ Production-ready code

---

### Phase 4: Future Enhancements (Post-Launch)

#### Advanced Features
- [ ] **Google Calendar Integration**:
  - Sync bookings to engineer's Google Calendar
  - Two-way sync (block studio time from personal calendar)
- [ ] **Payment Integration**:
  - Stripe or Square integration
  - Deposit collection on booking
  - Automated invoicing
- [ ] **Reporting & Analytics**:
  - Studio utilization rates
  - Revenue by room/engineer
  - Popular session types
  - Client retention metrics
- [ ] **Client Accounts**:
  - Optional client registration
  - Self-service booking history
  - Saved preferences
- [ ] **Waitlist**:
  - Join waitlist for fully booked times
  - Auto-notify when slots open
- [ ] **SMS Notifications**:
  - Twilio integration for text reminders
  - Booking confirmations via SMS
- [ ] **Advanced Scheduling**:
  - Buffer time between sessions
  - Setup/teardown time per room
  - Equipment rental tracking
- [ ] **Mobile App**:
  - React Native or Flutter
  - Push notifications

---

## API Design

### Public vs Authenticated API Access

- Public **read-only** booking status lookup uses confirmation codes and does not require authentication
- Public **state-changing** client actions (accept quote, decline quote, cancel, reschedule request) must use signed, time-bound tokens and must not expose internal booking IDs
- Internal authenticated engineer/admin endpoints may use booking IDs

### REST API Endpoints

#### Public Endpoints (No Auth Required)

##### `GET /api/availability`
**Description**: Get available time slots for booking

**Query Parameters**:
- `sessionType` (string, required): Type of session
- `startDate` (ISO date, required): Start of date range
- `endDate` (ISO date, required): End of date range
- `engineerId` (integer, optional): Filter by specific engineer
- `roomId` (integer, optional): Filter by specific room

**Response**:
```json
{
  "slots": [
    {
      "date": "2024-03-25",
      "startTime": "10:00",
      "endTime": "18:00",
      "engineer": {
        "id": 1,
        "name": "John Doe",
        "specialties": ["Tracking", "Mixing"]
      },
      "room": {
        "id": 1,
        "name": "Room A",
        "capabilities": ["Tracking", "Mixing", "Overdubs"]
      },
      "recommended": true
    }
  ]
}
```

##### `POST /api/bookings`
**Description**: Create a new booking request

**Request Body**:
```json
{
  "client": {
    "name": "The Awesome Band",
    "email": "band@example.com",
    "phone": "+1234567890"
  },
  "sessionType": "Tracking",
  "startTime": "2024-03-25T10:00:00Z",
  "endTime": "2024-03-25T14:00:00Z",
  "engineerId": 1,
  "roomId": 1,
  "notes": "Recording drums and bass",
  "recurring": {
    "enabled": true,
    "frequency": "WEEKLY",
    "dayOfWeek": "MONDAY",
    "endDate": "2024-04-29"
  }
}
```

**Response**:
```json
{
  "booking": {
    "id": 101,
    "status": "REQUESTED",
    "confirmationCode": "BK-101-ABC",
    "expiresAt": "2024-03-24T09:00:00Z",
    "message": "Your booking request has been sent to John Doe. You'll receive a quote via email within 72 hours."
  }
}
```

##### `GET /api/engineers`
**Description**: List all active engineers with their specialties (V1: no pricing exposed)

**Response**:
```json
{
  "engineers": [
    {
      "id": 1,
      "name": "John Doe",
      "specialties": [
        {"type": "Tracking", "priority": 10},
        {"type": "Mixing", "priority": 8}
      ],
      "bio": "Experienced tracking and mixing engineer specializing in rock and indie music"
    }
  ]
}
```

##### `GET /api/rooms`
**Description**: List all active rooms with capabilities (V1: no pricing exposed)

**Response**:
```json
{
  "rooms": [
    {
      "id": 1,
      "name": "Room A",
      "type": "Large Tracking + Control",
      "capabilities": ["Tracking", "Mixing", "Mastering"],
      "capacity": 10,
      "description": "Large tracking room ideal for bands and full production sessions"
    }
  ]
}
```

##### `GET /api/public/bookings/status/:confirmationCode`
**Description**: Check booking status using confirmation code (public endpoint - does not expose internal booking ID)

**Query Parameters**:
- `confirmationCode` (required): Confirmation code from booking confirmation email (e.g., "BK-101-ABC")

**Response**:
```json
{
  "booking": {
    "confirmationCode": "BK-101-ABC",
    "status": "BOOKED",
    "sessionType": "Tracking",
    "startTime": "2024-03-25T10:00:00Z",
    "endTime": "2024-03-25T14:00:00Z",
    "quotedAmount": 700.00,
    "quoteAcceptedAt": "2024-03-22T14:30:00Z",
    "engineer": "John Doe",
    "room": "Room A"
  }
}
```

**Security Note**: Internal booking IDs are never exposed in client-facing APIs. All client lookups use confirmation codes.

#### Protected Endpoints (Engineer Auth Required)

##### `POST /api/auth/login`
**Description**: Engineer login

**Request Body**:
```json
{
  "email": "engineer@studio.com",
  "password": "securepassword"
}
```

**Response**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "engineer": {
    "id": 1,
    "name": "John Doe",
    "email": "engineer@studio.com"
  }
}
```

##### `GET /api/engineers/me/bookings`
**Description**: Get logged-in engineer's bookings

**Query Parameters**:
- `status` (string, optional): Filter by status (REQUESTED, AWAITING_CLIENT_INFO, QUOTED, BOOKED, EXPIRED, CANCELLED, DECLINED, COMPLETED, FINAL_INVOICED)
- `startDate` (ISO date, optional)
- `endDate` (ISO date, optional)

**Response**:
```json
{
  "bookings": [
    {
      "id": 101,
      "client": {
        "name": "The Awesome Band",
        "email": "band@example.com"
      },
      "sessionType": "Tracking",
      "startTime": "2024-03-25T10:00:00Z",
      "endTime": "2024-03-25T14:00:00Z",
      "room": "Room A",
      "status": "REQUESTED",
      "expiresAt": "2024-03-23T09:00:00Z",
      "notes": "Recording drums and bass",
      "createdAt": "2024-03-20T09:00:00Z"
    }
  ]
}
```

##### `POST /api/bookings/:id/quote`
**Description**: Engineer generates and sends quote to client

**Request Body**:
```json
{
  "manualAdjustment": -50.00,  // Optional: discount or override
  "engineerNotes": "Happy to offer a discount for a repeat client!",
  "quoteExpirationHours": 48  // Optional: defaults to studio setting
}
```

**Response**:
```json
{
  "booking": {
    "id": 101,
    "status": "QUOTED",
    "quotedAmount": 350.00,
    "roomRate": 100.00,
    "engineerRate": 100.00,
    "manualAdjustment": -50.00,
    "quoteSentAt": "2024-03-21T10:00:00Z",
    "expiresAt": "2024-03-23T10:00:00Z"
  },
  "message": "Quote sent to client. Quote expires in 48 hours."
}
```

##### `POST /api/public/bookings/accept-quote`
**Description**: Client accepts quote and confirms booking (public endpoint)

**Authentication Options**:
1. **Confirmation Code** (manual entry by client)
2. **Signed Token** (from magic link in email) - RECOMMENDED

**Request Body (Option 1: Confirmation Code)**:
```json
{
  "confirmationCode": "BK-A7X9-K4M2"
}
```

**Request Body (Option 2: Signed Token)**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."  // Signed token from email link
}
```

**Response**:
```json
{
  "booking": {
    "confirmationCode": "BK-A7X9-K4M2",
    "status": "BOOKED",
    "quoteAcceptedAt": "2024-03-21T11:30:00Z",
    "expiresAt": null  // Cleared after acceptance
  },
  "message": "Booking confirmed! You and the engineer will receive confirmation emails."
}
```

**State Guards**:
- Must be in `QUOTED` status
- Must not be expired (`expires_at` > NOW)
- Prevents accepting if quote already expired or declined

**Idempotency**: Returns success if already `BOOKED`

**Side Effects**:
- Sets `status = 'BOOKED'`
- Sets `quote_accepted_at = NOW()`
- Clears `expires_at` (booking no longer expires)
- Sends confirmation emails to client and engineer

##### `POST /api/public/bookings/decline-quote`
**Description**: Client explicitly declines quote (public endpoint - uses confirmation code OR signed token)

**Authentication Options**:
1. **Confirmation Code** (manual entry by client)
2. **Signed Token** (from magic link in email) - RECOMMENDED

**Request Body (Option 1: Confirmation Code)**:
```json
{
  "confirmationCode": "BK-A7X9-K4M2",
  "reasonCode": "DECLINED_BY_CLIENT_QUOTE",  // Optional
  "reasonText": "Budget constraints"          // Optional
}
```

**Request Body (Option 2: Signed Token)**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",  // Signed token from email link
  "reasonCode": "DECLINED_BY_CLIENT_QUOTE",            // Optional
  "reasonText": "Found a different studio"             // Optional
}
```

**Response**:
```json
{
  "booking": {
    "confirmationCode": "BK-A7X9-K4M2",
    "status": "DECLINED",
    "declinedAt": "2024-03-21T12:00:00Z",
    "reasonCode": "DECLINED_BY_CLIENT_QUOTE"
  },
  "message": "Quote declined. Soft hold released. Thank you for considering us!"
}
```

**Idempotency**: Returns success if already DECLINED or EXPIRED

**V1 Note**: Differentiate between EXPIRED (no response) and DECLINED (explicit rejection)

##### `POST /api/bookings/:id/request-info`
**Description**: Engineer requests more information from client

**Request Body**:
```json
{
  "questions": "Can you provide more details about the session goals and any specific equipment needs?"
}
```

**Response**:
```json
{
  "booking": {
    "id": 101,
    "status": "AWAITING_CLIENT_INFO",
    "expiresAt": "2024-03-23T10:00:00Z"
  },
  "message": "Information request sent to client. Request expires in 48 hours."
}
```

##### `POST /api/bookings/:id/decline`
**Description**: Engineer declines booking request with optional alternative time slots

**Request Body**:
```json
{
  "reasonCode": "DECLINED_BY_ENGINEER_UNAVAILABLE",  // Use standardized reason code
  "reasonText": "I have a personal conflict that day",  // Optional freeform text
  "alternatives": [  // Optional alternative time slots
    {
      "startTime": "2024-03-26T10:00:00Z",
      "endTime": "2024-03-26T14:00:00Z"
    },
    {
      "startTime": "2024-03-27T14:00:00Z",
      "endTime": "2024-03-27T18:00:00Z"
    }
  ]
}
```

**Response**:
```json
{
  "booking": {
    "id": 101,
    "status": "DECLINED",
    "reasonCode": "DECLINED_BY_ENGINEER_UNAVAILABLE",
    "declinedAt": "2024-03-21T12:30:00Z"
  },
  "message": "Booking declined. Client has been notified with alternative time slots. Soft hold released."
}
```

**Valid Reason Codes**: `DECLINED_BY_ENGINEER_UNAVAILABLE`, `DECLINED_BY_ENGINEER_OTHER`, `DECLINED_DUE_TO_ENGINEER_OOO` (system-set)

**Idempotency**: Returns success if already DECLINED

##### `POST /api/public/bookings/cancel`
**Description**: Cancel a booking (V1 scope - lightweight policy)

**Authentication**:
- Client cancellation: Can use signed token OR confirmation code
- Engineer/Admin cancellation: Requires authenticated session

**Request Body (Client cancellation)**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",  // OR "confirmationCode": "BK-A7X9-K4M2"
  "reasonCode": "CANCELLED_BY_CLIENT",  // Standardized reason code
  "reasonText": "Schedule conflict - unable to attend"  // Optional
}
```

**Request Body (Engineer/Admin cancellation)**:
```json
{
  "reasonCode": "CANCELLED_BY_ENGINEER",  // or "CANCELLED_BY_ADMIN"
  "reasonText": "Equipment malfunction - room unavailable"  // Optional
}
```

**Response**:
```json
{
  "booking": {
    "id": 101,
    "confirmationCode": "BK-A7X9-K4M2",
    "status": "CANCELLED",
    "cancelledAt": "2024-03-20T15:30:00Z",
    "cancelledBy": "CLIENT",
    "reasonCode": "CANCELLED_BY_CLIENT",
    "reasonText": "Schedule conflict - unable to attend"
  },
  "message": "Booking cancelled. Client and engineer notified. Time slot released."
}
```

**Valid Reason Codes**: `CANCELLED_BY_CLIENT`, `CANCELLED_BY_ENGINEER`, `CANCELLED_BY_ADMIN`, `CANCELLED_DUE_TO_RESCHEDULE` (system-set), `CANCELLED_DUE_TO_ENGINEER_OOO`

**Idempotency**: Returns success if already CANCELLED

**V1 Note**: No cancellation fees or deposit handling in V1. Track cancellation data for future policy enforcement.

##### `POST /api/public/bookings/request-reschedule`
**Description**: Initiate reschedule flow (creates new booking request linked to original)

**Request Body**:
```json
{
  "newStartTime": "2024-03-28T10:00:00Z",
  "newEndTime": "2024-03-28T14:00:00Z",
  "reason": "Original date no longer works for client"  // Optional
}
```

**Response**:
```json
{
  "originalBooking": {
    "id": 101,
    "status": "BOOKED",  // Remains BOOKED until new quote accepted
    "rescheduledToBookingId": 202,
    "rescheduleStatus": "PENDING"
  },
  "newBooking": {
    "id": 202,
    "status": "REQUESTED",
    "confirmationCode": "BK-202-XYZ",
    "rescheduledFromBookingId": 101,
    "startTime": "2024-03-28T10:00:00Z",
    "endTime": "2024-03-28T14:00:00Z"
  },
  "message": "Reschedule request created. Engineer will review and send a new quote. Original booking remains active until new quote is accepted."
}
```

**V1 Note**: Reschedule = new booking request. Original booking only cancelled once new quote is accepted.

##### `POST /api/engineers/me/ooo`
**Description**: Block out time (Out of Office)

**Request Body**:
```json
{
  "startTime": "2024-04-01T00:00:00Z",
  "endTime": "2024-04-07T23:59:59Z",
  "reason": "Vacation"
}
```

**Response**:
```json
{
  "ooo": {
    "id": 5,
    "startTime": "2024-04-01T00:00:00Z",
    "endTime": "2024-04-07T23:59:59Z",
    "reason": "Vacation"
  },
  "conflicts": [] // List of existing confirmed bookings (status: BOOKED) in this range
}
```

##### `GET /api/engineers/me/ooo`
**Description**: Get all OOO periods for logged-in engineer

**Response**:
```json
{
  "oooList": [
    {
      "id": 5,
      "startTime": "2024-04-01T00:00:00Z",
      "endTime": "2024-04-07T23:59:59Z",
      "reason": "Vacation"
    }
  ]
}
```

##### `DELETE /api/engineers/me/ooo/:id`
**Description**: Remove OOO period

##### `GET /api/clients`
**Description**: Get client list with search

**Query Parameters**:
- `search` (string): Search by name or email

**Response**:
```json
{
  "clients": [
    {
      "id": 10,
      "name": "The Awesome Band",
      "email": "band@example.com",
      "phone": "+1234567890",
      "totalBookings": 5,
      "lastBooking": "2024-03-15T10:00:00Z"
    }
  ]
}
```

##### `GET /api/clients/:id`
**Description**: Get client details with booking history

**Response**:
```json
{
  "client": {
    "id": 10,
    "name": "The Awesome Band",
    "email": "band@example.com",
    "phone": "+1234567890",
    "notes": "Prefer morning sessions",
    "bookings": [
      {
        "id": 95,
        "sessionType": "Tracking",
        "startTime": "2024-03-15T10:00:00Z",
        "endTime": "2024-03-15T14:00:00Z",
        "engineer": "John Doe",
        "room": "Room A",
        "status": "completed"
      }
    ]
  }
}
```

---

## Email Notification System

**Security Note**: All action links in emails (e.g., "Accept Quote", "Decline", "Cancel Booking") MUST use signed tokens as described in the Security section. Never expose raw booking IDs or use unsigned URLs.

Example secure link generation:
```javascript
// ✅ SECURE - signed token
const acceptToken = generateSignedToken(booking.id, booking.confirmation_code, 'accept-quote', 48);
const acceptLink = `https://studio.com/bookings/accept-quote?token=${acceptToken}`;

// ❌ INSECURE - exposes booking ID
const unsafeLink = `https://studio.com/bookings/accept-quote?id=${booking.id}`;
```

All email templates below use placeholder text like `[Accept Quote Button]` which should be rendered as secure signed links.

### Email Templates

#### 1. Booking Request to Engineer
**Subject**: New Booking Request from [Client Name]

```
Hi [Engineer Name],

You have a new booking request (soft hold created - slot temporarily reserved):

Client: [Client Name]
Email: [Client Email]
Phone: [Client Phone]

Session Details:
- Type: [Session Type]
- Date: [Date]
- Time: [Start Time] - [End Time] ([Duration] hours)
- Room: [Room Name]

Notes from Client:
[Client Notes]

⏰ Please respond within 72 hours or this request will expire:

[Send Quote]  [Request More Info]  [Decline]

Manage this request in your dashboard:
[Dashboard Link]

---
[Studio Name]
```

#### 2. Booking Request Confirmation to Client
**Subject**: Booking Request Received - [Date] at [Time]

```
Hi [Client Name],

Thank you for your booking request! We've received your request and have reserved the time slot.

Request Details:
- Session Type: [Session Type]
- Date: [Date]
- Time: [Start Time] - [End Time] ([Duration] hours)
- Engineer: [Engineer Name]
- Room: [Room Name]

What Happens Next:
1. Your engineer will review your request and send you a custom quote within 72 hours
2. Once you receive the quote, you'll have 48 hours to accept or decline
3. Your time slot is temporarily held during this process

You'll receive an email from [Engineer Name] with pricing details soon.

Questions? Reply to this email or call us at [Phone Number].

Confirmation Code: [Confirmation Code]

---
[Studio Name]
```

#### 3. Quote Sent to Client
**Subject**: Your Quote from [Engineer Name] - [Date] Session

```
Hi [Client Name],

[Engineer Name] has prepared a quote for your upcoming session:

Session Details:
- Type: [Session Type]
- Date: [Date]
- Time: [Start Time] - [End Time] ([Duration] hours)
- Room: [Room Name]

Pricing Breakdown:
- Room Rate: $[Room Rate]/hr × [Duration] hrs = $[Room Total]
- Engineer Rate: $[Engineer Rate]/hr × [Duration] hrs = $[Engineer Total]
[If adjustment exists]
- Adjustment: $[Manual Adjustment] ([Adjustment Reason])

Total Quote: $[Quoted Amount]

⏰ This quote expires: [Expires At] (48 hours from now)

[Accept Quote Button]

If you have questions about this quote, reply to this email to reach [Engineer Name] directly.

To decline this quote, simply let it expire or click here: [Decline Link]

Confirmation Code: [Confirmation Code]

---
[Studio Name]
```

#### 4. Booking Confirmation to Client (After Quote Accepted)
**Subject**: Booking Confirmed - [Date] at [Time]

```
Hi [Client Name],

Great news! Your booking has been confirmed.

Session Details:
- Engineer: [Engineer Name]
- Session Type: [Session Type]
- Date: [Date]
- Time: [Start Time] - [End Time] ([Duration] hours)
- Room: [Room Name]
- Total: $[Quoted Amount] (accepted quote)

[If Recurring - V1 Note: Each session is independent]
This is part of a recurring booking series. Your other sessions:
- [Date 1] at [Time] - [Status]
- [Date 2] at [Time] - [Status]
- [Date 3] at [Time] - [Status]

Studio Address:
[Address]

Need to make changes?
[Request Reschedule]  [Cancel Booking]

Or contact [Engineer Name] directly: [Engineer Email] | [Phone Number]

Looking forward to seeing you!

Confirmation Code: [Confirmation Code]
Track your booking: [Booking Status Link with Confirmation Code]

---
[Studio Name]
```

#### 5. Booking Declined with Alternatives
**Subject**: Booking Update - Alternative Times Available

```
Hi [Client Name],

Unfortunately, [Engineer Name] is unable to accommodate your booking request for [Date] at [Start Time].

Reason:
[Decline Reason]

However, [Engineer Name] has suggested these alternative times:

1. [Alt Date 1] at [Alt Time 1] - [Request This Slot] button
2. [Alt Date 2] at [Alt Time 2] - [Request This Slot] button
3. [Alt Date 3] at [Alt Time 3] - [Request This Slot] button

Click "Request This Slot" to submit a new booking request, or visit our booking page to explore more options:
[Booking Page Link]

We apologize for the inconvenience and hope to work with you soon!

---
[Studio Name]
```

#### 6. Escalation to Owner
**Subject**: [URGENT] Booking Request Pending for 72+ Hours

```
Hi [Owner Name],

A booking request has been pending for over 72 hours without a response from the assigned engineer. The request will expire soon and the soft hold will be released.

Booking Details:
- Request ID: #[Booking ID]
- Client: [Client Name] ([Client Email])
- Engineer: [Engineer Name]
- Requested Date: [Date] at [Time]
- Session Type: [Session Type]
- Requested On: [Created At]
- Expires In: [Hours Until Expiration] hours

⚠️ Action Required - Choose one:

[Send Quote]  [Request More Info]  [Decline Request]

Or assign to different engineer: [Reassign Link]

View full details:
[Admin Dashboard Link]

---
Automated Alert - [Studio Name]
```

#### 7. Client Declines Quote
**Subject**: Quote Declined - [Date] Session

**Trigger**: Client explicitly rejects quote (status: QUOTED → DECLINED)

**To**: Engineer

```
Hi [Engineer Name],

[Client Name] has declined your quote for the session on [Date] at [Start Time].

Original Quote Details:
- Session Type: [Session Type]
- Duration: [Duration] hours
- Quoted Amount: $[Quoted Amount]
- Quote Sent: [Quote Sent At]

The soft hold has been released and the time slot is now available for other bookings.

View client's original request:
[Dashboard Link]

---
[Studio Name]
```

#### 8. Cancellation Confirmation (to Client)
**Subject**: Booking Cancelled - [Date] Session

**Trigger**: Booking cancelled by client, engineer, or admin

```
Hi [Client Name],

Your booking has been cancelled as requested.

Cancelled Booking Details:
- Confirmation Code: [Confirmation Code]
- Session Type: [Session Type]
- Original Date: [Date]
- Original Time: [Start Time] - [End Time]
- Engineer: [Engineer Name]
- Room: [Room Name]
- Cancelled On: [Cancelled At]
- Cancelled By: [Cancelled By]
[If reason provided]
- Reason: [Cancellation Reason]

[If quoted amount was paid - Future Enhancement]
Refund Status: [Refund details if applicable]

We hope to work with you again soon! Book another session:
[Booking Page Link]

Questions? Contact us at [Email] or [Phone Number].

---
[Studio Name]
```

#### 9. Reschedule Request Confirmation (to Client)
**Subject**: Reschedule Request Received - New Quote Coming Soon

**Trigger**: Client or engineer initiates reschedule flow

```
Hi [Client Name],

We've received your request to reschedule your session.

Original Booking:
- Confirmation Code: [Original Confirmation Code]
- Original Date: [Original Date]
- Original Time: [Original Start Time] - [Original End Time]

Requested New Time:
- New Date: [New Date]
- New Time: [New Start Time] - [New End Time]

What Happens Next:
1. Your engineer will review your reschedule request and send a new quote
2. Your original booking remains active until you accept the new quote
3. Once you accept the new quote, your original booking will be automatically cancelled

New Confirmation Code: [New Confirmation Code]

Track your reschedule request:
[Booking Status Link with New Confirmation Code]

Questions? Contact [Engineer Name]: [Engineer Email] | [Phone Number]

---
[Studio Name]
```

### Email Configuration

```javascript
// config/email.js
const emailConfig = {
  smtp: {
    host: process.env.SMTP_HOST || 'smtp.gmail.com',
    port: process.env.SMTP_PORT || 587,
    secure: false, // true for 465, false for other ports
    auth: {
      user: process.env.SMTP_USER,
      pass: process.env.SMTP_PASS
    }
  },
  from: {
    name: process.env.STUDIO_NAME || 'Audio Studio',
    email: process.env.STUDIO_EMAIL || 'bookings@studio.com'
  },
  ownerEmail: process.env.OWNER_EMAIL || 'owner@studio.com',
  escalationDays: 3
};
```

---

## Future Extensibility

### 1. Google Calendar Integration

**Architecture**:
```
[Studio Scheduler] <---> [Google Calendar API]
                           - OAuth 2.0 for engineer authorization
                           - Sync bookings to engineer's calendar
                           - Read engineer's personal events to block availability
```

**Implementation Approach**:
- Add `google_calendar_id` column to `engineers` table
- Store OAuth refresh token securely
- Create background sync job:
  - Runs every 15 minutes
  - Pushes confirmed bookings (status: BOOKED) to Google Calendar
  - Pulls busy times from Google Calendar to mark unavailable
- Use Google Calendar API webhooks for real-time updates

**Code Structure**:
```
/services
  /integrations
    /google-calendar
      - auth.js (OAuth flow)
      - sync.js (two-way sync logic)
      - webhooks.js (handle Google Calendar changes)
```

### 2. Payment Integration

**Architecture**:
```
[Booking Flow] --> [Payment Gateway (Stripe/Square)]
                    - Collect deposit (e.g., 50%)
                    - Create invoice
                    - Send payment link to client
```

**Database Changes**:
- Add tables:
  - `payments` (id, booking_id, amount, status, payment_method, transaction_id)
  - `invoices` (id, booking_id, total_amount, paid_amount, status, due_date)

**Implementation Approach**:
- Add payment step during or after quote acceptance
- Quote generation calculates total:
  - Engineer fee: [hours] × [engineer_rate_for_service_type]
    - Rate determined by engineer's pricing for the specific service type
  - Room fee: [hours] × [room_rate_for_session_type]
    - Rate determined by room's pricing (base rate or session-specific rate)
  - Manual adjustments: [optional discount or fee modifications]
  - Total: [engineer_fee + room_fee + manual_adjustment]
- **Payment Options**:
  1. **Deposit on quote acceptance**: Client pays % deposit to confirm booking
  2. **Full payment on acceptance**: Client pays full amount when accepting quote
  3. **Pay later**: Client accepts quote, pays closer to session date
- Send payment link via email or embed in quote acceptance page
- Webhook handler for payment confirmation
- Update booking status to "PAID" (variant of BOOKED)

**Code Structure**:
```
/services
  /payments
    - stripe.js or square.js
    - invoice-generator.js
    - payment-webhooks.js
```

### 3. Reporting & Analytics Module

**Key Metrics**:
- Room utilization percentage
- Engineer productivity (hours booked vs. available)
- Revenue per room, per engineer, per session type
- Client retention rate
- Average session duration
- Popular time slots

**Implementation**:
- Create read-only analytics endpoints
- Use database views or materialized views for performance
- Build admin dashboard with charts (Chart.js or Recharts)

**Example Query for Room Utilization**:
```sql
SELECT
  r.name,
  COUNT(b.id) as total_bookings,
  SUM(b.total_hours) as total_hours_booked,
  SUM(b.total_hours) / (DAYS_IN_MONTH * 10) * 100 as utilization_percent
FROM rooms r
LEFT JOIN bookings b ON r.id = b.room_id
WHERE b.status = 'completed'
  AND b.start_time >= '2024-03-01'
  AND b.start_time < '2024-04-01'
GROUP BY r.id, r.name;
```

### 4. Mobile Application

**Tech Stack**:
- React Native (shares code with web frontend)
- Expo for easier development
- Push notifications via Firebase Cloud Messaging

**Features**:
- Engineer: Generate quotes, request info, or decline bookings on the go
- Client: Request sessions and accept quotes from mobile
- Push notifications for booking state changes (REQUESTED, QUOTED, BOOKED, EXPIRED)

### 5. Multi-Studio Support

**For future scaling if studio expands to multiple locations**:

**Database Changes**:
- Add `studios` table (id, name, address, timezone)
- Add `studio_id` foreign key to `rooms` and `engineers`
- Filter all queries by studio_id

**UI Changes**:
- Add studio selector on booking page
- Engineer can be associated with multiple studios

---

## Next Steps

### Immediate Actions

1. **Choose Technology Stack**:
   - Review Option A (Node.js) vs. Option B (Python)
   - Consider team expertise and preferences
   - Make decision by: [Date]

2. **Set Up Development Environment**:
   - Install required software (Node.js/Python, PostgreSQL, Docker)
   - Create Git repository
   - Set up project structure

3. **Define Initial Data**:
   - Confirm engineer names and contact info
   - Define room specifications
   - List all session types and pricing

4. **Design Review** (Optional):
   - Review wireframes/mockups for UI
   - Confirm user flows match expectations

5. **Sprint Planning**:
   - Assign tasks from Phase 1
   - Set up project management (Trello, Jira, or GitHub Projects)
   - Schedule weekly check-ins

### Questions to Resolve Before Starting

1. **Engineer Authentication**:
   - How should engineers create accounts? (admin creates, self-registration with approval, etc.)
   - Password reset flow needed?

2. **Booking Modifications**:
   - Can clients cancel/reschedule after accepting a quote (status: BOOKED)? If so, what's the deadline?
   - Can engineers cancel confirmed bookings (status: BOOKED)? What's the notification process?
   - Can clients modify requests while in REQUESTED or QUOTED state?

3. **Time Zone Handling**:
   - Assume all times are in studio's local timezone?
   - Need to support clients in different timezones?

4. **Business Hours**:
   - What are studio operating hours?
   - Can bookings be made outside business hours (e.g., overnight sessions)?

5. **Session Preparation/Teardown**:
   - Is there buffer time needed between sessions?
   - Should system automatically block 30 min before/after for setup?

6. **Pricing Complexity**:
   - Is it simple hourly rate (engineer + room), or are there other factors?
   - Different rates for weekends/holidays?
   - Discounts for long sessions or repeat clients?

---

## Conclusion

This implementation plan provides a comprehensive roadmap for building the Audio Studio Scheduler. The phased approach ensures we deliver a functional MVP quickly while maintaining flexibility for future enhancements.

### Success Criteria

**MVP Success** (End of Phase 1):
- ✅ Clients can successfully request sessions through web portal (with lead time validation)
- ✅ Soft holds prevent double-bookings
- ✅ Engineers receive email notifications and can send quotes with custom pricing
- ✅ Clients can accept quotes to confirm bookings
- ✅ Automatic expiration releases holds after TTL timeout
- ✅ System prevents double-bookings and enforces room eligibility rules
- ✅ Basic recommendation engine suggests appropriate engineer/room
- ✅ All data persists correctly in database

**Full Product Success** (End of Phase 3):
- ✅ Recurring bookings work reliably (each booking independent in V1)
- ✅ Auto-escalation catches unresponsive engineers
- ✅ Client database tracks history and preferences
- ✅ Quote pricing calculated accurately (room + engineer rates + adjustments)
- ✅ System handles edge cases gracefully (concurrency, expiration, conflicts)
- ✅ Mobile-responsive design
- ✅ Production-ready security and performance

### Timeline Summary

- **Phase 1 (MVP)**: 4 weeks
- **Phase 2 (Enhanced Features)**: 2 weeks
- **Phase 3 (Polish & Testing)**: 2 weeks
- **Total to Production-Ready**: ~8 weeks

### Getting Started

Once you're ready to begin implementation, we'll start with:
1. Setting up the Git repository
2. Creating the database schema
3. Building the first API endpoint
4. Creating the booking form UI

Let's build something great! 🎵🎙️
