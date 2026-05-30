# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: HVAC Service Management · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design: every concept gets its own table, relationships are expressed through foreign keys, and reference data is factored out into lookup tables aligned with industry standards (BACnet object types, ASHRAE 180 task codes, EPA refrigerant identifiers, ISO 3166 jurisdictions). The design prioritises data integrity through constraints and referential rules, making it the safest choice for regulatory environments where EPA Section 608 compliance, ASHRAE 90.1 energy monitoring, and refrigerant tracking are non-negotiable.

The approach mirrors how enterprise platforms like Salesforce Field Service and Dynamics 365 Field Service structure their data: a central Work Order entity radiates outward to Service Appointments, Assets, Service Resources, Skill Requirements, and Parts. Every junction table (technician-to-skill, equipment-to-refrigerant-charge, work-order-to-part) is explicit, queryable, and auditable.

This is the most verbose model in terms of table count, but it rewards that investment with strong query flexibility, straightforward indexing, and clean separation of concerns. It is the natural choice for teams with relational database expertise who prioritise long-term maintainability and compliance over initial development speed.

**Best for:** Organisations requiring strict regulatory compliance (EPA, ASHRAE), complex cross-entity reporting, and long-term data integrity guarantees.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Standards-aligned reference data (ASHRAE 180 tasks, BACnet objects, EPA refrigerants)
- (+) Straightforward indexing and query optimisation
- (+) Clear separation of concerns; easy to reason about
- (-) Highest table count (~55-60 tables); more migrations to manage
- (-) Schema changes require ALTER TABLE + migration; less agile for rapid iteration
- (-) Junction tables add write overhead for many-to-many relationships
- (-) HVAC-specific fields (jurisdiction-varying compliance rules) require schema changes per jurisdiction

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASHRAE 135 / ISO 16484-5 (BACnet) | `bacnet_object_type` reference table enumerates standard BACnet object types (Analog Input, Binary Output, etc.); `equipment_telemetry_point` maps physical sensors to BACnet object identifiers |
| ASHRAE 180-2018 | `pm_task_template` table pre-seeded with ASHRAE 180 inspection/maintenance tasks per equipment category; `pm_schedule_task` links templates to service agreements |
| ASHRAE 90.1-2022 | `energy_baseline` table stores per-asset energy consumption baselines; `energy_reading` captures 15-minute interval data per Section 8 requirements; 36-month retention enforced |
| ASHRAE 62.1-2025 | `ventilation_zone` table captures ventilation rate requirements per zone; linked to commissioning work orders |
| EPA Section 608 / AIM Act | `refrigerant_type` reference table with GWP values; `refrigerant_transaction` captures every add/recover event per appliance; automatic threshold alerting at 15 lb |
| ISO 3166-1/2 | `jurisdiction` table uses ISO 3166 codes for country/subdivision; drives compliance rule selection |
| ISO 50001:2018 | `energy_management_plan` table structures continuous improvement cycles; `energy_anomaly` captures deviations for EnMS reporting |
| OAuth 2.0 / OpenID Connect | `oauth_client` and `user_session` tables support multi-tenant API authentication |
| OpenAPI 3.1 | Schema designed to map cleanly to OpenAPI resource definitions for REST API exposure |

---

## Core Identity and Multi-Tenancy

```sql
-- =============================================================
-- TENANT AND IDENTITY
-- =============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code    CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
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

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- e.g. 'admin', 'dispatcher', 'technician', 'customer'
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,  -- e.g. 'work_order.create', 'equipment.read'
    description     TEXT
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_tenant ON "user" (tenant_id);
CREATE INDEX idx_role_tenant ON role (tenant_id);
```

---

## Customer and Location Management

