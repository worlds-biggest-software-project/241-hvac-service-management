# Standards & API Reference

> Project: HVAC Service Management · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ASHRAE / Building Automation Standards

**ANSI/ASHRAE 135-2020 — BACnet: A Data Communication Protocol for Building Automation and Control Networks**
- URL: https://bacnet.org / https://www.ashrae.org/technical-resources/bookstore/bacnet
- The dominant open protocol for HVAC and building-automation device communication, enabling interoperability between equipment (chillers, AHUs, VAVs, sensors) and management software. Also published internationally as ISO 16484-5:2022. Specified in more than 60% of projects globally. Any HVAC platform seeking to ingest live equipment data must support BACnet/IP or integrate with a BACnet gateway.

**ISO 16484-5:2022 — Building Automation and Control Systems (BACS), Part 5: Data Communication Protocol**
- URL: https://www.iso.org/standard/71354.html
- The ISO harmonised version of ASHRAE 135 BACnet. Required for international certification and procurement compliance in EU and Asian markets. Provides the same protocol specification under an internationally recognised number.

**ANSI/ASHRAE/ACCA Standard 180-2018 — Standard Practice for Inspection and Maintenance of Commercial Building HVAC Systems**
- URL: https://www.ashrae.org/technical-resources/bookstore/standards-180-and-211
- The only ASHRAE standard that prescribes how to maintain HVAC systems, not just how to design them. Defines task-level PM schedules, inspection frequencies, and documentation requirements for AHUs, chillers, boilers, cooling towers, terminal units, and controls. A service management platform targeting commercial clients should align PM workflow templates with ASHRAE 180 task lists.

**ANSI/ASHRAE 90.1-2022 — Energy Standard for Buildings Except Low-Rise Residential Buildings**
- URL: https://www.ashrae.org/technical-resources/bookstore/standard-90-1
- Commercial buildings energy efficiency standard. Section 6 mandates automatic HVAC controls including optimal start and setback/shutdown; Section 8 requires energy monitoring by load category at 15-minute intervals with 36-month retention for buildings over 25,000 sqft. An AI-native HVAC platform can differentiate by providing automated 90.1 compliance dashboards from equipment telemetry.

**ANSI/ASHRAE 62.1-2025 — Ventilation and Acceptable Indoor Air Quality in Nonresidential Buildings**
- URL: https://www.ashrae.org/technical-resources/bookstore/standards-62-1-62-2
- Updated in late 2025 with revised ventilation rate tables. Relevant to seasonal commissioning checks and IAQ compliance documentation that technicians produce during service visits.

**SMACNA — Sheet Metal and Air Conditioning Contractors' National Association Standards**
- URL: https://www.smacna.org/technical/technical-documents
- Industry-standard duct construction and installation specifications referenced in commercial HVAC service and commissioning work. Service agreements for ductwork maintenance should reference SMACNA standards.

---

### Refrigerant and Environmental Compliance

**EPA Section 608 — Clean Air Act Refrigerant Handling Regulations**
- URL: https://www.epa.gov/section608
- Federal US regulation requiring technician certification for any work involving refrigerants. Mandates leak rate tracking, repair timelines, and disposal procedures. As of 2026 the AIM Act expansion lowered the threshold from 50 lb to 15 lb of HFC refrigerant, significantly expanding the universe of systems requiring formal compliance logging. An HVAC service platform must capture refrigerant amounts added/recovered per job and flag threshold breaches automatically.

**EPA AIM Act (American Innovation and Manufacturing Act) — HFC Phasedown Rules**
- URL: https://www.epa.gov/climate-hfcs-reduction
- From January 2025, new residential and light commercial systems must use refrigerants with GWP ≤ 700 (R-410A no longer permitted in new equipment). Service platforms need to track refrigerant type per unit, flag non-compliant replacements, and assist contractors in documenting transitions to low-GWP alternatives (R-32, R-454B, R-410A blends).

---

### Energy Management

**ISO 50001:2018 — Energy Management Systems**
- URL: https://www.iso.org/iso-50001-energy-management.html
- International standard for organisational energy management, analogous to ISO 9001 for quality. Requires monitoring, measurement, and continuous improvement of energy consumption. Facility managers operating under ISO 50001 certification will require HVAC service data (energy baselines, anomaly events, maintenance records) to feed their EnMS. An AI-native platform providing structured energy data exports serves this market.

---

