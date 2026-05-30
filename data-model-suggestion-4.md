# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: HVAC Service Management · Created: 2026-05-22

## Philosophy

This model maintains conventional relational tables for day-to-day CRUD operations (work orders, invoicing, scheduling) but adds a property graph layer for the relationship-heavy queries that define HVAC service management: equipment-to-building-to-customer ownership chains, technician-to-skill-to-certification networks, refrigerant flow between cylinders and equipment across service visits, and equipment component dependency trees. The graph is implemented either as PostgreSQL adjacency tables (`graph_node` / `graph_edge`) with Apache AGE extension for Cypher queries, or as a dedicated Neo4j instance federated with the relational store.

HVAC service management is inherently a graph problem. A single rooftop unit connects to ductwork, VAV boxes, thermostats, and a building automation controller. It is covered by a service agreement tied to a customer who manages multiple buildings. The refrigerant in that unit came from a specific cylinder, was added by a certified technician driving a particular vehicle, and the service was triggered by a BACnet alarm that traversed a network path through a gateway. Understanding these relationships — and traversing them efficiently — is what separates this model from the others.

The graph layer excels at queries that are expensive or awkward in pure relational SQL: "find all equipment downstream of chiller CH-1 in the chilled water loop," "show me every piece of equipment that technician Mike has serviced in the last year and the parts he used," "which buildings share the same model of compressor that failed at site X?" These multi-hop traversal queries power AI-driven fault correlation, conflict-of-interest detection (technician servicing equipment they installed), and portfolio-wide failure pattern analysis.

**Best for:** Organisations with complex equipment hierarchies (commercial/industrial HVAC), multi-building portfolios requiring cross-site analytics, or AI-driven fault correlation across interconnected systems.

**Trade-offs:**
- (+) Multi-hop relationship queries execute in milliseconds vs. multiple JOINs
- (+) Equipment dependency trees, refrigerant flow tracing, and technician history are first-class
- (+) Natural fit for AI knowledge graphs powering fault diagnosis and recommendation engines
- (+) Flexible relationship types — new edge types require no schema migration
- (-) Two data stores to maintain (relational + graph) or PostgreSQL AGE extension dependency
- (-) Graph queries (Cypher/GQL) require different developer skills than SQL
- (-) Transaction consistency across relational and graph stores adds complexity
- (-) More infrastructure to deploy and monitor
- (-) Over-engineering for small contractors with simple equipment inventories

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASHRAE 135 / ISO 16484-5 (BACnet) | BACnet device network topology modelled as graph: Device → Network → Gateway → Cloud. Enables traversal queries for "which devices are reachable through gateway X?" |
| ASHRAE 180-2018 | PM task applicability modelled as edges: `(EquipmentCategory)-[:REQUIRES_TASK]->(PMTask)`. New equipment types inherit applicable tasks by graph traversal |
| ASHRAE 90.1-2022 | Energy monitoring relationships: `(Equipment)-[:MEASURED_BY]->(TelemetryPoint)-[:FEEDS]->(EnergyBaseline)`. Anomaly propagation traced through graph |
| EPA Section 608 / AIM Act | Refrigerant provenance as a directed graph: `(Cylinder)-[:CHARGED_INTO]->(Equipment)` with quantity, date, and technician on the edge. Full chain of custody |
| ISO 3166-1/2 | Jurisdiction hierarchy as graph: `(City)-[:IN]->(State)-[:IN]->(Country)`. Compliance rules inherit from parent jurisdictions |
| W3C Web of Things (WoT) | Thing Description model maps naturally to graph: `(Device)-[:HAS_PROPERTY]->(Property)`, `(Device)-[:HAS_ACTION]->(Action)` |

---

## Relational Core (Operational CRUD)