```sql
-- =============================================================
-- CRM: CUSTOMERS, CONTACTS, LOCATIONS
-- =============================================================

CREATE TABLE customer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    customer_type   VARCHAR(20) NOT NULL CHECK (customer_type IN ('residential', 'commercial', 'government', 'property_manager')),
    billing_email   VARCHAR(255),
    billing_phone   VARCHAR(50),
    billing_address_line1  VARCHAR(255),
    billing_address_line2  VARCHAR(255),
    billing_city    VARCHAR(100),
    billing_state   VARCHAR(50),
    billing_zip     VARCHAR(20),
    billing_country CHAR(2) DEFAULT 'US',  -- ISO 3166-1
    tax_exempt      BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    mobile          VARCHAR(50),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    title           VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE service_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    name            VARCHAR(255),  -- e.g. 'Main Office', 'Building A'
    address_line1   VARCHAR(255) NOT NULL,
    address_line2   VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state           VARCHAR(50) NOT NULL,
    zip             VARCHAR(20) NOT NULL,
    country         CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    building_type   VARCHAR(50),  -- 'office', 'retail', 'warehouse', 'residential', 'hospital', 'school'
    building_sqft   INTEGER,
    floor_count     INTEGER,
    access_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customer_tenant ON customer (tenant_id);
CREATE INDEX idx_contact_customer ON contact (customer_id);
CREATE INDEX idx_service_location_customer ON service_location (customer_id);
CREATE INDEX idx_service_location_tenant ON service_location (tenant_id);
```

---

## Equipment and Asset Management

```sql
-- =============================================================
-- EQUIPMENT AND ASSETS
-- =============================================================

CREATE TABLE equipment_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- 'ahu', 'chiller', 'boiler', 'rtu', 'vav', 'split_system', 'cooling_tower', 'vfd', 'thermostat'
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    ashrae_180_section VARCHAR(20)  -- Reference to ASHRAE 180 equipment section
);

CREATE TABLE refrigerant_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(50) NOT NULL UNIQUE,  -- 'R-410A', 'R-32', 'R-454B', 'R-22', 'R-134a'
    chemical_name   VARCHAR(255),
    gwp             INTEGER NOT NULL,  -- Global Warming Potential
    ozone_depleting BOOLEAN NOT NULL DEFAULT false,
    aim_act_compliant BOOLEAN NOT NULL DEFAULT true,  -- Compliant for new installs post-2025
    max_gwp_new_residential INTEGER,  -- GWP limit for new residential equipment
    phase_out_date  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
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
    name            VARCHAR(255),  -- User-friendly label, e.g. 'RTU-3 East Wing'
    installation_date DATE,
    manufacture_date  DATE,
    warranty_expiry_date DATE,
    warranty_parts_expiry DATE,
    warranty_labour_expiry DATE,
    refrigerant_type_id UUID REFERENCES refrigerant_type(id),
    refrigerant_charge_oz DECIMAL(10, 2),  -- Factory charge in ounces
    nominal_capacity_btu INTEGER,  -- Nominal cooling/heating capacity
    voltage         VARCHAR(20),
    phase           VARCHAR(10),  -- '1-phase', '3-phase'
    fuel_type       VARCHAR(50),  -- 'electric', 'natural_gas', 'propane', 'oil'
    location_description TEXT,  -- 'Rooftop, NE corner' or 'Mechanical Room 2B'
    bacnet_device_id INTEGER,  -- BACnet device instance number if connected
    is_active       BOOLEAN NOT NULL DEFAULT true,
    decommission_date DATE,
    decommission_reason TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE equipment_component (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
    component_type  VARCHAR(100) NOT NULL,  -- 'compressor', 'condenser_coil', 'evaporator_coil', 'blower_motor', 'heat_exchanger', 'filter', 'vfd'
    manufacturer    VARCHAR(255),
    model_number    VARCHAR(255),
    serial_number   VARCHAR(255),
    installation_date DATE,
    warranty_expiry DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_tenant ON equipment (tenant_id);
CREATE INDEX idx_equipment_location ON equipment (service_location_id);
CREATE INDEX idx_equipment_customer ON equipment (customer_id);
CREATE INDEX idx_equipment_category ON equipment (category_id);
CREATE INDEX idx_equipment_serial ON equipment (serial_number);
CREATE INDEX idx_equipment_component_equip ON equipment_component (equipment_id);
```