### Building and Industrial Communication Protocols

**Modbus RTU / TCP — IDA Modbus Application Protocol Specification v1.1b3**
- URL: https://modbus.org/specs.php
- Simple, widely deployed serial/Ethernet protocol used on chillers, RTUs, variable-frequency drives (VFDs), and energy meters. Modbus/TCP (port 502) is the network variant. Any IoT gateway layer bridging field equipment to a cloud HVAC platform will encounter Modbus. The specification is free to download from modbus.org.

**ANSI/CEA-709 (LonTalk / LonWorks)**
- URL: https://www.lonmark.org/technical_resources/
- Peer-to-peer control network protocol common in older commercial HVAC installations. Brownfield integration projects at legacy sites will require Lon-to-BACnet or Lon-to-MQTT bridging. Less critical for new builds but cannot be ignored in retrofit service contracts.

---

### API and Data Exchange Standards

**OpenAPI Specification 3.1 (formerly Swagger)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for documenting REST APIs. All major HVAC FSM platforms either publish or generate OpenAPI specs. A new AI-native platform should publish a complete OpenAPI 3.1 spec to ease third-party integrations and developer adoption.

**GraphQL — June 2018 Specification**
- URL: https://spec.graphql.org/June2018/
- Query language and runtime used by Jobber as its primary API. GraphQL enables clients to request exactly the fields they need, reducing over-fetching — valuable for mobile technician apps on constrained networks.

**JSON Schema — Draft 2020-12**
- URL: https://json-schema.org/specification
- Standard for describing the structure and validation of JSON documents. Should be used to define and validate work order, asset, equipment, and compliance record data models for an HVAC service platform.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Standard authorisation protocol used by all major HVAC FSM APIs (ServiceTitan, Jobber, Housecall Pro, Salesforce). Any HVAC platform providing an API for third-party integrations must implement OAuth 2.0. The Authorization Code + PKCE flow is current best practice for mobile and single-page apps.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Provides standardised user authentication tokens (ID tokens as JWTs) for multi-tenant SaaS deployments where technician and customer portal users authenticate via an identity provider.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Standard for expressing relationships between web resources via HTTP Link headers. Relevant for REST API pagination and HATEOAS patterns in HVAC platform API design.

---

### Security Standards

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- The authoritative checklist for securing web APIs. Covers broken object-level authorisation, authentication weaknesses, excessive data exposure, and injection attacks. An HVAC platform handling sensitive building access data and customer records must be designed against these risks.

**NIST SP 800-53 Rev 5 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- US government control framework. Relevant for HVAC software sold to federal facilities, government buildings, or defence contractors where FedRAMP compliance may be required.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic / Open Specification**
- URL: https://modelcontextprotocol.io/specification
- Open protocol for connecting AI models to external data sources and tools. An HVAC service platform could expose an MCP server allowing AI assistants to query work order status, equipment history, technician availability, and compliance records in natural language — directly enabling the "natural-language service history query" capability identified as an AI-native opportunity.

---

## Similar Products — Developer Documentation & APIs

### ServiceTitan

- **Description:** End-to-end field service platform for HVAC, plumbing, and electrical contractors. Market leader for mid-market and enterprise residential/commercial service companies.
- **API Documentation:** https://developer.servicetitan.io/docs/apis
- **SDKs/Libraries:** PHP client: https://github.com/compwright/servicetitan; Postman/Insomnia collections available via developer portal
- **Developer Guide:** https://developer.servicetitan.io/docs/get-going-first-api-call/
- **Standards:** REST/JSON, OAuth 2.0, OpenAPI/Swagger specs published
- **Authentication:** OAuth 2.0 (Client ID + Secret Key → access token + app key)

---

### Housecall Pro

- **Description:** SMB-focused HVAC and home services management platform covering scheduling, dispatching, invoicing, and customer communication.
- **API Documentation:** https://docs.housecallpro.com/
- **SDKs/Libraries:** OpenAPI definition available; third-party scrapers on Apify
- **Developer Guide:** https://help.housecallpro.com/en/articles/8505035-api-overview
- **Standards:** REST/JSON, Webhooks (at-least-once delivery), OpenAPI 3.x
- **Authentication:** API key (MAX plan required); OAuth 2.0 for third-party integrations

---

### Jobber