```sql
-- =============================================================
-- RELATIONAL TABLES: DAY-TO-DAY OPERATIONS
-- =============================================================
-- These tables handle the transactional workload: creating work orders,
-- dispatching technicians, processing invoices. They are the system of record
-- for ACID-critical operations.

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    roles           TEXT[] NOT NULL DEFAULT ARRAY['viewer'],
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE customer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    customer_type   VARCHAR(20) NOT NULL,
    billing_email   VARCHAR(255),
    billing_phone   VARCHAR(50),
    tax_exempt      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE service_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    name            VARCHAR(255),
    address_line1   VARCHAR(255) NOT NULL,
    city            VARCHAR(100) NOT NULL,
    state           VARCHAR(50) NOT NULL,
    zip             VARCHAR(20) NOT NULL,
    country         CHAR(2) NOT NULL DEFAULT 'US',
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    jurisdiction_code VARCHAR(10),  -- ISO 3166-2
    building_type   VARCHAR(50),
    building_sqft   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE equipment_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    ashrae_180_section VARCHAR(20),
    description     TEXT
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_location_id UUID NOT NULL REFERENCES service_location(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    category_id     UUID NOT NULL REFERENCES equipment_category(id),
    manufacturer    VARCHAR(255),
    model_number    VARCHAR(255),
    serial_number   VARCHAR(255),
    name            VARCHAR(255),
    installation_date DATE,
    refrigerant_type VARCHAR(50),
    refrigerant_gwp  INTEGER,
    factory_charge_oz DECIMAL(10, 2),
    bacnet_device_id INTEGER,
    type_attributes JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE technician (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    employee_number VARCHAR(50),
    hire_date       DATE,
    hourly_rate     DECIMAL(10, 2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_order_number VARCHAR(50) NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customer(id),
    service_location_id UUID NOT NULL REFERENCES service_location(id),
    equipment_id    UUID REFERENCES equipment(id),
    work_order_type VARCHAR(50) NOT NULL DEFAULT 'corrective',
    status          VARCHAR(30) NOT NULL DEFAULT 'new',
    priority        INTEGER NOT NULL DEFAULT 3,
    summary         VARCHAR(500) NOT NULL,
    description     TEXT,
    diagnosis       TEXT,
    resolution      TEXT,
    source          VARCHAR(30) DEFAULT 'manual',
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    created_by      UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, work_order_number)
);

CREATE TABLE service_agreement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    agreement_number VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    billing_frequency VARCHAR(20) NOT NULL DEFAULT 'monthly',
    billing_amount  DECIMAL(12, 2) NOT NULL,
    coverage_type   VARCHAR(30) NOT NULL DEFAULT 'preventive',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    work_order_id   UUID REFERENCES work_order(id),
    invoice_number  VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    subtotal        DECIMAL(12, 2) NOT NULL,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    due_date        DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

-- Standard indexes
CREATE INDEX idx_customer_tenant ON customer (tenant_id);
CREATE INDEX idx_location_customer ON service_location (customer_id);
CREATE INDEX idx_equipment_tenant ON equipment (tenant_id);
CREATE INDEX idx_equipment_location ON equipment (service_location_id);
CREATE INDEX idx_equipment_serial ON equipment (serial_number);
CREATE INDEX idx_technician_tenant ON technician (tenant_id);
CREATE INDEX idx_wo_tenant ON work_order (tenant_id);
CREATE INDEX idx_wo_status ON work_order (tenant_id, status);
CREATE INDEX idx_wo_customer ON work_order (customer_id);
CREATE INDEX idx_wo_equipment ON work_order (equipment_id);
CREATE INDEX idx_wo_scheduled ON work_order (scheduled_start);
CREATE INDEX idx_invoice_tenant ON invoice (tenant_id);
CREATE INDEX idx_invoice_customer ON invoice (customer_id);
```

---

## Property Graph Layer

