# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: HVAC Service Management · Created: 2026-05-22

## Philosophy

This model keeps the structural backbone relational — foreign keys enforce relationships between customers, equipment, work orders, and technicians — but uses PostgreSQL JSONB columns for fields that vary by equipment type, jurisdiction, or tenant configuration. The core idea is that 80% of the schema is stable and benefits from relational constraints, while the remaining 20% (jurisdiction-specific compliance fields, equipment-type-specific attributes, tenant custom fields) belongs in typed JSONB columns with GIN indexes.

This approach is particularly well-suited to HVAC service management because equipment attributes vary dramatically by type: a chiller has tonnage, condenser water flow rate, and refrigerant circuit count; a VAV box has minimum/maximum airflow CFM and reheat coil type; a boiler has BTU input rating and combustion efficiency. In a normalised model, each equipment type would need its own attributes table or a sprawling EAV pattern. With JSONB, a single `equipment.type_attributes` column accommodates all of them, with JSON Schema validation ensuring data quality per equipment category.

The hybrid approach also handles the multi-jurisdiction compliance challenge elegantly. EPA Section 608 refrigerant tracking is federal, but states like California (CARB), New York, and Massachusetts have additional requirements. Rather than adding nullable columns for each jurisdiction's rules, a `compliance_context` JSONB column captures jurisdiction-specific data alongside the universal relational fields.

**Best for:** Teams building an MVP or early product that must handle diverse equipment types, multi-jurisdiction compliance, and tenant-specific customisation without constant schema migrations.

**Trade-offs:**
- (+) Rapid development: new equipment attributes or compliance fields require no migration
- (+) Per-equipment-type flexibility without EAV complexity
- (+) Tenant custom fields without schema changes
- (+) JSON Schema validation provides structure within JSONB columns
- (+) GIN indexes on JSONB columns enable efficient containment queries
- (-) JSONB fields lack foreign key constraints — referential integrity is application-enforced
- (-) Complex JSONB queries can be slower than indexed relational columns
- (-) JSONB contents are less discoverable than explicit columns for new developers
- (-) Reporting tools (BI, SQL clients) handle JSONB less ergonomically than flat columns
- (-) Risk of schema drift if JSON Schema validation is not enforced consistently

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASHRAE 135 / ISO 16484-5 (BACnet) | `telemetry_config` JSONB on equipment stores BACnet device instance, object list, and COV subscription config per equipment type |
| ASHRAE 180-2018 | PM task templates stored relationally; equipment-type-specific task parameters (measurements, tolerances) in `task_params` JSONB |
| ASHRAE 90.1-2022 | Energy monitoring configuration in `equipment.energy_config` JSONB — baseline kWh, monitoring interval, ASHRAE 90.1 section reference |
| EPA Section 608 / AIM Act | Core refrigerant fields (type, quantity, cert number) relational; jurisdiction-specific extensions (CARB rules, state leak repair timelines) in `compliance_context` JSONB |
| ISO 3166-1/2 | `service_location.jurisdiction_code` uses ISO 3166-2 codes to drive compliance rule selection |
| ISO 50001:2018 | Energy management plan structure in JSONB — flexible enough to model Plan-Do-Check-Act cycles per ISO 50001 |
| JSON Schema Draft 2020-12 | Every JSONB column has a corresponding JSON Schema document defining its expected structure and validation rules |

---

## Core Tables with JSONB Extensions

