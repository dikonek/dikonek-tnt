# Software Requirements Specification (SRS)
Project: dikonek-tnt - Track & Trace Platform  
Version: 0.3 (Draft)  
Owner: PT Dikonek  
Reference: dikonek-oee (stack, folder conventions, auth/MQTT patterns; batch/shift/sku/operator handling)

---

## Table of Contents
- 1. Introduction
- 2. Product Overview
- 3. User Classes
- 4. Functional Requirements - Server
- 5. Functional Requirements - Station Software
- 6. External Interface Requirements
- 7. Non-Functional Requirements
- 8. Data Model & Database Entities
- 9. Future Extensions

---

## 1. Introduction
### 1.1 Purpose
Define the functional and non-functional requirements for dikonek-tnt: a Track & Trace platform consisting of a central web server (dikonek-tnt) and Windows-based Station Software running on production lines. This SRS is aimed at:
- Product owner / CTO (scope and roadmap)
- Backend and frontend developers
- Desktop/.NET developers for Station Software
- QA/Testing team
- System integrators (PLC/field devices, ERP integration)

### 1.2 Scope
dikonek-tnt manages unique codes at scale, tracks code lifecycle across production and logistics, coordinates line devices via Station Software, and integrates with ERP. The server is a web app (same stack as dikonek-oee); Station Software is a .NET Windows Forms app. All production events are tagged with batch, shift, SKU, and operator context (aligned to dikonek-oee).

### 1.3 Definitions, Acronyms, Abbreviations
- Track & Trace: Tracking serialized codes through their lifecycle.
- Unique Code: Serialized identifier printed on each unit.
- Code Status: Logical state of a unique code in its lifecycle.
- Station Software: Desktop app per production line interfacing with devices and dikonek-tnt.
- ERP: Enterprise Resource Planning system.
- SKU: Stock Keeping Unit.
- Batch: Production batch / lot.
- Shift: Production time window (recurring calendar definition).
- Operator: Logged-in production user running a station.
- MQTT, Modbus TCP, EtherNet/IP: Industrial protocols in use.

### 1.4 References
- dikonek-oee codebase (technology stack, folder structure, coding standards).
- Device manuals: Gressler GT2000 TIJ Printer; Hikrobot code readers; A&D AD-4412-CW check weigher; Anritsu metal detector; Remote I/O + Modbus TCP docs.

### 1.5 Overview
This document covers product description, architecture, functional requirements (server + Station Software), external interfaces, non-functional requirements, and data model (with database entities).

---

## 2. Product Overview
### 2.1 Product Perspective
Central server + edge stations:
- **Server (dikonek-tnt):** Web UI, REST API, MQTT endpoints, central database for master data, codes, code history, station/device configuration.
- **Station Software (per line):** Windows desktop app; connects to local devices via Ethernet/IP, Modbus TCP, MQTT; publishes events/statuses to server; optional local DB for offline use.
- May share MQTT broker and infrastructure patterns with dikonek-oee.

### 2.2 Operating Environment
- **Backend:** Node.js 20+, NestJS, TypeScript; Postgres 14+; optional TimescaleDB.
- **Frontend:** React 18+, TypeScript, MUI; modern desktop browsers.
- **Station Software:** Windows 10/11, .NET (WinForms); connected to plant network with HTTP/MQTT access.
- **Deployment:** Docker/Kubernetes similar to dikonek-oee; Linux for production.

### 2.3 Design & Implementation Constraints
- Align with dikonek-oee backend/frontend structure, auth (JWT + refresh), RBAC, MQTT helpers, config style, and production context modeling (batch/shift/sku/operator).
- Reuse dikonek-oee front-end theming (shared MUI theme tokens: palette, typography, spacing, shape) for a consistent UI look and feel.
- Support up to 100 Station Software instances and multi-million codes per campaign.
- Industrial protocols handled by Station Software (server only receives abstracted events/commands).

### 2.4 Assumptions & Dependencies
- Stable network with occasional outages; MQTT broker sized for load.
- ERP connectors vary by customer (REST/DB/file); implemented via adapters.
- Device command sets and timing correctly implemented by .NET device libraries.

---

