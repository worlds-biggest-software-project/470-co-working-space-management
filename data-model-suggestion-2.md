# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Co-Working Space Management (Candidate #470)
> Generated: 2026-05-26

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the co-working space management domain. Every state change -- a membership activation, a booking confirmation, a payment received, a door unlocked -- is captured as an immutable event in an append-only event store. The current state of any aggregate (member, booking, resource, invoice) is derived by replaying its event stream. Separate read-model projections are maintained for queries, optimized for the specific access patterns of the operator dashboard, member portal, and analytics views.

This architecture is particularly well-suited to co-working management because:

1. **Booking conflicts require temporal reasoning**: Event sourcing naturally captures the sequence of reservation attempts, confirmations, and cancellations, enabling conflict resolution based on event ordering rather than optimistic locking.
2. **Billing demands a complete audit trail**: Every charge, credit, proration, and refund is recorded as an event, making financial reconciliation and dispute resolution trivial.
3. **Access control is event-driven by nature**: Granting and revoking door access based on membership status or booking windows maps directly to an event stream.
4. **Multi-system integration benefits from event-driven architecture**: Stripe webhooks, smart-lock status changes, and calendar sync are all naturally event-based interactions.

The primary event store uses PostgreSQL for its transactional guarantees, JSONB support for event payloads, and the ability to co-exist with read-model tables in the same database during early stages. For production scale, EventStoreDB or Apache Kafka can serve as the primary event log with PostgreSQL read models.

---

## Event Store Schema

```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- The core event store table
CREATE TABLE event_store (
    -- Global ordering
    global_position   BIGSERIAL PRIMARY KEY,
    -- Stream identification
    stream_id         VARCHAR(500) NOT NULL,  -- e.g. "membership-{uuid}", "booking-{uuid}"
    stream_type       VARCHAR(100) NOT NULL,  -- e.g. "Membership", "Booking", "Invoice"
    -- Event position within stream
    stream_position   INT NOT NULL,
    -- Event metadata
    event_id          UUID NOT NULL DEFAULT uuid_generate_v4() UNIQUE,
    event_type        VARCHAR(200) NOT NULL,  -- e.g. "MembershipActivated", "BookingConfirmed"
    -- Payload
    data              JSONB NOT NULL,         -- the event payload
    metadata          JSONB NOT NULL DEFAULT '{}',
    -- metadata contains: correlation_id, causation_id, user_id, ip_address, timestamp, schema_version
    -- Timing
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Optimistic concurrency control
    UNIQUE(stream_id, stream_position)
);

-- Indexes for common access patterns
CREATE INDEX idx_event_store_stream ON event_store(stream_id, stream_position);
CREATE INDEX idx_event_store_type ON event_store(event_type, global_position);
CREATE INDEX idx_event_store_stream_type ON event_store(stream_type, global_position);
CREATE INDEX idx_event_store_created ON event_store(created_at);
CREATE INDEX idx_event_store_correlation ON event_store((metadata->>'correlation_id'));

-- Snapshot store (for aggregates with long event histories)
CREATE TABLE snapshots (
    stream_id         VARCHAR(500) PRIMARY KEY,
    stream_type       VARCHAR(100) NOT NULL,
    stream_position   INT NOT NULL,
    state             JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Projection checkpoints (track where each projection has read up to)
CREATE TABLE projection_checkpoints (
    projection_name   VARCHAR(200) PRIMARY KEY,
    last_position     BIGINT NOT NULL DEFAULT 0,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_events (
    id                BIGSERIAL PRIMARY KEY,
    global_position   BIGINT NOT NULL,
    event_id          UUID NOT NULL,
    event_type        VARCHAR(200) NOT NULL,
    projection_name   VARCHAR(200) NOT NULL,
    error_message     TEXT NOT NULL,
    retry_count       INT NOT NULL DEFAULT 0,
    max_retries       INT NOT NULL DEFAULT 5,
    next_retry_at     TIMESTAMPTZ,
    resolved_at       TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Aggregate Roots and Event Types

### Membership Aggregate

The Membership aggregate manages the lifecycle of a member's subscription to a plan.

**Stream ID pattern**: `membership-{membership_id}`

```
Commands:
  CreateMembership        -> MembershipCreated
  ActivateMembership      -> MembershipActivated
  SuspendMembership       -> MembershipSuspended
  ReactivateMembership    -> MembershipReactivated
  ChangePlan              -> PlanChangeRequested -> PlanChangeEffected
  CancelMembership        -> MembershipCancellationRequested -> MembershipCancelled
  RenewMembership         -> MembershipRenewed
  AssignResource          -> ResourceAssigned
  UnassignResource        -> ResourceUnassigned
  AdjustCredits           -> CreditsAdjusted
  RollOverCredits         -> CreditsRolledOver
  ExpireCredits           -> CreditsExpired