```sql
-- =============================================================
-- PROPERTY GRAPH: RELATIONSHIPS AND TRAVERSALS
-- =============================================================
-- This layer captures relationships that are expensive to query
-- in relational SQL: equipment hierarchies, refrigerant flow,
-- technician-equipment-skill networks, and building system topologies.
--
-- Option A: PostgreSQL with Apache AGE extension (recommended for single-store)
-- Option B: Neo4j with CDC sync from relational tables
--
-- The schema below uses PostgreSQL tables that can be queried with
-- standard SQL or with Cypher via Apache AGE.

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types: 'Equipment', 'Component', 'Building', 'Floor', 'Zone',
    -- 'Technician', 'Skill', 'Certification', 'Customer', 'Manufacturer',
    -- 'RefrigerantCylinder', 'BACnetDevice', 'BACnetNetwork', 'Gateway',
    -- 'WorkOrder', 'PMTask', 'Part', 'EquipmentModel', 'Jurisdiction'
    external_id     UUID NOT NULL,  -- References the relational table's PK
    label           VARCHAR(255) NOT NULL,  -- Human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Denormalised key properties for graph query filtering without
    -- joining back to relational tables
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, external_id)
);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types (see catalogue below)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Edge-specific data: quantities, dates, scores
    weight          DECIMAL(10, 4),  -- Optional numeric weight for ranking/pathfinding
    valid_from      TIMESTAMPTZ,     -- Temporal edges: when the relationship started
    valid_to        TIMESTAMPTZ,     -- Temporal edges: when the relationship ended (null = current)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_node_tenant ON graph_node (tenant_id);
CREATE INDEX idx_graph_node_type ON graph_node (node_type);
CREATE INDEX idx_graph_node_external ON graph_node (external_id);
CREATE INDEX idx_graph_node_props ON graph_node USING GIN (properties);
CREATE INDEX idx_graph_edge_source ON graph_edge (source_node_id);
CREATE INDEX idx_graph_edge_target ON graph_edge (target_node_id);
CREATE INDEX idx_graph_edge_type ON graph_edge (edge_type);
CREATE INDEX idx_graph_edge_tenant ON graph_edge (tenant_id);
CREATE INDEX idx_graph_edge_temporal ON graph_edge (valid_from, valid_to);
CREATE INDEX idx_graph_edge_props ON graph_edge USING GIN (properties);
```

### Edge Type Catalogue