```sql
-- =============================================================
-- TENANT, IDENTITY, AND CONFIGURATION
-- =============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "work_order_prefix": "WO",
    --   "auto_number_start": 1000,
    --   "default_tax_rate": 0.0825,
    --   "require_customer_signature": true,
    --   "refrigerant_alert_threshold_lb": 15,
    --   "accounting_integration": "quickbooks_online",
    --   "notification_channels": ["email", "sms"],
    --   "custom_fields_schema": { ... }  -- Tenant-defined custom field definitions
    -- }
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
    -- PostgreSQL array for roles: ['admin', 'dispatcher', 'technician']
    -- Simpler than junction tables for most HVAC companies with <20 users
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions example:
    -- {
    --   "work_orders": ["create", "read", "update"],
    --   "equipment": ["read", "update"],
    --   "invoicing": ["read"],
    --   "refrigerant": ["create", "read"]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user" (tenant_id);
CREATE INDEX idx_user_roles ON "user" USING GIN (roles);
```

---

## Customer and Location

```sql
-- =============================================================
-- CRM: CUSTOMERS AND SERVICE LOCATIONS
-- =============================================================

CREATE TABLE customer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    customer_type   VARCHAR(20) NOT NULL CHECK (customer_type IN ('residential', 'commercial', 'government', 'property_manager')),
    billing_address JSONB NOT NULL DEFAULT '{}',
    -- billing_address example:
    -- {
    --   "line1": "123 Main St",
    --   "line2": "Suite 400",
    --   "city": "Austin",
    --   "state": "TX",
    --   "zip": "78701",
    --   "country": "US"
    -- }
    contact_info    JSONB NOT NULL DEFAULT '{}',
    -- contact_info example:
    -- {
    --   "primary_email": "john@acme.com",
    --   "billing_email": "ap@acme.com",
    --   "phone": "+15125551234",
    --   "contacts": [
    --     {"name": "John Smith", "title": "Facilities Manager", "email": "john@acme.com", "phone": "+15125551234", "is_primary": true},
    --     {"name": "Jane Doe", "title": "AP Clerk", "email": "jane@acme.com", "phone": "+15125555678"}
    --   ]
    -- }
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Tenant-defined custom fields validated against tenant.settings.custom_fields_schema
    tax_exempt      BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE service_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    name            VARCHAR(255),
    address         JSONB NOT NULL,
    -- address example:
    -- {
    --   "line1": "456 Commerce Blvd",
    --   "city": "Austin",
    --   "state": "TX",
    --   "zip": "78702",
    --   "country": "US",
    --   "latitude": 30.2672,
    --   "longitude": -97.7431
    -- }
    jurisdiction_code VARCHAR(10),  -- ISO 3166-2, e.g. 'US-TX' — drives compliance rule selection
    building_info   JSONB NOT NULL DEFAULT '{}',
    -- building_info example:
    -- {
    --   "type": "office",
    --   "sqft": 45000,
    --   "floors": 3,
    --   "year_built": 1998,
    --   "building_automation_system": "Tridium Niagara",
    --   "bacnet_network_id": 1001,
    --   "access_instructions": "Check in at security desk. Roof access via stairwell C.",
    --   "operating_hours": {"weekdays": "07:00-19:00", "weekends": "closed"}
    -- }
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customer_tenant ON customer (tenant_id);
CREATE INDEX idx_customer_type ON customer (customer_type);
CREATE INDEX idx_customer_contacts ON customer USING GIN (contact_info);
CREATE INDEX idx_location_customer ON service_location (customer_id);
CREATE INDEX idx_location_tenant ON service_location (tenant_id);
CREATE INDEX idx_location_jurisdiction ON service_location (jurisdiction_code);
```

---

## Equipment with Type-Specific JSONB Attributes