```

**Event payload examples**:

```json
// MembershipCreated
{
    "event_type": "MembershipCreated",
    "data": {
        "membership_id": "550e8400-e29b-41d4-a716-446655440001",
        "user_id": "550e8400-e29b-41d4-a716-446655440002",
        "organization_id": "550e8400-e29b-41d4-a716-446655440003",
        "plan_id": "550e8400-e29b-41d4-a716-446655440004",
        "location_id": "550e8400-e29b-41d4-a716-446655440005",
        "plan_type": "dedicated_desk",
        "start_date": "2026-06-01",
        "base_price": 450.00,
        "currency": "USD",
        "billing_cycle": "monthly",
        "trial_end_date": "2026-06-14",
        "included_credits": 10,
        "stripe_subscription_id": "sub_1NQJcF2eZvKYlo2C"
    },
    "metadata": {
        "correlation_id": "req-abc123",
        "causation_id": null,
        "user_id": "550e8400-e29b-41d4-a716-446655440002",
        "schema_version": 1
    }
}

// PlanChangeEffected
{
    "event_type": "PlanChangeEffected",
    "data": {
        "membership_id": "550e8400-e29b-41d4-a716-446655440001",
        "previous_plan_id": "550e8400-e29b-41d4-a716-446655440004",
        "new_plan_id": "550e8400-e29b-41d4-a716-446655440010",
        "previous_plan_type": "dedicated_desk",
        "new_plan_type": "private_office",
        "effective_date": "2026-07-01",
        "proration_amount": -75.00,
        "new_price": 850.00,
        "reason": "upgrade"
    },
    "metadata": {
        "correlation_id": "req-def456",
        "causation_id": "cmd-planchange-789",
        "user_id": "admin-550e8400",
        "schema_version": 1
    }
}

// CreditsAdjusted
{
    "event_type": "CreditsAdjusted",
    "data": {
        "membership_id": "550e8400-e29b-41d4-a716-446655440001",
        "adjustment": -2,
        "balance_before": 10,
        "balance_after": 8,
        "reason": "meeting_room_booking",
        "booking_id": "550e8400-e29b-41d4-a716-446655440020"
    },
    "metadata": {
        "correlation_id": "req-ghi789",
        "schema_version": 1
    }
}
```

### Booking Aggregate

The Booking aggregate handles reservation lifecycle from request through completion.

**Stream ID pattern**: `booking-{booking_id}`

```
Commands:
  RequestBooking          -> BookingRequested
  ConfirmBooking          -> BookingConfirmed
  CheckIn                 -> BookingCheckedIn
  CheckOut                -> BookingCheckedOut
  CancelBooking           -> BookingCancelled
  ModifyBooking           -> BookingModified
  MarkNoShow              -> BookingMarkedNoShow
  AddAttendee             -> AttendeeAdded
  RemoveAttendee          -> AttendeeRemoved
  LinkToRecurrence        -> RecurrenceLinked
```

**Event payload examples**:

```json
// BookingRequested
{
    "event_type": "BookingRequested",
    "data": {
        "booking_id": "550e8400-e29b-41d4-a716-446655440020",
        "resource_id": "550e8400-e29b-41d4-a716-446655440030",
        "user_id": "550e8400-e29b-41d4-a716-446655440002",
        "organization_id": "550e8400-e29b-41d4-a716-446655440003",
        "location_id": "550e8400-e29b-41d4-a716-446655440005",
        "resource_type": "meeting_room",
        "resource_name": "Conference Room A",
        "start_time": "2026-06-15T10:00:00Z",
        "end_time": "2026-06-15T11:30:00Z",
        "title": "Product Review Meeting",
        "attendee_count": 6,
        "total_price": 45.00,
        "credits_used": 0,
        "membership_id": "550e8400-e29b-41d4-a716-446655440001"
    },
    "metadata": {
        "correlation_id": "req-booking-001",
        "schema_version": 1
    }
}

// BookingConfirmed
{
    "event_type": "BookingConfirmed",
    "data": {
        "booking_id": "550e8400-e29b-41d4-a716-446655440020",
        "confirmed_at": "2026-06-10T14:32:00Z",
        "confirmation_method": "automatic",
        "access_granted": true,
        "access_point_ids": ["ap-001", "ap-002"],
        "calendar_event_synced": true
    },
    "metadata": {
        "correlation_id": "req-booking-001",
        "causation_id": "evt-BookingRequested-020",
        "schema_version": 1
    }
}
```

### Invoice Aggregate

**Stream ID pattern**: `invoice-{invoice_id}`

```
Commands:
  CreateInvoice           -> InvoiceCreated
  AddLineItem             -> LineItemAdded
  RemoveLineItem          -> LineItemRemoved
  FinalizeInvoice         -> InvoiceFinalized
  RecordPayment           -> PaymentRecorded
  RecordPartialPayment    -> PartialPaymentRecorded
  MarkOverdue             -> InvoiceMarkedOverdue
  ApplyDiscount           -> DiscountApplied
  VoidInvoice             -> InvoiceVoided
  RefundPayment           -> PaymentRefunded
  SendDunningNotice       -> DunningNoticeSent
  RecordStripeWebhook     -> StripeWebhookProcessed
