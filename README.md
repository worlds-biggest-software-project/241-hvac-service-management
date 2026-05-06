# HVAC Service Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source HVAC field service platform that turns equipment history, seasonal maintenance, and energy optimisation into first-class workflows.

HVAC Service Management is a candidate project for an open, AI-native alternative to proprietary field service platforms such as ServiceTitan, Housecall Pro, FieldEdge, and ServiceMax. It is aimed at HVAC contractors, commercial mechanical service companies, and property management firms that need deep equipment history, planned preventive maintenance, and energy-aware service workflows without paying enterprise SaaS prices.

---

## Why HVAC Service Management?

- Enterprise platforms like ServiceTitan start around $245/user/month and routinely require dedicated implementation partners, putting them out of reach of small contractors.
- SMB-focused tools (Jobber, Housecall Pro) onboard quickly but lack HVAC-specific preventive maintenance workflows, depth of equipment history, and flat-rate pricing for trades.
- HVAC specialists like FieldEdge carry desktop-era UX and limited analytics; mid-market platforms like Commusoft have thin North American presence.
- No SMB or mid-market tool ingests BACnet/Modbus equipment telemetry end-to-end into automatic fault-diagnosis service tickets.
- Energy-efficiency monitoring against ASHRAE 90.1 baselines and ESG reporting is largely absent from every incumbent surveyed.

---

## Key Features

### Scheduling, Dispatch, and Mobile Field Work

- Job scheduling with drag-and-drop dispatch board
- Technician mobile app with job details, customer history, and payment collection
- Offline support for field technicians
- Automated customer notifications for booking confirmation, technician ETA, and job completion

### Customer, Asset, and Service History

- Customer CRM with per-equipment service history and asset records
- Equipment records capturing make, model, serial number, installation date, and warranty
- Service agreement and maintenance contract management
- Customer self-service portal with job history, reports, and online invoice payment

### Planned Preventive Maintenance and Compliance

- Service agreements and planned preventive maintenance scheduling with automated job creation from contract schedules
- EPA refrigerant handling log with automatic threshold alerts aligned to AIM Act thresholds
- Digital forms with signature capture and compliance documentation
- Automated compliance record population from technician notes and sensor readings

### Pricing, Invoicing, and Accounting

- Flat-rate pricing catalogue with technician-guided upsell presentation
- Invoicing, payment collection, and accounting sync (QuickBooks at minimum)
- Recurring invoicing for service contracts
- AI-driven upsell recommendations at point of service

### AI and Telemetry Integration

- AI-assisted fault diagnosis from connected equipment telemetry via BACnet/Modbus ingestion
- Energy-efficiency baseline monitoring per asset with anomaly alerting aligned to ASHRAE 90.1
- Natural-language service history query interface for technicians and managers
- Voice-to-structured-field-note transcription for technicians in the field

---

## AI-Native Advantage

Incumbents bolt analytics and "AI" features onto workflows designed for manual data entry. This project treats AI as the substrate: equipment telemetry from BACnet, LonWorks, and Modbus feeds models that identify failure precursors days before breakdown and auto-generate service tickets with part recommendations. LLM-assisted scheduling factors weather forecasts, occupancy calendars, and runtime hours into preventive maintenance timing. Continuous energy-signature monitoring flags refrigerant leaks and coil degradation proactively, and natural-language querying lets staff ask plain-English questions across years of equipment history instead of navigating legacy reports.

---

## Tech Stack & Deployment

The project targets integration with established HVAC and building-automation standards: BACnet (ASHRAE 135 / ISO 16484-5), LonTalk / LonWorks (ANSI/CEA-709), and Modbus RTU/TCP for equipment data ingest, with workflows aligned to ASHRAE 90.1, 62.1, 62.2, SMACNA, and ISO 50001. Integration patterns follow incumbent norms — REST and GraphQL APIs with OAuth 2.0, webhooks for job and invoice events, and accounting connectors (QuickBooks, Xero, Sage). Deployment modes and SDK details are still to be specified.

---

## Market Context

The global HVAC service management software market is estimated at ~$1.51 billion in 2026, projected to reach ~$6.3–7.0 billion by 2035 at a CAGR of approximately 17%. Incumbent pricing spans $39–$109/month for entry-level tools (Jobber, Housecall Pro), $200–$500/month for mid-market (FieldEdge, Commusoft), and $245+/user/month with six-figure annual contracts at the enterprise tier (ServiceTitan, ServiceMax). Primary buyers are independent HVAC contractors (1–10 technicians), regional commercial HVAC service companies (10–100 technicians), national facility management firms, and property management companies seeking vendor-neutral equipment history.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