## 3. User Classes
- **System Administrator:** Global configuration, users, security, status model.
- **IT/Integration Engineer:** Stations, devices, protocols, ERP connectors, MQTT setup.
- **Production Engineer / Line Supervisor:** Batch setup, shift planning, code pools, monitor stations.
- **Warehouse Operator:** Scans codes, receives/dispatches inventory.
- **Distributor / Retail Operator (optional):** Inbound/outbound scanning.
- **Quality / Traceability Officer:** Investigates code history, reports.
- **Operator:** Runs a station during a shift; logs in/out and executes batches.
- **Customer (external):** Scans code via mobile for validation.

---

## 4. Functional Requirements - Server (dikonek-tnt)
IDs: FR-SRV-xx. Priority: M = Must, S = Should, C = Could.

### 4.1 User Management & Access Control
- FR-SRV-01 (M) Login with email/username + password.
- FR-SRV-02 (M) RBAC with predefined roles (Admin, IT Engineer, Production Engineer, Supervisor, Warehouse, Viewer, Operator); custom roles later.
- FR-SRV-03 (M) Authentication required for all non-public APIs.
- FR-SRV-04 (M) Password reset (self or admin).
- FR-SRV-05 (S) Log login/logout attempts and security events.

### 4.2 Master Data (SKU, Batch, Locations, Shifts)
- FR-SRV-10 (M) Manage SKU master data (code, name, description, packaging, active flag).
- FR-SRV-11 (M) Manage batch/lot (batch ID, SKU ref, planned qty, production window, status).
- FR-SRV-12 (S) Manage locations (site, warehouse, distributor, retail) with type and code.
- FR-SRV-13 (S) Manage business partners (distributors, retailers).
- FR-SRV-14 (M) Manage shift definitions (name/code, start/end, timezone, day-of-week pattern).
- FR-SRV-15 (S) Manage production schedules linking line/station to batch, shift, and SKU (planned vs. actual).

### 4.3 Unique Code Generation & Management
- FR-SRV-20 (M) Support formula-based and random code generation.
- FR-SRV-21 (M) Configurable templates (name, SKU/batch, length, charset, checksum).
- FR-SRV-22 (M) Generate and store multi-million codes per template/batch.
- FR-SRV-23 (M) Store code with SKU, batch, current status, timestamps, expiry (optional).
- FR-SRV-24 (M) API for Station Software to allocate/release codes.
- FR-SRV-25 (S) Range-based generation/allocation (future, seed/range storage).

### 4.4 Dynamic Code Status & Lifecycle
- FR-SRV-30 (M) Configurable code statuses (not hard-coded); default flow provided.
- FR-SRV-31 (M) Add/edit/delete statuses; enable/disable without losing history.
- FR-SRV-32 (S) Different status flows per SKU/group or packaging level.
- FR-SRV-33 (M) Persist status change history (code, from/to, source, time, station/device, payload).
- FR-SRV-34 (M) Accept status changes via MQTT, REST, and customer scan endpoint.

### 4.5 Station & Device Management
- FR-SRV-40 (M) Station registry (ID, name, site, line, location, MQTT/API params, status, last heartbeat).
- FR-SRV-41 (M) Device configs per station (ID, type, name, IP/endpoint, protocol, metadata, enabled).
- FR-SRV-42 (M) API for Station Software to fetch configs and send health/status.
- FR-SRV-43 (M) Monitor station/device status via MQTT (heartbeat, online/offline, last message).
- FR-SRV-44 (S) Remote commands to stations (start/stop batch, reload config, pause/resume printing, test prints).

### 4.6 Real-Time Monitoring & Dashboard
- FR-SRV-50 (M) Dashboards per line/station: batch/SKU/shift, station/device online status, throughput (codes/min), errors/rejects.
- FR-SRV-51 (M) Near real-time UI updates (<2s) from MQTT events.
- FR-SRV-52 (S) Alert rules (e.g., camera fail rate threshold, offline duration).