```sql
-- =============================================================
-- EQUIPMENT AND ASSETS
-- =============================================================

CREATE TABLE equipment_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    ashrae_180_section VARCHAR(20),
    attribute_schema JSONB NOT NULL DEFAULT '{}',
    -- JSON Schema defining the expected structure of equipment.type_attributes
    -- for this category. Example for 'chiller':
    -- {
    --   "type": "object",
    --   "properties": {
    --     "tonnage": {"type": "number"},
    --     "chiller_type": {"type": "string", "enum": ["air_cooled", "water_cooled"]},
    --     "compressor_type": {"type": "string", "enum": ["scroll", "screw", "centrifugal", "reciprocating"]},
    --     "refrigerant_circuits": {"type": "integer"},
    --     "condenser_water_gpm": {"type": "number"},
    --     "chilled_water_gpm": {"type": "number"},
    --     "design_cop": {"type": "number"}
    --   },
    --   "required": ["tonnage", "chiller_type"]
    -- }
    description     TEXT
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_location_id UUID NOT NULL REFERENCES service_location(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    category_id     UUID NOT NULL REFERENCES equipment_category(id),

    -- Universal relational fields (present for all equipment types)
    manufacturer    VARCHAR(255),
    model_number    VARCHAR(255),
    serial_number   VARCHAR(255),
    name            VARCHAR(255),
    installation_date DATE,
    warranty_info   JSONB NOT NULL DEFAULT '{}',
    -- warranty_info example:
    -- {
    --   "parts_expiry": "2031-06-15",
    --   "labour_expiry": "2027-06-15",
    --   "compressor_expiry": "2036-06-15",
    --   "warranty_provider": "Carrier",
    --   "warranty_number": "WRN-2026-44521",
    --   "registration_date": "2026-06-20"
    -- }

    -- Refrigerant info (relational for EPA compliance querying)
    refrigerant_type VARCHAR(50),      -- 'R-410A', 'R-32', 'R-454B'
    refrigerant_gwp  INTEGER,
    factory_charge_oz DECIMAL(10, 2),

    -- Type-specific attributes in JSONB (validated against equipment_category.attribute_schema)
    type_attributes JSONB NOT NULL DEFAULT '{}',
    -- Example for a Rooftop Unit (RTU):
    -- {
    --   "cooling_capacity_btu": 120000,
    --   "heating_capacity_btu": 150000,
    --   "seer_rating": 16.5,
    --   "hspf_rating": 9.0,
    --   "economizer_type": "differential_enthalpy",
    --   "stages_cooling": 2,
    --   "stages_heating": 2,
    --   "supply_fan_hp": 3,
    --   "voltage": "460V",
    --   "phase": "3-phase",
    --   "filter_size": "20x25x4",
    --   "filter_count": 4
    -- }
    --
    -- Example for a Variable Air Volume (VAV) Box:
    -- {
    --   "min_cfm": 200,
    --   "max_cfm": 1200,
    --   "reheat_type": "hot_water",
    --   "reheat_coil_kw": null,
    --   "damper_actuator": "Belimo LRB24-3-T",
    --   "zone_served": "East Wing 3rd Floor"
    -- }

    -- IoT/telemetry configuration
    telemetry_config JSONB NOT NULL DEFAULT '{}',
    -- telemetry_config example:
    -- {
    --   "protocol": "bacnet",
    --   "bacnet_device_id": 100042,
    --   "bacnet_network": 1001,
    --   "points": [
    --     {"name": "Supply Air Temp", "object_type": "analog_input", "instance": 1, "unit": "degF", "normal_range": [50, 65]},
    --     {"name": "Return Air Temp", "object_type": "analog_input", "instance": 2, "unit": "degF", "normal_range": [68, 78]},
    --     {"name": "Compressor Status", "object_type": "binary_input", "instance": 1},
    --     {"name": "Discharge Pressure", "object_type": "analog_input", "instance": 5, "unit": "psi", "alert_max": 425}
    --   ],
    --   "polling_interval_seconds": 300,
    --   "cov_subscriptions": ["Supply Air Temp", "Compressor Status"]
    -- }

    location_description TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    decommission_date DATE,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_tenant ON equipment (tenant_id);
CREATE INDEX idx_equipment_location ON equipment (service_location_id);
CREATE INDEX idx_equipment_customer ON equipment (customer_id);
CREATE INDEX idx_equipment_serial ON equipment (serial_number);
CREATE INDEX idx_equipment_refrigerant ON equipment (refrigerant_type);
CREATE INDEX idx_equipment_type_attrs ON equipment USING GIN (type_attributes);
CREATE INDEX idx_equipment_telemetry ON equipment USING GIN (telemetry_config);
```