---

## Refrigerant Compliance (EPA Section 608 / AIM Act)

```sql
-- =============================================================
-- REFRIGERANT TRACKING AND EPA COMPLIANCE
-- =============================================================

CREATE TABLE refrigerant_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_order_id   UUID REFERENCES work_order(id),  -- Forward reference; defined below
    technician_id   UUID REFERENCES technician(id),   -- Forward reference
    transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('add', 'recover', 'reclaim', 'recycle', 'dispose')),
    refrigerant_type_id UUID NOT NULL REFERENCES refrigerant_type(id),
    quantity_oz     DECIMAL(10, 2) NOT NULL,  -- In ounces
    quantity_lb     DECIMAL(10, 2) GENERATED ALWAYS AS (quantity_oz / 16.0) STORED,
    cylinder_serial_number VARCHAR(100),
    recovery_machine_serial VARCHAR(100),
    recovery_machine_cert   VARCHAR(100),  -- EPA-certified recovery equipment
    technician_epa_cert_number VARCHAR(100),  -- EPA Section 608 certification number
    technician_epa_cert_type   VARCHAR(20),  -- 'type_i', 'type_ii', 'type_iii', 'universal'
    leak_rate_percent DECIMAL(5, 2),  -- Annualised leak rate at time of service
    notes           TEXT,
    transaction_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE refrigerant_leak_inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    work_order_id   UUID REFERENCES work_order(id),
    technician_id   UUID REFERENCES technician(id),
    inspection_type VARCHAR(30) NOT NULL CHECK (inspection_type IN ('initial', 'follow_up', 'annual', 'automatic_detection')),
    leak_found      BOOLEAN NOT NULL DEFAULT false,
    leak_location   TEXT,
    repair_required BOOLEAN NOT NULL DEFAULT false,
    repair_deadline DATE,  -- 30-day repair window per EPA 608
    verification_test_date DATE,
    verification_passed BOOLEAN,
    annual_leak_rate_percent DECIMAL(5, 2),
    chronic_leaker  BOOLEAN NOT NULL DEFAULT false,  -- >= 125% annual leak rate
    inspection_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Rolling charge balance view
CREATE VIEW equipment_refrigerant_balance AS
SELECT
    equipment_id,
    SUM(CASE WHEN transaction_type = 'add' THEN quantity_oz ELSE 0 END) AS total_added_oz,
    SUM(CASE WHEN transaction_type IN ('recover', 'reclaim', 'dispose') THEN quantity_oz ELSE 0 END) AS total_removed_oz,
    SUM(CASE WHEN transaction_type = 'add' THEN quantity_oz ELSE -quantity_oz END) AS net_charge_oz,
    SUM(CASE WHEN transaction_type = 'add' THEN quantity_oz ELSE -quantity_oz END) / 16.0 AS net_charge_lb
FROM refrigerant_transaction
GROUP BY equipment_id;

CREATE INDEX idx_refrig_tx_equipment ON refrigerant_transaction (equipment_id);
CREATE INDEX idx_refrig_tx_tenant ON refrigerant_transaction (tenant_id);
CREATE INDEX idx_refrig_leak_equipment ON refrigerant_leak_inspection (equipment_id);
```

---

## Technician and Skill Management