```

**Event payload example**:

```json
// InvoiceCreated
{
    "event_type": "InvoiceCreated",
    "data": {
        "invoice_id": "550e8400-e29b-41d4-a716-446655440040",
        "organization_id": "550e8400-e29b-41d4-a716-446655440003",
        "user_id": "550e8400-e29b-41d4-a716-446655440002",
        "membership_id": "550e8400-e29b-41d4-a716-446655440001",
        "invoice_number": "INV-2026-001234",
        "billing_period_start": "2026-06-01",
        "billing_period_end": "2026-06-30",
        "issue_date": "2026-06-01",
        "due_date": "2026-06-15",
        "currency": "USD",
        "line_items": [
            {
                "description": "Dedicated Desk - Monthly",
                "quantity": 1,
                "unit_price": 450.00,
                "amount": 450.00
            },
            {
                "description": "Meeting Room - 3 hours",
                "quantity": 3,
                "unit_price": 30.00,
                "amount": 90.00
            }
        ],
        "subtotal": 540.00,
        "tax_rate": 0.08,
        "tax_amount": 43.20,
        "total": 583.20,
        "stripe_invoice_id": "in_1NQJcF2eZvKYlo2C"
    },
    "metadata": {
        "correlation_id": "billing-cycle-2026-06",
        "schema_version": 1
    }
}
```

### Access Control Aggregate

**Stream ID pattern**: `access-{user_id}-{location_id}`

```
Commands:
  GrantAccess             -> AccessGranted
  RevokeAccess            -> AccessRevoked
  RegisterCredential      -> CredentialRegistered
  DeactivateCredential    -> CredentialDeactivated
  RecordAccessEvent       -> DoorUnlocked / DoorLocked / AccessDenied
  SyncToProvider          -> ProviderSyncCompleted / ProviderSyncFailed
  GrantTemporaryAccess    -> TemporaryAccessGranted
  RevokeTemporaryAccess   -> TemporaryAccessRevoked
```

### Community Aggregate

**Stream ID pattern**: `post-{post_id}`, `thread-{thread_id}`

```
Commands:
  CreatePost              -> PostCreated
  EditPost                -> PostEdited
  DeletePost              -> PostDeleted
  LikePost                -> PostLiked
  UnlikePost              -> PostUnliked
  CommentOnPost           -> CommentAdded
  ModeratePost            -> PostModerated
  PinPost                 -> PostPinned
  UnpinPost               -> PostUnpinned
  FlagPost                -> PostFlagged
  SendDirectMessage       -> DirectMessageSent
  ReadMessages            -> MessagesRead
```

### Visitor Aggregate

**Stream ID pattern**: `visitor-{visitor_pass_id}`

```
Commands:
  PreRegisterVisitor      -> VisitorPreRegistered
  CheckInVisitor          -> VisitorCheckedIn
  CheckOutVisitor         -> VisitorCheckedOut
  NotifyHost              -> HostNotified
  GrantTemporaryAccess    -> VisitorAccessGranted
  DenyVisitor             -> VisitorDenied
  CancelVisit             -> VisitCancelled
```

### Event (Calendar Event) Aggregate

**Stream ID pattern**: `event-{event_id}`

```
Commands:
  CreateEvent             -> EventCreated
  UpdateEvent             -> EventUpdated
  PublishEvent            -> EventPublished
  CancelEvent             -> EventCancelled
  RSVP                    -> RSVPRecorded
  CancelRSVP              -> RSVPCancelled
  CheckInAttendee         -> AttendeeCheckedIn
  LinkRoomBooking         -> RoomBookingLinked
```

---

## Process Managers (Sagas)

Process managers coordinate multi-aggregate workflows that span several event streams.

### Membership Activation Saga

Triggered by `MembershipCreated`, orchestrates:
1. Create Stripe subscription -> listen for `StripeWebhookProcessed`
2. Grant access permissions -> emit `AccessGranted` for each access point
3. Send welcome email -> emit `WelcomeEmailSent`
4. Create first invoice -> emit `InvoiceCreated`
5. Sync to calendar if applicable

```
MembershipCreated
  -> CreateStripeSubscription (command)
      -> StripeWebhookProcessed (subscription.created)
          -> GrantAccess (command)
              -> AccessGranted
          -> CreateInvoice (command)
              -> InvoiceCreated
          -> SendWelcomeNotification (command)
              -> NotificationSent
  -> MembershipActivated (final state)
