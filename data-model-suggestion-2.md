# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: HVAC Service Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to an event store. The current state of any entity (work order, equipment, refrigerant charge) is derived by replaying its event stream. Read-optimised materialised views (CQRS pattern) serve the dispatch board, mobile app, and dashboards, while the event store serves as the single source of truth for compliance, audit, and AI analytics.

This approach is particularly well-suited to HVAC service management because of three domain characteristics: (1) EPA Section 608 and AIM Act compliance demand provable, tamper-evident records of every refrigerant transaction and leak inspection, (2) ASHRAE 90.1 requires 36 months of energy monitoring data at 15-minute intervals, and (3) AI-driven fault diagnosis benefits from rich event sequences showing how equipment state evolves over time. An event-sourced model gives all three capabilities natively.

The pattern is proven in domains with similar regulatory rigour. Financial trading systems, healthcare EHRs, and supply chain platforms use event sourcing to guarantee complete audit trails. For HVAC service management, the event store captures every dispatch, technician arrival, diagnosis, refrigerant addition, and invoice — creating a dataset that AI models can train on to predict equipment failures and optimise maintenance schedules.

**Best for:** Organisations that need provable audit trails for regulatory compliance (EPA, ASHRAE), want to leverage AI/ML on rich historical event data, or need to answer temporal queries ("what was the state of equipment X on date Y?").

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is a first-class record
- (+) Temporal queries are trivial: replay events to any point in time
- (+) AI/ML-ready: event sequences are natural training data for predictive models
- (+) EPA compliance is inherent: refrigerant events cannot be modified or deleted
- (+) Easy to add new event types without schema migration
- (-) Higher storage requirements — events accumulate indefinitely
- (-) Read queries require materialised views (CQRS); more infrastructure complexity
- (-) Eventual consistency between event store and read models
- (-) Steeper learning curve for developers unfamiliar with event sourcing
- (-) Debugging requires understanding event replay, not just "look at the row"

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASHRAE 135 / ISO 16484-5 (BACnet) | Telemetry readings ingested as `TelemetryRecorded` events with BACnet object identifiers; event stream enables full replay of sensor history |
| ASHRAE 180-2018 | PM task completions recorded as `MaintenanceTaskCompleted` events with ASHRAE 180 task references; compliance derived by replaying task events against schedules |
| ASHRAE 90.1-2022 | Energy readings are events; 36-month retention is natural (events are never deleted); baselines computed from event aggregation |
| EPA Section 608 / AIM Act | Every refrigerant add/recover is a `RefrigerantTransacted` event with technician cert, equipment ID, and quantity — immutable by design, satisfying EPA recordkeeping |
| ISO 3166-1/2 | Jurisdiction codes embedded in tenant and location events |
| ISO 50001:2018 | Energy management improvement cycles tracked as event sequences; deviation events feed EnMS reporting |
| OCSF (Open Cybersecurity Schema Framework) | Audit events follow OCSF-inspired structured event schema with category, activity, severity, and actor fields |

---

## Event Store Core

```sql
-- =============================================================
-- EVENT STORE: THE SINGLE SOURCE OF TRUTH
-- =============================================================

-- Central event store — all domain events are appended here
CREATE TABLE event_store (
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,         -- Aggregate root ID (e.g. work_order_id, equipment_id)
    stream_type     VARCHAR(100) NOT NULL,  -- 'WorkOrder', 'Equipment', 'ServiceAgreement', 'RefrigerantCharge'
    event_type      VARCHAR(100) NOT NULL,  -- 'WorkOrderCreated', 'TechnicianDispatched', 'RefrigerantAdded'
    event_version   INTEGER NOT NULL,       -- Sequence number within the stream (optimistic concurrency)
    tenant_id       UUID NOT NULL,
    actor_id        UUID,                   -- User or system that caused the event
    actor_type      VARCHAR(20) DEFAULT 'user' CHECK (actor_type IN ('user', 'system', 'iot_gateway', 'scheduler')),
    payload         JSONB NOT NULL,         -- Event-specific data
    metadata        JSONB,                  -- Correlation IDs, source IP, device info
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
) PARTITION BY RANGE (occurred_at);

-- Monthly partitions for event store
CREATE TABLE event_store_2026_01 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE event_store_2026_02 PARTITION OF event_store
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by automation

-- Unique constraint on event_id for global lookups
CREATE UNIQUE INDEX idx_event_store_event_id ON event_store (event_id);
CREATE INDEX idx_event_store_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_event_store_type ON event_store (event_type, occurred_at);
CREATE INDEX idx_event_store_tenant ON event_store (tenant_id, occurred_at);
CREATE INDEX idx_event_store_actor ON event_store (actor_id, occurred_at);

-- Snapshot table for performance: stores periodic aggregate state snapshots
-- to avoid replaying entire event streams for long-lived aggregates
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,  -- event_version at time of snapshot
    state           JSONB NOT NULL,     -- Serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Type Registry

```sql
-- =============================================================
-- EVENT TYPE CATALOGUE
-- =============================================================

