# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Co-Working Space Management (Candidate #470)
> Generated: 2026-05-26

## Overview

This model uses a fully normalized PostgreSQL relational schema, enforcing referential integrity and data consistency across all core domains: organizations, locations, members, memberships, billing, bookings, access control, community, and events. PostgreSQL is chosen for its ACID compliance, native support for range types and exclusion constraints (critical for preventing double-bookings), robust JSON support for edge cases, and mature ecosystem of extensions (PostGIS for location data, pg_cron for scheduled jobs, btree_gist for temporal exclusion constraints).

The schema follows third normal form (3NF) throughout, with strategic denormalization only where PostgreSQL materialized views can serve read-heavy analytics queries without compromising write-path integrity.

---

## Enumerated Types

```sql
-- Extensions required
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "btree_gist";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Core enums
CREATE TYPE membership_status AS ENUM (
    'pending', 'active', 'suspended', 'cancelled', 'expired'
);

CREATE TYPE plan_type AS ENUM (
    'hot_desk', 'dedicated_desk', 'private_office', 'virtual_office',
    'day_pass', 'credit_pack', 'custom'
);

CREATE TYPE billing_cycle AS ENUM (
    'daily', 'weekly', 'monthly', 'quarterly', 'annually'
);

CREATE TYPE booking_status AS ENUM (
    'pending', 'confirmed', 'checked_in', 'completed', 'cancelled', 'no_show'
);

CREATE TYPE resource_type AS ENUM (
    'hot_desk', 'dedicated_desk', 'private_office', 'meeting_room',
    'phone_booth', 'event_space', 'parking_spot', 'locker', 'custom'
);

CREATE TYPE invoice_status AS ENUM (
    'draft', 'open', 'paid', 'partially_paid', 'overdue', 'void', 'uncollectible'
);

CREATE TYPE payment_status AS ENUM (
    'pending', 'processing', 'succeeded', 'failed', 'refunded', 'partially_refunded'
);

CREATE TYPE payment_method_type AS ENUM (
    'credit_card', 'debit_card', 'bank_transfer', 'paypal', 'cash', 'credit_balance'
);

CREATE TYPE access_credential_type AS ENUM (
    'mobile_ble', 'rfid_card', 'pin_code', 'qr_code', 'biometric'
);

CREATE TYPE access_event_type AS ENUM (
    'entry', 'exit', 'denied', 'credential_expired', 'forced_entry'
);

CREATE TYPE post_type AS ENUM (
    'announcement', 'discussion', 'question', 'event_share', 'marketplace'
);

CREATE TYPE moderation_status AS ENUM (
    'pending', 'approved', 'flagged', 'removed', 'auto_approved'
);

CREATE TYPE event_rsvp_status AS ENUM (
    'invited', 'attending', 'maybe', 'declined', 'waitlisted'
);

CREATE TYPE visitor_status AS ENUM (
    'pre_registered', 'checked_in', 'checked_out', 'no_show', 'denied'
);

CREATE TYPE notification_channel AS ENUM (
    'email', 'sms', 'push', 'in_app', 'slack', 'teams'
);

CREATE TYPE user_role AS ENUM (
    'super_admin', 'org_admin', 'location_manager', 'front_desk', 'member', 'guest'
);
```

---

## Core Identity and Organization Tables