- **Description:** Field service management platform designed for small residential service businesses covering quoting, scheduling, invoicing, and client management.
- **API Documentation:** https://developer.getjobber.com/docs/
- **SDKs/Libraries:** Ruby on Rails app template: https://github.com/GetJobber/Jobber-AppTemplate-RailsAPI; GraphiQL playground via Developer Center
- **Developer Guide:** https://developer.getjobber.com/docs/getting_started/
- **Standards:** GraphQL (POST to https://api.getjobber.com/api/graphql); Webhooks with at-least-once delivery
- **Authentication:** OAuth 2.0 (Authorization Code flow)

---

### FieldEdge

- **Description:** HVAC and plumbing specialist platform with flat-rate pricing, equipment history tracking, and service agreement management.
- **API Documentation:** https://docs.api.fieldedge.com/ (Azure API Management portal)
- **SDKs/Libraries:** Not publicly listed
- **Developer Guide:** https://docs.api.fieldedge.com/ (sign-in required for full access)
- **Standards:** REST/JSON
- **Authentication:** API key (sign-in to developer portal required)

---

### BuildOps

- **Description:** Commercial HVAC and mechanical contractor platform covering estimating, project management, dispatch, job costing, and EPA compliance logging.
- **API Documentation:** https://developer.buildops.com/
- **SDKs/Libraries:** Not publicly listed
- **Developer Guide:** https://developer.buildops.com/
- **Standards:** REST/JSON
- **Authentication:** Not publicly documented; contact for developer access

---

### Commusoft

- **Description:** HVAC and plumbing service platform with deep planned preventive maintenance, service contract management, and customer portal capabilities.
- **API Documentation:** https://commusoft.docs.apiary.io/
- **SDKs/Libraries:** Zapier app available
- **Developer Guide:** https://commusoft.docs.apiary.io/
- **Standards:** REST/JSON (documented via Apiary/API Blueprint)
- **Authentication:** API key (details in Apiary docs)

---

### Salesforce Field Service (ServiceMax / Asset 360)

- **Description:** Enterprise field service and asset lifecycle management built on the Salesforce platform. Used by large HVAC and mechanical service organisations requiring deep CRM and IoT integration.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.field_service_dev.meta/field_service_dev/fsl_dev_rest.htm
- **SDKs/Libraries:** Salesforce SDKs for JavaScript, Java, Python, .NET, iOS, Android; APEX for platform code
- **Developer Guide:** https://trailhead.salesforce.com (Field Service Trailhead modules)
- **Standards:** REST and SOAP (v66.0 Spring '26); SOQL for data queries; Lightning Web Components
- **Authentication:** OAuth 2.0 (Salesforce Connected Apps)

---

### IFS Cloud Field Service Management

- **Description:** Enterprise FSM platform for asset-intensive industries (HVAC, utilities, manufacturing). Recognised in IDC MarketScape Leaders Category 2026.
- **API Documentation:** https://docs.ifs.com (login required for full API reference)
- **SDKs/Libraries:** REST client documentation within IFS developer portal
- **Developer Guide:** https://www.ifs.com/en/solutions/field-service-management/
- **Standards:** REST/JSON; OData for data queries; OpenAPI specs for selected endpoints
- **Authentication:** OAuth 2.0

---

## Notes

**Emerging standard to watch — MQTT and Sparkplug B:** MQTT (ISO/IEC 20922:2016) is a lightweight publish/subscribe messaging protocol increasingly used for IoT-to-cloud HVAC telemetry as an alternative to BACnet/IP in cloud-native architectures. Sparkplug B (Eclipse Foundation) adds a standardised topic namespace and payload encoding (Protobuf) for industrial IoT over MQTT. A next-generation HVAC service platform could accept telemetry via MQTT/Sparkplug B as well as BACnet to cover both legacy building controllers and modern IoT-native equipment.

**W3C Web of Things (WoT) Thing Description:** The W3C WoT working group has published a Thing Description standard (https://www.w3.org/TR/wot-thing-description/) that provides a machine-readable metadata format for IoT devices including HVAC sensors and controllers. While not yet widely adopted by HVAC OEMs, it represents the likely direction of equipment interoperability standards and is worth tracking for long-term platform architecture decisions.

**Refrigerant transition data gap:** There is currently no standardised API or data exchange format for tracking refrigerant type transitions across a fleet of equipment as owners upgrade from R-410A to low-GWP alternatives. This is an open opportunity to define a data model (JSON Schema) that becomes a de-facto industry standard and drives platform adoption.