### 4.7 ERP Integration
- FR-SRV-60 (M) ERP-independent integration via CSV/Excel upload and REST API; export code usage/status and shipment confirmations.
- FR-SRV-61 (S) Configurable ERP connectors (REST, DB, file-based).
- FR-SRV-62 (S) Integration logs (requests, responses, errors, last sync per object).
- FR-SRV-63 (M) Operate standalone without ERP (create/manage all data internally).

### 4.8 Code Query & Traceability
- FR-SRV-70 (M) Search by code string (exact), optional partial, batch, SKU, shift, operator, status, date range.
- FR-SRV-71 (M) Show full lifecycle trace (status changes with station/device/time and device data).
- FR-SRV-72 (M) Public endpoint to validate code; returns validity, used/not, basic product info; record customer scan if configured.

### 4.9 Reporting & Data Export
- FR-SRV-80 (M) Reports: codes generated vs. used per batch/shift; reject/NG ratios per device; status distribution per batch/shift.
- FR-SRV-81 (M) Export to CSV/Excel for main entities and reports.
- FR-SRV-82 (S) Traceability report (batch/SKU/shift -> all codes + locations).
- FR-SRV-83 (C) PDF report generation.

### 4.10 Administration & Settings
- FR-SRV-90 (M) Global settings: MQTT broker URL/credentials; ERP connector config; default status flow/mapping.
- FR-SRV-91 (M) Configure default allocation size; retention periods for logs/histories.
- FR-SRV-92 (S) Audit log of configuration changes.

### 4.11 Batch/Shift/SKU/Operator Context (Aligned with dikonek-oee)
- FR-SRV-100 (M) Every production event stored server-side shall be tagged with batch_id, sku_id, shift_id, station_id, and operator_id (if present).
- FR-SRV-101 (M) Server shall expose APIs to open/close a station run (batch + shift + operator) and reflect active context in dashboards.
- FR-SRV-102 (S) Allow reassignment or correction of operator/shift context with audit logging.

---

## 5. Functional Requirements - Station Software (Windows .NET)
IDs: FR-STA-xx; applies to each Station Software instance.

### 5.1 General Requirements
- FR-STA-01 (M) Windows Forms .NET desktop app.
- FR-STA-02 (M) Service-like operation (auto-start on boot; run unattended).
- FR-STA-03 (M) Configurable Station ID, MQTT broker, server API endpoint/auth.
- FR-STA-04 (M) Local configuration UI for IT engineer (manage devices, map types/protocols, test connections).
- FR-STA-05 (M) Fetch config from server and merge/override per policy.
- FR-STA-06 (M) Optional local database for offline buffering (codes/events) and sync.
- FR-STA-07 (M) Operator login/logout per station, enforcing role = Operator (or higher) before running production.

### 5.2 Device Abstraction & Protocols
- FR-STA-10 (M) Multiple devices per station (printers, cameras, check weigher, metal detector, remote I/O).
- FR-STA-11 (M) Protocol libraries with abstract interface (connect/disconnect/read/write/subscribe) for Ethernet/IP, Modbus TCP, MQTT.
- FR-STA-12 (M) Select protocol per device (e.g., printer via Ethernet/IP; weigher via Modbus TCP; remote I/O via Modbus).
- FR-STA-13 (M) Monitor device connectivity in local UI and via MQTT (online/offline).

### 5.3 Printer Integration (Gressler GT2000 TIJ)
- FR-STA-20 (M) Ethernet integration.
- FR-STA-21 (M) Functions: list/select templates; send variables (code, batch, date, etc.); start/stop print.
- FR-STA-22 (M) Send unique codes to printer at line speed (from server or local buffer).
- FR-STA-23 (S) Support multiple printers per station.

### 5.4 Camera Integration (Hikrobot)
- FR-STA-30 (M) Ethernet integration.
- FR-STA-31 (M) Functions: QR read (OK/NG, decoded code); present/absence check (OK/NG); read reject signal (digital I/O or interface).
- FR-STA-32 (M) Correlate reads to expected codes; publish events with code, result, timestamp, station/device.

### 5.5 Check Weigher Integration (A&D AD-4412-CW)
- FR-STA-40 (M) Modbus TCP integration.
- FR-STA-41 (M) Functions: read weight (with unit/stable flag); read reject signal; write reject; read/write velocity.
- FR-STA-42 (M) Correlate weight measurements with codes; store weight and pass/fail.