---

## Technicians

```sql
-- =============================================================
-- TECHNICIANS
-- =============================================================

CREATE TABLE technician (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    employee_number VARCHAR(50),
    hire_date       DATE,
    hourly_rate     DECIMAL(10, 2),
    skills          JSONB NOT NULL DEFAULT '[]',
    -- skills example:
    -- [
    --   {"code": "chiller_repair", "proficiency": "expert", "certified_date": "2020-03-15"},
    --   {"code": "controls_programming", "proficiency": "advanced"},
    --   {"code": "refrigerant_handling", "proficiency": "expert"}
    -- ]
    certifications  JSONB NOT NULL DEFAULT '[]',
    -- certifications example:
    -- [
    --   {"type": "epa_608_universal", "number": "608-UNI-2019-44521", "issued": "2019-06-01", "expires": null},
    --   {"type": "nate_ac", "number": "NATE-AC-2024-1122", "issued": "2024-01-15", "expires": "2026-01-15"},
    --   {"type": "osha_30", "number": "OSHA30-2023-8891", "issued": "2023-09-01", "expires": null}
    -- ]
    vehicle_info    JSONB NOT NULL DEFAULT '{}',
    -- {"vehicle_id": "VH-012", "type": "van", "license_plate": "TX-ABC-1234"}
    home_location   JSONB,
    -- {"latitude": 30.2672, "longitude": -97.7431}
    max_daily_jobs  INTEGER DEFAULT 8,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_technician_tenant ON technician (tenant_id);
CREATE INDEX idx_technician_skills ON technician USING GIN (skills);
CREATE INDEX idx_technician_certs ON technician USING GIN (certifications);
```

---

## Service Agreements and PM

```sql
-- =============================================================
-- SERVICE AGREEMENTS AND PREVENTIVE MAINTENANCE
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
    billing         JSONB NOT NULL DEFAULT '{}',
    -- billing example:
    -- {
    --   "frequency": "quarterly",
    --   "amount": 1250.00,
    --   "payment_terms_days": 30,
    --   "auto_invoice": true,
    --   "prorate_first_period": true,
    --   "tax_rate": 0.0825
    -- }
    coverage        JSONB NOT NULL DEFAULT '{}',
    -- coverage example:
    -- {
    --   "type": "full_service",
    --   "response_time_hours": 4,
    --   "includes_parts": true,
    --   "parts_limit_annual": 5000.00,
    --   "includes_refrigerant": true,
    --   "after_hours_coverage": true,
    --   "excluded_equipment_types": ["cooling_tower"]
    -- }
    equipment_ids   UUID[] NOT NULL DEFAULT '{}',
    -- PostgreSQL array of covered equipment IDs
    notes           TEXT,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_task_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_category_id UUID NOT NULL REFERENCES equipment_category(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    ashrae_180_ref  VARCHAR(50),
    frequency       VARCHAR(30) NOT NULL,
    estimated_duration_minutes INTEGER,
    is_regulatory   BOOLEAN NOT NULL DEFAULT false,
    task_params     JSONB NOT NULL DEFAULT '{}',
    -- task_params example for "Check refrigerant charge":
    -- {
    --   "measurements": [
    --     {"name": "Suction Pressure", "unit": "psi", "expected_range": [60, 80]},
    --     {"name": "Discharge Pressure", "unit": "psi", "expected_range": [200, 350]},
    --     {"name": "Superheat", "unit": "degF", "expected_range": [8, 15]},
    --     {"name": "Subcooling", "unit": "degF", "expected_range": [8, 14]}
    --   ],
    --   "tools_required": ["manifold_gauges", "thermometer", "leak_detector"],
    --   "safety_notes": "Ensure power is locked out before accessing electrical compartment"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_agreement_id UUID NOT NULL REFERENCES service_agreement(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    pm_task_template_id UUID NOT NULL REFERENCES pm_task_template(id),
    next_due_date   DATE NOT NULL,
    last_completed  DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agreement_tenant ON service_agreement (tenant_id);
CREATE INDEX idx_agreement_customer ON service_agreement (customer_id);
CREATE INDEX idx_agreement_status ON service_agreement (status);
CREATE INDEX idx_agreement_equipment ON service_agreement USING GIN (equipment_ids);
CREATE INDEX idx_pm_schedule_due ON pm_schedule (next_due_date);
```

