# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Co-Working Space Management (Candidate #470)
> Generated: 2026-05-26

## Overview

This model takes a pragmatic hybrid approach: normalized relational tables for data with stable, well-understood schemas (users, locations, invoices, payments) combined with JSONB columns for data that varies by operator, evolves rapidly, or has a naturally hierarchical structure (plan configurations, resource attributes, operating hours, custom fields, integration configs, branding settings).

The key insight driving this design is that co-working space management straddles two worlds. The financial and access-control domains demand strict relational integrity -- you cannot afford ambiguity in who paid what, or who has access to which door. But the space configuration and community domains are highly variable -- one operator's "phone booth" is another's "focus pod," custom amenity lists differ wildly, and every smart-lock provider sends different event payloads. A hybrid model lets each domain use the storage pattern that fits best.

PostgreSQL's JSONB type is not a compromise -- it is a first-class data type with GIN indexing, containment operators (`@>`), path queries (`->`, `->>`), and partial indexing. When used with discipline (indexed paths for frequent queries, JSON Schema validation in the application layer), JSONB columns perform comparably to normalized columns for read workloads and dramatically reduce schema migration overhead for evolving features.

---

## Design Principles

1. **Normalize financial data**: Invoices, payments, line items, and credit transactions use fully normalized tables with strict constraints. Money is never stored in JSONB.
2. **Normalize identity and relationships**: Users, organizations, locations, and their relationships are relational. Foreign keys enforce integrity.
3. **JSONB for operator-configurable schemas**: Plan configurations, resource attributes, operating hours, custom fields, and branding use JSONB. Operators can extend these without database migrations.
4. **JSONB for integration payloads**: Smart-lock events, Stripe webhooks, calendar sync data, and notification delivery receipts are stored as JSONB for flexibility across provider variations.
5. **JSONB for community content metadata**: Post attachments, event details, and visitor information use JSONB for extensibility.
6. **GIN indexes on frequently queried JSONB paths**: Not all JSONB is created equal. Hot paths get dedicated indexes.
7. **Application-layer JSON Schema validation**: The database stores JSONB flexibly; the application validates structure before writes.

---

## Schema Definition

### Extensions and Utility Types

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "btree_gist";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Enums for stable domain concepts
CREATE TYPE membership_status AS ENUM ('pending', 'active', 'suspended', 'cancelled', 'expired');
CREATE TYPE booking_status AS ENUM ('pending', 'confirmed', 'checked_in', 'completed', 'cancelled', 'no_show');
CREATE TYPE invoice_status AS ENUM ('draft', 'open', 'paid', 'partially_paid', 'overdue', 'void', 'uncollectible');
CREATE TYPE payment_status AS ENUM ('pending', 'processing', 'succeeded', 'failed', 'refunded', 'partially_refunded');
```

### Organizations and Locations

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    -- Stable fields (always present, frequently queried)
    billing_email   VARCHAR(255),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    stripe_account_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Flexible fields (vary by operator, evolve over time)
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings schema:
    -- {
    --   "legal_name": "string",
    --   "tax_id": "string",
    --   "phone": "string",
    --   "website": "string",
    --   "billing": {
    --     "payment_terms_days": 30,
    --     "auto_charge": true,
    --     "dunning_enabled": true,
    --     "dunning_schedule": [3, 7, 14],
    --     "tax_rate": 0.08,
    --     "invoice_prefix": "INV",
    --     "invoice_footer": "Thank you for being a member!"
    --   },
    --   "notifications": {
    --     "booking_reminder_hours": 1,
    --     "welcome_email_enabled": true,
    --     "digest_frequency": "weekly"
    --   },
    --   "community": {
    --     "moderation_mode": "auto_approve",
    --     "allow_member_posts": true,
    --     "allow_direct_messages": true
    --   },
    --   "custom_fields": {
    --     "members": [
    --       {"key": "dietary_preference", "label": "Dietary Preference", "type": "select", "options": ["none", "vegetarian", "vegan", "halal"]},
    --       {"key": "tshirt_size", "label": "T-Shirt Size", "type": "select", "options": ["S", "M", "L", "XL"]}
    --     ],
    --     "resources": [
    --       {"key": "noise_level", "label": "Noise Level", "type": "select", "options": ["silent", "quiet", "moderate"]}
    --     ]
    --   }
    -- }
    branding        JSONB NOT NULL DEFAULT '{}',
    -- branding schema:
    -- {
    --   "custom_domain": "members.acmespace.com",
    --   "primary_color": "#2563eb",
    --   "secondary_color": "#1e40af",
    --   "accent_color": "#f59e0b",
    --   "logo_url": "https://cdn.example.com/logo.png",
    --   "favicon_url": "https://cdn.example.com/favicon.ico",
    --   "login_bg_url": "https://cdn.example.com/bg.jpg",
    --   "email_from_name": "Acme Coworking",
    --   "email_from_address": "hello@acmespace.com",
    --   "email_header_html": "<div>...</div>",
    --   "email_footer_html": "<div>...</div>",
    --   "app_name": "Acme Space",
    --   "custom_css": ".header { ... }"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

-- GIN index on settings for containment queries
CREATE INDEX idx_org_settings ON organizations USING gin(settings);

CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    -- Stable address fields (frequently queried, filtered)
    city            VARCHAR(100) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Flexible location details (varies by location, includes operating hours)
    details         JSONB NOT NULL DEFAULT '{}',
    -- details schema:
    -- {
    --   "address": {
    --     "line1": "123 Main St",
    --     "line2": "Suite 400",
    --     "state_province": "CA",
    --     "postal_code": "94105"
    --   },
    --   "coordinates": { "lat": 37.7749, "lng": -122.4194 },
    --   "phone": "+1-555-0100",
    --   "email": "sf@acmespace.com",
    --   "operating_hours": {
    --     "mon": { "open": "07:00", "close": "22:00" },
    --     "tue": { "open": "07:00", "close": "22:00" },
    --     "wed": { "open": "07:00", "close": "22:00" },
    --     "thu": { "open": "07:00", "close": "22:00" },
    --     "fri": { "open": "07:00", "close": "20:00" },
    --     "sat": { "open": "09:00", "close": "17:00" },
    --     "sun": null
    --   },
    --   "holiday_closures": [
    --     { "date": "2026-12-25", "name": "Christmas Day" },
    --     { "date": "2026-01-01", "name": "New Year's Day" }
    --   ],
    --   "total_area_sqm": 850.5,
    --   "max_capacity": 120,
    --   "floor_count": 3,
    --   "amenities": ["wifi", "printing", "kitchen", "showers", "bike_storage", "ev_charging"],
    --   "photos": ["https://cdn.example.com/sf-1.jpg", "https://cdn.example.com/sf-2.jpg"],
    --   "description": "A modern workspace in the heart of SOMA...",
    --   "parking_info": "Street metered parking available on 2nd St."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE(organization_id, slug)
);

CREATE INDEX idx_locations_org ON locations(organization_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_locations_city ON locations(city, country_code) WHERE is_active = true;
CREATE INDEX idx_locations_details ON locations USING gin(details);
```