```sql
-- Organizations represent the operator entity (may manage multiple locations)
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(50),
    billing_email   VARCHAR(255),
    phone           VARCHAR(50),
    website         VARCHAR(500),
    logo_url        VARCHAR(500),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    stripe_account_id VARCHAR(255),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_organizations_slug ON organizations(slug) WHERE deleted_at IS NULL;

-- Locations (physical co-working spaces)
CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    address_line1   VARCHAR(255) NOT NULL,
    address_line2   VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    timezone        VARCHAR(50) NOT NULL,
    phone           VARCHAR(50),
    email           VARCHAR(255),
    operating_hours JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"mon": {"open": "08:00", "close": "20:00"}, "sat": {"open": "09:00", "close": "17:00"}, "sun": null}
    total_area_sqm  DECIMAL(10, 2),
    max_capacity    INT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE(organization_id, slug)
);

CREATE INDEX idx_locations_org ON locations(organization_id) WHERE deleted_at IS NULL;

-- Users: unified table for all user types (members, staff, admins)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    display_name    VARCHAR(200),
    phone           VARCHAR(50),
    avatar_url      VARCHAR(500),
    bio             TEXT,
    company_name    VARCHAR(255),
    job_title       VARCHAR(150),
    oauth_provider  VARCHAR(50),
    oauth_uid       VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;

-- User-organization membership with role assignment
CREATE TABLE user_org_roles (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    role            user_role NOT NULL DEFAULT 'member',
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    granted_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at      TIMESTAMPTZ,
    UNIQUE(user_id, organization_id, role)
);

CREATE INDEX idx_user_org_roles_user ON user_org_roles(user_id) WHERE revoked_at IS NULL;
CREATE INDEX idx_user_org_roles_org ON user_org_roles(organization_id) WHERE revoked_at IS NULL;

-- Corporate accounts (companies that manage multiple member seats)
CREATE TABLE corporate_accounts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    company_name    VARCHAR(255) NOT NULL,
    billing_email   VARCHAR(255) NOT NULL,
    billing_address TEXT,
    tax_id          VARCHAR(50),
    primary_contact_id UUID REFERENCES users(id),
    stripe_customer_id VARCHAR(255),
    max_seats       INT,
    invoice_consolidation BOOLEAN NOT NULL DEFAULT true,
    payment_terms_days INT NOT NULL DEFAULT 30,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

-- Link members to corporate accounts
CREATE TABLE corporate_members (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    corporate_account_id UUID NOT NULL REFERENCES corporate_accounts(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    is_admin        BOOLEAN NOT NULL DEFAULT false,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    removed_at      TIMESTAMPTZ,
    UNIQUE(corporate_account_id, user_id)
);
```

---

## Membership and Plans

```sql
-- Membership plan definitions (templates for what a member gets)
CREATE TABLE membership_plans (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    plan_type       plan_type NOT NULL,
    billing_cycle   billing_cycle NOT NULL DEFAULT 'monthly',
    base_price      DECIMAL(10, 2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- Resource inclusions
    included_desk_hours  INT,          -- NULL = unlimited
    included_room_minutes INT,         -- NULL = unlimited
    included_guest_passes INT DEFAULT 0,
    included_print_pages  INT DEFAULT 0,
    -- Credit system
    included_credits INT DEFAULT 0,
    credit_rollover  BOOLEAN NOT NULL DEFAULT false,
    max_rollover_credits INT,
    -- Access rules
    access_24_7     BOOLEAN NOT NULL DEFAULT false,
    access_start_time TIME,            -- e.g. 08:00
    access_end_time   TIME,            -- e.g. 20:00
    access_days     INT[] DEFAULT '{1,2,3,4,5}',  -- ISO day-of-week (1=Mon)
    -- Location scope
    all_locations   BOOLEAN NOT NULL DEFAULT false,
    -- Community features
    community_post_allowed BOOLEAN NOT NULL DEFAULT true,
    event_creation_allowed BOOLEAN NOT NULL DEFAULT false,
    -- Stripe
    stripe_product_id VARCHAR(255),
    stripe_price_id   VARCHAR(255),
    -- Lifecycle
    is_publicly_listed BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    trial_days      INT DEFAULT 0,
    min_commitment_months INT DEFAULT 0,
    cancellation_notice_days INT DEFAULT 30,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_plans_org ON membership_plans(organization_id) WHERE deleted_at IS NULL;

-- Plans available at specific locations (when all_locations = false)
CREATE TABLE plan_locations (
    plan_id         UUID NOT NULL REFERENCES membership_plans(id),
    location_id     UUID NOT NULL REFERENCES locations(id),
    PRIMARY KEY (plan_id, location_id)
);

-- Active memberships (a member subscribed to a plan)
CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    plan_id         UUID NOT NULL REFERENCES membership_plans(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    location_id     UUID REFERENCES locations(id),  -- NULL if multi-location plan
    corporate_account_id UUID REFERENCES corporate_accounts(id),
    status          membership_status NOT NULL DEFAULT 'pending',
    start_date      DATE NOT NULL,
    end_date        DATE,               -- NULL = ongoing
    trial_end_date  DATE,
    current_period_start DATE,
    current_period_end   DATE,
    -- Billing
    price_override  DECIMAL(10, 2),     -- NULL = use plan price
    stripe_subscription_id VARCHAR(255),
    -- Credits
    credit_balance  INT NOT NULL DEFAULT 0,
    -- Assigned resource (for dedicated desks / private offices)
    assigned_resource_id UUID,           -- FK added after resources table
    -- Cancellation
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    cancellation_effective_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memberships_user ON memberships(user_id);
CREATE INDEX idx_memberships_org ON memberships(organization_id);
CREATE INDEX idx_memberships_status ON memberships(status) WHERE status = 'active';
CREATE INDEX idx_memberships_stripe ON memberships(stripe_subscription_id) WHERE stripe_subscription_id IS NOT NULL;

-- Membership status change log (audit trail)
CREATE TABLE membership_status_changes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    membership_id   UUID NOT NULL REFERENCES memberships(id),
    from_status     membership_status,
    to_status       membership_status NOT NULL,
    changed_by      UUID REFERENCES users(id),
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_membership_changes ON membership_status_changes(membership_id, created_at);
```