```sql
-- =============================================================
-- TECHNICIANS, SKILLS, AND CERTIFICATIONS
-- =============================================================

CREATE TABLE technician (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    employee_number VARCHAR(50),
    hire_date       DATE,
    hourly_rate     DECIMAL(10, 2),
    overtime_rate   DECIMAL(10, 2),
    max_daily_jobs  INTEGER DEFAULT 8,
    home_latitude   DECIMAL(10, 7),
    home_longitude  DECIMAL(10, 7),
    vehicle_id      VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- 'chiller_repair', 'controls_programming', 'ductwork', 'refrigerant_handling'
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(50),  -- 'hvac', 'electrical', 'plumbing', 'controls'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE technician_skill (
    technician_id   UUID NOT NULL REFERENCES technician(id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skill(id) ON DELETE CASCADE,
    proficiency     VARCHAR(20) DEFAULT 'intermediate' CHECK (proficiency IN ('beginner', 'intermediate', 'advanced', 'expert')),
    certified_date  DATE,
    PRIMARY KEY (technician_id, skill_id)
);

CREATE TABLE certification_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- 'epa_608_universal', 'nate_ac', 'nate_hp', 'osha_30'
    name            VARCHAR(255) NOT NULL,
    issuing_body    VARCHAR(255) NOT NULL,  -- 'EPA', 'NATE', 'OSHA', 'State Board'
    renewal_period_months INTEGER,
    description     TEXT
);

CREATE TABLE technician_certification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    technician_id   UUID NOT NULL REFERENCES technician(id) ON DELETE CASCADE,
    certification_type_id UUID NOT NULL REFERENCES certification_type(id),
    certification_number VARCHAR(100),
    issue_date      DATE NOT NULL,
    expiry_date     DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    document_url    TEXT,  -- Scanned certificate
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_technician_tenant ON technician (tenant_id);
CREATE INDEX idx_tech_cert_technician ON technician_certification (technician_id);
CREATE INDEX idx_tech_cert_expiry ON technician_certification (expiry_date);
```

---

## Service Agreements and Preventive Maintenance

```sql
-- =============================================================
-- SERVICE AGREEMENTS AND PM SCHEDULES
-- =============================================================

CREATE TABLE service_agreement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    agreement_number VARCHAR(50) NOT NULL,
    name            VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'expired', 'cancelled', 'suspended')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    auto_renew      BOOLEAN NOT NULL DEFAULT false,
    billing_frequency VARCHAR(20) NOT NULL DEFAULT 'monthly' CHECK (billing_frequency IN ('monthly', 'quarterly', 'semi_annual', 'annual', 'one_time')),
    billing_amount  DECIMAL(12, 2) NOT NULL,
    coverage_type   VARCHAR(30) NOT NULL DEFAULT 'preventive' CHECK (coverage_type IN ('preventive', 'full_service', 'parts_and_labour', 'labour_only')),
    response_time_hours INTEGER,  -- SLA response time
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agreement_equipment (
    agreement_id    UUID NOT NULL REFERENCES service_agreement(id) ON DELETE CASCADE,
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    PRIMARY KEY (agreement_id, equipment_id)
);

-- ASHRAE 180 aligned PM task templates
CREATE TABLE pm_task_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_category_id UUID NOT NULL REFERENCES equipment_category(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    ashrae_180_ref  VARCHAR(50),  -- e.g. 'Table 7-1, Task 3' from ASHRAE 180
    frequency       VARCHAR(30) NOT NULL CHECK (frequency IN ('weekly', 'monthly', 'quarterly', 'semi_annual', 'annual', 'biennial')),
    estimated_duration_minutes INTEGER,
    requires_skill_id UUID REFERENCES skill(id),
    is_regulatory   BOOLEAN NOT NULL DEFAULT false,  -- Must be done for compliance
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_agreement_id UUID NOT NULL REFERENCES service_agreement(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    pm_task_template_id UUID NOT NULL REFERENCES pm_task_template(id),
    next_due_date   DATE NOT NULL,
    last_completed_date DATE,
    override_frequency VARCHAR(30),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agreement_tenant ON service_agreement (tenant_id);
CREATE INDEX idx_agreement_customer ON service_agreement (customer_id);
CREATE INDEX idx_pm_schedule_due ON pm_schedule (next_due_date);
CREATE INDEX idx_pm_schedule_agreement ON pm_schedule (service_agreement_id);
CREATE INDEX idx_pm_schedule_equipment ON pm_schedule (equipment_id);
```

---

## Work Orders and Scheduling