### Users and Roles

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    -- Stable identity fields (queried constantly)
    email           VARCHAR(255) UNIQUE NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    password_hash   VARCHAR(255),
    -- Flexible profile data (varies by member, operator can add custom fields)
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile schema:
    -- {
    --   "display_name": "Jane Doe",
    --   "avatar_url": "https://cdn.example.com/avatar/jane.jpg",
    --   "bio": "Product designer and coffee enthusiast",
    --   "company_name": "Acme Corp",
    --   "job_title": "Senior Designer",
    --   "linkedin_url": "https://linkedin.com/in/janedoe",
    --   "twitter_handle": "@janedoe",
    --   "website": "https://janedoe.design",
    --   "skills": ["UX Design", "Figma", "Prototyping"],
    --   "interests": ["Design", "AI", "Sustainability"],
    --   "open_to_networking": true,
    --   "dietary_preference": "vegetarian",    <-- operator custom field
    --   "tshirt_size": "M"                    <-- operator custom field
    -- }
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences schema:
    -- {
    --   "notifications": {
    --     "email": { "bookings": true, "billing": true, "community": true, "events": true },
    --     "push": { "bookings": true, "billing": false, "community": true, "events": true },
    --     "sms": { "bookings": false, "billing": true, "community": false, "events": false }
    --   },
    --   "locale": "en-US",
    --   "timezone_override": null,
    --   "preferred_location_id": "uuid...",
    --   "preferred_desk_zone": "quiet_area",
    --   "calendar_sync": {
    --     "google": { "enabled": true, "calendar_id": "primary" },
    --     "outlook": { "enabled": false }
    --   }
    -- }
    -- Auth metadata
    oauth_provider  VARCHAR(50),
    oauth_uid       VARCHAR(255),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_profile ON users USING gin(profile);
-- Partial GIN index for networking searches
CREATE INDEX idx_users_skills ON users USING gin((profile->'skills')) WHERE is_active = true;

CREATE TABLE user_org_roles (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    -- CHECK (role IN ('super_admin','org_admin','location_manager','front_desk','member','guest'))
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions schema (overrides for fine-grained control):
    -- {
    --   "can_manage_members": true,
    --   "can_manage_billing": false,
    --   "can_manage_bookings": true,
    --   "can_moderate_community": true,
    --   "can_manage_access_control": false,
    --   "location_ids": ["uuid1", "uuid2"]  -- null = all locations
    -- }
    granted_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at      TIMESTAMPTZ,
    UNIQUE(user_id, organization_id, role)
);

CREATE INDEX idx_user_roles ON user_org_roles(user_id) WHERE revoked_at IS NULL;

