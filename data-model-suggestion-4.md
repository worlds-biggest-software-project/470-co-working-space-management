# Data Model Suggestion 4: Polyglot Persistence with Time-Series and Graph Specialization (PostgreSQL + TimescaleDB + Apache AGE + Redis)

> Project: Co-Working Space Management (Candidate #470)
> Generated: 2026-05-26

## Overview

This model adopts a polyglot persistence architecture that uses domain-specific storage engines for the workloads where they excel, unified under a single PostgreSQL process where possible. The core insight is that co-working space management is not one problem -- it is at least four distinct data domains that each have a natural storage paradigm:

1. **Transactional data** (memberships, billing, invoices, plans): Classical relational with strict ACID guarantees. Handled by standard PostgreSQL tables.
2. **Temporal event streams** (occupancy sensor readings, access control events, check-in/check-out logs, booking utilization telemetry, HVAC and environmental data): Append-heavy, time-partitioned, aggregation-intensive. Handled by **TimescaleDB** hypertables -- a PostgreSQL extension that provides automatic time-partitioning, columnar compression, and continuous aggregates without leaving the PostgreSQL ecosystem.
3. **Community and relationship data** (member connections, interest graphs, collaboration networks, event co-attendance, referral chains, community detection): Highly connected, traversal-heavy, schema-flexible. Handled by **Apache AGE** -- a PostgreSQL extension that adds openCypher graph query capability directly inside PostgreSQL, enabling graph traversals alongside SQL queries in the same database.
4. **Real-time operational state** (current desk availability, active sessions, booking slot locks, rate limiters, notification queues): Sub-millisecond reads, ephemeral, high-throughput. Handled by **Redis** as an in-memory cache and real-time state layer.

This architecture avoids the operational complexity of running four separate database systems. TimescaleDB and Apache AGE are PostgreSQL extensions -- they run inside the same PostgreSQL instance, share the same connection pool, participate in the same transactions, and are backed up with the same pg_dump. Redis is the only additional system, and it serves a clearly bounded role as a volatile cache layer. The result is a system that matches each workload to its optimal storage paradigm while keeping operational complexity close to that of a single-database deployment.

---

## Architecture Overview

```
                    +-------------------------------+
                    |        Application Layer       |
                    |   (API Gateway / Service Mesh) |
                    +------+--------+--------+------+
                           |        |        |
              +------------+   +----+----+   +------------+
              |                |         |                 |
     +--------v--------+ +----v----+ +--v-----------+ +---v---------+
     |   PostgreSQL     | |Timescale| | Apache AGE   | |   Redis     |
     |   Core Tables    | |DB Hyper-| | Graph Layer  | |   Cache &   |
     |   (Relational)   | |tables   | | (Cypher in   | |   Real-Time |
     |                  | |(Time-   | |  PostgreSQL) | |   State     |
     |  - Members       | | Series) | |              | |             |
     |  - Organizations | |         | | - Member     | | - Avail.    |
     |  - Locations     | | - Sensor| |   Network    | |   Matrix    |
     |  - Memberships   | |   Data  | | - Interest   | | - Session   |
     |  - Invoices      | | - Access|   Groups     | |   Tokens    |
     |  - Payments      | |   Logs  | | - Collab.   | | - Booking   |
     |  - Plans         | | - Check-| |   Clusters  | |   Locks     |
     |  - Bookings      | |   ins   | | - Referral  | | - Rate      |
     |  - Resources     | | - Env.  | |   Chains    | |   Limits    |
     |                  | |   Data  | |             | | - Pub/Sub   |
     +------------------+ +---------+ +-------------+ +-------------+
              |                |              |
              +-------+--------+--------------+
                      |
              +-------v--------+
              |   Shared       |
              |   PostgreSQL   |
              |   Instance     |
              +----------------+
```

---

## Part 1: Core Relational Tables (Standard PostgreSQL)

These tables handle transactional integrity for the financial, membership, and booking domains. They follow a normalized relational design and are referenced by the time-series and graph layers via foreign keys.

### Extensions Required

```sql
-- Core PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "btree_gist";

-- Time-series extension
CREATE EXTENSION IF NOT EXISTS "timescaledb";

-- Graph extension
CREATE EXTENSION IF NOT EXISTS "age";
LOAD 'age';
SET search_path = ag_catalog, "$user", public;
```

### Enumerated Types

```sql
CREATE TYPE membership_status AS ENUM (
    'pending', 'active', 'paused', 'cancelled', 'expired', 'suspended'
);

CREATE TYPE plan_type AS ENUM (
    'hot_desk', 'dedicated_desk', 'private_office',
    'virtual_office', 'day_pass', 'credit_pack', 'custom'
);

CREATE TYPE billing_cycle AS ENUM (
    'daily', 'weekly', 'monthly', 'quarterly', 'annual'
);

CREATE TYPE booking_status AS ENUM (
    'tentative', 'confirmed', 'checked_in', 'completed',
    'cancelled', 'no_show'
);

CREATE TYPE resource_type AS ENUM (
    'hot_desk', 'dedicated_desk', 'private_office',
    'meeting_room', 'phone_booth', 'event_space',
    'parking_spot', 'locker', 'custom'
);

CREATE TYPE invoice_status AS ENUM (
    'draft', 'issued', 'paid', 'overdue', 'void', 'refunded'
);

CREATE TYPE payment_method AS ENUM (
    'stripe', 'paypal', 'square', 'bank_transfer',
    'cash', 'credit_note'
);

CREATE TYPE access_credential_type AS ENUM (
    'pin_code', 'rfid_card', 'mobile_ble', 'qr_code',
    'biometric', 'smart_lock_token'
);

CREATE TYPE visitor_status AS ENUM (
    'pre_registered', 'checked_in', 'checked_out', 'cancelled'
);
```

### Organizations and Locations

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    legal_name      TEXT,
    tax_id          TEXT,
    billing_email   TEXT,
    phone           TEXT,
    website         TEXT,
    logo_url        TEXT,
    branding_config JSONB DEFAULT '{}',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    address_line1   TEXT NOT NULL,
    address_line2   TEXT,
    city            TEXT NOT NULL,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    latitude        DOUBLE PRECISION,
    longitude       DOUBLE PRECISION,
    operating_hours JSONB NOT NULL DEFAULT '{}',
    amenities       JSONB DEFAULT '[]',
    capacity        INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_locations_org ON locations(organization_id);
CREATE INDEX idx_locations_city ON locations(city, country_code);
```

### Members and Authentication

```sql
CREATE TABLE members (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT NOT NULL,
    password_hash   TEXT,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    display_name    TEXT,
    avatar_url      TEXT,
    phone           TEXT,
    company_name    TEXT,
    job_title       TEXT,
    bio             TEXT,
    interests       TEXT[] DEFAULT '{}',
    skills          TEXT[] DEFAULT '{}',
    industry        TEXT,
    preferred_location_id UUID REFERENCES locations(id),
    notification_prefs    JSONB DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_members_org ON members(organization_id);
CREATE INDEX idx_members_email ON members(email);
CREATE INDEX idx_members_interests ON members USING GIN(interests);
CREATE INDEX idx_members_skills ON members USING GIN(skills);
```

### Plans and Memberships

```sql
CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    location_id     UUID REFERENCES locations(id),  -- NULL = available at all locations
    name            TEXT NOT NULL,
    plan_type       plan_type NOT NULL,
    billing_cycle   billing_cycle NOT NULL,
    price_cents     INTEGER NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    included_credits INTEGER DEFAULT 0,
    credit_rollover BOOLEAN DEFAULT false,
    max_rollover_credits INTEGER,
    features        JSONB DEFAULT '{}',
    access_hours    JSONB DEFAULT '{}',  -- e.g., {"mon": "08:00-22:00", ...}
    booking_limits  JSONB DEFAULT '{}',  -- e.g., {"meeting_rooms_per_day": 2}
    community_permissions JSONB DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    member_id       UUID NOT NULL REFERENCES members(id),
    plan_id         UUID NOT NULL REFERENCES plans(id),
    location_id     UUID NOT NULL REFERENCES locations(id),
    status          membership_status NOT NULL DEFAULT 'pending',
    start_date      DATE NOT NULL,
    end_date        DATE,
    trial_end_date  DATE,
    billing_anchor  DATE NOT NULL,
    remaining_credits INTEGER DEFAULT 0,
    stripe_subscription_id TEXT,
    cancellation_reason TEXT,
    cancelled_at    TIMESTAMPTZ,
    paused_at       TIMESTAMPTZ,
    resume_date     DATE,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_memberships_member ON memberships(member_id);
CREATE INDEX idx_memberships_status ON memberships(status) WHERE status = 'active';
CREATE INDEX idx_memberships_location ON memberships(location_id);
```

### Resources and Bookings

```sql
CREATE TABLE resources (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    resource_type   resource_type NOT NULL,
    floor           TEXT,
    zone            TEXT,
    capacity        INTEGER DEFAULT 1,
    amenities       JSONB DEFAULT '[]',
    hourly_rate_cents INTEGER,
    half_day_rate_cents INTEGER,
    daily_rate_cents INTEGER,
    attributes      JSONB DEFAULT '{}',
    is_bookable     BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    smart_lock_id   TEXT,  -- external smart lock device identifier
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resources_location ON resources(location_id);
CREATE INDEX idx_resources_type ON resources(location_id, resource_type);

CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    member_id       UUID NOT NULL REFERENCES members(id),
    location_id     UUID NOT NULL REFERENCES locations(id),
    status          booking_status NOT NULL DEFAULT 'tentative',
    time_range      TSTZRANGE NOT NULL,
    title           TEXT,
    notes           TEXT,
    attendee_count  INTEGER DEFAULT 1,
    total_price_cents INTEGER,
    credits_used    INTEGER DEFAULT 0,
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    recurring_rule  JSONB,  -- iCal RRULE as JSON
    parent_booking_id UUID REFERENCES bookings(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Prevent double-booking using GiST exclusion constraint
    EXCLUDE USING GIST (
        resource_id WITH =,
        time_range WITH &&
    ) WHERE (status NOT IN ('cancelled', 'no_show'))
);

CREATE INDEX idx_bookings_member ON bookings(member_id);
CREATE INDEX idx_bookings_resource ON bookings(resource_id);
CREATE INDEX idx_bookings_time ON bookings USING GIST(time_range);
CREATE INDEX idx_bookings_status ON bookings(status);
```

### Invoices and Payments

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    member_id       UUID NOT NULL REFERENCES members(id),
    membership_id   UUID REFERENCES memberships(id),
    invoice_number  TEXT NOT NULL UNIQUE,
    status          invoice_status NOT NULL DEFAULT 'draft',
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal_cents  INTEGER NOT NULL DEFAULT 0,
    tax_cents       INTEGER NOT NULL DEFAULT 0,
    discount_cents  INTEGER NOT NULL DEFAULT 0,
    total_cents     INTEGER NOT NULL DEFAULT 0,
    due_date        DATE NOT NULL,
    paid_at         TIMESTAMPTZ,
    stripe_invoice_id TEXT,
    notes           TEXT,
    line_items      JSONB NOT NULL DEFAULT '[]',
    billing_address JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_member ON invoices(member_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due ON invoices(due_date) WHERE status IN ('issued', 'overdue');

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    member_id       UUID NOT NULL REFERENCES members(id),
    amount_cents    INTEGER NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  payment_method NOT NULL,
    stripe_payment_id TEXT,
    status          TEXT NOT NULL DEFAULT 'pending',
    failure_reason  TEXT,
    refunded_amount_cents INTEGER DEFAULT 0,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_member ON payments(member_id);
```

### Access Credentials and Visitors

```sql
CREATE TABLE access_credentials (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    member_id       UUID NOT NULL REFERENCES members(id),
    credential_type access_credential_type NOT NULL,
    credential_value TEXT NOT NULL,  -- encrypted
    device_name     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until     TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credentials_member ON access_credentials(member_id);

CREATE TABLE visitors (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id     UUID NOT NULL REFERENCES locations(id),
    host_member_id  UUID NOT NULL REFERENCES members(id),
    visitor_name    TEXT NOT NULL,
    visitor_email   TEXT,
    visitor_phone   TEXT,
    visitor_company TEXT,
    status          visitor_status NOT NULL DEFAULT 'pre_registered',
    expected_arrival TIMESTAMPTZ NOT NULL,
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    temp_access_code TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_visitors_location ON visitors(location_id);
CREATE INDEX idx_visitors_host ON visitors(host_member_id);
CREATE INDEX idx_visitors_arrival ON visitors(expected_arrival);
```

---

## Part 2: Time-Series Layer (TimescaleDB Hypertables)

TimescaleDB transforms specific tables into hypertables -- automatically time-partitioned structures optimized for append-heavy workloads with time-range queries. This layer captures the high-velocity event streams that a co-working space generates: every door unlock, every sensor reading, every check-in, every Wi-Fi association.

### Why TimescaleDB for Co-Working Spaces

Co-working spaces generate substantial volumes of time-stamped data:

- **Access control events**: Every door unlock, lock attempt, credential scan. A 200-desk space with 10 access points generates 2,000-5,000 events per day.
- **Occupancy sensor readings**: PIR, thermal, and ToF sensors reporting presence every 30-60 seconds. A 50-sensor deployment generates 72,000-144,000 readings per day.
- **Environmental telemetry**: Temperature, humidity, CO2, noise level, light intensity. Critical for automated HVAC control and comfort optimization.
- **Check-in/check-out events**: Member arrivals and departures, desk claims and releases.
- **Wi-Fi and network events**: Device associations, bandwidth usage, connection quality metrics.

Storing this data in regular PostgreSQL tables would require manual partitioning, custom aggregation logic, and careful index management. TimescaleDB provides all of this automatically as a PostgreSQL extension.

### Access Control Event Stream

```sql
CREATE TABLE access_events (
    time            TIMESTAMPTZ NOT NULL,
    location_id     UUID NOT NULL,
    device_id       TEXT NOT NULL,       -- smart lock device identifier
    member_id       UUID,                -- NULL for failed/unknown attempts
    credential_id   UUID,
    event_type      TEXT NOT NULL,       -- 'unlock', 'lock', 'denied', 'tailgate', 'forced'
    access_point    TEXT NOT NULL,       -- 'main_entrance', 'floor_3_east', 'meeting_room_a'
    direction       TEXT,                -- 'entry', 'exit'
    method          TEXT,                -- 'ble', 'rfid', 'pin', 'qr', 'remote'
    latency_ms      INTEGER,            -- lock response time
    battery_pct     SMALLINT,           -- lock battery level
    metadata        JSONB DEFAULT '{}'
);

-- Convert to hypertable with 1-day chunks
SELECT create_hypertable('access_events', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

-- Composite index for member access history
CREATE INDEX idx_access_events_member ON access_events (member_id, time DESC)
    WHERE member_id IS NOT NULL;

-- Index for device health monitoring
CREATE INDEX idx_access_events_device ON access_events (device_id, time DESC);

-- Index for security queries (denied access attempts)
CREATE INDEX idx_access_events_denied ON access_events (location_id, time DESC)
    WHERE event_type IN ('denied', 'tailgate', 'forced');
```

### Occupancy Sensor Readings

```sql
CREATE TABLE occupancy_readings (
    time            TIMESTAMPTZ NOT NULL,
    location_id     UUID NOT NULL,
    sensor_id       TEXT NOT NULL,
    zone            TEXT NOT NULL,       -- 'open_plan_east', 'meeting_room_b', 'kitchen'
    floor           TEXT,
    sensor_type     TEXT NOT NULL,       -- 'pir', 'tof', 'thermal', 'camera_count'
    occupant_count  INTEGER NOT NULL DEFAULT 0,
    capacity        INTEGER,            -- zone capacity for utilization calculation
    is_occupied     BOOLEAN NOT NULL DEFAULT false,
    confidence      REAL                -- sensor confidence score 0.0-1.0
);

SELECT create_hypertable('occupancy_readings', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

-- Primary query pattern: zone occupancy over time
CREATE INDEX idx_occupancy_zone ON occupancy_readings (location_id, zone, time DESC);

-- Sensor health monitoring
CREATE INDEX idx_occupancy_sensor ON occupancy_readings (sensor_id, time DESC);
```

### Environmental Telemetry

```sql
CREATE TABLE environmental_readings (
    time            TIMESTAMPTZ NOT NULL,
    location_id     UUID NOT NULL,
    sensor_id       TEXT NOT NULL,
    zone            TEXT NOT NULL,
    temperature_c   REAL,
    humidity_pct    REAL,
    co2_ppm         REAL,
    noise_db        REAL,
    light_lux       REAL,
    air_quality_idx REAL,
    pm25            REAL,               -- particulate matter
    voc_ppb         REAL                -- volatile organic compounds
);

SELECT create_hypertable('environmental_readings', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_environmental_zone ON environmental_readings (location_id, zone, time DESC);
```

### Member Check-In/Check-Out Events

```sql
CREATE TABLE checkin_events (
    time            TIMESTAMPTZ NOT NULL,
    location_id     UUID NOT NULL,
    member_id       UUID NOT NULL,
    resource_id     UUID,               -- specific desk/room if applicable
    event_type      TEXT NOT NULL,       -- 'checkin', 'checkout', 'auto_checkout'
    method          TEXT NOT NULL,       -- 'app', 'kiosk', 'access_point', 'wifi_detect', 'manual'
    booking_id      UUID,               -- linked booking if applicable
    duration_minutes INTEGER             -- populated on checkout
);

SELECT create_hypertable('checkin_events', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_checkin_member ON checkin_events (member_id, time DESC);
CREATE INDEX idx_checkin_location ON checkin_events (location_id, time DESC);
CREATE INDEX idx_checkin_resource ON checkin_events (resource_id, time DESC)
    WHERE resource_id IS NOT NULL;
```

### Wi-Fi and Network Events

```sql
CREATE TABLE wifi_events (
    time            TIMESTAMPTZ NOT NULL,
    location_id     UUID NOT NULL,
    access_point_id TEXT NOT NULL,
    mac_address     TEXT NOT NULL,       -- hashed for privacy
    member_id       UUID,               -- matched via device registration
    event_type      TEXT NOT NULL,       -- 'associate', 'disassociate', 'roam'
    signal_strength INTEGER,            -- dBm
    bandwidth_mbps  REAL,
    channel         INTEGER,
    ssid            TEXT
);

SELECT create_hypertable('wifi_events', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_wifi_member ON wifi_events (member_id, time DESC)
    WHERE member_id IS NOT NULL;
CREATE INDEX idx_wifi_ap ON wifi_events (access_point_id, time DESC);
```

### Continuous Aggregates for Real-Time Analytics

Continuous aggregates are materialized views that TimescaleDB incrementally refreshes in the background. They provide pre-computed rollups for dashboards and reports without the cost of scanning raw data.

```sql
-- Hourly occupancy summary per zone
CREATE MATERIALIZED VIEW occupancy_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    location_id,
    zone,
    AVG(occupant_count)::REAL AS avg_occupants,
    MAX(occupant_count) AS peak_occupants,
    MIN(occupant_count) AS min_occupants,
    AVG(CASE WHEN capacity > 0
        THEN (occupant_count::REAL / capacity) * 100
        ELSE 0
    END)::REAL AS avg_utilization_pct,
    COUNT(*) AS reading_count
FROM occupancy_readings
GROUP BY bucket, location_id, zone
WITH NO DATA;

-- Refresh policy: update every 30 minutes, covering the last 2 hours
SELECT add_continuous_aggregate_policy('occupancy_hourly',
    start_offset    => INTERVAL '2 hours',
    end_offset      => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes'
);

-- Daily occupancy summary for long-term trend analysis
CREATE MATERIALIZED VIEW occupancy_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS day,
    location_id,
    zone,
    AVG(avg_occupants)::REAL AS avg_occupants,
    MAX(peak_occupants) AS peak_occupants,
    AVG(avg_utilization_pct)::REAL AS avg_utilization_pct,
    SUM(reading_count)::BIGINT AS total_readings
FROM occupancy_hourly
GROUP BY day, location_id, zone
WITH NO DATA;

SELECT add_continuous_aggregate_policy('occupancy_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- Hourly access control summary
CREATE MATERIALIZED VIEW access_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    location_id,
    access_point,
    event_type,
    COUNT(*) AS event_count,
    COUNT(DISTINCT member_id) AS unique_members,
    AVG(latency_ms)::REAL AS avg_latency_ms,
    MIN(battery_pct) AS min_battery_pct
FROM access_events
GROUP BY bucket, location_id, access_point, event_type
WITH NO DATA;

SELECT add_continuous_aggregate_policy('access_hourly',
    start_offset    => INTERVAL '2 hours',
    end_offset      => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes'
);

-- Daily check-in summary for membership utilization reporting
CREATE MATERIALIZED VIEW checkin_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    location_id,
    member_id,
    COUNT(*) FILTER (WHERE event_type = 'checkin') AS checkin_count,
    COUNT(*) FILTER (WHERE event_type = 'checkout') AS checkout_count,
    SUM(duration_minutes) FILTER (WHERE duration_minutes IS NOT NULL) AS total_minutes,
    AVG(duration_minutes)::REAL FILTER (WHERE duration_minutes IS NOT NULL) AS avg_duration_minutes,
    MIN(time) FILTER (WHERE event_type = 'checkin') AS first_checkin,
    MAX(time) FILTER (WHERE event_type = 'checkout') AS last_checkout
FROM checkin_events
GROUP BY day, location_id, member_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('checkin_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- Hourly environmental comfort index
CREATE MATERIALIZED VIEW environmental_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    location_id,
    zone,
    AVG(temperature_c)::REAL AS avg_temp_c,
    AVG(humidity_pct)::REAL AS avg_humidity_pct,
    AVG(co2_ppm)::REAL AS avg_co2_ppm,
    MAX(co2_ppm)::REAL AS peak_co2_ppm,
    AVG(noise_db)::REAL AS avg_noise_db,
    AVG(light_lux)::REAL AS avg_light_lux,
    AVG(air_quality_idx)::REAL AS avg_aqi
FROM environmental_readings
GROUP BY bucket, location_id, zone
WITH NO DATA;

SELECT add_continuous_aggregate_policy('environmental_hourly',
    start_offset    => INTERVAL '2 hours',
    end_offset      => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes'
);
```

### Compression and Retention Policies

TimescaleDB's native compression achieves 10-20x reduction for time-series data. Combined with retention policies, this creates an automated data lifecycle: hot data (recent, uncompressed) for real-time queries, warm data (compressed) for historical analysis, and automatic deletion of data beyond its useful life.

```sql
-- Enable compression on all hypertables
ALTER TABLE access_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'location_id, device_id',
    timescaledb.compress_orderby = 'time DESC'
);

ALTER TABLE occupancy_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'location_id, zone',
    timescaledb.compress_orderby = 'time DESC'
);

ALTER TABLE environmental_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'location_id, zone',
    timescaledb.compress_orderby = 'time DESC'
);

ALTER TABLE checkin_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'location_id, member_id',
    timescaledb.compress_orderby = 'time DESC'
);

ALTER TABLE wifi_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'location_id, access_point_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Compression policies: compress chunks older than 7 days
SELECT add_compression_policy('access_events', INTERVAL '7 days');
SELECT add_compression_policy('occupancy_readings', INTERVAL '7 days');
SELECT add_compression_policy('environmental_readings', INTERVAL '7 days');
SELECT add_compression_policy('checkin_events', INTERVAL '7 days');
SELECT add_compression_policy('wifi_events', INTERVAL '7 days');

-- Retention policies: drop raw data after defined periods
-- (aggregates in continuous aggregates are retained longer)
SELECT add_retention_policy('occupancy_readings', INTERVAL '90 days');
SELECT add_retention_policy('environmental_readings', INTERVAL '90 days');
SELECT add_retention_policy('wifi_events', INTERVAL '90 days');
SELECT add_retention_policy('access_events', INTERVAL '365 days');  -- longer for security audit
SELECT add_retention_policy('checkin_events', INTERVAL '365 days'); -- longer for membership analytics
```

---

## Part 3: Graph Layer (Apache AGE)

Apache AGE adds openCypher graph query capability directly inside PostgreSQL. The graph layer models the community and relationship dimensions of the co-working space -- the connections between members, their shared interests, collaboration patterns, event co-attendance, and referral networks. These are inherently graph problems: finding communities, recommending connections, identifying influential members, and tracing referral chains all require multi-hop traversals that are expensive or impossible in SQL but natural in Cypher.

### Graph Schema Creation

```sql
-- Create the graph namespace
SELECT create_graph('coworking_community');
```

### Node Types

```sql
-- Member nodes (synced from relational members table)
-- Properties mirror key fields from the relational layer
SELECT * FROM cypher('coworking_community', $$
    CREATE (:Member {
        member_id: 'uuid-placeholder',
        name: 'Jane Smith',
        company: 'Acme Corp',
        job_title: 'Product Manager',
        industry: 'Technology',
        location_id: 'uuid-placeholder',
        joined_at: '2025-01-15T00:00:00Z'
    })
$$) AS (v agtype);

-- Interest nodes (canonical interest taxonomy)
SELECT * FROM cypher('coworking_community', $$
    CREATE (:Interest {name: 'Machine Learning', category: 'Technology'})
$$) AS (v agtype);

-- Skill nodes (professional skills for matching)
SELECT * FROM cypher('coworking_community', $$
    CREATE (:Skill {name: 'Python', category: 'Programming', level: 'advanced'})
$$) AS (v agtype);

-- Event nodes (community events for co-attendance tracking)
SELECT * FROM cypher('coworking_community', $$
    CREATE (:Event {
        event_id: 'uuid-placeholder',
        title: 'AI/ML Meetup',
        date: '2025-03-20',
        location_id: 'uuid-placeholder',
        category: 'Technology'
    })
$$) AS (v agtype);

-- Company nodes (for organizational networking)
SELECT * FROM cypher('coworking_community', $$
    CREATE (:Company {
        name: 'Acme Corp',
        industry: 'Technology',
        size_category: 'startup'
    })
$$) AS (v agtype);

-- Location nodes (for cross-location community analysis)
SELECT * FROM cypher('coworking_community', $$
    CREATE (:Location {
        location_id: 'uuid-placeholder',
        name: 'Downtown Hub',
        city: 'San Francisco'
    })
$$) AS (v agtype);

-- Community nodes (detected or curated groups)
SELECT * FROM cypher('coworking_community', $$
    CREATE (:Community {
        name: 'AI Enthusiasts',
        type: 'detected',
        created_at: '2025-06-01'
    })
$$) AS (v agtype);
```

### Relationship Types

```sql
-- Member connections (bidirectional social connections)
SELECT * FROM cypher('coworking_community', $$
    MATCH (a:Member {member_id: 'uuid-1'}), (b:Member {member_id: 'uuid-2'})
    CREATE (a)-[:CONNECTED_TO {
        since: '2025-02-10',
        source: 'community_feed',
        interaction_count: 0
    }]->(b)
$$) AS (e agtype);

-- Interest associations
SELECT * FROM cypher('coworking_community', $$
    MATCH (m:Member {member_id: 'uuid-1'}), (i:Interest {name: 'Machine Learning'})
    CREATE (m)-[:INTERESTED_IN {weight: 0.9, declared: true}]->(i)
$$) AS (e agtype);

-- Skill declarations
SELECT * FROM cypher('coworking_community', $$
    MATCH (m:Member {member_id: 'uuid-1'}), (s:Skill {name: 'Python'})
    CREATE (m)-[:HAS_SKILL {proficiency: 'expert', endorsed_by: 3}]->(s)
$$) AS (e agtype);

-- Event attendance
SELECT * FROM cypher('coworking_community', $$
    MATCH (m:Member {member_id: 'uuid-1'}), (e:Event {event_id: 'uuid-event-1'})
    CREATE (m)-[:ATTENDED {rsvp_time: '2025-03-18', checked_in: true}]->(e)
$$) AS (e agtype);

-- Company employment
SELECT * FROM cypher('coworking_community', $$
    MATCH (m:Member {member_id: 'uuid-1'}), (c:Company {name: 'Acme Corp'})
    CREATE (m)-[:WORKS_AT {role: 'Product Manager', since: '2024-06-01'}]->(c)
$$) AS (e agtype);

-- Member-Location affinity (based on check-in patterns)
SELECT * FROM cypher('coworking_community', $$
    MATCH (m:Member {member_id: 'uuid-1'}), (l:Location {location_id: 'uuid-loc-1'})
    CREATE (m)-[:FREQUENTS {
        visit_count: 42,
        avg_duration_hours: 6.5,
        preferred_zone: 'quiet_area',
        last_visit: '2025-06-15'
    }]->(l)
$$) AS (e agtype);

-- Referral chains
SELECT * FROM cypher('coworking_community', $$
    MATCH (a:Member {member_id: 'uuid-1'}), (b:Member {member_id: 'uuid-2'})
    CREATE (a)-[:REFERRED {
        date: '2025-01-20',
        plan_type: 'dedicated_desk',
        converted: true
    }]->(b)
$$) AS (e agtype);

-- Community membership
SELECT * FROM cypher('coworking_community', $$
    MATCH (m:Member {member_id: 'uuid-1'}), (c:Community {name: 'AI Enthusiasts'})
    CREATE (m)-[:MEMBER_OF {joined: '2025-06-01', role: 'member'}]->(c)
$$) AS (e agtype);

-- Collaboration edges (derived from shared booking rooms, coworking proximity)
SELECT * FROM cypher('coworking_community', $$
    MATCH (a:Member {member_id: 'uuid-1'}), (b:Member {member_id: 'uuid-2'})
    CREATE (a)-[:COLLABORATED_WITH {
        meeting_count: 5,
        last_meeting: '2025-06-10',
        shared_bookings: 3
    }]->(b)
$$) AS (e agtype);
```

### Graph Queries for Co-Working Community Features

```sql
-- MEMBER RECOMMENDATION: Find members with shared interests
-- who are not yet connected (friend-of-friend style)
SELECT * FROM cypher('coworking_community', $$
    MATCH (me:Member {member_id: $member_id})-[:INTERESTED_IN]->(i:Interest)<-[:INTERESTED_IN]-(other:Member)
    WHERE NOT (me)-[:CONNECTED_TO]-(other) AND me <> other
    WITH other, COLLECT(i.name) AS shared_interests, COUNT(i) AS overlap
    ORDER BY overlap DESC
    LIMIT 10
    RETURN other.member_id, other.name, other.company, shared_interests, overlap
$$) AS (member_id agtype, name agtype, company agtype, shared_interests agtype, overlap agtype);

-- COMMUNITY DETECTION: Find clusters of members who frequently
-- attend the same events (co-attendance graph)
SELECT * FROM cypher('coworking_community', $$
    MATCH (a:Member)-[:ATTENDED]->(e:Event)<-[:ATTENDED]-(b:Member)
    WHERE a <> b
    WITH a, b, COUNT(e) AS shared_events
    WHERE shared_events >= 3
    RETURN a.member_id, a.name, b.member_id, b.name, shared_events
    ORDER BY shared_events DESC
$$) AS (a_id agtype, a_name agtype, b_id agtype, b_name agtype, shared_events agtype);

-- REFERRAL CHAIN ANALYSIS: Trace multi-hop referral paths
-- to identify top referrers and viral growth patterns
SELECT * FROM cypher('coworking_community', $$
    MATCH path = (root:Member)-[:REFERRED*1..5]->(leaf:Member)
    WHERE NOT ()-[:REFERRED]->(root)
    WITH root, LENGTH(path) AS chain_depth, COUNT(leaf) AS total_referrals
    RETURN root.member_id, root.name, chain_depth, total_referrals
    ORDER BY total_referrals DESC
    LIMIT 20
$$) AS (member_id agtype, name agtype, depth agtype, referrals agtype);

-- NETWORKING SUGGESTIONS: Find members at the same location
-- in complementary industries who share skills
SELECT * FROM cypher('coworking_community', $$
    MATCH (me:Member {member_id: $member_id})-[:FREQUENTS]->(l:Location)<-[:FREQUENTS]-(other:Member)
    WHERE me <> other AND me.industry <> other.industry
    OPTIONAL MATCH (me)-[:HAS_SKILL]->(s:Skill)<-[:HAS_SKILL]-(other)
    WITH other, l, COLLECT(s.name) AS shared_skills, COUNT(s) AS skill_overlap
    RETURN other.member_id, other.name, other.company, other.industry,
           l.name AS shared_location, shared_skills, skill_overlap
    ORDER BY skill_overlap DESC
    LIMIT 15
$$) AS (member_id agtype, name agtype, company agtype, industry agtype,
        location agtype, shared_skills agtype, skill_overlap agtype);

-- INFLUENCE SCORING: Identify the most connected members
-- across multiple relationship types
SELECT * FROM cypher('coworking_community', $$
    MATCH (m:Member)
    OPTIONAL MATCH (m)-[:CONNECTED_TO]-(connections)
    OPTIONAL MATCH (m)-[:REFERRED]->(referrals)
    OPTIONAL MATCH (m)-[:ATTENDED]->(events)
    OPTIONAL MATCH (m)-[:COLLABORATED_WITH]-(collaborators)
    WITH m,
         COUNT(DISTINCT connections) AS connection_count,
         COUNT(DISTINCT referrals) AS referral_count,
         COUNT(DISTINCT events) AS event_count,
         COUNT(DISTINCT collaborators) AS collab_count
    WITH m,
         connection_count + (referral_count * 3) + event_count + (collab_count * 2) AS influence_score
    RETURN m.member_id, m.name, m.company, influence_score
    ORDER BY influence_score DESC
    LIMIT 25
$$) AS (member_id agtype, name agtype, company agtype, influence_score agtype);
```

---

## Part 4: Real-Time Cache Layer (Redis)

Redis serves as the volatile, sub-millisecond state layer for operations that cannot tolerate database round-trips. It is the only component outside PostgreSQL, serving a bounded role as a cache and real-time coordination layer.

### Availability Matrix

```
Key Pattern:
  avail:{location_id}:{date}:{resource_type}

Value: Redis Bitmap
  Each bit represents a 15-minute slot (96 bits per day)
  Bit 0 = 00:00-00:15, Bit 1 = 00:15-00:30, ..., Bit 95 = 23:45-00:00

Commands:
  SETBIT avail:loc-123:2025-06-20:meeting_room 36 1   # Mark 09:00 slot as booked
  GETBIT avail:loc-123:2025-06-20:meeting_room 36      # Check slot
  BITCOUNT avail:loc-123:2025-06-20:meeting_room        # Count booked slots

TTL: 48 hours after the date passes (auto-cleanup)
```

### Active Session Tracking

```
Key Pattern:
  session:{location_id}:active

Value: Redis Sorted Set
  Score: check-in timestamp (Unix epoch)
  Member: member_id

Commands:
  ZADD session:loc-123:active 1718880000 "member-uuid-456"    # Check in
  ZREM session:loc-123:active "member-uuid-456"                # Check out
  ZCARD session:loc-123:active                                  # Current occupancy
  ZRANGEBYSCORE session:loc-123:active -inf +inf                # All active members
  ZCOUNT session:loc-123:active -inf +inf                       # Count for dashboard

TTL: No expiry (managed by check-out events)
```

### Booking Slot Locks (Preventing Double-Booking Race Conditions)

```
Key Pattern:
  lock:booking:{resource_id}:{slot_hash}

Value: member_id of the lock holder

Commands:
  SET lock:booking:res-789:20250620T0900 "member-uuid-456" NX EX 120
  # NX = only if not exists (atomic lock acquisition)
  # EX 120 = auto-expire after 120 seconds (abandoned lock cleanup)

  DEL lock:booking:res-789:20250620T0900  # Release after confirmed booking
```

### Real-Time Occupancy Counters

```
Key Pattern:
  occupancy:{location_id}:{zone}

Value: Integer (current occupant count)

Commands:
  INCR occupancy:loc-123:open_plan_east        # Sensor detects entry
  DECR occupancy:loc-123:open_plan_east        # Sensor detects exit
  GET occupancy:loc-123:open_plan_east          # Current count
  MGET occupancy:loc-123:*                      # All zones at location

Key Pattern:
  occupancy:capacity:{location_id}:{zone}

Value: Integer (maximum capacity)

Commands:
  SET occupancy:capacity:loc-123:open_plan_east 50
  # Compare GET occupancy vs capacity for utilization percentage
```

### Rate Limiting (API and Booking Abuse Prevention)

```
Key Pattern:
  ratelimit:{action}:{member_id}:{window}

Value: Counter

Commands:
  INCR ratelimit:booking:member-456:20250620-09   # Increment hourly counter
  EXPIRE ratelimit:booking:member-456:20250620-09 3600  # Expire after 1 hour
  GET ratelimit:booking:member-456:20250620-09     # Check against limit
```

### Pub/Sub for Real-Time Notifications

```
Channels:
  notifications:{location_id}          # Location-wide announcements
  notifications:member:{member_id}     # Personal notifications
  occupancy_updates:{location_id}      # Real-time occupancy changes
  booking_updates:{location_id}        # Booking confirmations/cancellations
  access_alerts:{location_id}          # Security alerts (denied access, tailgate)

Message Format (JSON):
  {
    "type": "booking_confirmed",
    "booking_id": "uuid",
    "resource_name": "Meeting Room A",
    "member_id": "uuid",
    "time_range": {"start": "2025-06-20T09:00Z", "end": "2025-06-20T10:00Z"},
    "timestamp": "2025-06-19T14:30:00Z"
  }
```

### Cache Warming and Invalidation Strategy

```
Pattern: Write-Through for Bookings
  1. Application writes booking to PostgreSQL
  2. On success, update Redis availability bitmap
  3. Publish booking event to Redis Pub/Sub channel

Pattern: Read-Through for Member Profiles
  Key: member_profile:{member_id}
  TTL: 15 minutes
  On miss: Query PostgreSQL, cache result in Redis

Pattern: Event-Driven Invalidation
  When membership status changes in PostgreSQL:
  1. Delete cached access permissions: DEL access_perms:{member_id}
  2. Delete cached member profile: DEL member_profile:{member_id}
  3. Publish event to access control channel for smart lock update
```

---

## Part 5: Data Synchronization Between Layers

### Relational-to-Graph Synchronization

The graph layer mirrors a subset of relational data (members, events, companies) as graph nodes. Synchronization is event-driven:

```sql
-- PostgreSQL trigger to queue graph sync on member creation/update
CREATE OR REPLACE FUNCTION sync_member_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    -- Insert into a sync queue table (processed by background worker)
    INSERT INTO graph_sync_queue (
        entity_type, entity_id, operation, payload, created_at
    ) VALUES (
        'member',
        NEW.id,
        CASE WHEN TG_OP = 'INSERT' THEN 'create' ELSE 'update' END,
        jsonb_build_object(
            'member_id', NEW.id,
            'name', NEW.first_name || ' ' || NEW.last_name,
            'company', NEW.company_name,
            'job_title', NEW.job_title,
            'industry', NEW.industry,
            'interests', NEW.interests,
            'skills', NEW.skills,
            'location_id', NEW.preferred_location_id
        ),
        now()
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_member_graph_sync
    AFTER INSERT OR UPDATE ON members
    FOR EACH ROW
    EXECUTE FUNCTION sync_member_to_graph();

-- Graph sync queue table
CREATE TABLE graph_sync_queue (
    id          BIGSERIAL PRIMARY KEY,
    entity_type TEXT NOT NULL,
    entity_id   UUID NOT NULL,
    operation   TEXT NOT NULL,
    payload     JSONB NOT NULL,
    processed   BOOLEAN NOT NULL DEFAULT false,
    processed_at TIMESTAMPTZ,
    error       TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_sync_pending ON graph_sync_queue (created_at)
    WHERE processed = false;
```

### Time-Series to Graph Derivation

Collaboration and co-attendance relationships are derived from time-series data:

```sql
-- Periodic job: derive collaboration edges from shared meeting room bookings
-- Run daily via pg_cron or application scheduler
WITH shared_bookings AS (
    SELECT
        a.member_id AS member_a,
        b.member_id AS member_b,
        COUNT(*) AS shared_count,
        MAX(a.time) AS last_shared
    FROM checkin_events a
    JOIN checkin_events b ON a.resource_id = b.resource_id
        AND a.location_id = b.location_id
        AND a.time >= b.time - INTERVAL '30 minutes'
        AND a.time <= b.time + INTERVAL '30 minutes'
        AND a.member_id < b.member_id  -- avoid duplicates
        AND a.event_type = 'checkin'
        AND b.event_type = 'checkin'
    WHERE a.time > now() - INTERVAL '30 days'
        AND a.resource_id IS NOT NULL
    GROUP BY a.member_id, b.member_id
    HAVING COUNT(*) >= 2
)
-- Queue these for graph edge creation/update
INSERT INTO graph_sync_queue (entity_type, entity_id, operation, payload, created_at)
SELECT
    'collaboration',
    uuid_generate_v4(),
    'upsert',
    jsonb_build_object(
        'member_a', member_a,
        'member_b', member_b,
        'meeting_count', shared_count,
        'last_meeting', last_shared
    ),
    now()
FROM shared_bookings;
```

### Time-Series to Graph: Location Affinity

```sql
-- Derive FREQUENTS relationships from check-in data
WITH visit_stats AS (
    SELECT
        member_id,
        location_id,
        COUNT(*) AS visit_count,
        AVG(duration_minutes)::REAL / 60.0 AS avg_hours,
        MAX(time) AS last_visit
    FROM checkin_events
    WHERE event_type = 'checkin'
        AND time > now() - INTERVAL '90 days'
    GROUP BY member_id, location_id
    HAVING COUNT(*) >= 3
)
INSERT INTO graph_sync_queue (entity_type, entity_id, operation, payload, created_at)
SELECT
    'location_affinity',
    uuid_generate_v4(),
    'upsert',
    jsonb_build_object(
        'member_id', member_id,
        'location_id', location_id,
        'visit_count', visit_count,
        'avg_duration_hours', avg_hours,
        'last_visit', last_visit
    ),
    now()
FROM visit_stats;
```

---

## Pros and Cons

### Pros

1. **Each workload uses its optimal storage paradigm.** Financial transactions get ACID relational integrity. Sensor streams get automatic time-partitioning, columnar compression, and continuous aggregates. Community relationships get native graph traversals. Real-time state gets sub-millisecond in-memory access. No single storage engine is asked to do what it was not designed for.

2. **PostgreSQL remains the single source of truth.** TimescaleDB and Apache AGE are PostgreSQL extensions, not separate databases. They share the same backup, replication, connection pool, monitoring, and transaction model. An operator running `pg_dump` captures relational tables, hypertables, and graph data in one backup.

3. **Continuous aggregates eliminate expensive analytical queries.** Occupancy dashboards, utilization reports, and environmental comfort scores are pre-computed and incrementally refreshed. The operator dashboard loads in milliseconds even with millions of raw sensor readings because it queries aggregates, not raw data.

4. **Graph queries solve problems that are intractable in SQL.** Finding "members who share 3+ interests with me, attend the same events, and are at my location today" requires multi-hop traversals across heterogeneous relationship types. In Cypher, this is a natural, readable query. In SQL, it would require multiple self-joins, CTEs, and careful optimization that degrades as the dataset grows.

5. **Compression and retention automate data lifecycle management.** Raw sensor data compresses 10-20x after 7 days and is automatically dropped after 90 days, while aggregates persist indefinitely. The operator never manages partitions, archive tables, or cleanup jobs.

6. **Redis provides real-time coordination without database contention.** Booking slot locks, live occupancy counters, and session tracking happen in memory without competing for PostgreSQL connections. The bitmap-based availability matrix gives O(1) slot availability checks.

7. **The architecture scales horizontally at the right points.** Time-series data (the highest-volume component) can scale via TimescaleDB's multi-node distributed hypertables. Redis can scale via Redis Cluster. The relational and graph layers scale vertically with PostgreSQL, which is sufficient for all but the largest multi-site operators.

8. **AI/ML workloads have direct access to pre-structured data.** Occupancy forecasting models consume continuous aggregates directly. Community detection algorithms operate on the graph layer natively. Churn prediction models combine relational membership data with graph-derived engagement scores and time-series check-in patterns -- all queryable from a single PostgreSQL connection.

### Cons

1. **Operational complexity exceeds a pure PostgreSQL deployment.** Even though TimescaleDB and Apache AGE are PostgreSQL extensions, they must be installed, configured, upgraded, and monitored as additional components. Redis adds a separate system with its own persistence, replication, and failure modes. The team must understand three query languages (SQL, Cypher, Redis commands).

2. **Data synchronization between layers introduces eventual consistency.** The graph layer is populated asynchronously via a sync queue. Between a member's creation in the relational layer and their appearance as a graph node, recommendation queries will not include them. The sync queue must be monitored for failures, and the system must handle the case where graph data is temporarily stale.

3. **Apache AGE is less mature than standalone Neo4j.** AGE supports a subset of openCypher. Complex graph algorithms (Louvain community detection, PageRank, betweenness centrality) are not built into AGE -- they require either application-level implementation, export to a GDS-capable system, or use of pgRouting for path-based algorithms. For operators who need sophisticated graph analytics, a standalone Neo4j instance may eventually become necessary.

4. **Testing complexity increases.** Integration tests must cover cross-layer scenarios: a booking in the relational layer must update the Redis availability bitmap and eventually create collaboration edges in the graph. Test fixtures must populate three different storage paradigms. CI/CD pipelines need PostgreSQL with extensions and Redis.

5. **TimescaleDB licensing considerations.** TimescaleDB Community Edition is open-source (Timescale License), but some advanced features (multi-node distributed hypertables, continuous aggregate enhancements) require TimescaleDB Enterprise. Self-hosted operators must evaluate which features they need and whether the community edition is sufficient.

6. **Redis as a single point of failure for real-time operations.** If Redis goes down, booking slot locks fail open (risking double-bookings until the PostgreSQL exclusion constraint catches them), real-time occupancy counters become stale, and live availability checks fall back to database queries. Redis Sentinel or Redis Cluster mitigates this but adds operational overhead.

7. **Schema evolution requires coordinated changes.** Adding a new member attribute that should appear in recommendations requires changes in three places: the relational table (ALTER TABLE), the graph sync trigger (update payload), and the graph node creation logic. This coordination overhead is the price of polyglot persistence.

8. **Higher memory requirements.** Redis requires dedicated RAM for the working set. TimescaleDB's chunk management and continuous aggregate materialization consume more memory than plain PostgreSQL. A minimum production deployment needs 16-32 GB RAM versus 8 GB for a pure relational approach.

---

## Technology Recommendations

### PostgreSQL

- **Version**: PostgreSQL 17+ for MERGE command, improved JSON processing, and logical replication improvements.
- **Configuration**: `shared_preload_libraries = 'timescaledb,age'` to load both extensions at startup.
- **Connection pooling**: PgBouncer in transaction mode, with separate pools for OLTP (relational/graph) and analytics (time-series aggregates) workloads.

### TimescaleDB

- **Version**: TimescaleDB 2.17+ (Community Edition) for continuous aggregate improvements, compression enhancements, and UUIDv7 support in retention policies.
- **Chunk interval**: 1-day chunks for sensor data (optimal for daily compression scheduling). 7-day chunks for access and check-in events (lower volume, longer retention).
- **Compression**: Enable after 7 days for all hypertables. Use `segmentby` on location_id and the most common filter column (zone, device_id, member_id) for each table.
- **Continuous aggregates**: Hourly aggregates for dashboards, daily aggregates for reports, weekly/monthly aggregates for trend analysis. Stack aggregates hierarchically (daily built on hourly) for efficiency.

### Apache AGE

- **Version**: Apache AGE 1.5+ for PostgreSQL 17 compatibility.
- **Graph size**: Suitable for up to ~10 million nodes and ~50 million edges (covers even the largest multi-site operators). Beyond this, consider Neo4j GDS for graph analytics with AGE for transactional queries.
- **Indexing**: Create property indexes on frequently queried node properties (member_id, name, location_id) for efficient MATCH operations.
- **Sync frequency**: Near-real-time (< 5 seconds) for member and event creation. Batch (daily) for derived relationships (collaboration, co-attendance).

### Redis

- **Version**: Redis 7.4+ for improved client-side caching, better ACL support, and Redis Functions.
- **Deployment**: Redis Sentinel for high availability in single-site deployments. Redis Cluster for multi-site operators needing geographic distribution.
- **Memory policy**: `allkeys-lfu` eviction policy to keep the most frequently accessed keys when memory pressure occurs.
- **Persistence**: AOF (Append Only File) with `appendfsync everysec` for durability without performance impact. RDB snapshots every 15 minutes as backup.
- **Key expiry discipline**: Every key must have a TTL except sorted sets managed by application logic (active sessions). Stale data in Redis is worse than a cache miss.

### Monitoring Stack

- **PostgreSQL/TimescaleDB**: pg_stat_statements for query performance, timescaledb_information views for chunk and compression status, custom continuous aggregate for hypertable ingestion rates.
- **Apache AGE**: Monitor graph_sync_queue depth and processing latency. Alert on queue depth > 1000 or processing lag > 60 seconds.
- **Redis**: Redis INFO metrics (memory usage, hit rate, connected clients), Pub/Sub channel activity, keyspace notifications for expiry monitoring.
- **Unified**: Prometheus exporters for all three (postgres_exporter, redis_exporter) feeding Grafana dashboards.

---

## Migration and Scaling Considerations

### Phase 1: Start Simple (0-3 Locations)

Begin with PostgreSQL + TimescaleDB only. Skip Apache AGE and Redis initially:

- Use standard PostgreSQL tables for all relational data.
- Use TimescaleDB hypertables for sensor and access event data from day one -- the schema is identical to regular tables, so there is no additional complexity, only the benefit of automatic partitioning.
- Use application-level in-memory caching (e.g., Node.js Map or Python dict) instead of Redis for booking locks and availability.
- Use SQL queries with array overlap operators (`&&`) for basic interest-based member matching instead of graph queries.

```sql
-- Simple SQL-based member matching (Phase 1 alternative to graph)
SELECT m.id, m.first_name, m.last_name, m.company_name,
       m.interests & ARRAY['AI', 'Machine Learning', 'Data Science'] AS shared_interests,
       array_length(m.interests & ARRAY['AI', 'Machine Learning', 'Data Science'], 1) AS overlap
FROM members m
WHERE m.id != $current_member_id
  AND m.organization_id = $org_id
  AND m.interests && ARRAY['AI', 'Machine Learning', 'Data Science']
ORDER BY overlap DESC
LIMIT 10;
```

### Phase 2: Add Real-Time Layer (3-10 Locations)

Introduce Redis when application-level caching becomes insufficient:

- Deploy Redis for booking slot locks (prevents race conditions under concurrent load).
- Move availability bitmaps to Redis for sub-millisecond availability checks.
- Add Redis Pub/Sub for real-time notifications (replaces polling).
- Active session tracking moves to Redis sorted sets.

**Migration path**: No data migration required. Redis is populated from PostgreSQL on first access (read-through) or on the next write (write-through). The application code adds a Redis layer that wraps existing PostgreSQL queries.

### Phase 3: Add Graph Layer (10+ Locations or Community-Heavy Operators)

Introduce Apache AGE when SQL-based member matching becomes too slow or too limited:

- Install Apache AGE extension.
- Run a one-time backfill script to create graph nodes from existing members, events, and companies.
- Deploy the graph sync trigger on the members table.
- Run the collaboration and co-attendance derivation jobs as daily scheduled tasks.
- Gradually migrate recommendation and networking queries from SQL to Cypher.

**Migration path**: The graph layer is additive. No existing data moves. The sync queue ensures new data flows to the graph automatically. Backfill scripts populate historical data:

```sql
-- One-time backfill: create Member nodes from existing relational data
DO $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN SELECT id, first_name || ' ' || last_name AS name,
                      company_name, job_title, industry,
                      preferred_location_id, created_at
               FROM members WHERE is_active = true
    LOOP
        EXECUTE format(
            'SELECT * FROM cypher(''coworking_community'', $$
                CREATE (:Member {
                    member_id: %L,
                    name: %L,
                    company: %L,
                    job_title: %L,
                    industry: %L,
                    location_id: %L,
                    joined_at: %L
                })
            $$) AS (v agtype)',
            rec.id, rec.name, rec.company_name, rec.job_title,
            rec.industry, rec.preferred_location_id, rec.created_at
        );
    END LOOP;
END $$;
```

### Phase 4: Scale for Large Networks (50+ Locations)

- **TimescaleDB multi-node**: Distribute hypertables across multiple PostgreSQL instances for sensor data ingestion at scale. Access nodes route queries; data nodes store chunks. This is transparent to the application -- queries use the same SQL.
- **Redis Cluster**: Shard Redis across multiple nodes for geographic distribution. Each location cluster can have a local Redis replica for sub-millisecond reads with asynchronous replication to the central cluster.
- **Read replicas**: PostgreSQL streaming replicas serve analytics queries and graph traversals without impacting the primary's transactional throughput.
- **Graph layer scaling**: If Apache AGE becomes a bottleneck for complex graph algorithms (community detection on 100K+ members), offload graph analytics to a nightly Neo4j GDS export while keeping AGE for transactional graph queries.

### Capacity Planning

| Component | 1 Location (50 desks) | 10 Locations (500 desks) | 50 Locations (2,500 desks) |
|---|---|---|---|
| **Relational rows** | ~50K | ~500K | ~2.5M |
| **Time-series events/day** | ~100K | ~1M | ~5M |
| **Time-series storage (90 days, compressed)** | ~2 GB | ~20 GB | ~100 GB |
| **Graph nodes** | ~500 | ~5K | ~25K |
| **Graph edges** | ~5K | ~100K | ~1M |
| **Redis memory** | ~50 MB | ~500 MB | ~2 GB |
| **PostgreSQL RAM** | 4 GB | 16 GB | 32-64 GB |
| **Recommended CPU** | 2 cores | 8 cores | 16-32 cores |

### Backup and Disaster Recovery

```bash
# Single backup captures relational, time-series, and graph data
pg_dump --format=custom --compress=9 \
    --file=coworking_full_$(date +%Y%m%d).dump \
    coworking_db

# Redis backup (supplementary -- Redis is reconstructable from PostgreSQL)
redis-cli BGSAVE
cp /var/lib/redis/dump.rdb /backups/redis_$(date +%Y%m%d).rdb

# Recovery priority:
# 1. Restore PostgreSQL (all persistent state)
# 2. Redis will warm from PostgreSQL on first access
# 3. Verify graph sync queue is processing
# 4. Verify continuous aggregates are refreshing
```

### Schema Evolution Workflow

When adding a new feature (e.g., "member preferred working hours"):

1. **Relational**: `ALTER TABLE members ADD COLUMN preferred_hours JSONB DEFAULT '{}';`
2. **Graph sync trigger**: Update the `sync_member_to_graph()` function payload to include `preferred_hours`.
3. **Graph node**: The sync worker adds the property to existing Member nodes on next update.
4. **Redis**: No change unless the feature requires real-time caching.
5. **Time-series**: No change unless the feature generates new event streams.
6. **Continuous aggregates**: No change unless new aggregations are needed.

Most features touch only one or two layers. The key discipline is identifying which layers a feature impacts before implementation and updating all affected sync mechanisms.

---

## Example Cross-Layer Queries

### "Show me the real-time dashboard for Downtown Hub"

```
1. Redis:  GET occupancy:loc-123:*                    → Current occupancy per zone
2. Redis:  ZCARD session:loc-123:active                → Active members count
3. SQL:    SELECT * FROM occupancy_hourly               → Hourly trend chart
           WHERE location_id = 'loc-123'
           AND bucket > now() - INTERVAL '24 hours'
4. SQL:    SELECT * FROM environmental_hourly            → Comfort metrics
           WHERE location_id = 'loc-123'
           AND bucket > now() - INTERVAL '8 hours'
5. SQL:    SELECT * FROM access_hourly                   → Entry/exit patterns
           WHERE location_id = 'loc-123'
           AND bucket > now() - INTERVAL '24 hours'
```

### "Recommend people I should meet today"

```
1. SQL:    Get current member's interests, skills, location from members table
2. Redis:  ZRANGEBYSCORE session:loc-123:active -inf +inf  → Who is here now
3. Cypher: MATCH (me:Member {member_id: $id})-[:INTERESTED_IN]->(i)<-[:INTERESTED_IN]-(other)
           WHERE other.member_id IN $active_member_ids
           AND NOT (me)-[:CONNECTED_TO]-(other)
           RETURN other, collect(i.name) AS shared_interests
           ORDER BY size(shared_interests) DESC
           LIMIT 5
```

### "Generate the monthly utilization report"

```
1. SQL:    SELECT * FROM occupancy_daily                 → Daily occupancy by zone
           WHERE location_id = $loc AND day >= $month_start AND day < $month_end
2. SQL:    SELECT * FROM checkin_daily                    → Member visit patterns
           WHERE location_id = $loc AND day >= $month_start AND day < $month_end
3. SQL:    SELECT resource_type, COUNT(*), SUM(total_price_cents)
           FROM bookings WHERE location_id = $loc        → Revenue per resource type
           AND lower(time_range) >= $month_start
           GROUP BY resource_type
4. Cypher: MATCH (m:Member)-[:FREQUENTS]->(l:Location {location_id: $loc})
           RETURN COUNT(m) AS active_community_size      → Community engagement metric
```

---

## Summary

This polyglot persistence architecture recognizes that co-working space management is a multi-domain problem that benefits from domain-specific storage engines. By using PostgreSQL as the unified platform -- with TimescaleDB for time-series, Apache AGE for graph queries, and Redis for real-time state -- operators get the best of each paradigm without the operational burden of running four separate database systems. The phased migration path allows operators to start simple and add specialized layers only when their scale and feature requirements justify the additional complexity.