---

## Resources and Bookings

```sql
-- Floors within a location (for spatial organization)
CREATE TABLE floors (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    name            VARCHAR(100) NOT NULL,
    level           INT NOT NULL,
    floor_plan_url  VARCHAR(500),
    area_sqm        DECIMAL(10, 2),
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Bookable resources (desks, rooms, phone booths, etc.)
CREATE TABLE resources (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    floor_id        UUID REFERENCES floors(id),
    name            VARCHAR(255) NOT NULL,
    resource_type   resource_type NOT NULL,
    description     TEXT,
    capacity        INT NOT NULL DEFAULT 1,
    -- Pricing
    hourly_rate     DECIMAL(10, 2),
    half_day_rate   DECIMAL(10, 2),
    daily_rate      DECIMAL(10, 2),
    credit_cost     INT,                -- credits consumed per booking unit
    -- Physical attributes
    area_sqm        DECIMAL(8, 2),
    has_monitor     BOOLEAN DEFAULT false,
    has_whiteboard  BOOLEAN DEFAULT false,
    has_video_conf  BOOLEAN DEFAULT false,
    has_phone       BOOLEAN DEFAULT false,
    has_standing_desk BOOLEAN DEFAULT false,
    -- Availability
    is_bookable     BOOLEAN NOT NULL DEFAULT true,
    min_booking_minutes INT DEFAULT 30,
    max_booking_minutes INT DEFAULT 480,
    buffer_minutes  INT DEFAULT 0,      -- cleanup time between bookings
    advance_booking_days INT DEFAULT 30, -- how far ahead bookings are allowed
    -- Access control
    access_point_id VARCHAR(255),       -- external smart lock ID
    -- Display
    image_url       VARCHAR(500),
    color_hex       CHAR(7),
    sort_order      INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_resources_location ON resources(location_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_resources_type ON resources(location_id, resource_type) WHERE deleted_at IS NULL AND is_active = true;

-- Add FK for assigned resource on memberships
ALTER TABLE memberships
    ADD CONSTRAINT fk_memberships_assigned_resource
    FOREIGN KEY (assigned_resource_id) REFERENCES resources(id);

-- Resource amenities (many-to-many for extensible amenity tagging)
CREATE TABLE amenities (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,
    icon            VARCHAR(50),
    category        VARCHAR(50),
    UNIQUE(organization_id, name)
);

CREATE TABLE resource_amenities (
    resource_id     UUID NOT NULL REFERENCES resources(id),
    amenity_id      UUID NOT NULL REFERENCES amenities(id),
    PRIMARY KEY (resource_id, amenity_id)
);

-- Bookings (the core reservation entity)
CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    membership_id   UUID REFERENCES memberships(id),
    title           VARCHAR(255),
    status          booking_status NOT NULL DEFAULT 'pending',
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    -- Temporal range for exclusion constraint
    time_range      TSTZRANGE GENERATED ALWAYS AS (tstzrange(start_time, end_time)) STORED,
    -- Pricing at time of booking
    total_price     DECIMAL(10, 2),
    credits_used    INT DEFAULT 0,
    currency        CHAR(3) DEFAULT 'USD',
    -- Recurrence
    recurrence_rule VARCHAR(255),       -- RFC 5545 RRULE
    parent_booking_id UUID REFERENCES bookings(id),
    -- Check-in tracking
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    -- Cancellation
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    -- Notes
    notes           TEXT,
    internal_notes  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_booking_times CHECK (end_time > start_time)
);

-- Exclusion constraint to prevent double-bookings at the database level
ALTER TABLE bookings
    ADD CONSTRAINT no_overlapping_bookings
    EXCLUDE USING gist (
        resource_id WITH =,
        time_range WITH &&
    ) WHERE (status NOT IN ('cancelled', 'no_show'));

CREATE INDEX idx_bookings_resource_time ON bookings(resource_id, start_time, end_time) WHERE status NOT IN ('cancelled', 'no_show');
CREATE INDEX idx_bookings_user ON bookings(user_id, start_time);
CREATE INDEX idx_bookings_org ON bookings(organization_id, start_time);

-- Booking attendees (for meeting rooms with multiple participants)
CREATE TABLE booking_attendees (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    booking_id      UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    email           VARCHAR(255),       -- for external attendees
    name            VARCHAR(200),
    is_organizer    BOOLEAN NOT NULL DEFAULT false,
    rsvp_status     event_rsvp_status DEFAULT 'invited',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_booking_attendees ON booking_attendees(booking_id);

-- Day passes (walk-in or online purchase, not tied to a membership)
CREATE TABLE day_passes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID REFERENCES users(id),     -- NULL for anonymous walk-ins
    location_id     UUID NOT NULL REFERENCES locations(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    pass_date       DATE NOT NULL,
    price           DECIMAL(10, 2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    payment_id      UUID,               -- FK added after payments table
    purchased_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    guest_name      VARCHAR(200),       -- for anonymous purchasers
    guest_email     VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_day_passes_location_date ON day_passes(location_id, pass_date);
```