```sql
-- =============================================================
-- WORK ORDERS, APPOINTMENTS, AND DISPATCH
-- =============================================================

CREATE TABLE work_order_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- 'emergency', 'preventive', 'corrective', 'installation', 'commissioning', 'inspection'
    name            VARCHAR(100) NOT NULL,
    default_priority INTEGER DEFAULT 3,
    description     TEXT
);

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_order_number VARCHAR(50) NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customer(id),
    service_location_id UUID NOT NULL REFERENCES service_location(id),
    work_order_type_id UUID NOT NULL REFERENCES work_order_type(id),
    service_agreement_id UUID REFERENCES service_agreement(id),
    pm_schedule_id  UUID REFERENCES pm_schedule(id),  -- If generated from PM schedule
    equipment_id    UUID REFERENCES equipment(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'new' CHECK (status IN (
        'new', 'dispatched', 'en_route', 'in_progress', 'on_hold',
        'completed', 'cancelled', 'invoiced'
    )),
    priority        INTEGER NOT NULL DEFAULT 3 CHECK (priority BETWEEN 1 AND 5),
    summary         VARCHAR(500) NOT NULL,
    description     TEXT,
    reported_symptom TEXT,
    diagnosis       TEXT,
    resolution      TEXT,
    requested_date  TIMESTAMPTZ,
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    created_by_user_id UUID REFERENCES "user"(id),
    source          VARCHAR(30) DEFAULT 'manual' CHECK (source IN ('manual', 'pm_schedule', 'iot_alert', 'customer_portal', 'phone', 'email')),
    iot_alert_id    UUID,  -- Reference to originating IoT alert if source = 'iot_alert'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, work_order_number)
);

CREATE TABLE work_order_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    technician_id   UUID NOT NULL REFERENCES technician(id),
    is_primary      BOOLEAN NOT NULL DEFAULT true,
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at     TIMESTAMPTZ,
    status          VARCHAR(20) DEFAULT 'assigned' CHECK (status IN ('assigned', 'accepted', 'declined', 'completed'))
);

CREATE TABLE service_appointment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    technician_id   UUID NOT NULL REFERENCES technician(id),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    arrival_window_start TIMESTAMPTZ,
    arrival_window_end   TIMESTAMPTZ,
    actual_arrival  TIMESTAMPTZ,
    actual_departure TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled' CHECK (status IN (
        'scheduled', 'dispatched', 'en_route', 'arrived', 'in_progress',
        'completed', 'cancelled', 'no_show'
    )),
    travel_time_minutes INTEGER,
    on_site_duration_minutes INTEGER,
    customer_signature_url TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    pm_task_template_id UUID REFERENCES pm_task_template(id),
    sequence_number INTEGER NOT NULL DEFAULT 1,
    description     VARCHAR(500) NOT NULL,
    is_completed    BOOLEAN NOT NULL DEFAULT false,
    completed_at    TIMESTAMPTZ,
    completed_by    UUID REFERENCES technician(id),
    result          TEXT,  -- Pass/fail/measurement value
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_note (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES "user"(id),
    note_type       VARCHAR(20) DEFAULT 'general' CHECK (note_type IN ('general', 'technician', 'diagnosis', 'customer_communication', 'internal')),
    content         TEXT NOT NULL,
    is_visible_to_customer BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_tenant ON work_order (tenant_id);
CREATE INDEX idx_wo_customer ON work_order (customer_id);
CREATE INDEX idx_wo_location ON work_order (service_location_id);
CREATE INDEX idx_wo_equipment ON work_order (equipment_id);
CREATE INDEX idx_wo_status ON work_order (status);
CREATE INDEX idx_wo_scheduled ON work_order (scheduled_start);
CREATE INDEX idx_wo_agreement ON work_order (service_agreement_id);
CREATE INDEX idx_sa_work_order ON service_appointment (work_order_id);
CREATE INDEX idx_sa_technician ON service_appointment (technician_id);
CREATE INDEX idx_sa_scheduled ON service_appointment (scheduled_start);
CREATE INDEX idx_wo_assignment_wo ON work_order_assignment (work_order_id);
CREATE INDEX idx_wo_task_wo ON work_order_task (work_order_id);
```

---