```sql
-- =============================================================
-- EDGE TYPE REFERENCE
-- =============================================================
-- Documents all relationship types in the graph.

CREATE TABLE graph_edge_type (
    edge_type       VARCHAR(100) PRIMARY KEY,
    source_node_type VARCHAR(50) NOT NULL,
    target_node_type VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    is_temporal     BOOLEAN NOT NULL DEFAULT false,
    properties_schema JSONB  -- JSON Schema for edge properties
);

-- Seed data for edge types:
INSERT INTO graph_edge_type (edge_type, source_node_type, target_node_type, description, is_temporal) VALUES

-- Equipment topology
('LOCATED_IN',         'Equipment',     'Building',        'Equipment is physically located in a building', false),
('INSTALLED_ON',       'Equipment',     'Floor',           'Equipment installed on a specific floor', false),
('SERVES_ZONE',        'Equipment',     'Zone',            'Equipment serves a thermal zone', false),
('PARENT_OF',          'Equipment',     'Equipment',       'Parent equipment contains child (e.g., AHU contains VFD)', false),
('DOWNSTREAM_OF',      'Equipment',     'Equipment',       'Equipment is downstream in fluid/air loop', false),
('CONTROLLED_BY',      'Equipment',     'Equipment',       'Equipment is controlled by a controller/BAS', false),
('HAS_COMPONENT',      'Equipment',     'Component',       'Equipment has a specific component', false),

-- BACnet network topology
('ON_NETWORK',         'BACnetDevice',  'BACnetNetwork',   'BACnet device is on a specific network', false),
('GATEWAYED_BY',       'BACnetNetwork', 'Gateway',         'Network reaches cloud via this gateway', false),

-- Ownership and service
('OWNED_BY',           'Equipment',     'Customer',        'Equipment is owned by customer', true),
('MANAGED_BY',         'Building',      'Customer',        'Building is managed by customer/property manager', true),
('COVERED_BY',         'Equipment',     'ServiceAgreement','Equipment is covered by agreement', true),

-- Technician network
('HAS_SKILL',          'Technician',    'Skill',           'Technician possesses skill', true),
('HOLDS_CERT',         'Technician',    'Certification',   'Technician holds certification', true),
('SERVICED',           'Technician',    'Equipment',       'Technician performed service on equipment', true),
('ASSIGNED_TO',        'Technician',    'WorkOrder',       'Technician assigned to work order', true),
('INSTALLED',          'Technician',    'Equipment',       'Technician installed equipment', false),

-- Refrigerant flow (directed graph for provenance)
('CHARGED_FROM',       'Equipment',     'RefrigerantCylinder', 'Refrigerant added from cylinder to equipment', false),
('RECOVERED_TO',       'Equipment',     'RefrigerantCylinder', 'Refrigerant recovered from equipment to cylinder', false),

-- Work order relationships
('FOR_EQUIPMENT',      'WorkOrder',     'Equipment',       'Work order is for this equipment', false),
('AT_LOCATION',        'WorkOrder',     'Building',        'Work order is at this building', false),
('GENERATED_FROM',     'WorkOrder',     'WorkOrder',       'Follow-up work order generated from parent', false),
('TRIGGERED_BY_ANOMALY','WorkOrder',    'Equipment',       'Work order auto-generated from anomaly detection', false),

-- Parts and manufacturing
('MANUFACTURED_BY',    'Equipment',     'Manufacturer',    'Equipment made by manufacturer', false),
('USES_PART',          'WorkOrder',     'Part',            'Part used in work order', false),
('COMPATIBLE_WITH',    'Part',          'EquipmentModel',  'Part is compatible with equipment model', false),
('REPLACED_BY',        'Part',          'Part',            'Part superseded by newer part', false),

-- Jurisdiction hierarchy
('IN_JURISDICTION',    'Building',      'Jurisdiction',    'Building is in jurisdiction', false),
('PARENT_JURISDICTION','Jurisdiction',  'Jurisdiction',    'Jurisdiction hierarchy (city→state→country)', false);
```

---

## Refrigerant Provenance Graph

```sql
-- =============================================================
-- REFRIGERANT TRACKING: RELATIONAL + GRAPH
-- =============================================================
-- Relational table for transactional integrity; graph edges for provenance tracing.

CREATE TABLE refrigerant_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_order_id   UUID REFERENCES work_order(id),
    technician_id   UUID REFERENCES technician(id),
    transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('add', 'recover', 'reclaim', 'recycle', 'dispose')),
    refrigerant_type VARCHAR(50) NOT NULL,
    refrigerant_gwp  INTEGER NOT NULL,
    quantity_oz     DECIMAL(10, 2) NOT NULL,
    cylinder_serial VARCHAR(100),
    technician_epa_cert VARCHAR(100),
    technician_epa_cert_type VARCHAR(20),
    recovery_machine_serial VARCHAR(100),
    leak_rate_percent DECIMAL(5, 2),
    notes           TEXT,
    transaction_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Graph edges created by trigger or application code on each transaction:
--
-- For 'add' transaction:
--   (Equipment)-[:CHARGED_FROM {quantity_oz: 48, date: '2026-05-22', work_order: 'WO-142', technician: 'Mike'}]->(Cylinder)
--   (Technician)-[:SERVICED {date: '2026-05-22', action: 'refrigerant_add', work_order: 'WO-142'}]->(Equipment)
--
-- For 'recover' transaction:
--   (Equipment)-[:RECOVERED_TO {quantity_oz: 32, date: '2026-05-22'}]->(Cylinder)
--
-- This enables Cypher queries like:
--   MATCH (e:Equipment)-[:CHARGED_FROM]->(c:Cylinder)-[:CHARGED_FROM]-(other:Equipment)
--   WHERE e.id = $equipment_id
--   RETURN other
--   -- "Find all equipment that shared refrigerant from the same cylinder"
--   -- (useful for contamination tracing)

CREATE INDEX idx_refrig_tx_equipment ON refrigerant_transaction (equipment_id);
CREATE INDEX idx_refrig_tx_tenant ON refrigerant_transaction (tenant_id);
CREATE INDEX idx_refrig_tx_cylinder ON refrigerant_transaction (cylinder_serial);
```