---

## Work Orders

```sql
-- =============================================================
-- WORK ORDERS
-- =============================================================

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_order_number VARCHAR(50) NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customer(id),
    service_location_id UUID NOT NULL REFERENCES service_location(id),
    equipment_id    UUID REFERENCES equipment(id),
    service_agreement_id UUID REFERENCES service_agreement(id),
    pm_schedule_id  UUID REFERENCES pm_schedule(id),

    -- Core relational fields
    work_order_type VARCHAR(50) NOT NULL DEFAULT 'corrective',
    status          VARCHAR(30) NOT NULL DEFAULT 'new' CHECK (status IN (
        'new', 'dispatched', 'en_route', 'in_progress', 'on_hold',
        'completed', 'cancelled', 'invoiced'
    )),
    priority        INTEGER NOT NULL DEFAULT 3 CHECK (priority BETWEEN 1 AND 5),
    source          VARCHAR(30) DEFAULT 'manual',
    summary         VARCHAR(500) NOT NULL,

    -- Flexible detail fields
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "reported_symptom": "Unit not cooling. Warm air from supply vents.",
    --   "diagnosis": "Low refrigerant charge. Found leak at service valve Schrader core.",
    --   "resolution": "Replaced Schrader core, pressure tested, added 3 lb R-410A.",
    --   "fault_codes": ["E04", "HP_CUTOUT"],
    --   "ai_diagnosis_confidence": 0.87,
    --   "ai_suggested_parts": ["Schrader core", "Valve cap"],
    --   "measurements": {
    --     "suction_pressure_psi": 72,
    --     "discharge_pressure_psi": 310,
    --     "superheat_degf": 18,
    --     "subcooling_degf": 6,
    --     "supply_air_temp_degf": 58
    --   }
    -- }

    -- Scheduling
    scheduling      JSONB NOT NULL DEFAULT '{}',
    -- scheduling example:
    -- {
    --   "requested_date": "2026-05-22T14:00:00Z",
    --   "scheduled_start": "2026-05-23T08:00:00Z",
    --   "scheduled_end": "2026-05-23T10:00:00Z",
    --   "actual_start": "2026-05-23T08:15:00Z",
    --   "actual_end": "2026-05-23T09:45:00Z",
    --   "arrival_window": {"start": "2026-05-23T08:00:00Z", "end": "2026-05-23T10:00:00Z"},
    --   "technician_ids": ["uuid1", "uuid2"],
    --   "primary_technician_id": "uuid1",
    --   "travel_time_minutes": 22
    -- }

    -- Compliance context (jurisdiction-specific)
    compliance_context JSONB NOT NULL DEFAULT '{}',
    -- compliance_context example (for a job in California):
    -- {
    --   "jurisdiction": "US-CA",
    --   "epa_608_applicable": true,
    --   "carb_applicable": true,
    --   "carb_registration_required": true,
    --   "refrigerant_recovery_cert": "608-UNI-2019-44521",
    --   "refrigerant_transactions": [
    --     {"type": "recover", "refrigerant": "R-410A", "quantity_oz": 8, "cylinder_serial": "CYL-001"},
    --     {"type": "add", "refrigerant": "R-410A", "quantity_oz": 56, "cylinder_serial": "CYL-002"}
    --   ],
    --   "leak_inspection_performed": true,
    --   "leak_rate_percent": 8.5,
    --   "customer_signature_url": "https://storage.example.com/signatures/wo-142.png"
    -- }

    -- Financials
    financials      JSONB NOT NULL DEFAULT '{}',
    -- financials example:
    -- {
    --   "line_items": [
    --     {"description": "Diagnostic fee", "type": "service", "qty": 1, "unit_price": 89.00, "total": 89.00},
    --     {"description": "Schrader core replacement", "type": "part", "sku": "SC-1420", "qty": 1, "unit_price": 12.50, "total": 12.50},
    --     {"description": "R-410A refrigerant (3 lb)", "type": "part", "sku": "R410A-LB", "qty": 3, "unit_price": 45.00, "total": 135.00},
    --     {"description": "Labour (1.5 hrs)", "type": "labour", "qty": 1.5, "unit_price": 125.00, "total": 187.50}
    --   ],
    --   "subtotal": 424.00,
    --   "tax_rate": 0.0825,
    --   "tax_amount": 34.98,
    --   "total": 458.98,
    --   "payment_status": "pending",
    --   "invoice_id": "inv-uuid"
    -- }

    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, work_order_number)
);

-- Status history is tracked as a JSONB array on the work order itself
-- (avoids a separate status_history table for simpler querying)
-- Application appends to this array on each status change:
-- work_order.details->'status_history' = [
--   {"status": "new", "at": "2026-05-22T10:00:00Z", "by": "user-uuid"},
--   {"status": "dispatched", "at": "2026-05-22T10:30:00Z", "by": "user-uuid"},
--   {"status": "in_progress", "at": "2026-05-23T08:15:00Z", "by": "tech-uuid"}
-- ]

CREATE INDEX idx_wo_tenant ON work_order (tenant_id);
CREATE INDEX idx_wo_customer ON work_order (customer_id);
CREATE INDEX idx_wo_status ON work_order (tenant_id, status);
CREATE INDEX idx_wo_equipment ON work_order (equipment_id);
CREATE INDEX idx_wo_type ON work_order (work_order_type);
CREATE INDEX idx_wo_details ON work_order USING GIN (details);
CREATE INDEX idx_wo_compliance ON work_order USING GIN (compliance_context);
CREATE INDEX idx_wo_financials ON work_order USING GIN (financials);
```