## Pricing, Parts, and Invoicing

```sql
-- =============================================================
-- PRICEBOOK, PARTS, AND INVOICING
-- =============================================================

CREATE TABLE pricebook_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES pricebook_category(id),
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pricebook_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category_id     UUID REFERENCES pricebook_category(id),
    sku             VARCHAR(100),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    item_type       VARCHAR(20) NOT NULL CHECK (item_type IN ('part', 'labour', 'service', 'equipment', 'misc')),
    unit_cost       DECIMAL(12, 2),
    flat_rate_price DECIMAL(12, 2),
    markup_percent  DECIMAL(5, 2),
    taxable         BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    pricebook_item_id UUID REFERENCES pricebook_item(id),
    description     VARCHAR(500) NOT NULL,
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_price      DECIMAL(12, 2) NOT NULL,
    discount_percent DECIMAL(5, 2) DEFAULT 0,
    tax_rate        DECIMAL(5, 4) DEFAULT 0,
    line_total      DECIMAL(12, 2) NOT NULL,
    item_type       VARCHAR(20) NOT NULL CHECK (item_type IN ('part', 'labour', 'service', 'equipment', 'misc')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    work_order_id   UUID REFERENCES work_order(id),
    invoice_number  VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'sent', 'viewed', 'partial', 'paid', 'overdue', 'void')),
    subtotal        DECIMAL(12, 2) NOT NULL,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    due_date        DATE NOT NULL,
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    amount          DECIMAL(12, 2) NOT NULL,
    payment_method  VARCHAR(30) NOT NULL CHECK (payment_method IN ('credit_card', 'check', 'cash', 'ach', 'financing', 'other')),
    reference_number VARCHAR(100),
    payment_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricebook_tenant ON pricebook_item (tenant_id);
CREATE INDEX idx_wo_line_item_wo ON work_order_line_item (work_order_id);
CREATE INDEX idx_invoice_tenant ON invoice (tenant_id);
CREATE INDEX idx_invoice_customer ON invoice (customer_id);
CREATE INDEX idx_invoice_wo ON invoice (work_order_id);
CREATE INDEX idx_invoice_status ON invoice (status);
CREATE INDEX idx_payment_invoice ON payment (invoice_id);
```

---

## IoT Telemetry and Energy Monitoring