---

## Billing and Payments

```sql
-- Invoices
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    corporate_account_id UUID REFERENCES corporate_accounts(id),
    membership_id   UUID REFERENCES memberships(id),
    invoice_number  VARCHAR(50) UNIQUE NOT NULL,
    status          invoice_status NOT NULL DEFAULT 'draft',
    -- Dates
    issue_date      DATE NOT NULL,
    due_date        DATE NOT NULL,
    paid_date       DATE,
    -- Amounts
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_rate        DECIMAL(5, 4) DEFAULT 0,
    discount_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_due      DECIMAL(12, 2) GENERATED ALWAYS AS (total - amount_paid) STORED,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- Stripe
    stripe_invoice_id VARCHAR(255),
    stripe_payment_intent_id VARCHAR(255),
    -- Metadata
    notes           TEXT,
    billing_period_start DATE,
    billing_period_end   DATE,
    -- Dunning
    dunning_attempts INT NOT NULL DEFAULT 0,
    last_dunning_at  TIMESTAMPTZ,
    next_dunning_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoices_user ON invoices(user_id, issue_date);
CREATE INDEX idx_invoices_corp ON invoices(corporate_account_id) WHERE corporate_account_id IS NOT NULL;
CREATE INDEX idx_invoices_status ON invoices(status) WHERE status IN ('open', 'overdue');
CREATE INDEX idx_invoices_stripe ON invoices(stripe_invoice_id) WHERE stripe_invoice_id IS NOT NULL;

-- Invoice line items
CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_price      DECIMAL(10, 2) NOT NULL,
    amount          DECIMAL(12, 2) NOT NULL,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    -- Link to source (one of these)
    membership_id   UUID REFERENCES memberships(id),
    booking_id      UUID REFERENCES bookings(id),
    day_pass_id     UUID REFERENCES day_passes(id),
    -- Billing period
    period_start    DATE,
    period_end      DATE,
    -- Proration
    is_prorated     BOOLEAN NOT NULL DEFAULT false,
    proration_factor DECIMAL(6, 5),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

-- Payments received
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_id      UUID REFERENCES invoices(id),
    user_id         UUID REFERENCES users(id),
    amount          DECIMAL(12, 2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          payment_status NOT NULL DEFAULT 'pending',
    payment_method  payment_method_type,
    -- Stripe
    stripe_payment_id VARCHAR(255),
    stripe_charge_id  VARCHAR(255),
    stripe_refund_id  VARCHAR(255),
    -- Details
    description     TEXT,
    failure_reason  TEXT,
    refund_amount   DECIMAL(12, 2) DEFAULT 0,
    refund_reason   TEXT,
    -- Metadata
    payment_date    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_user ON payments(user_id, payment_date);
CREATE INDEX idx_payments_stripe ON payments(stripe_payment_id) WHERE stripe_payment_id IS NOT NULL;

-- Add FK for day_pass payment
ALTER TABLE day_passes
    ADD CONSTRAINT fk_day_passes_payment
    FOREIGN KEY (payment_id) REFERENCES payments(id);

-- Credit transactions (track credit balance changes)
CREATE TABLE credit_transactions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    membership_id   UUID NOT NULL REFERENCES memberships(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    amount          INT NOT NULL,       -- positive = credit, negative = debit
    balance_after   INT NOT NULL,
    description     VARCHAR(500) NOT NULL,
    -- Source
    booking_id      UUID REFERENCES bookings(id),
    invoice_id      UUID REFERENCES invoices(id),
    -- Expiry
    expires_at      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_credit_tx_membership ON credit_transactions(membership_id, created_at);

-- Discount codes / coupons
CREATE TABLE discount_codes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    code            VARCHAR(50) NOT NULL,
    description     TEXT,
    discount_type   VARCHAR(20) NOT NULL CHECK (discount_type IN ('percentage', 'fixed_amount')),
    discount_value  DECIMAL(10, 2) NOT NULL,
    currency        CHAR(3) DEFAULT 'USD',
    max_uses        INT,
    current_uses    INT NOT NULL DEFAULT 0,
    valid_from      TIMESTAMPTZ,
    valid_until     TIMESTAMPTZ,
    applicable_plan_ids UUID[],         -- empty = all plans
    stripe_coupon_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organization_id, code)
);
```