---

## Invoicing

```sql
-- =============================================================
-- INVOICING AND PAYMENTS
-- =============================================================

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    customer_id     UUID NOT NULL REFERENCES customer(id),
    work_order_id   UUID REFERENCES work_order(id),
    service_agreement_id UUID REFERENCES service_agreement(id),
    invoice_number  VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'sent', 'viewed', 'partial', 'paid', 'overdue', 'void')),
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- Mirrors work_order.financials.line_items or agreement billing
    subtotal        DECIMAL(12, 2) NOT NULL,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    due_date        DATE NOT NULL,
    payments        JSONB NOT NULL DEFAULT '[]',
    -- payments example:
    -- [
    --   {"amount": 458.98, "method": "credit_card", "reference": "ch_1234", "date": "2026-05-24T11:00:00Z"}
    -- ]
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoice_tenant ON invoice (tenant_id);
CREATE INDEX idx_invoice_customer ON invoice (customer_id);
CREATE INDEX idx_invoice_status ON invoice (status);
CREATE INDEX idx_invoice_wo ON invoice (work_order_id);
```

---

## Pricebook

```sql
-- =============================================================
-- PRICEBOOK
-- =============================================================

CREATE TABLE pricebook_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    item_type       VARCHAR(20) NOT NULL CHECK (item_type IN ('part', 'labour', 'service', 'equipment', 'misc')),
    pricing         JSONB NOT NULL DEFAULT '{}',
    -- pricing example (supports both cost-plus and flat-rate):
    -- {
    --   "unit_cost": 12.50,
    --   "flat_rate_price": 45.00,
    --   "markup_percent": 260,
    --   "taxable": true,
    --   "tiered_pricing": {
    --     "good": 45.00,
    --     "better": 65.00,
    --     "best": 89.00
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricebook_tenant ON pricebook_item (tenant_id);
CREATE INDEX idx_pricebook_sku ON pricebook_item (sku);
CREATE INDEX idx_pricebook_type ON pricebook_item (item_type);
```