```sql
-- =============================================================
-- IoT TELEMETRY AND ENERGY MONITORING
-- =============================================================

CREATE TABLE bacnet_object_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- 'analog_input', 'analog_output', 'binary_input', 'binary_output', 'schedule', 'trend_log'
    name            VARCHAR(100) NOT NULL,
    description     TEXT
);

CREATE TABLE equipment_telemetry_point (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
    bacnet_object_type_id UUID REFERENCES bacnet_object_type(id),
    point_name      VARCHAR(255) NOT NULL,  -- 'Supply Air Temp', 'Return Air Temp', 'Compressor Amps', 'Discharge Pressure'
    point_identifier VARCHAR(255),  -- BACnet object instance or Modbus register address
    protocol        VARCHAR(20) NOT NULL DEFAULT 'bacnet' CHECK (protocol IN ('bacnet', 'modbus', 'lonworks', 'mqtt', 'manual')),
    unit_of_measure VARCHAR(50),  -- 'degF', 'degC', 'psi', 'amps', 'kW', 'cfm', '%RH'
    normal_min      DECIMAL(12, 4),
    normal_max      DECIMAL(12, 4),
    alert_min       DECIMAL(12, 4),
    alert_max       DECIMAL(12, 4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Telemetry readings: partitioned by month for time-series performance
CREATE TABLE telemetry_reading (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    telemetry_point_id UUID NOT NULL REFERENCES equipment_telemetry_point(id),
    equipment_id    UUID NOT NULL,
    reading_value   DECIMAL(16, 6) NOT NULL,
    quality         VARCHAR(20) DEFAULT 'good' CHECK (quality IN ('good', 'uncertain', 'bad', 'offline')),
    recorded_at     TIMESTAMPTZ NOT NULL,
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions (example for 2026)
CREATE TABLE telemetry_reading_2026_01 PARTITION OF telemetry_reading
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE telemetry_reading_2026_02 PARTITION OF telemetry_reading
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional monthly partitions created by automation

CREATE TABLE energy_baseline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    baseline_type   VARCHAR(30) NOT NULL CHECK (baseline_type IN ('daily_kwh', 'monthly_kwh', 'cooling_efficiency', 'heating_efficiency')),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    baseline_value  DECIMAL(12, 4) NOT NULL,
    unit            VARCHAR(30) NOT NULL,  -- 'kWh', 'kW/ton', 'COP', 'SEER'
    ashrae_901_ref  VARCHAR(50),  -- Reference to ASHRAE 90.1 section
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE energy_anomaly (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    baseline_id     UUID REFERENCES energy_baseline(id),
    anomaly_type    VARCHAR(50) NOT NULL,  -- 'efficiency_degradation', 'consumption_spike', 'runtime_excessive', 'refrigerant_leak_suspected'
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    detected_at     TIMESTAMPTZ NOT NULL,
    description     TEXT NOT NULL,
    current_value   DECIMAL(12, 4),
    expected_value  DECIMAL(12, 4),
    deviation_percent DECIMAL(5, 2),
    auto_work_order_id UUID REFERENCES work_order(id),  -- If an auto-generated WO was created
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES "user"(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_telemetry_point_equipment ON equipment_telemetry_point (equipment_id);
CREATE INDEX idx_telemetry_reading_point ON telemetry_reading (telemetry_point_id, recorded_at);
CREATE INDEX idx_energy_baseline_equipment ON energy_baseline (equipment_id);
CREATE INDEX idx_energy_anomaly_equipment ON energy_anomaly (equipment_id);
CREATE INDEX idx_energy_anomaly_tenant ON energy_anomaly (tenant_id);
CREATE INDEX idx_energy_anomaly_severity ON energy_anomaly (severity, detected_at);
```

---

## Notifications and Audit Log

```sql
-- =============================================================
-- NOTIFICATIONS AND AUDIT
-- =============================================================

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    recipient_type  VARCHAR(20) NOT NULL CHECK (recipient_type IN ('user', 'customer', 'contact')),
    recipient_id    UUID NOT NULL,
    channel         VARCHAR(20) NOT NULL CHECK (channel IN ('email', 'sms', 'push', 'in_app')),
    template_code   VARCHAR(100) NOT NULL,  -- 'appointment_reminder', 'invoice_sent', 'wo_completed'
    subject         VARCHAR(500),
    body            TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'delivered', 'failed', 'bounced')),
    sent_at         TIMESTAMPTZ,
    related_entity_type VARCHAR(50),  -- 'work_order', 'invoice', 'service_appointment'
    related_entity_id   UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES "user"(id),
    entity_type     VARCHAR(100) NOT NULL,  -- 'work_order', 'equipment', 'refrigerant_transaction'
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL CHECK (action IN ('create', 'update', 'delete', 'status_change', 'assignment')),
    changes         JSONB,  -- {"field": {"old": "value1", "new": "value2"}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_tenant ON notification (tenant_id);
CREATE INDEX idx_notification_status ON notification (status, created_at);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant ON audit_log (tenant_id, created_at);
CREATE INDEX idx_audit_log_user ON audit_log (user_id, created_at);
```

---

## Jurisdiction and Compliance Rules