---

## Access Control

```sql
-- Access points (doors, gates, turnstiles)
CREATE TABLE access_points (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    floor_id        UUID REFERENCES floors(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- Hardware integration
    provider        VARCHAR(50) NOT NULL,  -- 'kisi', 'salto', 'brivo', 'hakuna'
    external_id     VARCHAR(255) NOT NULL, -- ID in the provider's system
    provider_config JSONB DEFAULT '{}',
    -- Access rules
    direction       VARCHAR(10) DEFAULT 'both' CHECK (direction IN ('entry', 'exit', 'both')),
    requires_booking BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_access_points_location ON access_points(location_id);

-- Link access points to resources (e.g., a meeting room door)
CREATE TABLE resource_access_points (
    resource_id     UUID NOT NULL REFERENCES resources(id),
    access_point_id UUID NOT NULL REFERENCES access_points(id),
    PRIMARY KEY (resource_id, access_point_id)
);

-- Member access credentials
CREATE TABLE access_credentials (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    credential_type access_credential_type NOT NULL,
    credential_value VARCHAR(500) NOT NULL,  -- encrypted for sensitive types
    label           VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    revoked_reason  TEXT
);

CREATE INDEX idx_credentials_user ON access_credentials(user_id) WHERE is_active = true;

-- Access permissions (what a user can access, derived from membership/booking)
CREATE TABLE access_permissions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    access_point_id UUID NOT NULL REFERENCES access_points(id),
    -- Source of permission
    membership_id   UUID REFERENCES memberships(id),
    booking_id      UUID REFERENCES bookings(id),
    visitor_pass_id UUID,               -- FK added after visitors table
    -- Time window
    valid_from      TIMESTAMPTZ NOT NULL,
    valid_until     TIMESTAMPTZ,
    -- Schedule restrictions
    allowed_days    INT[],
    allowed_start_time TIME,
    allowed_end_time   TIME,
    -- Status
    is_active       BOOLEAN NOT NULL DEFAULT true,
    synced_to_provider BOOLEAN NOT NULL DEFAULT false,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_access_perms_user ON access_permissions(user_id) WHERE is_active = true;
CREATE INDEX idx_access_perms_point ON access_permissions(access_point_id) WHERE is_active = true;

-- Access event log (audit trail of all access events)
CREATE TABLE access_events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    access_point_id UUID NOT NULL REFERENCES access_points(id),
    user_id         UUID REFERENCES users(id),
    credential_id   UUID REFERENCES access_credentials(id),
    event_type      access_event_type NOT NULL,
    success         BOOLEAN NOT NULL,
    failure_reason  VARCHAR(255),
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Provider data
    provider_event_id VARCHAR(255),
    raw_event       JSONB
);

-- Partition access events by month for performance
CREATE INDEX idx_access_events_point ON access_events(access_point_id, occurred_at);
CREATE INDEX idx_access_events_user ON access_events(user_id, occurred_at);
```

---

## Community and Events