-- Documents all known event types and their expected payload schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,  -- JSON Schema defining the expected payload structure
    version         INTEGER NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types that would be registered:
--
-- WorkOrder stream:
--   WorkOrderCreated, WorkOrderUpdated, WorkOrderDispatched, WorkOrderAccepted,
--   WorkOrderStarted, WorkOrderCompleted, WorkOrderCancelled, WorkOrderInvoiced,
--   TechnicianAssigned, TechnicianUnassigned, TaskCompleted, NoteAdded,
--   LineItemAdded, LineItemRemoved, DiagnosisRecorded, ResolutionRecorded
--
-- Equipment stream:
--   EquipmentRegistered, EquipmentUpdated, EquipmentDecommissioned,
--   ComponentReplaced, WarrantyUpdated, TelemetryPointConfigured
--
-- RefrigerantCharge stream (per equipment):
--   RefrigerantAdded, RefrigerantRecovered, RefrigerantReclaimed,
--   LeakInspectionPerformed, LeakDetected, LeakRepaired,
--   ChronicLeakerFlagged, ThresholdBreached
--
-- ServiceAgreement stream:
--   AgreementCreated, AgreementActivated, AgreementRenewed,
--   AgreementCancelled, EquipmentCovered, EquipmentUncovered,
--   PMScheduleGenerated, BillingCycleCompleted
--
-- Telemetry stream (per equipment):
--   TelemetryRecorded, AnomalyDetected, BaselineEstablished,
--   EfficiencyDegraded, AlertTriggered, AlertAcknowledged
```

### Example Event Payloads

```sql
-- Example: WorkOrderCreated event payload
-- {
--   "work_order_number": "WO-2026-00142",
--   "customer_id": "550e8400-...",
--   "service_location_id": "6ba7b810-...",
--   "equipment_id": "7c9e6679-...",
--   "work_order_type": "corrective",
--   "priority": 2,
--   "summary": "RTU-3 not cooling — compressor cycling on high-pressure cutout",
--   "reported_symptom": "Building occupants report warm air from east wing vents",
--   "source": "customer_portal",
--   "requested_date": "2026-05-22T14:00:00Z"
-- }

-- Example: RefrigerantAdded event payload
-- {
--   "refrigerant_type": "R-410A",
--   "refrigerant_gwp": 2088,
--   "quantity_oz": 48,
--   "quantity_lb": 3.0,
--   "running_total_lb": 18.5,
--   "cylinder_serial": "CYL-20240918-A",
--   "recovery_machine_serial": "RM-2023-445",
--   "technician_epa_cert": "608-UNI-2019-44521",
--   "technician_epa_cert_type": "universal",
--   "leak_rate_percent": 12.5,
--   "threshold_15lb_exceeded": true,
--   "aim_act_compliant": false,
--   "notes": "Added refrigerant after condenser coil repair. System now above 15lb AIM Act threshold."
-- }

-- Example: AnomalyDetected event payload
-- {
--   "anomaly_type": "efficiency_degradation",
--   "severity": "warning",
--   "telemetry_point_id": "a1b2c3d4-...",
--   "point_name": "Compressor Amps",
--   "current_value": 18.7,
--   "expected_value": 14.2,
--   "deviation_percent": 31.7,
--   "baseline_id": "e5f6a7b8-...",
--   "probable_cause": "Dirty condenser coil or low refrigerant charge",
--   "recommended_action": "Schedule inspection — check condenser coil and refrigerant charge",
--   "auto_work_order_generated": true
-- }
```

---

## Materialised Read Models (CQRS)

These tables are projections built by event handlers that consume the event store. They are disposable and can be rebuilt from events at any time.

```sql
-- =============================================================
-- READ MODEL: CURRENT STATE PROJECTIONS
-- =============================================================