```

### Booking Confirmation Saga

Triggered by `BookingRequested`, orchestrates:
1. Check resource availability (read model query)
2. Reserve the resource (optimistic reservation)
3. Deduct credits or charge payment
4. Grant temporary access for the booking window
5. Sync to external calendars
6. Send confirmation notification

```
BookingRequested
  -> ValidateAvailability (query against read model)
      -> [Available] ConfirmBooking (command)
          -> BookingConfirmed
              -> GrantTemporaryAccess (command)
              -> DeductCredits (command) OR CreateCharge (command)
              -> SyncCalendar (command)
              -> SendConfirmation (command)
      -> [Unavailable] RejectBooking (command)
          -> BookingRejected
```

### Billing Cycle Saga

Triggered on a schedule (e.g., first of each month), orchestrates:
1. For each active membership, create an invoice
2. Add line items for base plan + overages
3. Finalize and send to Stripe
4. Listen for payment confirmation
5. Handle failed payments with dunning sequence

```
BillingCycleTriggered
  -> [For each active membership]
      -> CreateInvoice (command) + AddLineItems
          -> InvoiceFinalized
              -> ChargeStripe (command)
                  -> [Success] PaymentRecorded
                  -> [Failure] PaymentFailed
                      -> ScheduleDunning
                          -> DunningNoticeSent (1st, 2nd, 3rd attempt)
                              -> [Still failed] SuspendMembership