---

## Telemetry (Time-Series)

```sql
-- =============================================================
-- IoT TELEMETRY
-- =============================================================

CREATE TABLE telemetry_reading (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL,
    point_name      VARCHAR(255) NOT NULL,
    reading_value   DECIMAL(16, 6) NOT NULL,
    unit            VARCHAR(50),
    quality         VARCHAR(20) DEFAULT 'good',
    recorded_at     TIMESTAMPTZ NOT NULL,
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE TABLE telemetry_reading_2026_q1 PARTITION OF telemetry_reading
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE telemetry_reading_2026_q2 PARTITION OF telemetry_reading
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE TABLE energy_anomaly (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    anomaly_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL,
    description     TEXT NOT NULL,
    current_value   DECIMAL(12, 4),
    expected_value  DECIMAL(12, 4),
    deviation_percent DECIMAL(5, 2),
    auto_work_order_id UUID REFERENCES work_order(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_telemetry_equip_time ON telemetry_reading (equipment_id, recorded_at);
CREATE INDEX idx_anomaly_equipment ON energy_anomaly (equipment_id);
CREATE INDEX idx_anomaly_tenant ON energy_anomaly (tenant_id);
```

---

## Example Graph Queries

### Equipment dependency tree (Cypher via Apache AGE)

```sql
-- Find all equipment downstream of Chiller CH-1 in the chilled water loop
-- (AHUs, fan coils, etc. that depend on this chiller)
SELECT * FROM cypher('hvac_graph', $$
    MATCH (chiller:Equipment {name: 'CH-1'})-[:DOWNSTREAM_OF*1..5]->(downstream:Equipment)
    RETURN downstream.label, downstream.properties->>'category' AS category
$$) AS (label agtype, category agtype);
```

### Technician service history network

```sql
-- Show all equipment serviced by technician Mike and the skills used
SELECT * FROM cypher('hvac_graph', $$
    MATCH (t:Technician {label: 'Mike Johnson'})-[s:SERVICED]->(e:Equipment)
    WHERE s.properties->>'date' >= '2025-01-01'
    RETURN e.label AS equipment,
           s.properties->>'action' AS service_type,
           s.properties->>'date' AS service_date,
           e.properties->>'category' AS equipment_type
    ORDER BY s.properties->>'date' DESC
$$) AS (equipment agtype, service_type agtype, service_date agtype, equipment_type agtype);
```

### Refrigerant contamination tracing

```sql
-- Find all equipment that received refrigerant from the same cylinder as equipment X
-- (critical for contamination investigations)
SELECT * FROM cypher('hvac_graph', $$
    MATCH (source:Equipment {external_id: '7c9e6679-...'})-[:CHARGED_FROM]->(cyl:RefrigerantCylinder)<-[:CHARGED_FROM]-(other:Equipment)
    WHERE other.external_id <> source.external_id
    RETURN other.label AS equipment,
           other.properties->>'serial_number' AS serial,
           cyl.label AS cylinder
$$) AS (equipment agtype, serial agtype, cylinder agtype);
```

### Equipment affected by a specific part failure pattern

```sql
-- Find all equipment using the same compressor model that failed in equipment X
-- (portfolio-wide risk assessment)
SELECT * FROM cypher('hvac_graph', $$
    MATCH (failed:Equipment {external_id: '7c9e6679-...'})-[:HAS_COMPONENT]->(comp:Component)
    WHERE comp.properties->>'component_type' = 'compressor'
    WITH comp.properties->>'model_number' AS failed_model
    MATCH (other:Equipment)-[:HAS_COMPONENT]->(other_comp:Component)
    WHERE other_comp.properties->>'model_number' = failed_model
      AND other_comp.properties->>'component_type' = 'compressor'
    RETURN other.label AS equipment,
           other.properties->>'serial_number' AS serial,
           other.properties->>'location' AS location
$$) AS (equipment agtype, serial agtype, location agtype);
```