```sql
-- Community posts (social feed)
CREATE TABLE community_posts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    location_id     UUID REFERENCES locations(id),  -- NULL = org-wide
    author_id       UUID NOT NULL REFERENCES users(id),
    post_type       post_type NOT NULL DEFAULT 'discussion',
    title           VARCHAR(500),
    body            TEXT NOT NULL,
    -- Media attachments stored as array of URLs
    media_urls      VARCHAR(500)[],
    -- Moderation
    moderation_status moderation_status NOT NULL DEFAULT 'auto_approved',
    moderated_by    UUID REFERENCES users(id),
    moderated_at    TIMESTAMPTZ,
    moderation_note TEXT,
    -- Engagement counters (denormalized for read performance)
    like_count      INT NOT NULL DEFAULT 0,
    comment_count   INT NOT NULL DEFAULT 0,
    -- Visibility
    is_pinned       BOOLEAN NOT NULL DEFAULT false,
    is_visible      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_posts_org ON community_posts(organization_id, created_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_posts_author ON community_posts(author_id) WHERE deleted_at IS NULL;

-- Post likes
CREATE TABLE post_likes (
    post_id         UUID NOT NULL REFERENCES community_posts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (post_id, user_id)
);

-- Post comments
CREATE TABLE post_comments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id         UUID NOT NULL REFERENCES community_posts(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    parent_comment_id UUID REFERENCES post_comments(id),  -- threading
    body            TEXT NOT NULL,
    moderation_status moderation_status NOT NULL DEFAULT 'auto_approved',
    like_count      INT NOT NULL DEFAULT 0,
    is_visible      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_comments_post ON post_comments(post_id, created_at) WHERE deleted_at IS NULL;

-- Direct messages between members
CREATE TABLE direct_message_threads (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE direct_message_participants (
    thread_id       UUID NOT NULL REFERENCES direct_message_threads(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    last_read_at    TIMESTAMPTZ,
    is_muted        BOOLEAN NOT NULL DEFAULT false,
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    left_at         TIMESTAMPTZ,
    PRIMARY KEY (thread_id, user_id)
);

CREATE TABLE direct_messages (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    thread_id       UUID NOT NULL REFERENCES direct_message_threads(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    media_urls      VARCHAR(500)[],
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_dm_thread ON direct_messages(thread_id, created_at);

-- Events
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    location_id     UUID REFERENCES locations(id),
    organizer_id    UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    -- Timing
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    timezone        VARCHAR(50) NOT NULL,
    is_all_day      BOOLEAN NOT NULL DEFAULT false,
    -- Location
    venue_name      VARCHAR(255),
    booking_id      UUID REFERENCES bookings(id),  -- linked room booking
    is_virtual      BOOLEAN NOT NULL DEFAULT false,
    virtual_meeting_url VARCHAR(500),
    -- Capacity
    max_attendees   INT,
    -- Ticketing
    is_free         BOOLEAN NOT NULL DEFAULT true,
    ticket_price    DECIMAL(10, 2),
    currency        CHAR(3) DEFAULT 'USD',
    -- Visibility
    is_public       BOOLEAN NOT NULL DEFAULT true,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    cover_image_url VARCHAR(500),
    -- Engagement counters
    rsvp_count      INT NOT NULL DEFAULT 0,
    attendee_count  INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_events_org ON events(organization_id, start_time) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_location ON events(location_id, start_time) WHERE deleted_at IS NULL;

-- Event RSVPs
CREATE TABLE event_rsvps (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    status          event_rsvp_status NOT NULL DEFAULT 'attending',
    ticket_count    INT NOT NULL DEFAULT 1,
    payment_id      UUID REFERENCES payments(id),
    checked_in_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(event_id, user_id)
);

CREATE INDEX idx_event_rsvps ON event_rsvps(event_id);

-- Member directory profiles (opt-in public profile for networking)
CREATE TABLE member_profiles (
    user_id         UUID PRIMARY KEY REFERENCES users(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    headline        VARCHAR(255),
    skills          VARCHAR(100)[],
    interests       VARCHAR(100)[],
    linkedin_url    VARCHAR(500),
    twitter_handle  VARCHAR(100),
    website_url     VARCHAR(500),
    is_visible      BOOLEAN NOT NULL DEFAULT true,
    open_to_networking BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Visitors and Guest Management

```sql
-- Visitor passes
CREATE TABLE visitor_passes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    host_id         UUID NOT NULL REFERENCES users(id),
    -- Visitor details
    visitor_name    VARCHAR(200) NOT NULL,
    visitor_email   VARCHAR(255),
    visitor_phone   VARCHAR(50),
    visitor_company VARCHAR(255),
    -- Visit details
    purpose         VARCHAR(500),
    expected_arrival TIMESTAMPTZ NOT NULL,
    expected_departure TIMESTAMPTZ,
    -- Status
    status          visitor_status NOT NULL DEFAULT 'pre_registered',
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    -- Access
    temporary_credential VARCHAR(255),   -- PIN or QR code
    access_point_ids UUID[],
    -- Notifications
    host_notified   BOOLEAN NOT NULL DEFAULT false,
    host_notified_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_visitor_passes_location ON visitor_passes(location_id, expected_arrival);
