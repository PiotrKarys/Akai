# CHANGELOG

Wszystkie istotne zmiany w projekcie będą dokumentowane w tym pliku.

Format wzorowany na [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

> [!IMPORTANT]
> Agent AI MUSI aktualizować ten plik przy każdej zmianie API, schematu danych lub architektury.

---

## [Faza 1 - unreleased]

### Added
- CRUD obiektów, paneli, stref (`/api/objects`, `/api/panels`, `/api/zones`)
- Auth JWT RS256 + RBAC (6 ról: SYSTEM, MASTER, ADMIN, OPERATOR, TECHNICIAN, FIELD_WORKER)
- Flutter Desktop: lista obiektów, szczegóły, logowanie, Secret Reveal
- Flutter Mobile: read-only, SQLite sync
- Structured Logging (`structlog` + `request_id`)
- Feature Flags (`feature_flags.yaml`)
- Seed Data Strategy (`seed_dev.py`)
- Konfiguracja per obiekt: `OBJECT_CONTACTS`, `OBJECT_PROCEDURES`, `OBJECT_ALARM_CONFIG`
- Tryb Serwisowy (config): kolumny `service_mode_*` w OBJECTS
- Import Excel: `POST /api/admin/import/objects` + tabela `IMPORT_LOG`
- Error Code Registry (ustandaryzowane kody błędów)

### Changed
- Harmonogram: 4 fazy (MVP/v1.0/v2.0/Nice-to-have) → 6 faz

---

## [Faza 2 - unreleased]

### Added
- Satel Worker (TCP/IP ↔ ETHM-1)
- RabbitMQ (topic + direct exchanges)
- Redis (Live State Cache)
- WebSocket (real-time push alarmów)
- Outbox Relay (atomowe dostarczanie eventów)
- Alarm Lifecycle (bundling, dedup, statusy: NEW→CLOSED)
- Tryb Serwisowy (efekt alarmowy): tłumienie priorytetu, brak dźwięku
- Push Notifications (FCM Android, CRITICAL only)
- Dziennik Dyspozytora (`DISPATCH_LOG`)
- Historia Stanu Centrali (`PANEL_STATE_HISTORY`)
- Event schema: `panel.state.changed`

---

## [Faza 3 - unreleased]

### Added
- Multi-object/panel support (50+ obiektów)
- Prometheus + Grafana dashboardy
- Alertmanager (CRITICAL/WARNING/INFO)
- Synthetic Probes (health check co 5 min)

---

## [Faza 4 - unreleased]

### Added
- SMS Agent (modem GSM, parser Efento/Bluelog)
- Alarmy temperaturowe (TEMP_ALARM/TEMP_NORMAL)
- PII Isolation (SHA-256 → `SMS_RAW_ARCHIVE`)

---

## [Faza 5 - unreleased]

### Added
- Komendy sterujące ARM/DISARM z UI
- Priority Queues (`cmd.satel.high` / `cmd.satel.low`)
- Auto-Arm harmonogramy

---

## [Faza 6 - unreleased]

### Added
- PostgreSQL HA (Patroni + etcd + PgBouncer)
- Multi-instance Backend (2+ za Nginx)
- Redis Sentinel (3 instancje)
- RabbitMQ Quorum Queues (3 nody)
- Raporty (CSV/PDF, Read Replica)
- Service Tickets
- Object Documentation
- Offline Mode (SQLite + Intent Queue)
- Chaos Engineering