### Building system topology traversal

```sql
-- Map the complete HVAC system topology for a building
SELECT * FROM cypher('hvac_graph', $$
    MATCH (b:Building {external_id: '6ba7b810-...'})<-[:LOCATED_IN]-(e:Equipment)
    OPTIONAL MATCH (e)-[:PARENT_OF]->(child:Equipment)
    OPTIONAL MATCH (e)-[:CONTROLLED_BY]->(controller:Equipment)
    RETURN e.label AS equipment,
           e.properties->>'category' AS type,
           collect(DISTINCT child.label) AS children,
           controller.label AS controlled_by
    ORDER BY e.properties->>'category'
$$) AS (equipment agtype, type agtype, children agtype, controlled_by agtype);
```

### Standard SQL fallback: multi-hop traversal without Cypher

```sql
-- For environments without Apache AGE: recursive CTE for equipment hierarchy
WITH RECURSIVE equipment_tree AS (
    -- Base: start from a specific parent equipment
    SELECT
        gn.external_id AS equipment_id,
        gn.label AS equipment_name,
        gn.properties->>'category' AS category,
        0 AS depth,
        ARRAY[gn.label] AS path
    FROM graph_node gn
    WHERE gn.external_id = '7c9e6679-...'
      AND gn.node_type = 'Equipment'

    UNION ALL

    -- Recursive: follow PARENT_OF edges
    SELECT
        child.external_id,
        child.label,
        child.properties->>'category',
        et.depth + 1,
        et.path || child.label
    FROM equipment_tree et
    JOIN graph_edge ge ON ge.source_node_id = (
        SELECT id FROM graph_node WHERE external_id = et.equipment_id AND node_type = 'Equipment'
    )
    JOIN graph_node child ON child.id = ge.target_node_id
    WHERE ge.edge_type = 'PARENT_OF'
      AND et.depth < 10  -- Safety limit
)
SELECT * FROM equipment_tree ORDER BY depth, equipment_name;
```

---

## Graph Sync Mechanism

```sql
-- =============================================================
-- GRAPH SYNC: KEEPING GRAPH IN SYNC WITH RELATIONAL TABLES
-- =============================================================

-- Change data capture table — populated by triggers on relational tables
CREATE TABLE graph_sync_queue (
    id              BIGSERIAL PRIMARY KEY,
    operation       VARCHAR(10) NOT NULL CHECK (operation IN ('upsert_node', 'upsert_edge', 'delete_node', 'delete_edge')),
    node_type       VARCHAR(50),
    edge_type       VARCHAR(100),
    source_table    VARCHAR(100) NOT NULL,
    source_id       UUID NOT NULL,
    payload         JSONB NOT NULL,
    processed       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ
);

CREATE INDEX idx_sync_queue_unprocessed ON graph_sync_queue (processed, created_at)
    WHERE processed = false;

-- Example trigger: sync equipment inserts/updates to graph
-- CREATE OR REPLACE FUNCTION sync_equipment_to_graph()
-- RETURNS TRIGGER AS $$
-- BEGIN
--     INSERT INTO graph_sync_queue (operation, node_type, source_table, source_id, payload)
--     VALUES (
--         'upsert_node',
--         'Equipment',
--         'equipment',
--         NEW.id,
--         jsonb_build_object(
--             'tenant_id', NEW.tenant_id,
--             'label', COALESCE(NEW.name, NEW.serial_number, NEW.id::text),
--             'properties', jsonb_build_object(
--                 'category', (SELECT code FROM equipment_category WHERE id = NEW.category_id),
--                 'manufacturer', NEW.manufacturer,
--                 'model_number', NEW.model_number,
--                 'serial_number', NEW.serial_number,
--                 'refrigerant_type', NEW.refrigerant_type,
--                 'bacnet_device_id', NEW.bacnet_device_id,
--                 'is_active', NEW.is_active
--             )
--         )
--     );
--     RETURN NEW;
-- END;
-- $$ LANGUAGE plpgsql;
--
-- CREATE TRIGGER trg_equipment_graph_sync
--     AFTER INSERT OR UPDATE ON equipment
--     FOR EACH ROW EXECUTE FUNCTION sync_equipment_to_graph();
```