-- Materialised from WorkOrder stream events
CREATE TABLE rm_work_order (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    work_order_number VARCHAR(50) NOT NULL,
    customer_id     UUID NOT NULL,
    customer_name   VARCHAR(255),  -- Denormalised for read performance
    service_location_id UUID NOT NULL,
    location_address TEXT,          -- Denormalised
    equipment_id    UUID,
    equipment_name  VARCHAR(255),  -- Denormalised
    work_order_type VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    priority        INTEGER NOT NULL,
    summary         VARCHAR(500),
    diagnosis       TEXT,
    resolution      TEXT,
    source          VARCHAR(30),
    primary_technician_id UUID,
    primary_technician_name VARCHAR(255),  -- Denormalised
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    total_amount    DECIMAL(12, 2),
    invoice_status  VARCHAR(20),
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_version INTEGER NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Materialised from Equipment + RefrigerantCharge + Telemetry stream events
CREATE TABLE rm_equipment (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    customer_name   VARCHAR(255),
    service_location_id UUID NOT NULL,
    location_address TEXT,
    category_code   VARCHAR(50),
    category_name   VARCHAR(100),
    manufacturer    VARCHAR(255),
    model_number    VARCHAR(255),
    serial_number   VARCHAR(255),
    name            VARCHAR(255),
    installation_date DATE,
    warranty_expiry DATE,
    refrigerant_type VARCHAR(50),
    refrigerant_gwp INTEGER,
    current_charge_lb DECIMAL(10, 2),
    annual_leak_rate_percent DECIMAL(5, 2),
    is_aim_act_compliant BOOLEAN,
    is_chronic_leaker BOOLEAN DEFAULT false,
    last_service_date DATE,
    next_pm_due_date DATE,
    active_work_order_count INTEGER DEFAULT 0,
    total_work_order_count INTEGER DEFAULT 0,
    bacnet_device_id INTEGER,
    telemetry_status VARCHAR(20),  -- 'online', 'offline', 'degraded'
    last_telemetry_at TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_event_version INTEGER NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Materialised from ServiceAgreement stream events
CREATE TABLE rm_service_agreement (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    customer_name   VARCHAR(255),
    agreement_number VARCHAR(50),
    name            VARCHAR(255),
    status          VARCHAR(20),
    start_date      DATE,
    end_date        DATE,
    billing_frequency VARCHAR(20),
    billing_amount  DECIMAL(12, 2),
    coverage_type   VARCHAR(30),
    covered_equipment_count INTEGER DEFAULT 0,
    overdue_pm_count INTEGER DEFAULT 0,
    next_pm_due     DATE,
    total_revenue   DECIMAL(12, 2) DEFAULT 0,
    last_event_version INTEGER NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

-- Dispatch board projection: optimised for the real-time scheduling view
CREATE TABLE rm_dispatch_board (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    date            DATE NOT NULL,
    technician_id   UUID NOT NULL,
    technician_name VARCHAR(255) NOT NULL,
    work_order_id   UUID NOT NULL,
    work_order_number VARCHAR(50),
    customer_name   VARCHAR(255),
    location_address TEXT,
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    status          VARCHAR(30) NOT NULL,
    priority        INTEGER,
    work_order_type VARCHAR(50),
    equipment_name  VARCHAR(255),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Refrigerant compliance dashboard projection
CREATE TABLE rm_refrigerant_compliance (
    equipment_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    equipment_name  VARCHAR(255),
    location_address TEXT,
    refrigerant_type VARCHAR(50),
    refrigerant_gwp INTEGER,
    factory_charge_lb DECIMAL(10, 2),
    current_charge_lb DECIMAL(10, 2),
    total_added_lb  DECIMAL(10, 2),
    total_recovered_lb DECIMAL(10, 2),
    annual_leak_rate_percent DECIMAL(5, 2),
    is_chronic_leaker BOOLEAN DEFAULT false,
    exceeds_15lb_threshold BOOLEAN DEFAULT false,
    aim_act_compliant BOOLEAN,
    last_leak_inspection DATE,
    next_leak_inspection_due DATE,
    open_repair_deadline DATE,
    transaction_count INTEGER DEFAULT 0,
    last_transaction_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_wo_tenant_status ON rm_work_order (tenant_id, status);
CREATE INDEX idx_rm_wo_scheduled ON rm_work_order (scheduled_start);
CREATE INDEX idx_rm_wo_technician ON rm_work_order (primary_technician_id);
CREATE INDEX idx_rm_equip_tenant ON rm_equipment (tenant_id);
CREATE INDEX idx_rm_equip_location ON rm_equipment (service_location_id);
CREATE INDEX idx_rm_equip_serial ON rm_equipment (serial_number);
CREATE INDEX idx_rm_dispatch_date ON rm_dispatch_board (tenant_id, date, technician_id);
CREATE INDEX idx_rm_refrig_tenant ON rm_refrigerant_compliance (tenant_id);
CREATE INDEX idx_rm_refrig_chronic ON rm_refrigerant_compliance (is_chronic_leaker) WHERE is_chronic_leaker = true;
```

---

## Reference Data Tables

```sql
-- =============================================================
-- REFERENCE DATA (not event-sourced; static/seed data)
-- =============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE equipment_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    ashrae_180_section VARCHAR(20),
    description     TEXT
);

CREATE TABLE refrigerant_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(50) NOT NULL UNIQUE,
    gwp             INTEGER NOT NULL,
    ozone_depleting BOOLEAN NOT NULL DEFAULT false,
    aim_act_compliant BOOLEAN NOT NULL DEFAULT true,
    phase_out_date  DATE
);

CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(50)
);

CREATE TABLE certification_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    issuing_body    VARCHAR(255) NOT NULL,
    renewal_period_months INTEGER
);

CREATE TABLE pm_task_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_category_id UUID NOT NULL REFERENCES equipment_category(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    ashrae_180_ref  VARCHAR(50),
    frequency       VARCHAR(30) NOT NULL,
    estimated_duration_minutes INTEGER,
    is_regulatory   BOOLEAN NOT NULL DEFAULT false
);
```

---

## Telemetry Event Ingestion

```sql
-- =============================================================
-- TELEMETRY: HIGH-FREQUENCY EVENTS
-- =============================================================

-- Telemetry uses a separate event table for performance isolation
-- (telemetry events vastly outnumber business events)
CREATE TABLE telemetry_event (
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    point_name      VARCHAR(255) NOT NULL,
    point_identifier VARCHAR(255),  -- BACnet object instance or Modbus register
    protocol        VARCHAR(20) NOT NULL,
    reading_value   DECIMAL(16, 6) NOT NULL,
    unit_of_measure VARCHAR(50),
    quality         VARCHAR(20) DEFAULT 'good',
    recorded_at     TIMESTAMPTZ NOT NULL,
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions
CREATE TABLE telemetry_event_2026_01 PARTITION OF telemetry_event
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional partitions

-- Downsampled aggregates for long-term queries
CREATE TABLE telemetry_aggregate (
    equipment_id    UUID NOT NULL,
    point_name      VARCHAR(255) NOT NULL,
    bucket_start    TIMESTAMPTZ NOT NULL,
    bucket_duration INTERVAL NOT NULL,  -- '15 minutes', '1 hour', '1 day'
    min_value       DECIMAL(16, 6),
    max_value       DECIMAL(16, 6),
    avg_value       DECIMAL(16, 6),
    sample_count    INTEGER NOT NULL,
    PRIMARY KEY (equipment_id, point_name, bucket_start, bucket_duration)
);

CREATE INDEX idx_telemetry_event_equip ON telemetry_event (equipment_id, recorded_at);
CREATE INDEX idx_telemetry_agg_equip ON telemetry_aggregate (equipment_id, point_name, bucket_start);
```

---

## Event Processing Infrastructure

```sql
-- =============================================================
-- EVENT PROCESSING: SUBSCRIPTIONS AND PROJECTIONS
-- =============================================================

-- Tracks where each projection/consumer has read up to
CREATE TABLE event_subscription (
    subscription_id VARCHAR(100) PRIMARY KEY,  -- 'rm_work_order_projector', 'notification_handler', 'ai_feature_pipeline'
    description     TEXT,
    last_processed_event_id UUID,
    last_processed_at TIMESTAMPTZ,
    lag_events      INTEGER DEFAULT 0,
    status          VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'paused', 'rebuilding', 'error')),
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed processing
CREATE TABLE event_dead_letter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id VARCHAR(100) NOT NULL REFERENCES event_subscription(subscription_id),
    event_id        UUID NOT NULL,
    stream_id       UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 3,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_dead_letter_subscription ON event_dead_letter (subscription_id, created_at);
CREATE INDEX idx_dead_letter_unresolved ON event_dead_letter (resolved_at) WHERE resolved_at IS NULL;
```

---

## Example Queries

### Replay equipment state at a specific point in time

```sql
-- "What was the refrigerant charge of equipment X on 2026-03-15?"
SELECT payload
FROM event_store
WHERE stream_id = '7c9e6679-...'
  AND stream_type = 'RefrigerantCharge'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- Application code replays these events to compute the charge at that date
```

### Find all equipment that had refrigerant added in the last 90 days

```sql
SELECT DISTINCT stream_id AS equipment_id,
       payload->>'refrigerant_type' AS refrigerant,
       payload->>'quantity_lb' AS quantity_lb,
       occurred_at
FROM event_store
WHERE event_type = 'RefrigerantAdded'
  AND tenant_id = 'abc123...'
  AND occurred_at >= now() - INTERVAL '90 days'
ORDER BY occurred_at DESC;
```

### Build a work order timeline

```sql
-- Complete timeline of a single work order
SELECT event_type,
       actor_id,
       payload,
       occurred_at
FROM event_store
WHERE stream_id = '550e8400-...'
  AND stream_type = 'WorkOrder'
ORDER BY event_version ASC;

-- Returns: WorkOrderCreated → TechnicianAssigned → WorkOrderDispatched →
--          WorkOrderStarted → DiagnosisRecorded → RefrigerantAdded →
--          TaskCompleted → LineItemAdded → WorkOrderCompleted → WorkOrderInvoiced
```

### AI training data extraction

```sql
-- Extract fault-to-resolution event sequences for ML training
SELECT
    e1.stream_id AS equipment_id,
    e1.payload->>'anomaly_type' AS fault_type,
    e1.payload->>'probable_cause' AS predicted_cause,
    e1.occurred_at AS fault_detected_at,
    e2.payload->>'diagnosis' AS actual_diagnosis,
    e2.payload->>'resolution' AS resolution,
    e2.occurred_at AS resolved_at,
    EXTRACT(EPOCH FROM (e2.occurred_at - e1.occurred_at)) / 3600 AS hours_to_resolve
FROM event_store e1
JOIN event_store e2 ON e2.payload->>'equipment_id' = e1.stream_id::text
    AND e2.event_type = 'ResolutionRecorded'
    AND e2.occurred_at > e1.occurred_at
    AND e2.occurred_at < e1.occurred_at + INTERVAL '30 days'
WHERE e1.event_type = 'AnomalyDetected'
  AND e1.tenant_id = 'abc123...'
ORDER BY e1.occurred_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store Core | 3 | event_store (partitioned), event_snapshot, event_type_registry |
| Read Models | 5 | rm_work_order, rm_equipment, rm_service_agreement, rm_dispatch_board, rm_refrigerant_compliance |
| Reference Data | 8 | tenant, user, equipment_category, refrigerant_type, skill, certification_type, pm_task_template + 1 more |
| Telemetry | 2 | telemetry_event (partitioned), telemetry_aggregate |
| Event Processing | 2 | event_subscription, event_dead_letter |
| **Total** | **~20 tables** | Plus partitions; read models are disposable/rebuildable |

---

## Key Design Decisions

1. **Single event store table** — all business domain events go into one partitioned table. This simplifies cross-aggregate queries (e.g., "show me all events for customer X across work orders, equipment, and agreements") and enables a single event bus for downstream consumers.

2. **Separate telemetry event table** — IoT sensor readings are orders of magnitude more voluminous than business events (potentially millions per day vs. thousands). Isolating them prevents telemetry from overwhelming the business event store.

3. **JSONB payloads with schema registry** — event payloads are stored as JSONB for flexibility, but the `event_type_registry` table documents the expected JSON Schema for each event type. This balances schema evolution (new fields can be added without migration) with documentation and validation.

4. **Snapshots for long-lived aggregates** — equipment that has been in service for years may accumulate thousands of events. Periodic snapshots in `event_snapshot` allow state reconstruction from the most recent snapshot + subsequent events, keeping replay time bounded.

5. **Read models are disposable** — every `rm_*` table can be dropped and rebuilt from the event store. This means read model schema can evolve independently of the event schema — a new dashboard requirement just needs a new projector, not a data migration.

6. **Event subscriptions with dead-letter queue** — each consumer (projector, notification handler, AI pipeline) tracks its position in the event stream via `event_subscription`. Failed events go to `event_dead_letter` for retry, preventing a single bad event from blocking the pipeline.

7. **Immutable refrigerant events satisfy EPA Section 608** — once a `RefrigerantAdded` or `RefrigerantRecovered` event is written, it cannot be modified or deleted. The event payload includes EPA certification numbers, quantities, and cylinder serials — exactly what an EPA inspector would request.

8. **Temporal queries are first-class** — "what was the state of this equipment on March 15?" is answered by replaying events up to that date. No need for slowly-changing-dimension patterns or bi-temporal columns.

9. **AI pipeline as a first-class event consumer** — the AI/ML feature extraction pipeline is just another event subscription. It processes `AnomalyDetected`, `DiagnosisRecorded`, and `ResolutionRecorded` events to build training datasets for predictive maintenance models.

10. **Optimistic concurrency via event_version** — the `(stream_id, event_version)` primary key prevents concurrent writes from corrupting aggregate state. If two dispatchers try to assign the same work order simultaneously, the second write fails and can be retried with the updated state.