CREATE INDEX idx_visitor_passes_host ON visitor_passes(host_id);

-- Add FK for visitor pass on access_permissions
ALTER TABLE access_permissions
    ADD CONSTRAINT fk_access_perms_visitor
    FOREIGN KEY (visitor_pass_id) REFERENCES visitor_passes(id);
```

---

## Notifications

```sql
CREATE TABLE notification_preferences (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    channel         notification_channel NOT NULL,
    -- Categories
    booking_confirmations BOOLEAN NOT NULL DEFAULT true,
    booking_reminders     BOOLEAN NOT NULL DEFAULT true,
    billing_alerts        BOOLEAN NOT NULL DEFAULT true,
    community_mentions    BOOLEAN NOT NULL DEFAULT true,
    community_digest      BOOLEAN NOT NULL DEFAULT false,
    event_announcements   BOOLEAN NOT NULL DEFAULT true,
    visitor_arrivals      BOOLEAN NOT NULL DEFAULT true,
    system_updates        BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(user_id, channel)
);

CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    channel         notification_channel NOT NULL,
    category        VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    action_url      VARCHAR(500),
    is_read         BOOLEAN NOT NULL DEFAULT false,
    read_at         TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    delivery_status VARCHAR(20) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, created_at DESC);
CREATE INDEX idx_notifications_unread ON notifications(user_id) WHERE is_read = false;
```

---

## White-Label and Branding

```sql
CREATE TABLE branding_configs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    -- Domain
    custom_domain   VARCHAR(255),
    -- Colors
    primary_color   VARCHAR(7),
    secondary_color VARCHAR(7),
    accent_color    VARCHAR(7),
    -- Assets
    logo_url        VARCHAR(500),
    favicon_url     VARCHAR(500),
    login_bg_url    VARCHAR(500),
    -- Email templates
    email_from_name VARCHAR(100),
    email_from_address VARCHAR(255),
    email_header_html TEXT,
    email_footer_html TEXT,
    -- Mobile app
    app_name        VARCHAR(100),
    ios_bundle_id   VARCHAR(255),
    android_package VARCHAR(255),
    -- Custom CSS override
    custom_css      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organization_id)
);
```

---

## Analytics Materialized Views

```sql
-- Occupancy summary (refreshed periodically)
CREATE MATERIALIZED VIEW mv_daily_occupancy AS
SELECT
    r.location_id,
    r.resource_type,
    DATE(b.start_time AT TIME ZONE l.timezone) AS booking_date,
    COUNT(DISTINCT b.id) AS total_bookings,
    COUNT(DISTINCT b.user_id) AS unique_users,
    SUM(EXTRACT(EPOCH FROM (b.end_time - b.start_time)) / 3600) AS total_hours_booked,
    COUNT(DISTINCT r.id) AS resources_used,
    (SELECT COUNT(*) FROM resources r2
     WHERE r2.location_id = r.location_id
     AND r2.resource_type = r.resource_type
     AND r2.is_active = true
     AND r2.deleted_at IS NULL) AS total_resources
FROM bookings b
JOIN resources r ON b.resource_id = r.id
JOIN locations l ON r.location_id = l.id
WHERE b.status NOT IN ('cancelled', 'no_show')
GROUP BY r.location_id, r.resource_type, DATE(b.start_time AT TIME ZONE l.timezone), l.timezone;

CREATE UNIQUE INDEX idx_mv_daily_occupancy ON mv_daily_occupancy(location_id, resource_type, booking_date);

-- Revenue summary
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT
    i.organization_id,
    DATE_TRUNC('month', i.issue_date) AS revenue_month,
    i.currency,
    COUNT(*) AS invoice_count,
    SUM(i.total) AS total_invoiced,
    SUM(i.amount_paid) AS total_collected,
    SUM(CASE WHEN i.status = 'overdue' THEN i.total - i.amount_paid ELSE 0 END) AS outstanding_amount,
    COUNT(CASE WHEN i.status = 'paid' THEN 1 END) AS paid_count,
    COUNT(CASE WHEN i.status = 'overdue' THEN 1 END) AS overdue_count
FROM invoices i
GROUP BY i.organization_id, DATE_TRUNC('month', i.issue_date), i.currency;