---

## Audit Log

```sql
-- =============================================================
-- AUDIT LOG
-- =============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, occurred_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenant, user |
| CRM | 2 | customer, service_location |
| Equipment | 2 | equipment_category, equipment |
| Technicians | 1 | technician |
| Work Orders | 1 | work_order |
| Service Agreements | 1 | service_agreement |
| Invoicing | 1 | invoice |
| Refrigerant Compliance | 1 | refrigerant_transaction |
| Graph Layer | 4 | graph_node, graph_edge, graph_edge_type, graph_sync_queue |
| Telemetry | 2 | telemetry_reading (partitioned), energy_anomaly |
| Audit | 1 | audit_log |
| **Total** | **~18 relational + 4 graph = 22 tables** | Graph layer adds the relationship intelligence |

---

## Key Design Decisions

1. **Relational for transactions, graph for relationships** — the relational tables handle ACID-critical operations (creating work orders, processing payments, recording refrigerant transactions). The graph layer is a derived view optimised for traversal queries. If the graph is lost, it can be rebuilt from relational data.

2. **Property graph with JSONB properties** — both `graph_node.properties` and `graph_edge.properties` use JSONB to store denormalised key attributes. This allows graph queries to filter without joining back to relational tables for common predicates.

3. **Temporal edges with valid_from/valid_to** — relationships like `SERVICED`, `COVERED_BY`, and `HAS_SKILL` have time ranges. This lets queries like "who was the technician certified to handle R-410A on the date this work was performed?" work correctly even after certifications expire or skills are updated.

4. **Edge type catalogue** — the `graph_edge_type` table documents all valid relationship types with their source/target node types and expected properties. This serves as a living schema for the graph layer and prevents arbitrary edge types from proliferating.

5. **Refrigerant provenance as a directed graph** — the `CHARGED_FROM` and `RECOVERED_TO` edges create a complete chain of custody for refrigerant. This enables contamination tracing, cylinder inventory auditing, and EPA compliance reporting through graph traversal rather than multi-table joins.

6. **Equipment hierarchy via PARENT_OF and DOWNSTREAM_OF edges** — a chiller connects to AHUs which connect to VAV boxes. These relationships are trivial to query in a graph but require recursive CTEs in pure SQL. The graph layer makes topology queries fast and readable.

7. **Change data capture sync** — a `graph_sync_queue` table, populated by database triggers, ensures the graph layer stays consistent with relational tables. A background worker processes the queue, upserting nodes and edges. This async approach avoids slowing down transactional writes.

8. **Apache AGE for single-store deployment** — Apache AGE adds Cypher query support to PostgreSQL, eliminating the need for a separate Neo4j instance. For teams that want graph query power without the operational overhead of a second database, AGE provides Cypher queries directly alongside SQL in the same PostgreSQL database.

9. **Compatible parts network** — the `COMPATIBLE_WITH` and `REPLACED_BY` edges create a parts compatibility graph. When a technician needs a replacement part, the system can traverse the graph to find compatible alternatives and superseding part numbers — enabling smarter parts recommendations.

10. **AI knowledge graph foundation** — the property graph layer serves as the foundation for an AI knowledge graph. Fault patterns, equipment relationships, technician expertise, and part compatibility form a rich knowledge base that LLM-powered agents can query via Cypher to provide contextual recommendations during diagnosis and dispatch.