---

## Telemetry and Energy

```sql
-- =============================================================
-- IoT TELEMETRY AND ENERGY MONITORING
-- =============================================================

-- Telemetry readings remain relational+partitioned for time-series performance
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
-- ... quarterly partitions

CREATE TABLE energy_baseline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    baseline_config JSONB NOT NULL,
    -- baseline_config example:
    -- {
    --   "type": "monthly_kwh",
    --   "period": {"start": "2026-01-01", "end": "2026-01-31"},
    --   "baseline_value": 4250.5,
    --   "unit": "kWh",
    --   "ashrae_901_section": "8.4.3",
    --   "deviation_threshold_percent": 15,
    --   "weather_normalized": true,
    --   "cooling_degree_days": 142
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE energy_anomaly (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    baseline_id     UUID REFERENCES energy_baseline(id),
    anomaly_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "current_value": 5200.3,
    --   "expected_value": 4250.5,
    --   "deviation_percent": 22.3,
    --   "probable_cause": "Compressor short-cycling — possible low refrigerant",
    --   "recommended_action": "Inspect refrigerant charge and compressor contactor",
    --   "ai_confidence": 0.82
    -- }
    detected_at     TIMESTAMPTZ NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    auto_work_order_id UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_telemetry_equip_time ON telemetry_reading (equipment_id, recorded_at);
CREATE INDEX idx_energy_baseline_equip ON energy_baseline (equipment_id);
CREATE INDEX idx_energy_anomaly_tenant ON energy_anomaly (tenant_id);
CREATE INDEX idx_energy_anomaly_equip ON energy_anomaly (equipment_id);
```

---

## Audit and Notifications

```sql
-- =============================================================
-- AUDIT LOG AND NOTIFICATIONS
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

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    recipient_type  VARCHAR(20) NOT NULL,
    recipient_id    UUID NOT NULL,
    channel         VARCHAR(20) NOT NULL,
    template_code   VARCHAR(100) NOT NULL,
    content         JSONB NOT NULL DEFAULT '{}',
    -- content example:
    -- {
    --   "subject": "Work Order WO-2026-00142 Completed",
    --   "body": "Your HVAC service appointment has been completed...",
    --   "work_order_number": "WO-2026-00142",
    --   "technician_name": "Mike Johnson"
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, occurred_at);
CREATE INDEX idx_notification_tenant ON notification (tenant_id, status);
```

---

## Example Queries

### Find all equipment with a specific attribute using JSONB containment

```sql
-- Find all chillers with tonnage > 200 in a tenant
SELECT id, name, serial_number,
       type_attributes->>'tonnage' AS tonnage,
       type_attributes->>'chiller_type' AS chiller_type
FROM equipment
WHERE tenant_id = 'abc...'
  AND category_id = (SELECT id FROM equipment_category WHERE code = 'chiller')
  AND (type_attributes->>'tonnage')::numeric > 200;
```

### Find technicians with a specific certification

```sql
-- Find all technicians with valid EPA Universal certification
SELECT t.id, u.full_name, cert->>'number' AS cert_number
FROM technician t
JOIN "user" u ON u.id = t.user_id
CROSS JOIN LATERAL jsonb_array_elements(t.certifications) AS cert
WHERE t.tenant_id = 'abc...'
  AND cert->>'type' = 'epa_608_universal'
  AND t.is_active = true;
```