```

### Member Offboarding Saga

Triggered by `MembershipCancelled`, orchestrates:
1. Revoke all access permissions
2. Cancel future bookings
3. Final invoice and proration
4. Remove from community groups (optional)
5. Archive data per retention policy

---

## Read Model Projections

Read models are separate PostgreSQL tables, optimized for specific query patterns. Each projection subscribes to the event stream and updates its tables as events arrive.

### Member Dashboard Projection

```sql
-- Current state of all memberships (for member portal and admin dashboard)
CREATE TABLE rm_memberships (
    membership_id     UUID PRIMARY KEY,
    user_id           UUID NOT NULL,
    user_email        VARCHAR(255) NOT NULL,
    user_name         VARCHAR(200) NOT NULL,
    organization_id   UUID NOT NULL,
    location_id       UUID,
    plan_id           UUID NOT NULL,
    plan_name         VARCHAR(255) NOT NULL,
    plan_type         VARCHAR(50) NOT NULL,
    status            VARCHAR(50) NOT NULL,
    start_date        DATE NOT NULL,
    end_date          DATE,
    current_period_start DATE,
    current_period_end   DATE,
    base_price        DECIMAL(10, 2) NOT NULL,
    price_override    DECIMAL(10, 2),
    credit_balance    INT NOT NULL DEFAULT 0,
    assigned_resource_name VARCHAR(255),
    stripe_subscription_id VARCHAR(255),
    cancelled_at      TIMESTAMPTZ,
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_memberships_user ON rm_memberships(user_id);
CREATE INDEX idx_rm_memberships_org ON rm_memberships(organization_id, status);
```

### Booking Calendar Projection

```sql
-- Real-time resource availability and booking calendar
CREATE TABLE rm_resource_availability (
    resource_id       UUID NOT NULL,
    booking_date      DATE NOT NULL,
    location_id       UUID NOT NULL,
    resource_type     VARCHAR(50) NOT NULL,
    resource_name     VARCHAR(255) NOT NULL,
    capacity          INT NOT NULL,
    is_available      BOOLEAN NOT NULL DEFAULT true,
    PRIMARY KEY (resource_id, booking_date)
);

CREATE TABLE rm_bookings (
    booking_id        UUID PRIMARY KEY,
    resource_id       UUID NOT NULL,
    resource_name     VARCHAR(255) NOT NULL,
    resource_type     VARCHAR(50) NOT NULL,
    location_id       UUID NOT NULL,
    location_name     VARCHAR(255) NOT NULL,
    user_id           UUID NOT NULL,
    user_name         VARCHAR(200) NOT NULL,
    title             VARCHAR(255),
    status            VARCHAR(50) NOT NULL,
    start_time        TIMESTAMPTZ NOT NULL,
    end_time          TIMESTAMPTZ NOT NULL,
    total_price       DECIMAL(10, 2),
    credits_used      INT DEFAULT 0,
    checked_in_at     TIMESTAMPTZ,
    attendee_count    INT DEFAULT 0,
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_bookings_resource ON rm_bookings(resource_id, start_time, end_time) WHERE status NOT IN ('cancelled', 'no_show');
CREATE INDEX idx_rm_bookings_user ON rm_bookings(user_id, start_time);
CREATE INDEX idx_rm_bookings_location ON rm_bookings(location_id, start_time);

-- Time-slot view for availability checks (materialized for performance)
CREATE TABLE rm_time_slots (
    resource_id       UUID NOT NULL,
    slot_date         DATE NOT NULL,
    slot_start        TIME NOT NULL,
    slot_end          TIME NOT NULL,
    is_booked         BOOLEAN NOT NULL DEFAULT false,
    booking_id        UUID,
    booked_by_user_id UUID,
    PRIMARY KEY (resource_id, slot_date, slot_start)
);

CREATE INDEX idx_rm_time_slots ON rm_time_slots(resource_id, slot_date) WHERE is_booked = false;
```

### Billing Projection

```sql
-- Invoice read model (for member portal invoice list and admin billing dashboard)
CREATE TABLE rm_invoices (
    invoice_id        UUID PRIMARY KEY,
    invoice_number    VARCHAR(50) UNIQUE NOT NULL,
    organization_id   UUID NOT NULL,
    user_id           UUID,
    user_name         VARCHAR(200),
    user_email        VARCHAR(255),
    corporate_account_id UUID,
    corporate_name    VARCHAR(255),
    membership_id     UUID,
    status            VARCHAR(50) NOT NULL,
    issue_date        DATE NOT NULL,
    due_date          DATE NOT NULL,
    paid_date         DATE,
    subtotal          DECIMAL(12, 2) NOT NULL,
    tax_amount        DECIMAL(12, 2) NOT NULL,
    discount_amount   DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total             DECIMAL(12, 2) NOT NULL,
    amount_paid       DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_due        DECIMAL(12, 2) NOT NULL,
    currency          CHAR(3) NOT NULL,
    line_items        JSONB NOT NULL DEFAULT '[]',
    stripe_invoice_id VARCHAR(255),
    dunning_attempts  INT NOT NULL DEFAULT 0,
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_invoices_user ON rm_invoices(user_id, issue_date);
CREATE INDEX idx_rm_invoices_org ON rm_invoices(organization_id, status);
CREATE INDEX idx_rm_invoices_overdue ON rm_invoices(due_date) WHERE status IN ('open', 'overdue');

-- Payment history
CREATE TABLE rm_payments (
    payment_id        UUID PRIMARY KEY,
    invoice_id        UUID,
    user_id           UUID,
    amount            DECIMAL(12, 2) NOT NULL,
    currency          CHAR(3) NOT NULL,
    status            VARCHAR(50) NOT NULL,
    payment_method    VARCHAR(50),
    stripe_payment_id VARCHAR(255),
    payment_date      TIMESTAMPTZ,
    failure_reason    TEXT,
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Access Control Projection

```sql
-- Current access permissions (read model for access control decisions)
CREATE TABLE rm_access_permissions (
    user_id           UUID NOT NULL,
    access_point_id   UUID NOT NULL,
    location_id       UUID NOT NULL,
    source_type       VARCHAR(50) NOT NULL,  -- 'membership', 'booking', 'visitor'
    source_id         UUID NOT NULL,
    valid_from        TIMESTAMPTZ NOT NULL,
    valid_until       TIMESTAMPTZ,
    allowed_days      INT[],
    allowed_start_time TIME,
    allowed_end_time   TIME,
    is_active         BOOLEAN NOT NULL DEFAULT true,
    synced_to_provider BOOLEAN NOT NULL DEFAULT false,
    provider          VARCHAR(50),
    external_id       VARCHAR(255),
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, access_point_id, source_id)
);

CREATE INDEX idx_rm_access_active ON rm_access_permissions(access_point_id) WHERE is_active = true;

-- Access event log (append-only, used for security audit and occupancy tracking)
CREATE TABLE rm_access_log (
    id                BIGSERIAL PRIMARY KEY,
    access_point_id   UUID NOT NULL,
    location_id       UUID NOT NULL,
    user_id           UUID,
    event_type        VARCHAR(50) NOT NULL,
    success           BOOLEAN NOT NULL,
    occurred_at       TIMESTAMPTZ NOT NULL,
    credential_type   VARCHAR(50),
    global_position   BIGINT NOT NULL
);

CREATE INDEX idx_rm_access_log_time ON rm_access_log(location_id, occurred_at);
CREATE INDEX idx_rm_access_log_user ON rm_access_log(user_id, occurred_at);
```

### Community Projection

```sql
-- Community feed (denormalized for fast feed rendering)
CREATE TABLE rm_community_feed (
    post_id           UUID PRIMARY KEY,
    organization_id   UUID NOT NULL,
    location_id       UUID,
    author_id         UUID NOT NULL,
    author_name       VARCHAR(200) NOT NULL,
    author_avatar_url VARCHAR(500),
    post_type         VARCHAR(50) NOT NULL,
    title             VARCHAR(500),
    body              TEXT NOT NULL,
    media_urls        VARCHAR(500)[],
    like_count        INT NOT NULL DEFAULT 0,
    comment_count     INT NOT NULL DEFAULT 0,
    is_pinned         BOOLEAN NOT NULL DEFAULT false,
    moderation_status VARCHAR(50) NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL,
    last_event_position BIGINT NOT NULL
);

CREATE INDEX idx_rm_feed_org ON rm_community_feed(organization_id, created_at DESC);
CREATE INDEX idx_rm_feed_pinned ON rm_community_feed(organization_id, is_pinned) WHERE is_pinned = true;
```

### Analytics Projection

```sql
-- Daily occupancy stats (rebuilt from access and booking events)
CREATE TABLE rm_daily_occupancy (
    location_id       UUID NOT NULL,
    resource_type     VARCHAR(50) NOT NULL,
    stat_date         DATE NOT NULL,
    total_bookings    INT NOT NULL DEFAULT 0,
    unique_users      INT NOT NULL DEFAULT 0,
    total_hours_booked DECIMAL(10, 2) NOT NULL DEFAULT 0,
    total_resources   INT NOT NULL DEFAULT 0,
    utilization_pct   DECIMAL(5, 2),
    peak_hour         INT,  -- hour of day with most bookings
    check_ins         INT NOT NULL DEFAULT 0,
    no_shows          INT NOT NULL DEFAULT 0,
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (location_id, resource_type, stat_date)
);

-- Monthly revenue stats
CREATE TABLE rm_monthly_revenue (
    organization_id   UUID NOT NULL,
    revenue_month     DATE NOT NULL,  -- first day of month
    currency          CHAR(3) NOT NULL,
    total_invoiced    DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total_collected   DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total_outstanding DECIMAL(12, 2) NOT NULL DEFAULT 0,
    new_memberships   INT NOT NULL DEFAULT 0,
    churned_memberships INT NOT NULL DEFAULT 0,
    active_memberships INT NOT NULL DEFAULT 0,
    mrr               DECIMAL(12, 2) NOT NULL DEFAULT 0,
    arr               DECIMAL(12, 2) NOT NULL DEFAULT 0,
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (organization_id, revenue_month, currency)
);

-- Member engagement scoring
CREATE TABLE rm_member_engagement (
    user_id           UUID NOT NULL,
    organization_id   UUID NOT NULL,
    -- Activity counts (rolling 30 days)
    bookings_30d      INT NOT NULL DEFAULT 0,
    check_ins_30d     INT NOT NULL DEFAULT 0,
    community_posts_30d INT NOT NULL DEFAULT 0,
    events_attended_30d INT NOT NULL DEFAULT 0,
    messages_sent_30d   INT NOT NULL DEFAULT 0,
    -- Engagement score (calculated)
    engagement_score  DECIMAL(5, 2) NOT NULL DEFAULT 0,
    churn_risk        VARCHAR(20) DEFAULT 'low',  -- 'low', 'medium', 'high'
    last_activity_at  TIMESTAMPTZ,
    last_event_position BIGINT NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, organization_id)
);

CREATE INDEX idx_rm_engagement_risk ON rm_member_engagement(organization_id, churn_risk) WHERE churn_risk IN ('medium', 'high');
```

---

## Event Processing Architecture

### Appending Events (Write Path)

```python
# Pseudocode for appending events with optimistic concurrency
async def append_events(stream_id: str, stream_type: str,
                        expected_position: int, events: list[Event]) -> int:
    """
    Append events to a stream with optimistic concurrency control.
    Raises ConcurrencyConflict if expected_position doesn't match.
    """
    async with db.transaction():
        # Check current position
        current = await db.fetchval(
            "SELECT MAX(stream_position) FROM event_store WHERE stream_id = $1",
            stream_id
        )
        current = current if current is not None else -1

        if current != expected_position:
            raise ConcurrencyConflict(
                f"Stream {stream_id}: expected position {expected_position}, "
                f"found {current}"
            )

        # Insert events
        position = expected_position
        for event in events:
            position += 1
            await db.execute("""
                INSERT INTO event_store
                    (stream_id, stream_type, stream_position,
                     event_type, data, metadata)
                VALUES ($1, $2, $3, $4, $5, $6)
            """, stream_id, stream_type, position,
                 event.type, event.data, event.metadata)

        # Notify projections via LISTEN/NOTIFY
        await db.execute(
            "NOTIFY event_appended, $1",
            json.dumps({"stream_id": stream_id, "position": position})
        )

        return position
```

### Projection Processing (Read Path)

```python
# Pseudocode for catch-up subscription projection
class BookingProjection:
    PROJECTION_NAME = "booking_calendar"

    HANDLERS = {
        "BookingRequested": "_on_booking_requested",
        "BookingConfirmed": "_on_booking_confirmed",
        "BookingCancelled": "_on_booking_cancelled",
        "BookingCheckedIn": "_on_booking_checked_in",
        "BookingCheckedOut": "_on_booking_checked_out",
        "BookingMarkedNoShow": "_on_booking_no_show",
    }

    async def process(self):
        """Catch up from last checkpoint, then listen for new events."""
        checkpoint = await self._get_checkpoint()

        # Catch-up phase: process all events since last checkpoint
        events = await db.fetch("""
            SELECT * FROM event_store
            WHERE global_position > $1
            AND event_type = ANY($2)
            ORDER BY global_position
        """, checkpoint, list(self.HANDLERS.keys()))

        for event in events:
            handler = getattr(self, self.HANDLERS[event.event_type])
            try:
                await handler(event)
                await self._update_checkpoint(event.global_position)
            except Exception as e:
                await self._dead_letter(event, str(e))

        # Live phase: listen for new events via NOTIFY
        await self._subscribe_live()

    async def _on_booking_requested(self, event):
        data = event.data
        await db.execute("""
            INSERT INTO rm_bookings
                (booking_id, resource_id, resource_name, resource_type,
                 location_id, location_name, user_id, user_name,
                 title, status, start_time, end_time,
                 total_price, credits_used, last_event_position, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, 'pending',
                    $10, $11, $12, $13, $14, NOW())
        """, ...)

        # Update time slots
        await self._mark_slots_booked(
            data["resource_id"],
            data["start_time"],
            data["end_time"],
            data["booking_id"]
        )

    async def _on_booking_cancelled(self, event):
        data = event.data
        await db.execute("""
            UPDATE rm_bookings
            SET status = 'cancelled', last_event_position = $2, updated_at = NOW()
            WHERE booking_id = $1
        """, data["booking_id"], event.global_position)

        # Free up time slots
        await self._mark_slots_available(
            data["resource_id"],
            data["start_time"],
            data["end_time"]
        )
```

---

## Booking Conflict Resolution via Event Ordering

One of the critical challenges in co-working booking systems is preventing double-bookings. In the event-sourced model, this is handled by the Booking aggregate's decision function:

```python
class BookingAggregate:
    """
    The aggregate checks availability against a read model
    before emitting BookingRequested. Concurrent requests for the
    same resource/time are serialized via the stream's optimistic
    concurrency control.

    For hot desks (fungible resources), the aggregate checks a
    resource pool counter rather than individual resource assignment.
    """

    def handle_request_booking(self, command, availability_read_model):
        # Query the read model for current availability
        is_available = availability_read_model.check_slot(
            resource_id=command.resource_id,
            start_time=command.start_time,
            end_time=command.end_time
        )

        if not is_available:
            return [BookingRejected(
                booking_id=command.booking_id,
                reason="resource_unavailable",
                resource_id=command.resource_id,
                requested_start=command.start_time,
                requested_end=command.end_time
            )]

        # For fungible resources (hot desks), use a reservation counter
        if command.resource_type == "hot_desk":
            available_count = availability_read_model.get_available_count(
                location_id=command.location_id,
                resource_type="hot_desk",
                date=command.start_time.date()
            )
            if available_count <= 0:
                return [BookingRejected(...)]

        return [BookingRequested(
            booking_id=command.booking_id,
            resource_id=command.resource_id,
            # ... all booking details
        )]
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by default**: Every state change is recorded as an immutable event. This is invaluable for billing disputes ("when was this charge created and why?"), access control auditing ("who entered the building at 3 AM?"), and membership lifecycle tracking. No additional audit logging infrastructure is needed.

2. **Temporal queries are natural**: "What was this member's plan on March 15th?" is answered by replaying events up to that date. Traditional CRUD systems cannot answer temporal questions without separate audit tables.

3. **Perfect fit for Stripe webhook integration**: Stripe's own architecture is event-driven (invoice.created, payment_intent.succeeded, subscription.updated). Mapping Stripe webhooks to domain events creates a seamless, idempotent billing pipeline.

4. **Supports complex billing scenarios**: Mid-month plan changes, prorations, credit rollovers, and corporate invoice consolidation are naturally modeled as sequences of events. The billing projection can reconstruct any invoice's complete history.

5. **Decoupled read and write scaling**: The booking calendar read model can be replicated across multiple instances for the member-facing portal, while the event store handles writes at a lower throughput. Read models can be rebuilt from scratch if corrupted or if new query patterns emerge.

6. **Natural integration architecture**: Smart lock status changes, calendar sync events, and notification delivery confirmations are all naturally modeled as events, making the system's integration layer consistent with its core architecture.

7. **Event replay enables powerful analytics**: Member engagement scoring, churn prediction, and occupancy forecasting can all be implemented as projections that replay historical event streams with new algorithms without modifying the core system.

### Cons

1. **Significant implementation complexity**: Event sourcing requires building or adopting an event store, projection engine, saga/process manager framework, and snapshot mechanism. This is substantially more work than a CRUD approach, especially for a team without prior event sourcing experience.

2. **Eventual consistency for read models**: The booking calendar read model may lag behind the event store by milliseconds to seconds. During this window, a member might see a stale availability view. For high-contention resources (popular meeting rooms), this could lead to user-facing booking rejections after an apparently successful request.

3. **Event schema evolution is painful**: Once events are stored, they are immutable. If the `BookingRequested` event schema changes (e.g., adding a new required field), you must implement upcasting logic to transform old events to the new format during replay. This complexity compounds over years of operation.

4. **Debugging is harder**: Instead of inspecting a single row in a `bookings` table, developers must replay an event stream and understand the sequence of state transitions. This increases cognitive load and slows down incident response.

5. **Storage growth**: The event store grows indefinitely. A busy location generating 1,000 bookings/day, 5,000 access events/day, and 500 community interactions/day will accumulate millions of events per year. Snapshotting mitigates replay time but not storage cost.

6. **Double-booking prevention requires care**: Unlike PostgreSQL's exclusion constraints, the event-sourced approach relies on read-model queries and optimistic concurrency. Under high concurrency, this can allow brief windows of inconsistency. Additional saga logic or reservation tokens are needed to close these windows.

7. **Team skill requirements**: Event sourcing is an advanced architectural pattern. Finding developers who can effectively maintain sagas, handle projection failures, manage schema evolution, and debug event replay issues is significantly harder than finding developers who can work with a relational schema.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| Event Store | PostgreSQL (early stage) -> EventStoreDB (production scale) |
| Read Model DB | PostgreSQL for relational projections |
| Event Bus | PostgreSQL LISTEN/NOTIFY (early) -> Apache Kafka or NATS (production) |
| Saga / Process Manager | Custom implementation or Temporal.io for workflow orchestration |
| Projection Engine | Custom catch-up subscription engine |
| Snapshots | PostgreSQL JSONB (co-located with event store) |
| API Layer | GraphQL for flexible read-model queries; REST for commands |
| Serialization | JSON with schema versioning; consider Avro for Kafka at scale |
| Monitoring | OpenTelemetry for distributed tracing across event processing |

---

## Migration and Scaling Considerations

### Phase 1: PostgreSQL-based event store (0-100 locations)
- Single PostgreSQL instance hosts both event store and read models
- Projections run in-process with the application
- LISTEN/NOTIFY for real-time projection updates
- Snapshot every 100 events per aggregate to keep replay times under 50ms

### Phase 2: Dedicated event infrastructure (100-500 locations)
- Migrate event store to EventStoreDB for native stream support and catch-up subscriptions
- Keep PostgreSQL for read models
- Introduce Kafka for cross-service event distribution
- Deploy projections as independent microservices with their own checkpoints
- Implement schema registry (Confluent Schema Registry) for event versioning

### Phase 3: Global scale (500+ locations)
- Partition event streams by organization (each org gets its own EventStoreDB stream category)
- Deploy read models per region (US, EU, APAC) for low-latency queries
- Archive old events (> 2 years) to cold storage (S3 + Athena for ad-hoc queries)
- Implement event compaction for high-volume streams (access events)

### Event store sizing estimates
- Average event size: ~500 bytes (JSONB payload + metadata)
- Per location per day (estimated):
  - Membership events: ~20
  - Booking events: ~200
  - Access events: ~2,000
  - Billing events: ~50
  - Community events: ~100
  - Total: ~2,370 events/location/day
- 100 locations: ~237,000 events/day = ~86.5M events/year = ~43 GB/year
- 1,000 locations: ~865M events/year = ~430 GB/year (before compression)

### Rebuilding projections
- Projection rebuild time (100M events, single-threaded): ~15-30 minutes
- With parallel processing (8 workers): ~2-4 minutes
- Rebuild strategy: deploy new projection version alongside old, catch up, then switch traffic