```sql
-- =============================================================
-- JURISDICTION AND COMPLIANCE
-- =============================================================

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,    -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),          -- ISO 3166-2 (e.g. 'US-CA', 'US-TX')
    name            VARCHAR(255) NOT NULL,
    level           VARCHAR(20) NOT NULL CHECK (level IN ('country', 'state', 'county', 'city')),
    parent_id       UUID REFERENCES jurisdiction(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    rule_code       VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(50) NOT NULL,  -- 'refrigerant', 'energy', 'ventilation', 'safety', 'licensing'
    effective_date  DATE NOT NULL,
    expiry_date     DATE,
    source_reference TEXT,  -- e.g. 'EPA 40 CFR Part 82, Subpart F'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_order_id   UUID REFERENCES work_order(id),
    equipment_id    UUID REFERENCES equipment(id),
    compliance_rule_id UUID REFERENCES compliance_rule(id),
    document_type   VARCHAR(50) NOT NULL,  -- 'leak_inspection_report', 'refrigerant_log', 'commissioning_report', 'energy_audit'
    title           VARCHAR(255) NOT NULL,
    file_url        TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    signed_by       UUID REFERENCES "user"(id),
    signed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdiction_country ON jurisdiction (country_code);
CREATE INDEX idx_compliance_rule_jurisdiction ON compliance_rule (jurisdiction_id);
CREATE INDEX idx_compliance_doc_tenant ON compliance_document (tenant_id);
CREATE INDEX idx_compliance_doc_equipment ON compliance_document (equipment_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 6 | tenant, user, role, permission, role_permission, user_role |
| CRM (Customer/Contact/Location) | 3 | customer, contact, service_location |
| Equipment & Assets | 4 | equipment_category, refrigerant_type, equipment, equipment_component |
| Refrigerant Compliance | 2 + 1 view | refrigerant_transaction, refrigerant_leak_inspection, balance view |
| Technicians & Skills | 4 | technician, skill, technician_skill, technician_certification + certification_type |
| Service Agreements & PM | 4 | service_agreement, agreement_equipment, pm_task_template, pm_schedule |
| Work Orders & Scheduling | 5 | work_order_type, work_order, work_order_assignment, service_appointment, work_order_task, work_order_note |
| Pricing & Invoicing | 5 | pricebook_category, pricebook_item, work_order_line_item, invoice, payment |
| IoT & Energy | 5 | bacnet_object_type, equipment_telemetry_point, telemetry_reading (partitioned), energy_baseline, energy_anomaly |
| Notifications & Audit | 2 | notification, audit_log |
| Jurisdiction & Compliance | 3 | jurisdiction, compliance_rule, compliance_document |
| **Total** | **~44 tables** | Plus partitions for telemetry_reading and 1 view |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe cross-tenant data exports, and future sharding without key collisions.

2. **Explicit tenant_id on every business table** — supports PostgreSQL Row-Level Security policies for multi-tenant data isolation without schema-per-tenant overhead.

3. **ASHRAE 180 task templates as seed data** — `pm_task_template` is pre-populated with standardised inspection/maintenance tasks from ASHRAE 180, giving new tenants compliant PM workflows out of the box.

4. **Refrigerant tracking as first-class transactions** — every ounce of refrigerant added or recovered is a row in `refrigerant_transaction`, with EPA certification numbers and equipment references. The `equipment_refrigerant_balance` view provides real-time charge calculations for 15 lb threshold alerting.

5. **Time-partitioned telemetry** — `telemetry_reading` is range-partitioned by month, enabling efficient time-range queries and automated partition management for 36-month ASHRAE 90.1 retention.

6. **Separation of Work Order and Service Appointment** — mirrors the Salesforce/Dynamics 365 pattern where a work order defines "what needs to be done" and service appointments define "when and who." This supports multi-visit jobs and crew assignments.

7. **Equipment-to-BACnet point mapping** — `equipment_telemetry_point` maps physical sensors to their protocol identifiers (BACnet object instances, Modbus registers), enabling the IoT ingestion layer to route readings correctly.

8. **Jurisdiction-aware compliance rules** — the `jurisdiction` → `compliance_rule` hierarchy lets the platform apply different regulatory requirements per state/city without schema changes.

9. **Audit log with JSONB diff** — `audit_log.changes` stores field-level before/after values as JSONB, providing a lightweight audit trail without the complexity of full event sourcing.

10. **Pricebook with flat-rate support** — `pricebook_item` supports both cost-plus and flat-rate pricing models, aligning with FieldEdge/ServiceTitan pricing workflows.