CREATE UNIQUE INDEX idx_mv_monthly_revenue ON mv_monthly_revenue(organization_id, revenue_month, currency);
```

---

## Pros and Cons

### Pros

1. **Database-level booking integrity**: The GiST exclusion constraint (`no_overlapping_bookings`) makes double-bookings structurally impossible at the storage layer, regardless of application bugs or race conditions. This is the single strongest technical advantage of PostgreSQL for this domain.

2. **Referential integrity everywhere**: Every relationship is enforced by foreign keys. A membership cannot reference a nonexistent plan; a booking cannot reference a deleted resource. This eliminates entire classes of data corruption bugs that plague document-store or event-sourced systems.

3. **Mature billing model**: The invoices/line-items/payments structure mirrors Stripe's own data model, making webhook reconciliation straightforward. Pro-rated charges, credit tracking, and corporate invoice consolidation are all first-class entities.

4. **Multi-tenant by design**: The organization/location hierarchy supports independent operators, multi-site chains, and enterprise networks from day one. Role-based access control is normalized into `user_org_roles`.

5. **Audit trails built in**: Membership status changes, access events, credit transactions, and payment history provide comprehensive audit capabilities required for financial compliance and access control accountability.

6. **Excellent tooling ecosystem**: PostgreSQL has first-class support in every major ORM (Prisma, Drizzle, SQLAlchemy, TypeORM, ActiveRecord), migration tool (Flyway, Liquibase, dbmate), and monitoring solution (pganalyze, Datadog).

7. **Standard SQL**: No vendor lock-in. The schema can be adapted to CockroachDB, YugabyteDB, or Aurora PostgreSQL for horizontal scaling without fundamental redesign.

### Cons

1. **Schema rigidity**: Adding a new resource attribute (e.g., "has_standing_desk") requires a migration. For a domain where operators may want custom fields on resources, members, or plans, this creates friction. Every custom field is either a migration or a JSONB escape hatch.

2. **Multi-location query complexity**: Cross-location analytics and reporting queries must join through the organization-location-resource hierarchy, which becomes verbose and potentially slow for operators with dozens of locations.

3. **Scaling write-heavy access events**: The `access_events` table will grow rapidly (thousands of events per location per day). Without partitioning, query performance degrades. Table partitioning by month is recommended but adds operational complexity.

4. **Denormalization temptation**: Counters like `like_count` and `comment_count` on community posts are denormalized for read performance but create update anomaly risks. Triggers or application-level consistency are required.

5. **Schema migration coordination**: In a multi-tenant SaaS deployment, schema migrations must be applied across all tenants simultaneously. This requires careful blue-green deployment strategies.

6. **No built-in time-series optimization**: Occupancy sensor data, real-time check-in streams, and IoT data do not fit well into standard relational tables. A companion time-series solution (TimescaleDB) would be needed for advanced occupancy analytics.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| Database | PostgreSQL 16+ with btree_gist extension |
| Migrations | Flyway or dbmate for version-controlled schema changes |
| ORM | Prisma (TypeScript) or SQLAlchemy (Python) |
| Connection pooling | PgBouncer in transaction mode |
| Read replicas | PostgreSQL streaming replication for analytics queries |
| Full-text search | PostgreSQL tsvector for community post search; Elasticsearch if scale demands |
| Caching | Redis for session data, booking availability snapshots, and rate limiting |
| Background jobs | pg_cron for materialized view refresh; Temporal or BullMQ for billing workflows |

---

## Migration and Scaling Considerations

### Phase 1: Single-instance PostgreSQL (0-50 locations)
- Single primary with one read replica
- Materialized views refreshed every 15 minutes via pg_cron
- Access events table partitioned by month from day one
- PgBouncer for connection pooling (target: 200 concurrent connections)

### Phase 2: Vertical scaling (50-500 locations)
- Move to managed PostgreSQL (RDS, Cloud SQL, or Supabase) with automated backups
- Add dedicated read replica for analytics/reporting workloads
- Implement row-level security (RLS) policies for multi-tenant isolation
- Partition bookings table by organization_id + month for large operators

### Phase 3: Horizontal scaling (500+ locations)
- Consider Citus extension for distributed PostgreSQL (shard by organization_id)
- Extract access events into TimescaleDB hypertable for IoT-scale sensor data
- Move community search to dedicated Elasticsearch cluster
- Implement change data capture (CDC) via Debezium for real-time analytics pipelines

### Data retention strategy
- Active data: indefinite retention for memberships, invoices, payments
- Access events: 2 years hot, archive to S3/cold storage after
- Community posts: indefinite (soft delete)
- Notifications: 90 days active, archive after
- Booking history: indefinite (required for utilization analytics)