### 5.6 Metal Detector Integration (Anritsu)
- FR-STA-50 (M) Remote I/O integration.
- FR-STA-51 (M) Read reject signal from relay/digital IO.
- FR-STA-52 (M) Correlate metal detector results with codes where timing allows.

### 5.7 Remote I/O via Modbus TCP
- FR-STA-60 (M) Support remote I/O modules via Modbus TCP.
- FR-STA-61 (M) Read digital inputs; write digital outputs.
- FR-STA-62 (M) Configurable mapping from I/O points to logical signals (e.g., Reject_Camera, Conveyor_Start).

### 5.8 MQTT Publishing & Subscription
- FR-STA-70 (M) Publish station heartbeat, device health, code events (allocated/printed/read/weighed/rejected), station events (batch start/stop, errors).
- FR-STA-71 (M) Consistent/configurable topic structure, e.g.:
  - tnt/{site}/{line}/{station}/status
  - tnt/{site}/{line}/{station}/device/{deviceId}/status
  - tnt/{site}/{line}/{station}/code/{code}/event
- FR-STA-72 (S) Subscribe to server commands: tnt/{site}/{line}/{station}/command/* (reload config, start/stop batch, tests).

### 5.9 Offline Mode & Synchronization
- FR-STA-80 (M) When server/MQTT unreachable: continue using local config; buffer codes/events in local DB.
- FR-STA-81 (M) On reconnect: sync buffered events in correct order.
- FR-STA-82 (S) If conflicts (e.g., code in different status), log/mark conflict; server decides resolution.

### 5.10 Production Context (Batch/Shift/SKU/Operator)
- FR-STA-90 (M) Station UI shall allow selecting/confirming active batch and shift before starting production; selection comes from server schedule when available.
- FR-STA-91 (M) All published events must include batch_id, sku_id, shift_id, operator_id, station_id, device_id where applicable (aligned with dikonek-oee telemetry).
- FR-STA-92 (S) Support quick operator switch with minimal downtime and clear audit trail.

---

## 6. External Interface Requirements
### 6.1 User Interface (Web)
- Responsive web UI for login, master data, code generation/status, station/device config, real-time monitoring, reports.
- Design aligned with dikonek-oee (React + MUI); use the same theme tokens (colors, typography scale, spacing, rounded corners) and support light/dark if available.

### 6.2 API Interfaces
- REST API (JSON/HTTPS): auth (login/refresh); CRUD master data; code operations (generate/allocate/query); station/device endpoints; ERP import/export; code status updates.
- MQTT: topic naming conventions; JSON payloads; auth via username/password or certificate.

### 6.3 Hardware/Protocol Interfaces (Station Software)
- Ethernet/IP, Modbus TCP, Remote I/O per device. Detailed register/address mapping provided in device integration specs.

---

## 7. Non-Functional Requirements
### 7.1 Performance
- Support up to 100 Station Software instances; each with multiple devices.
- Sustain tens to hundreds of events per second overall.
- API read 95th percentile < 500 ms; UI event-to-screen < 2s.
- Generate millions of codes within minutes per batch job.

### 7.2 Reliability & Availability
- 24/7 operation; MQTT broker and DB in HA where possible.
- Offline mode on stations for network/server outages.

### 7.3 Security
- HTTPS for external access; JWT with refresh (per dikonek-oee pattern); RBAC on all APIs.
- Protect code validation endpoint (rate limiting, basic bot protection).
- Secure storage for sensitive configs (MQTT/ERP credentials).

### 7.4 Maintainability
- Follow dikonek-oee backend/frontend folder structure and shared libraries (auth, user mgmt, MQTT common).
- Automated unit/integration tests for key modules.
- Config via environment variables and config files (no hard-coded secrets).

### 7.5 Portability
- Backend deployable on Linux with Docker; Station Software on Windows 10/11.

### 7.6 Localization
- Phase 1: English UI; future: Bahasa Indonesia and others.

---

## 8. Data Model & Database Entities (PostgreSQL)
Logical model aligned to dikonek-oee conventions (snake_case, UUID primary keys, created_at/updated_at audit columns). All production events carry batch_id, sku_id, shift_id, station_id, device_id, and operator_id when available.

### 8.1 Security & Access
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| user | Application users | id (uuid), name (text), email (text unique), password_hash (text), role_id (uuid), status (enum: active/locked/invited), last_login_at (timestamptz) | Reuse auth lib from dikonek-oee |
| role | RBAC roles | id (uuid), name (text), description (text) | Seed default roles incl. Operator |
| permission_map | Role permissions | role_id (uuid), resource (text), actions (jsonb) | Align format with dikonek-oee |
| session_token | Token tracking | id (uuid), user_id (uuid), refresh_token (text hashed), expires_at (timestamptz) | For revocation/audit |
| audit_log | Config/security changes | id (uuid), user_id (uuid), entity (text), action (text), before (jsonb), after (jsonb), created_at (timestamptz) | Covers FR-SRV-92 |

### 8.2 Master Data
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| sku | Product catalog | id (uuid), sku_code (text unique), name (text), description (text), packaging_info (jsonb), active (bool) | |
| batch | Production lots | id (uuid), batch_code (text unique), sku_id (uuid), planned_qty (int), start_time/end_time (timestamptz), status (enum: planned/running/completed/closed) | |
| shift | Shift definitions | id (uuid), code (text unique), name (text), start_time (time), end_time (time), timezone (text), days_of_week (int[]), description (text) | Matches dikonek-oee shift model |
| production_schedule | Planned runs | id (uuid), line (text), station_id (uuid null), batch_id (uuid), sku_id (uuid), shift_id (uuid), planned_start/end (timestamptz), planned_qty (int), status (enum: planned/confirmed/in_progress/completed/cancelled) | |
| location | Sites/warehouses | id (uuid), code (text unique), name (text), type (enum: site/warehouse/distributor/retail), parent_location_id (uuid null) | |
| partner | Business partners | id (uuid), code (text unique), name (text), type (enum: distributor/retailer), location_id (uuid null) | |

### 8.3 Station & Device Configuration
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| station | Station registry | id (uuid), name (text), site (text), line (text), location_id (uuid), status (enum: online/offline), last_heartbeat_at (timestamptz), mqtt_topic_base (text), api_key (text hashed) | |
| device | Devices per station | id (uuid), station_id (uuid), type (enum: printer/camera/weigher/metal_detector/remote_io), name (text), ip_address (inet/text), protocol (enum: ethernet_ip/modbus_tcp/mqtt/other), info1/info2/info3 (text), enabled (bool) | |
| device_setting | Device params | id (uuid), device_id (uuid), key (text), value (jsonb) | Optional KV for device-specific details |
| station_command | Remote commands log | id (uuid), station_id (uuid), command (text), payload (jsonb), status (enum: pending/sent/ack/failed), created_at (timestamptz) | Supports FR-SRV-44 |

### 8.4 Production Context & Sessions
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| station_run | Active production context | id (uuid), station_id (uuid), batch_id (uuid), sku_id (uuid), shift_id (uuid), operator_id (uuid), started_at (timestamptz), ended_at (timestamptz), status (enum: open/closed), notes (text) | Aligns with dikonek-oee run model |
| operator_session | Operator presence | id (uuid), user_id (uuid), station_id (uuid), shift_id (uuid), started_at (timestamptz), ended_at (timestamptz), status (enum: open/closed) | |

### 8.5 Code Templates & Pools
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| code_template | Generation templates | id (uuid), name (text), type (enum: formula/random), pattern (text/jsonb), length (int), charset (text), checksum_type (text), sku_id (uuid null), batch_id (uuid null) | |
| code_pool | Generated batches | id (uuid), template_id (uuid), sku_id (uuid), batch_id (uuid), total_codes (bigint), created_at (timestamptz), status (enum: new/generating/ready/archived) | |
| code | Unique codes | id (uuid), code_value (text unique), pool_id (uuid), sku_id (uuid), batch_id (uuid), current_status_id (uuid), created_at (timestamptz), expires_at (timestamptz null) | Index on code_value |
| code_allocation | Allocations to stations | id (uuid), code_id (uuid), station_id (uuid), device_id (uuid null), batch_id (uuid), sku_id (uuid), shift_id (uuid), operator_id (uuid null), allocated_at (timestamptz), released_at (timestamptz null), status (enum: allocated/released/cancelled) | Covers FR-SRV-24 |

### 8.6 Status & Traceability
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| status_definition | Status catalog | id (uuid), code (text unique), name (text), description (text), order_index (int), active (bool), scope (enum: global/per_flow), packaging_level (enum: unit/carton/pallet null) | |
| status_flow | Optional flow grouping | id (uuid), name (text), default_flag (bool) | |
| status_flow_transition | Allowed transitions | flow_id (uuid), from_status_id (uuid), to_status_id (uuid) | For FR-SRV-32 |
| code_status_history | Status events | id (uuid), code_id (uuid), status_id (uuid), event_source (enum: station/erp/manual/customer), station_id (uuid null), device_id (uuid null), batch_id (uuid), sku_id (uuid), shift_id (uuid), operator_id (uuid null), payload (jsonb), created_at (timestamptz) | Index on (code_id, created_at) |

### 8.7 Device Data & QA Results
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| code_camera_result | Camera reads | id (uuid), code_id (uuid), device_id (uuid), result (enum: ok/ng), decoded_value (text), image_ref (text null), reject_flag (bool), batch_id (uuid), sku_id (uuid), shift_id (uuid), operator_id (uuid null), captured_at (timestamptz) | |
| code_weight_result | Weigher data | id (uuid), code_id (uuid), device_id (uuid), weight (numeric), unit (text), stable (bool), pass_fail (enum: pass/fail), batch_id (uuid), sku_id (uuid), shift_id (uuid), operator_id (uuid null), captured_at (timestamptz) | |
| code_metal_check | Metal detector | id (uuid), code_id (uuid), device_id (uuid), detected (bool), batch_id (uuid), sku_id (uuid), shift_id (uuid), operator_id (uuid null), captured_at (timestamptz) | |
| code_reject_event | Consolidated rejects | id (uuid), code_id (uuid), device_id (uuid null), reason (text/enum), station_id (uuid), batch_id (uuid), sku_id (uuid), shift_id (uuid), operator_id (uuid null), created_at (timestamptz) | |

### 8.8 Integration & Logs
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| erp_integration_config | Connector configs | id (uuid), type (enum: rest/db/file), endpoint (text), credentials (jsonb encrypted), mapping (jsonb), enabled (bool) | |
| integration_job | Sync jobs | id (uuid), object_type (text), direction (enum: in/out), status (enum: pending/running/success/failed), last_run_at (timestamptz), next_run_at (timestamptz), retries (int) | |
| integration_log | Request/response log | id (uuid), job_id (uuid), status (enum: success/fail), details (jsonb), created_at (timestamptz) | |
| mqtt_message_log | Optional MQTT audit | id (uuid), station_id (uuid), topic (text), payload (jsonb), direction (enum: publish/subscribe), batch_id (uuid null), sku_id (uuid null), shift_id (uuid null), operator_id (uuid null), created_at (timestamptz) | For troubleshooting |

### 8.9 Settings & Retention
| Entity | Purpose | Key Fields (Type) | Notes |
| --- | --- | --- | --- |
| system_setting | Key/value settings | id (uuid), key (text unique), value (jsonb), scope (enum: global/site/station) | Covers FR-SRV-90/91 |
| retention_policy | Data retention | id (uuid), category (enum: code_history/logs/reports), duration_days (int), notes (text) | |

### 8.10 Data Retention
- Code and status history: configurable (default 3-5 years for traceability).
- Operational and MQTT message logs: configurable (e.g., 6-12 months).
- Large tables indexed for code_value lookups and ordered by created_at for efficient archive.

---

## 9. Future Extensions (Non-MVP)
- Hierarchical packaging (unit/box/carton/pallet).
- Geolocation tracking for customer scans.
- Anti-counterfeit patterns (geo/IP risk scoring).
- Deeper integration with dikonek-oee (linking codes with OEE events).
- Multi-tenant architecture.
- Advanced analytics dashboards and PDF-ready official reports.