### Query refrigerant transactions from work order compliance context

```sql
-- Total refrigerant added per equipment in the last year
SELECT wo.equipment_id, e.name,
       SUM((tx->>'quantity_oz')::numeric) AS total_added_oz
FROM work_order wo
CROSS JOIN LATERAL jsonb_array_elements(wo.compliance_context->'refrigerant_transactions') AS tx
JOIN equipment e ON e.id = wo.equipment_id
WHERE wo.tenant_id = 'abc...'
  AND tx->>'type' = 'add'
  AND wo.created_at >= now() - INTERVAL '1 year'
GROUP BY wo.equipment_id, e.name
ORDER BY total_added_oz DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenant, user (roles as array, permissions as JSONB) |
| CRM | 2 | customer, service_location (contacts embedded in customer JSONB) |
| Equipment | 2 | equipment_category, equipment (type_attributes, telemetry_config, warranty as JSONB) |
| Technicians | 1 | technician (skills, certifications as JSONB arrays) |
| Service Agreements & PM | 3 | service_agreement, pm_task_template, pm_schedule |
| Work Orders | 1 | work_order (details, scheduling, compliance_context, financials as JSONB) |
| Invoicing & Pricebook | 2 | invoice, pricebook_item (line_items, payments, pricing as JSONB) |
| Telemetry & Energy | 3 | telemetry_reading (partitioned), energy_baseline, energy_anomaly |
| Audit & Notifications | 2 | audit_log, notification |
| **Total** | **~18 tables** | Plus partitions for telemetry_reading |

---

## Key Design Decisions

1. **JSONB for variable attributes, relational for stable relationships** — the customer-to-equipment-to-work-order relationship chain is always relational with foreign keys. Only attributes that vary by type, jurisdiction, or tenant go into JSONB columns.

2. **Equipment type_attributes validated by JSON Schema** — each `equipment_category` row carries an `attribute_schema` (JSON Schema) that defines what fields are expected in `equipment.type_attributes`. Application code validates on write; the database stores flexibly.

3. **Contacts embedded in customer JSONB** — for most HVAC companies, a customer has 1-3 contacts. Embedding them in `customer.contact_info` avoids a junction table and simplifies the common "show me the customer with their contacts" query.

4. **Compliance context as JSONB on work order** — jurisdiction-specific compliance data (CARB rules in California, state-specific leak repair timelines) lives in `work_order.compliance_context`. This avoids adding nullable columns for every jurisdiction's requirements.

5. **Work order financials as JSONB** — line items, totals, and payment status are embedded in the work order. For companies that need full double-entry accounting, the `invoice` table provides a relational anchor, but the work order itself carries the financial snapshot for mobile technician display.

6. **Skills and certifications as JSONB arrays on technician** — avoids three junction tables (technician_skill, certification_type, technician_certification) at the cost of application-enforced integrity. Suitable for the typical HVAC company with 5-50 technicians and 3-10 skill/cert types.

7. **PostgreSQL arrays for simple lists** — `service_agreement.equipment_ids` and `user.roles` use native PostgreSQL arrays with GIN indexes. Simpler than junction tables for lists that are always read and written as a whole.

8. **Tiered pricing in pricebook JSONB** — the `pricebook_item.pricing` JSONB supports good/better/best pricing tiers natively, matching the Housecall Pro and FieldEdge "proposal builder" pattern without requiring a separate pricing tier table.

9. **Quarterly telemetry partitions** — telemetry data is partitioned quarterly rather than monthly, reducing partition management overhead while maintaining acceptable query performance for the ASHRAE 90.1 36-month retention window (12 partitions for 3 years).

10. **~18 tables vs. ~44 in the normalised model** — the JSONB hybrid cuts the table count by more than half. Each eliminated table reduces migration complexity, ORM mapping surface, and API endpoint count, accelerating MVP development.