-- Corporate accounts with flexible billing configuration
CREATE TABLE corporate_accounts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    company_name    VARCHAR(255) NOT NULL,
    billing_email   VARCHAR(255) NOT NULL,
    primary_contact_id UUID REFERENCES users(id),
    stripe_customer_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Flexible billing and company details
    config          JSONB NOT NULL DEFAULT '{}',
    -- config schema:
    -- {
    --   "billing_address": { "line1": "...", "city": "...", ... },
    --   "tax_id": "US12345678",
    --   "payment_terms_days": 30,
    --   "invoice_consolidation": true,
    --   "max_seats": 25,
    --   "department_codes": ["engineering", "design", "sales"],
    --   "purchase_order_required": true,
    --   "approved_plan_types": ["dedicated_desk", "private_office"],
    --   "budget_limit_monthly": 15000.00
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE corporate_members (
    corporate_account_id UUID NOT NULL REFERENCES corporate_accounts(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    is_admin        BOOLEAN NOT NULL DEFAULT false,
    department      VARCHAR(100),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    removed_at      TIMESTAMPTZ,
    PRIMARY KEY (corporate_account_id, user_id)
);
```

### Membership Plans (JSONB-Heavy for Operator Customization)

```sql
CREATE TABLE membership_plans (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    -- Stable, queryable fields
    name            VARCHAR(255) NOT NULL,
    plan_type       VARCHAR(50) NOT NULL,
    -- CHECK (plan_type IN ('hot_desk','dedicated_desk','private_office','virtual_office','day_pass','credit_pack','custom'))
    billing_cycle   VARCHAR(20) NOT NULL DEFAULT 'monthly',
    base_price      DECIMAL(10, 2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_publicly_listed BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    -- Stripe linkage
    stripe_product_id VARCHAR(255),
    stripe_price_id   VARCHAR(255),
    -- Flexible plan configuration (highly variable between operators and plan types)
    config          JSONB NOT NULL DEFAULT '{}',
    -- config schema:
    -- {
    --   "inclusions": {
    --     "desk_hours_per_month": null,        // null = unlimited
    --     "meeting_room_minutes_per_month": 120,
    --     "guest_passes_per_month": 3,
    --     "print_pages_per_month": 100,
    --     "credits_per_month": 10,
    --     "mail_handling": true,
    --     "registered_business_address": false,
    --     "dedicated_phone_line": false,
    --     "storage_locker": false
    --   },
    --   "credits": {
    --     "rollover": true,
    --     "max_rollover": 20,
    --     "expiry_months": 3
    --   },
    --   "access": {
    --     "twenty_four_seven": false,
    --     "start_time": "08:00",
    --     "end_time": "20:00",
    --     "allowed_days": [1, 2, 3, 4, 5],
    --     "all_locations": false,
    --     "allowed_location_ids": ["uuid1"]
    --   },
    --   "booking_rules": {
    --     "advance_booking_days": 14,
    --     "max_concurrent_bookings": 3,
    --     "min_booking_minutes": 30,
    --     "max_booking_minutes": 480,
    --     "auto_cancel_no_show_minutes": 15
    --   },
    --   "community": {
    --     "can_post": true,
    --     "can_create_events": false,
    --     "visible_in_directory": true
    --   },
    --   "commitment": {
    --     "trial_days": 14,
    --     "min_months": 3,
    --     "cancellation_notice_days": 30
    --   },
    --   "overage_rates": {
    --     "desk_hour": 5.00,
    --     "meeting_room_minute": 0.50,
    --     "guest_pass": 15.00,
    --     "print_page_bw": 0.05,
    --     "print_page_color": 0.15
    --   },
    --   "description_long": "Our Premium plan gives you...",
    --   "features_list": ["24/7 access", "Free meeting rooms", "Mail handling"],
    --   "badge_color": "#2563eb",
    --   "icon": "star"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_plans_org ON membership_plans(organization_id, is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_plans_type ON membership_plans(organization_id, plan_type) WHERE is_active = true;
-- GIN index for querying plan configurations
CREATE INDEX idx_plans_config ON membership_plans USING gin(config);
```

### Memberships (Relational with JSONB Metadata)

```sql
CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    plan_id         UUID NOT NULL REFERENCES membership_plans(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    location_id     UUID REFERENCES locations(id),
    corporate_account_id UUID REFERENCES corporate_accounts(id),
    -- Stable lifecycle fields (frequently queried, status-driven logic)
    status          membership_status NOT NULL DEFAULT 'pending',
    start_date      DATE NOT NULL,
    end_date        DATE,
    trial_end_date  DATE,
    current_period_start DATE,
    current_period_end   DATE,
    -- Financial (must be relational for integrity)
    price_override  DECIMAL(10, 2),
    credit_balance  INT NOT NULL DEFAULT 0,
    stripe_subscription_id VARCHAR(255),
    -- Flexible membership metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata schema:
    -- {
    --   "assigned_resource_id": "uuid...",
    --   "assigned_resource_name": "Desk 42",
    --   "cancellation": {
    --     "requested_at": "2026-07-15T10:00:00Z",
    --     "effective_date": "2026-08-01",
    --     "reason": "relocating",
    --     "feedback": "Great space, just moving cities"
    --   },
    --   "plan_snapshot": {
    --     "plan_name": "Dedicated Desk",
    --     "base_price": 450.00,
    --     "inclusions": { ... }
    --   },
    --   "onboarding": {
    --     "tour_completed": true,
    --     "wifi_setup": true,
    --     "access_card_issued": true,
    --     "welcome_email_sent": "2026-06-01T09:00:00Z"
    --   },
    --   "custom_fields": {
    --     "department_code": "engineering",
    --     "cost_center": "CC-1234"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memberships_user ON memberships(user_id);
CREATE INDEX idx_memberships_org_status ON memberships(organization_id, status);
CREATE INDEX idx_memberships_active ON memberships(organization_id) WHERE status = 'active';
CREATE INDEX idx_memberships_stripe ON memberships(stripe_subscription_id) WHERE stripe_subscription_id IS NOT NULL;
```

### Resources (JSONB Attributes for Operator Flexibility)

```sql
CREATE TABLE resources (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    -- Stable fields (frequently queried, filtered, sorted)
    name            VARCHAR(255) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    capacity        INT NOT NULL DEFAULT 1,
    is_bookable     BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Pricing (relational for financial integrity)
    hourly_rate     DECIMAL(10, 2),
    half_day_rate   DECIMAL(10, 2),
    daily_rate      DECIMAL(10, 2),
    credit_cost     INT,
    -- Flexible attributes (vary wildly between resource types and operators)
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes schema (varies by resource_type):
    --
    -- For desks:
    -- {
    --   "floor": "3rd Floor",
    --   "zone": "quiet_area",
    --   "position": { "x": 150, "y": 320 },  -- for floor plan rendering
    --   "has_monitor": true,
    --   "monitor_size": "27 inch",
    --   "has_standing_option": true,
    --   "has_ergonomic_chair": true,
    --   "power_outlets": 2,
    --   "natural_light": true,
    --   "noise_level": "quiet",
    --   "near_kitchen": false,
    --   "photos": ["https://cdn.example.com/desk42.jpg"]
    -- }
    --
    -- For meeting rooms:
    -- {
    --   "floor": "2nd Floor",
    --   "has_whiteboard": true,
    --   "has_video_conferencing": true,
    --   "video_system": "Zoom Rooms",
    --   "has_phone": true,
    --   "has_projector": false,
    --   "has_tv_display": true,
    --   "display_size": "65 inch",
    --   "catering_available": true,
    --   "layout_options": ["boardroom", "u-shape", "theater"],
    --   "current_layout": "boardroom",
    --   "accessible": true,
    --   "photos": ["https://cdn.example.com/room-a.jpg"]
    -- }
    --
    -- For phone booths:
    -- {
    --   "floor": "1st Floor",
    --   "soundproofing_rating": "high",
    --   "has_charger": true,
    --   "ventilation": "active"
    -- }
    --
    -- For parking spots:
    -- {
    --   "level": "B1",
    --   "spot_number": "P-42",
    --   "ev_charging": true,
    --   "covered": true,
    --   "size": "standard"
    -- }
    booking_rules   JSONB NOT NULL DEFAULT '{}',
    -- booking_rules schema:
    -- {
    --   "min_booking_minutes": 30,
    --   "max_booking_minutes": 480,
    --   "buffer_minutes": 10,
    --   "advance_booking_days": 30,
    --   "allowed_plan_types": ["dedicated_desk", "private_office"],
    --   "requires_approval": false,
    --   "auto_release_no_show_minutes": 15
    -- }
    -- Access control linkage
    access_config   JSONB NOT NULL DEFAULT '{}',
    -- access_config schema:
    -- {
    --   "access_point_ids": ["ap-001", "ap-002"],
    --   "provider": "kisi",
    --   "external_lock_id": "lock-12345",
    --   "unlock_duration_seconds": 5,
    --   "requires_active_booking": true
    -- }
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_resources_location ON resources(location_id, resource_type) WHERE deleted_at IS NULL AND is_active = true;
CREATE INDEX idx_resources_bookable ON resources(location_id) WHERE is_bookable = true AND is_active = true AND deleted_at IS NULL;
-- GIN index for attribute searches ("show me desks with monitors near a window")
CREATE INDEX idx_resources_attrs ON resources USING gin(attributes);
```

### Bookings (Fully Relational with JSONB Metadata)

```sql
CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    membership_id   UUID REFERENCES memberships(id),
    -- Strictly relational booking data (critical for conflict prevention)
    status          booking_status NOT NULL DEFAULT 'pending',
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    time_range      TSTZRANGE GENERATED ALWAYS AS (tstzrange(start_time, end_time)) STORED,
    -- Financial data (must be relational)
    total_price     DECIMAL(10, 2),
    credits_used    INT DEFAULT 0,
    currency        CHAR(3) DEFAULT 'USD',
    -- Flexible booking metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata schema:
    -- {
    --   "title": "Product Review Meeting",
    --   "notes": "Need the large whiteboard set up",
    --   "internal_notes": "VIP client meeting",
    --   "attendees": [
    --     { "user_id": "uuid...", "name": "Jane Doe", "email": "jane@example.com", "rsvp": "attending" },
    --     { "user_id": null, "name": "External Guest", "email": "guest@partner.com", "rsvp": "invited" }
    --   ],
    --   "recurrence": {
    --     "rule": "FREQ=WEEKLY;BYDAY=TU,TH;COUNT=10",
    --     "parent_booking_id": "uuid...",
    --     "occurrence_index": 3
    --   },
    --   "check_in": {
    --     "checked_in_at": "2026-06-15T09:58:00Z",
    --     "checked_in_by": "mobile_app",
    --     "checked_out_at": "2026-06-15T11:35:00Z"
    --   },
    --   "cancellation": {
    --     "cancelled_at": "2026-06-14T16:00:00Z",
    --     "reason": "meeting_rescheduled",
    --     "cancelled_by": "uuid...",
    --     "refund_amount": 30.00
    --   },
    --   "calendar_sync": {
    --     "google_event_id": "abc123",
    --     "outlook_event_id": "def456"
    --   },
    --   "catering_request": {
    --     "items": ["Coffee", "Pastries"],
    --     "headcount": 8,
    --     "special_requests": "Dairy-free milk option"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_booking_times CHECK (end_time > start_time)
);

-- Exclusion constraint prevents double-bookings at the database level
ALTER TABLE bookings
    ADD CONSTRAINT no_overlapping_bookings
    EXCLUDE USING gist (
        resource_id WITH =,
        time_range WITH &&
    ) WHERE (status NOT IN ('cancelled', 'no_show'));

CREATE INDEX idx_bookings_resource ON bookings(resource_id, start_time, end_time) WHERE status NOT IN ('cancelled', 'no_show');
CREATE INDEX idx_bookings_user ON bookings(user_id, start_time);
CREATE INDEX idx_bookings_org ON bookings(organization_id, start_time);
```

### Billing (Fully Relational -- No JSONB for Money)

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    corporate_account_id UUID REFERENCES corporate_accounts(id),
    membership_id   UUID REFERENCES memberships(id),
    invoice_number  VARCHAR(50) UNIQUE NOT NULL,
    status          invoice_status NOT NULL DEFAULT 'draft',
    issue_date      DATE NOT NULL,
    due_date        DATE NOT NULL,
    paid_date       DATE,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_rate        DECIMAL(5, 4) DEFAULT 0,
    discount_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_due      DECIMAL(12, 2) GENERATED ALWAYS AS (total - amount_paid) STORED,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    stripe_invoice_id VARCHAR(255),
    billing_period_start DATE,
    billing_period_end   DATE,
    dunning_attempts INT NOT NULL DEFAULT 0,
    -- Flexible invoice metadata (rendered PDF link, notes, etc.)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- { "notes": "...", "pdf_url": "...", "purchase_order": "PO-12345" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoices_user ON invoices(user_id, issue_date);
CREATE INDEX idx_invoices_status ON invoices(status) WHERE status IN ('open', 'overdue');

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_price      DECIMAL(10, 2) NOT NULL,
    amount          DECIMAL(12, 2) NOT NULL,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    membership_id   UUID REFERENCES memberships(id),
    booking_id      UUID REFERENCES bookings(id),
    period_start    DATE,
    period_end      DATE,
    is_prorated     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_id      UUID REFERENCES invoices(id),
    user_id         UUID REFERENCES users(id),
    amount          DECIMAL(12, 2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          payment_status NOT NULL DEFAULT 'pending',
    payment_method  VARCHAR(50),
    stripe_payment_id VARCHAR(255),
    payment_date    TIMESTAMPTZ,
    failure_reason  TEXT,
    refund_amount   DECIMAL(12, 2) DEFAULT 0,
    -- Stripe webhook raw data (preserve full provider response)
    provider_data   JSONB DEFAULT '{}',
    -- {
    --   "stripe_charge_id": "ch_...",
    --   "stripe_payment_intent_id": "pi_...",
    --   "card_last4": "4242",
    --   "card_brand": "visa",
    --   "receipt_url": "https://pay.stripe.com/...",
    --   "raw_webhook": { ... }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_stripe ON payments(stripe_payment_id) WHERE stripe_payment_id IS NOT NULL;

CREATE TABLE credit_transactions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    membership_id   UUID NOT NULL REFERENCES memberships(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    amount          INT NOT NULL,
    balance_after   INT NOT NULL,
    description     VARCHAR(500) NOT NULL,
    booking_id      UUID REFERENCES bookings(id),
    invoice_id      UUID REFERENCES invoices(id),
    expires_at      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_credit_tx ON credit_transactions(membership_id, created_at);
```

### Access Control (Relational + JSONB for Provider Data)

```sql
CREATE TABLE access_points (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Provider integration (highly variable between Kisi, Salto, Brivo, Hakuna)
    provider_config JSONB NOT NULL DEFAULT '{}',
    -- provider_config schema:
    -- {
    --   "provider": "kisi",
    --   "external_id": "lock-12345",
    --   "api_key_ref": "vault://kisi-api-key",
    --   "direction": "both",
    --   "unlock_duration_seconds": 5,
    --   "offline_capable": true,
    --   "ble_uuid": "550e8400-e29b-...",
    --   "firmware_version": "3.2.1",
    --   "last_heartbeat": "2026-06-15T10:00:00Z",
    --   "battery_level": 85,
    --   "custom_schedule": { ... }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_access_points ON access_points(location_id) WHERE is_active = true;

CREATE TABLE access_credentials (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    credential_type VARCHAR(50) NOT NULL,
    credential_value VARCHAR(500) NOT NULL,
    label           VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    -- Provider-specific credential data
    provider_data   JSONB DEFAULT '{}',
    -- { "provider": "kisi", "external_credential_id": "cred-abc", "ble_token": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_credentials_user ON access_credentials(user_id) WHERE is_active = true;

CREATE TABLE access_events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    access_point_id UUID NOT NULL REFERENCES access_points(id),
    user_id         UUID REFERENCES users(id),
    event_type      VARCHAR(50) NOT NULL,
    success         BOOLEAN NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Raw event from provider (preserve exact payload for debugging)
    raw_event       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "provider": "kisi",
    --   "provider_event_id": "evt-789",
    --   "credential_type": "mobile_ble",
    --   "credential_id": "uuid...",
    --   "failure_reason": null,
    --   "device_info": { "os": "iOS 19", "app_version": "2.3.1" },
    --   "location": { "lat": 37.7749, "lng": -122.4194 }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_access_events_point ON access_events(access_point_id, occurred_at);
CREATE INDEX idx_access_events_user ON access_events(user_id, occurred_at);
```

### Community and Events (JSONB for Flexible Content)

```sql
CREATE TABLE community_posts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    location_id     UUID REFERENCES locations(id),
    author_id       UUID NOT NULL REFERENCES users(id),
    -- Queryable fields
    post_type       VARCHAR(50) NOT NULL DEFAULT 'discussion',
    moderation_status VARCHAR(50) NOT NULL DEFAULT 'auto_approved',
    is_pinned       BOOLEAN NOT NULL DEFAULT false,
    is_visible      BOOLEAN NOT NULL DEFAULT true,
    like_count      INT NOT NULL DEFAULT 0,
    comment_count   INT NOT NULL DEFAULT 0,
    -- Flexible content (title, body, media, tags)
    content         JSONB NOT NULL,
    -- content schema:
    -- {
    --   "title": "Welcome to our new members!",
    --   "body": "We're excited to welcome...",
    --   "format": "markdown",
    --   "media": [
    --     { "url": "https://cdn.example.com/photo1.jpg", "type": "image", "alt": "New members" },
    --     { "url": "https://cdn.example.com/doc.pdf", "type": "document", "name": "Welcome Guide" }
    --   ],
    --   "tags": ["announcement", "community"],
    --   "mentions": ["uuid1", "uuid2"],
    --   "poll": {
    --     "question": "What event should we host next?",
    --     "options": ["Happy Hour", "Workshop", "Game Night"],
    --     "votes": { "Happy Hour": 12, "Workshop": 8, "Game Night": 15 },
    --     "ends_at": "2026-06-20T00:00:00Z"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_posts_feed ON community_posts(organization_id, created_at DESC) WHERE deleted_at IS NULL AND is_visible = true;
CREATE INDEX idx_posts_content ON community_posts USING gin(content);

CREATE TABLE post_interactions (
    post_id         UUID NOT NULL REFERENCES community_posts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    interaction_type VARCHAR(20) NOT NULL, -- 'like', 'bookmark', 'report'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (post_id, user_id, interaction_type)
);

CREATE TABLE post_comments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id         UUID NOT NULL REFERENCES community_posts(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    parent_comment_id UUID REFERENCES post_comments(id),
    body            TEXT NOT NULL,
    like_count      INT NOT NULL DEFAULT 0,
    is_visible      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_comments_post ON post_comments(post_id, created_at) WHERE deleted_at IS NULL;

-- Events with JSONB for flexible event details
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    location_id     UUID REFERENCES locations(id),
    organizer_id    UUID NOT NULL REFERENCES users(id),
    booking_id      UUID REFERENCES bookings(id),
    -- Queryable fields
    title           VARCHAR(500) NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    is_free         BOOLEAN NOT NULL DEFAULT true,
    rsvp_count      INT NOT NULL DEFAULT 0,
    max_attendees   INT,
    -- Flexible event details
    details         JSONB NOT NULL DEFAULT '{}',
    -- details schema:
    -- {
    --   "description": "Join us for...",
    --   "description_format": "markdown",
    --   "cover_image_url": "https://cdn.example.com/event.jpg",
    --   "venue": { "name": "Main Event Space", "address": "...", "is_virtual": false },
    --   "virtual": { "meeting_url": "https://zoom.us/...", "passcode": "12345" },
    --   "ticketing": {
    --     "price": 25.00,
    --     "currency": "USD",
    --     "stripe_product_id": "prod_...",
    --     "early_bird_price": 15.00,
    --     "early_bird_deadline": "2026-06-10T00:00:00Z",
    --     "refund_policy": "Full refund up to 48 hours before event"
    --   },
    --   "categories": ["networking", "workshop"],
    --   "speakers": [
    --     { "name": "Jane Smith", "title": "CTO", "bio": "...", "photo_url": "..." }
    --   ],
    --   "agenda": [
    --     { "time": "18:00", "title": "Registration & Networking" },
    --     { "time": "18:30", "title": "Keynote" },
    --     { "time": "19:30", "title": "Q&A" }
    --   ],
    --   "sponsor": { "name": "TechCorp", "logo_url": "..." }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_events_org ON events(organization_id, start_time) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_published ON events(organization_id, start_time) WHERE is_published = true AND deleted_at IS NULL;

CREATE TABLE event_rsvps (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'attending',
    ticket_count    INT NOT NULL DEFAULT 1,
    payment_id      UUID REFERENCES payments(id),
    checked_in_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(event_id, user_id)
);
```

### Visitors

```sql
CREATE TABLE visitor_passes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    host_id         UUID NOT NULL REFERENCES users(id),
    -- Queryable status fields
    status          VARCHAR(20) NOT NULL DEFAULT 'pre_registered',
    expected_arrival TIMESTAMPTZ NOT NULL,
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    -- Flexible visitor details (varies by operator requirements)
    visitor_info    JSONB NOT NULL DEFAULT '{}',
    -- visitor_info schema:
    -- {
    --   "name": "John Guest",
    --   "email": "john@partner.com",
    --   "phone": "+1-555-0200",
    --   "company": "Partner Corp",
    --   "purpose": "Client meeting",
    --   "expected_departure": "2026-06-15T17:00:00Z",
    --   "photo_url": null,
    --   "id_verified": false,
    --   "nda_signed": true,
    --   "parking_needed": true,
    --   "wifi_credentials": { "ssid": "Guest-WiFi", "password": "welcome2026" },
    --   "temporary_access": {
    --     "credential_type": "qr_code",
    --     "credential_value": "vis-abc123",
    --     "access_point_ids": ["ap-001"]
    --   },
    --   "dietary_requirements": "vegetarian",   <-- operator custom field
    --   "emergency_contact": { "name": "...", "phone": "..." }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_visitors_location ON visitor_passes(location_id, expected_arrival);
CREATE INDEX idx_visitors_host ON visitor_passes(host_id);
```

### Notifications and Audit Log

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    channel         VARCHAR(20) NOT NULL,
    category        VARCHAR(50) NOT NULL,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    -- Flexible notification content
    content         JSONB NOT NULL,
    -- {
    --   "title": "Booking Confirmed",
    --   "body": "Your booking for Conference Room A on June 15 is confirmed.",
    --   "action_url": "/bookings/uuid...",
    --   "icon": "calendar",
    --   "sent_at": "2026-06-10T14:32:00Z",
    --   "delivery_status": "delivered",
    --   "delivery_provider_id": "msg-abc123"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, created_at DESC);
CREATE INDEX idx_notifications_unread ON notifications(user_id) WHERE is_read = false;

-- Audit log with JSONB for flexible change tracking
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
    -- changes schema:
    -- {
    --   "before": { "status": "active", "price": 450.00 },
    --   "after": { "status": "suspended", "price": 450.00 },
    --   "diff": { "status": ["active", "suspended"] }
    -- }
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id, created_at);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at);
```

---

## Query Examples

### Finding desks with specific attributes

```sql
-- "Show me available quiet desks with monitors on the 3rd floor"
SELECT r.id, r.name, r.hourly_rate,
       r.attributes->>'zone' AS zone,
       r.attributes->>'monitor_size' AS monitor_size
FROM resources r
WHERE r.location_id = $1
  AND r.resource_type = 'hot_desk'
  AND r.is_active = true
  AND r.is_bookable = true
  AND r.attributes @> '{"has_monitor": true, "noise_level": "quiet"}'
  AND r.attributes->>'floor' = '3rd Floor'
  AND r.deleted_at IS NULL
ORDER BY r.sort_order;
```

### Querying custom member fields

```sql
-- "Find all members with dietary_preference = vegetarian for catering"
SELECT u.id, u.first_name, u.last_name, u.email,
       u.profile->>'dietary_preference' AS dietary_preference,
       u.profile->>'company_name' AS company
FROM users u
JOIN memberships m ON u.id = m.user_id
WHERE m.organization_id = $1
  AND m.status = 'active'
  AND u.profile->>'dietary_preference' = 'vegetarian';
```

### Plan comparison with JSONB inclusions

```sql
-- "Show all plans with their meeting room inclusions"
SELECT p.name, p.plan_type, p.base_price,
       (p.config->'inclusions'->>'meeting_room_minutes_per_month')::int AS room_minutes,
       (p.config->'inclusions'->>'desk_hours_per_month') AS desk_hours,  -- null = unlimited
       (p.config->'inclusions'->>'credits_per_month')::int AS credits,
       p.config->'access'->>'twenty_four_seven' AS has_24_7
FROM membership_plans p
WHERE p.organization_id = $1
  AND p.is_active = true
  AND p.deleted_at IS NULL
ORDER BY p.sort_order;
```

---

## Pros and Cons

### Pros

1. **Operator customization without migrations**: The biggest advantage. When an operator wants to add "noise_level" to desks or "dietary_preference" to member profiles, it is a configuration change, not a database migration. This is critical for a platform serving diverse operators with different needs.

2. **Single database engine**: Everything runs on PostgreSQL. No additional database technology to learn, deploy, monitor, or back up. Reduces operational complexity dramatically compared to polyglot persistence approaches.

3. **Financial integrity preserved**: All money-related data (invoices, payments, line items, credit transactions) uses normalized tables with strict types and foreign keys. The JSONB flexibility does not compromise financial data integrity.

4. **Double-booking prevention intact**: The `bookings` table keeps `start_time`, `end_time`, and `resource_id` as relational columns with a GiST exclusion constraint. The booking metadata (attendees, notes, catering) is in JSONB, but the conflict-prevention logic is purely relational.

5. **Reduced table count**: The normalized model (Suggestion 1) requires ~40 tables. This hybrid model achieves the same functionality with ~25 tables by consolidating variable attributes into JSONB columns. Fewer tables means simpler queries, fewer joins, and easier mental models for developers.

6. **Integration payload flexibility**: Smart-lock providers (Kisi, Salto, Brivo, Hakuna) send wildly different event payloads. Storing `raw_event` as JSONB preserves the full payload for debugging without creating provider-specific tables. The same applies to Stripe webhooks.

7. **GIN indexes make JSONB queries fast**: Queries like `attributes @> '{"has_monitor": true}'` use GIN indexes and perform well even at scale. Partial GIN indexes on specific paths (e.g., user skills for networking search) provide targeted performance.

8. **Schema evolution is painless for JSONB columns**: Adding a new field to a plan's `config` or a resource's `attributes` requires no migration. Old records simply lack the field, which the application handles with defaults. This makes rapid iteration during early product development much smoother.

### Cons

1. **No database-level validation of JSONB structure**: PostgreSQL does not enforce JSON Schema on JSONB columns. A bug in the application could write malformed plan configurations or resource attributes. Application-layer validation (JSON Schema, Zod, Pydantic) is mandatory but must be comprehensive and consistently applied.

2. **JSONB updates are whole-document**: Updating a single field in a JSONB column (e.g., changing one attendee's RSVP status in a booking's `metadata`) rewrites the entire JSONB value. For large documents with frequent partial updates, this creates unnecessary I/O. Keeping JSONB documents under 10KB mitigates this.

3. **Reporting complexity**: Aggregating across JSONB fields is more complex than across normalized columns. "Total meeting room minutes included across all active plans" requires `(config->'inclusions'->>'meeting_room_minutes_per_month')::int` instead of a simple `SUM(included_room_minutes)`. BI tools and report builders handle this less gracefully.

4. **Type safety is the application's responsibility**: JSONB stores everything as JSON types (string, number, boolean, null, array, object). There is no `DECIMAL(10,2)` precision guarantee inside JSONB. Storing prices in JSONB would be dangerous, which is why this model keeps all financial data in typed columns.

5. **Indexing discipline required**: Without GIN indexes, JSONB queries degrade to sequential scans. Developers must understand which JSONB paths are frequently queried and create appropriate indexes. Forgetting to index a new query pattern can cause unexpected performance problems.

6. **Migrations for JSONB data are application-side**: When the structure of a JSONB field changes (e.g., renaming `config.inclusions.room_minutes` to `config.inclusions.meeting_room_minutes_per_month`), the database migration is a data-backfill UPDATE, not a DDL change. These must be written and tested carefully.

7. **Potential for schema drift**: Without strict enforcement, different operators' data may drift in structure over time. One operator's resource attributes might use `"has_monitor": true` while another uses `"monitor": "yes"`. Strong application-layer validation and documentation are essential.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| Database | PostgreSQL 16+ with btree_gist extension |
| JSONB Validation | JSON Schema (ajv for Node.js, jsonschema for Python) or Zod/Pydantic in application layer |
| ORM | Prisma (TypeScript) with `Json` type support, or Drizzle ORM with custom JSONB types |
| Migrations | Flyway or dbmate; separate migration scripts for schema (DDL) and data (JSONB backfills) |
| Connection pooling | PgBouncer in transaction mode |
| Search | PostgreSQL GIN indexes for JSONB; pg_trgm for text search within JSONB body fields |
| Caching | Redis for session data and frequently-read JSONB configs (plan configs, branding) |
| API | GraphQL for flexible querying of JSONB attributes; REST for transactional commands |

---

## Migration and Scaling Considerations

### Phase 1: Startup (0-50 locations)
- Single PostgreSQL instance with one read replica
- GIN indexes on all JSONB columns from day one
- JSON Schema validation library integrated into the application from launch
- Monitor JSONB document sizes (alert if any exceed 50KB)
- Use `jsonb_set()` for surgical JSONB updates where possible

### Phase 2: Growth (50-500 locations)
- Managed PostgreSQL (RDS, Cloud SQL, Supabase) for automated backups and failover
- Read replicas dedicated to analytics queries
- Extract high-volume JSONB data to materialized views for reporting
- Implement JSONB schema versioning in application layer
- Consider extracting access_events into partitioned table or TimescaleDB hypertable
- Add expression indexes on specific JSONB paths that become hot:
  ```sql
  CREATE INDEX idx_resource_zone ON resources((attributes->>'zone')) WHERE is_active = true;
  CREATE INDEX idx_plan_24_7 ON membership_plans(organization_id) WHERE (config->'access'->>'twenty_four_seven')::boolean = true;
  ```

### Phase 3: Scale (500+ locations)
- Citus or partitioning by organization_id for horizontal scaling
- Extract community search to Elasticsearch with JSONB content indexed
- CDC pipeline (Debezium) to feed JSONB changes to analytics data warehouse
- Consider promoting frequently-queried JSONB paths to dedicated columns as patterns stabilize:
  ```sql
  -- When you discover that 90% of queries filter on noise_level, promote it:
  ALTER TABLE resources ADD COLUMN noise_level VARCHAR(20);
  UPDATE resources SET noise_level = attributes->>'noise_level' WHERE attributes ? 'noise_level';
  CREATE INDEX idx_resources_noise ON resources(noise_level) WHERE noise_level IS NOT NULL;
  ```

### JSONB size management
- Target maximum JSONB document size: 50KB (covers 99% of use cases)
- Monitor with: `SELECT pg_column_size(attributes) FROM resources ORDER BY pg_column_size(attributes) DESC LIMIT 10;`
- If a JSONB column consistently exceeds 50KB, extract the large sub-document into a related table
- Avoid storing binary data (images, files) in JSONB; use URLs pointing to object storage

### JSONB schema registry
- Maintain a schema registry (e.g., a `jsonb_schemas` table or a code-level registry) documenting the expected structure of each JSONB column
- Version schemas and support backward compatibility (new fields are always optional, never remove fields from validation without a data migration)
- Generate TypeScript/Python types from the schema registry for compile-time safety
