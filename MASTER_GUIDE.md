# MASTER GUIDE — STAM REPLACER

> **Wersja:** 1.0 | **Data:** 2026-02-20 | **Źródła:** Pliki 01–18 + MY_HONEST_OPINION.md
> Kompleksowy przewodnik implementacyjno-operacyjny systemu monitoringu bezpieczeństwa.

---

## SPIS TREŚCI

1. [Wizja i Zakres Systemu](#1-wizja-i-zakres-systemu)
2. [Architektura Systemu](#2-architektura-systemu)
3. [Model Danych](#3-model-danych)
4. [Przepływ Danych (Data Flow Masterplan)](#4-przepływ-danych)
5. [Harmonogram Implementacji (Fazy MVP → v2.0)](#5-harmonogram-implementacji)
6. [Moduły Funkcjonalne — Kompletna Mapa](#6-moduły-funkcjonalne)
7. [Bezpieczeństwo i Role](#7-bezpieczeństwo-i-role)
8. [Wysoka Dostępność i Disaster Recovery](#8-wysoka-dostępność-i-dr)
9. [Deployment i Infrastruktura](#9-deployment-i-infrastruktura)
10. [Monitoring i Alerty](#10-monitoring-i-alerty)
11. [Testowanie](#11-testowanie)
12. [Procedury Operacyjne (Runbook)](#12-procedury-operacyjne)
13. [Decyzje Architektoniczne (ADR)](#13-decyzje-architektoniczne)

---

## 1. Wizja i Zakres Systemu

### 1.1 Cel projektu

Stworzenie centralnej aplikacji do zarządzania bezpieczeństwem fizycznym firmy, integrującej:

- **Alarmy Satel** — bezpośrednia integracja TCP/IP z modułami ETHM-1 Plus
- **Monitoring temperatury** — SMS z czujników Efento i Bluelog
- **Dane obiektów** — adresy, kontakty, dokumentacja techniczna, CCTV
- **Raportowanie** — historia zdarzeń, metryki reakcji operatorów
- **Mobilny dostęp** — Flutter na Android/iOS (v1.0+)

System docelowo **zastąpi STAM** jako główne narzędzie operacyjne.

### 1.2 Czego system NIE robi

| Poza zakresem | Powód |
|---|---|
| Bezpośrednie sterowanie CCTV (nagrywanie, PTZ) | System przechowuje jedynie metadane dostępowe |
| Monitoring GPS pojazdów | Odrębny system floty |
| Zarządzanie HR / grafiki dyżurów | Dedykowane narzędzia |
| Strefy pożarowe (PPOŻ) | Osobny system certyfikowany |

### 1.3 Role użytkowników

| Rola | Dostęp | Uwagi |
|---|---|---|
| **SYSTEM** | Pełny, programowy | Sub-role: WORKER, SMS, RELAY |
| **MASTER** | Pełny, ręczny | Właściciel systemu |
| **ADMIN** | Zarządzanie obiektami, użytkownikami, panelami | Brak dostępu do audytu systemowego |
| **OPERATOR** | Obsługa alarmów, podgląd obiektów | Dyżurny monitoringu |
| **TECHNICIAN** | Dostęp do przypisanych obiektów + dokumentacja techniczna | Ograniczony scope |
| **FIELD_WORKER** | Minimum: przypisane obiekty read-only + Secret Reveal | Pracownik terenowy |

---

## 2. Architektura Systemu

### 2.1 Warstwy

```
┌─────────────────────────────────────────────────────────┐
│  CLIENT LAYER (Flutter Desktop + Mobile)                 │
│  WebSocket (real-time) + REST API (CRUD, raporty)       │
├─────────────────────────────────────────────────────────┤
│  NGINX (Reverse Proxy, Load Balancer, TLS)               │
├─────────────────────────────────────────────────────────┤
│  API LAYER — Backend-1 + Backend-2 (FastAPI, Python)     │
│  Outbox Relay × 2 (asyncio tasks)                       │
├─────────────────────────────────────────────────────────┤
│  MESSAGING — RabbitMQ (topic + direct exchanges)         │
├─────────────────────────────────────────────────────────┤
│  WORKERS — Satel Worker (asyncio) + SMS Agent            │
├─────────────────────────────────────────────────────────┤
│  DATA LAYER                                              │
│  PostgreSQL Primary + Replica + PgBouncer + etcd         │
│  Redis (live state cache)                                │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Zasady architektoniczne

| Zasada | Realizacja |
|---|---|
| **Event-driven** | RabbitMQ: `satel.events`, `sms.events`, `system.commands` |
| **Atomowość** | Outbox Pattern — INSERT event + outbox w 1 transakcji |
| **Push-based** | WebSocket z Tiered Catch-Up (Redis → PostgreSQL → REST) |
| **HA (99.9% SLO)** | Patroni, multi-instance backend, leader election |
| **Infrastruktura lokalna** | Docker Compose, bez chmury dla core loop |
| **UTC everywhere** | Konwersja na czas lokalny wyłącznie w Flutter UI |

### 2.3 Stack technologiczny

| Warstwa | Technologia | Wersja |
|---|---|---|
| Frontend | Flutter (Desktop + Mobile) | 3.x |
| Backend | Python + FastAPI | 3.11+ / 0.100+ |
| Messaging | RabbitMQ | 3.12+ |
| Database | PostgreSQL | 16+ |
| Cache | Redis | 7+ |
| HA (DB) | Patroni + etcd + PgBouncer | — |
| Reverse Proxy | Nginx | 1.25+ |
| Konteneryzacja | Docker + Docker Compose | 24+ |
| Monitoring | Prometheus + Grafana + Alertmanager | — |
| Logowanie | Structured JSON (opcjonalnie Loki v1.0) | — |
| CI/CD | GitHub Actions / GitLab CI | — |

---

## 3. Model Danych

### 3.1 Encje kluczowe (ERD)

```
OBJECTS ──1:N──> PANELS ──1:N──> ZONES
                   │
                   └──1:N──> SATEL_COMMANDS
                   
EVENTS ──N:1──> BUNDLE_ALARMS ──N:1──> OBJECTS
                   │
AUDIT_LOG          └── version (optimistic locking)

USERS ──N:M──> ROLES
USERS ──N:M──> OBJECTS (user_object_assignments)

OUTBOX (relay → RabbitMQ)
SMS_RAW_ARCHIVE (PII isolation)
```

### 3.2 Kluczowe pola i mechanizmy

| Mechanizm | Pole/Tabela | Opis |
|---|---|---|
| **Optimistic Locking** | `BUNDLE_ALARMS.version` | Każda zmiana statusu inkrementuje wersję; conflict → 409 |
| **Event Ordering** | `OUTBOX.sequence_id` (BIGINT) | Globalny monotoniczny ID dla WebSocket Catch-Up |
| **Deduplikacja** | `EVENTS.dedup_key` (UNIQUE) | Format: `{panel_id}:{event_code}:{zone_id}:{timestamp_minute}` |
| **Partycjonowanie** | `EVENTS` — Range by `created_at` | Automatyczne partycje miesięczne |
| **PII Isolation** | `SMS_RAW_ARCHIVE` | Surowe SMS oddzielnie, dostęp: MASTER/SYSTEM |
| **Priority Override** | `ZONES.priority_override` (JSONB) | Admin nadpisuje default priority per zone |

---

## 4. Przepływ Danych

### 4.1 Alarm Satel (end-to-end)

```
ETHM-1 (TCP:10004, VLAN)
    │
    ▼
Satel Worker ─── parse + CRC check
    │
    ├──▶ Redis: UPDATE panel:{id}:zones (live state, TTL 5 min)
    │
    └──▶ RabbitMQ: PUBLISH satel.alarm.{panel_id}
              │
              ▼
         Backend API: CONSUME events.processing
              │
              ├── Dedup check (dedup_key)
              ├── Priority enrichment (ZONES.priority_override)
              ├── INSERT EVENTS + BUNDLE_ALARMS + OUTBOX (1 transakcja)
              │
              ▼
         Outbox Relay: SELECT pending → PUBLISH to RabbitMQ
              │
              ▼
         WebSocket: PUSH alarm.new (via Redis pub/sub)
              │
              ▼
         Flutter UI: Dźwięk + migająca ikona (< 1s)
```

### 4.2 Alarm SMS (temperatury)

```
Czujnik Efento/Bluelog → SMS
    │
    ▼
Modem GSM → SMS Agent: parse + trusted sender check
    │
    ├── Nieznany nadawca → ignore + log WARNING
    │
    └── Znany nadawca → RabbitMQ: PUBLISH sms.temp.{source}
              │
              ▼
         Backend: CONSUME → INSERT event (TEMP_ALARM/TEMP_NORMAL)
         → Bundle z requires_note = true
         → WebSocket → Operator
```

### 4.3 Komenda sterująca (ARM/DISARM, v2.0)

```
Operator → Flutter UI: "Uzbrój strefę"
    │
    ▼
Backend API: POST /api/commands/satel
    │
    ├── INSERT satel_commands (status: PENDING)
    ├── Redis: partition → "ARMING_STAY" (stan przejściowy)
    │
    └── RabbitMQ: PUBLISH cmd.satel.high
              │
              ▼
         Satel Worker: wysyła 0x80/0x84 do ETHM-1
              │
              ├── ACK → satel_commands.status = ACK, Redis update
              ├── NACK → satel_commands.status = NACK, Redis revert
              └── TIMEOUT → satel_commands.status = TIMEOUT, Redis revert
```

### 4.4 Cykl życia alarmu (Alarm Lifecycle)

```
NEW ──[claim]──▶ CLAIMING (TTL 30s)
                    │
              ┌─[success]──▶ IN_PROGRESS
              │                  │
              │            [ack + notatka]
              │                  │
              │                  ▼
              │                 ACK
              │                  │
              │             [resolve]
              │                  │
              │                  ▼
              │              RESOLVED
              │                  │
              │     ┌─[close]────┤────[auto-close 30min]─┐
              │     ▼                                      ▼
              │   CLOSED                                CLOSED
              │
              └─[timeout 30s]──▶ NEW (revert, reaper job)
```

**Zasady bundlingu:**
- Nowy event dołącza do otwartego bundle'a dla tego samego obiektu/typu (okno: 15 min)
- Re-trigger: nowy event na RESOLVED bundle → nowy bundle
- Auto-close: RESOLVED bundle bez aktywności → CLOSED po 30 min

---

## 5. Harmonogram Implementacji

### Faza 1: MVP (3–4 miesiące)

> Cel: Działający system monitoringu z alarmami Satel i SMS, obsługiwany przez operatorów.

| # | Moduły | Opis |
|---|---|---|
| 1 | **Satel Worker** | TCP/IP ↔ ETHM-1, polling, events do RabbitMQ |
| 2 | **Backend Core** | FastAPI, CRUD obiektów/paneli, alarm processing, bundling |
| 3 | **SMS Agent** | Modem GSM, parser Efento/Bluelog, events do RabbitMQ |
| 4 | **Flutter Desktop** | Dashboard alarmów, lista obiektów, logowanie, Secret Reveal |
| 5 | **Auth & RBAC** | JWT RS256, 6 ról, permission matrix |
| 6 | **WebSocket** | Real-time push alarmów do operatorów |
| 7 | **Outbox Relay** | Atomowe dostarczanie eventów DB→RabbitMQ |
| 8 | **Baza danych** | PostgreSQL + schema + seed data |
| 9 | **Redis** | Live state cache paneli + token blacklist |
| 10 | **Deployment** | Docker Compose (DEV), CI/CD basic |

**User Stories MVP:** US-001 → US-016 (15_USER_STORIES_MVP.md)

### Faza 2: v1.0 (2–3 miesiące po MVP)

| # | Funkcjonalność | Źródło |
|---|---|---|
| 1 | **PostgreSQL HA** — Patroni + etcd + PgBouncer | 09_HA_RTO_RPO |
| 2 | **Multi-instance Backend** — 2+ instancje za Nginx | 11_DEPLOYMENT |
| 3 | **Priorytetyzacja komend RabbitMQ** — high/low queues | 13_EVENT_SCHEMAS |
| 4 | **Raporty** — historia alarmów, eksport CSV/PDF | 03_FUNCTIONAL_MODULES |
| 5 | **Flutter Mobile** — Android (iOS opcjonalnie) | 03_FUNCTIONAL_MODULES |
| 6 | **Monitoring** — Prometheus + Grafana + Alertmanager | 17_MONITORING |
| 7 | **Synthetic Probes** — end-to-end health check co 5 min | 17_MONITORING |
| 8 | **Read Replica** — PostgreSQL replica dla raportów | 11_DEPLOYMENT |
| 9 | **Loki** — centralne logowanie (opcjonalnie) | 17_MONITORING |

### Faza 3: v2.0 (3–6 miesięcy po v1.0)

| # | Funkcjonalność | Źródło |
|---|---|---|
| 1 | **Komendy sterujące Satel** — ARM/DISARM/OUTPUT z UI | 14_SATEL_COMMANDS |
| 2 | **Auto-Arm** — harmonogramy uzbrajania z conflict resolution | 03_FUNCTIONAL_MODULES |
| 3 | **Redis Sentinel** — HA dla Redis | 09_HA_RTO_RPO |
| 4 | **RabbitMQ Quorum Queues** — cluster 3 nody | 09_HA_RTO_RPO |
| 5 | **Offline Mode (Flutter)** — SQLite cache + Intent Queue | 03_FUNCTIONAL_MODULES |
| 6 | **Object Documentation** — pliki, schematy, zdjęcia per obiekt | 03_FUNCTIONAL_MODULES |
| 7 | **Webhook Efento** — HMAC-SHA256 zamiast SMS | 06_INTEGRATIONS |
| 8 | **Chaos Engineering** — miesięczne testy awarii | 16_TESTING_STRATEGY |

### Faza 4: Nice-to-have

| Funkcjonalność | Źródło |
|---|---|
| CCTV metadata (kamera, login, hasło) | 03_FUNCTIONAL_MODULES |
| Moduł zgłoszeń serwisowych (SERVICE_TICKETS) | 03_FUNCTIONAL_MODULES |
| Eskalacja alarmów (auto-escalate po timeout) | 03_FUNCTIONAL_MODULES |
| Ulubione/ostatnie obiekty (per-user) | 03_FUNCTIONAL_MODULES |
| Migracja na Kubernetes | 11_DEPLOYMENT |

---

## 6. Moduły Funkcjonalne

### 6.1 Obiekty (CRUD)

- Centralna encja systemu — wszystko przypisane do obiektu
- Pola: nazwa, adres, typ, lat/lon (geokodowanie), kontakty, status (`ACTIVE/SERVICE/CLOSED`)
- Admin może dodawać/edytować/dezaktywować
- Operator widzi listę z wyszukiwaniem i filtrami

### 6.2 Panele i Strefy

- Panel = centrala Satel ETHM-1 przypisana do obiektu
- Strefa = wejście/partycja w centrali z konfiguracją priority_override
- Widok live: stan stref z Redis (Armed/Disarmed/Alarm/Unknown)

### 6.3 Alarmy i Eventy

- **Raw Event** → `EVENTS` (insert via Backend z dedup)
- **Bundle Alarm** → `BUNDLE_ALARMS` (grupowanie wg obiektu/typu, okno 15 min)
- Priority enrichment: Worker → `default_priority`, Backend → `priority` (z override strefy)
- Wymagane notatki: alarmy temperaturowe (`requires_note = true`)

### 6.4 Integracja Satel (ETHM-1)

| Aspekt | Szczegóły |
|---|---|
| Protokół | TCP/IP, port 10004, CRC-CCITT (init: 0x147A) |
| Heartbeat | 0x7F co 1s, timeout 3s → reconnect |
| Polling | Zones (0x00-0x03), Partitions (0x09), Outputs (0x17), Troubles (0x1C-0x20) |
| Reconnect | Exponential backoff: 1s→2s→4s→8s→16s→30s (cap) |
| ETHM-1 limit | **1 połączenie TCP** — Worker = exclusive |
| DLOAD release | Connection Release API → Worker rozłącza na czas programowania |
| Firmware | ETHM-1 Plus v1.06+ wymagana |

### 6.5 Integracja SMS

| Aspekt | Szczegóły |
|---|---|
| Modem | GSM USB, podłączony do serwera |
| Źródła | Efento (sensor temp.), Bluelog (rejestrator) |
| Trusted senders | Whitelist numerów — nieznany numer = ignore |
| PII | Surowy SMS: SHA-256 hash → `sms_raw_archive` |
| Quality flag | `complete / truncated / garbled / unparseable` |

### 6.6 Timed Secret Reveal

- Hasła CCTV/rejestrator ukryte domyślnie → przycisk "Pokaż"
- Wymagany powód (min. 5 znaków)
- Widoczne 60s z countdownem → auto-hide
- Wpis w `AUDIT_LOG` (kto, co, kiedy, powód)
- Limity/h: Operator 10, Technician 20, Auditor 20

### 6.7 UI Resilience (Flutter)

| Scenariusz | Zachowanie |
|---|---|
| WebSocket disconnect | Skeleton loader + banner "Ponowne łączenie..." |
| Redis down | Stan paneli → "UNKNOWN", banner ostrzegawczy |
| Backend down | Offline mode (read-only z SQLite, v2.0) |
| 409 Conflict | Toast: "Dane zmienione — odśwież" |
| Staleness indicator | "⚠️ Dane z [timestamp]" gdy brak refresh > 30s |

---

## 7. Bezpieczeństwo i Role

### 7.1 Autentykacja

| Element | Implementacja |
|---|---|
| JWT | RS256 (klucz asymetryczny), `kid` header |
| Access Token | 15 min TTL |
| Refresh Token | 7 dni TTL |
| Rotacja kluczy | Nowy klucz co 90 dni, stary akceptowany 7 dni |
| Password hash | Argon2id |
| Brute-force | 5 prób/min → block IP 1 min; 10 prób → lock konto 30 min |
| Mobile | PIN/Biometrics (opcjonalnie v1.0) |

### 7.2 WebSocket Auth

```
1. Client: POST /api/auth/ws-ticket (Bearer header)
2. Backend: generuje ticket (UUID, TTL 30s, single-use)
3. Client: WebSocket connect → /ws?ticket={ticket_id}
4. Backend: waliduje ticket → OK → upgrade do WS
```

### 7.3 Infrastruktura

| Warstwa | Zabezpieczenie |
|---|---|
| Docker Secrets | Wszystkie hasła, klucze JWT, kody Satel |
| VLAN | Izolacja ruchu ETHM-1 (macvlan w Docker) |
| TLS | HTTPS (Nginx) + TLS internal (opcjonalnie) |
| Network | Brak publicznego dostępu do Redis/RabbitMQ/PostgreSQL |

### 7.4 Macierz Uprawnień (skrót)

| Zasób | OPERATOR | TECHNICIAN | FIELD_WORKER | ADMIN | MASTER |
|---|---|---|---|---|---|
| Alarmy: view | ✅ all | ✅ assigned | ❌ | ✅ all | ✅ all |
| Alarmy: claim/ack | ✅ | ❌ | ❌ | ❌ | ✅ |
| Obiekty: edit | ❌ | ❌ | ❌ | ✅ | ✅ |
| Secret Reveal | ✅ (10/h) | ✅ (20/h) | ✅ (20/h) | ✅ | ✅ |
| Użytkownicy: manage | ❌ | ❌ | ❌ | ✅ | ✅ |
| Komendy Satel | ❌ | ❌ | ❌ | ✅ | ✅ |
| Audit Log: view | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 8. Wysoka Dostępność i DR

### 8.1 Cele HA

| Komponent | RTO | RPO | Mechanizm |
|---|---|---|---|
| PostgreSQL | < 30s | < 5 min (async repl.) | Patroni + etcd + PgBouncer |
| Backend API | < 15s | — | Multi-instance za Nginx |
| Satel Worker | < 60s | — | Redis leader election, restart policy |
| Redis | < 30s | — | Sentinel (v2.0), fallback na DB |
| RabbitMQ | < 60s | — | Quorum Queues (v2.0), Outbox kompensuje |

### 8.2 Graceful Shutdown

- FastAPI: SIGTERM → odmów nowych requestów, dokończ in-flight (30s grace)
- Satel Worker: SIGTERM → snapshot stanu do Redis/DB, zamknij TCP, graceful
- Docker: `stop_grace_period: 35s`

### 8.3 System Modes

| Tryb | Warunek | Zachowanie |
|---|---|---|
| NORMAL | Wszystko OK | Pełna funkcjonalność |
| DEGRADED | Redis down / partial | Alarmy przychodzą, live state niedostępny |
| MAINTENANCE | Admin / deploy | Read-only, banner w UI |
| OFFLINE (client) | WS + REST failure | Read-only z SQLite, Intent Queue |

---

## 9. Deployment i Infrastruktura

### 9.1 Kontenery (13 serwisów)

| # | Serwis | Rola |
|---|---|---|
| 1 | `nginx` | Reverse proxy, TLS, load balancer |
| 2 | `backend-1` | API instance #1 |
| 3 | `backend-2` | API instance #2 |
| 4 | `outbox-relay-1` | Outbox → RabbitMQ relay #1 |
| 5 | `outbox-relay-2` | Outbox → RabbitMQ relay #2 |
| 6 | `satel-worker` | TCP/IP ↔ ETHM-1 |
| 7 | `pg-primary` | PostgreSQL primary |
| 8 | `pg-replica` | PostgreSQL read replica |
| 9 | `etcd` | Patroni consensus store |
| 10 | `pgbouncer` | Connection pooling |
| 11 | `redis` | Live state cache |
| 12 | `rabbitmq` | Message broker |
| 13 | `sms-agent` | GSM modem ↔ SMS parsing |

### 9.2 Sieci Docker

| Sieć | Typ | Serwisy |
|---|---|---|
| `stam_internal` | bridge | Wszystkie |
| `satel_vlan` | macvlan | `satel-worker` (izolacja ETHM-1) |

### 9.3 Środowiska

| Env | Cel | HA | CI/CD |
|---|---|---|---|
| **DEV** | Rozwój local | 1 instancja | — |
| **STAGE** | Testy, chaos engineering | Pełna replikacja PROD | Auto-deploy z main |
| **PROD** | Produkcja | Pełne HA | Manual approval |

### 9.4 Backup

| Element | Metoda | Częstotliwość | Retencja |
|---|---|---|---|
| PostgreSQL | `pg_dump` + WAL archiving | Codziennie (full) + ciągły WAL | 30 dni |
| Redis | RDB snapshot | Nie backupowany (rebuilt z DB) | — |
| Docker Compose files | Git repo | Przy każdej zmianie | ∞ |
| Logi | Rotacja + Loki (v1.0) | Ciągle | 90 dni (PROD) |

---

## 10. Monitoring i Alerty

### 10.1 Stack

Prometheus (scrape 15s) → Grafana (dashboardy) → Alertmanager (SMS/email)

### 10.2 Kluczowe metryki

| Kategoria | Metryka | Alert threshold |
|---|---|---|
| **Biznesowe** | `alarms_active` | > 50 → WARNING |
| | `alarm_response_time_seconds` | p95 > 5 min → WARNING |
| **Satel** | `satel_connection_status` | 0 > 5 min → CRITICAL |
| **RabbitMQ** | `rabbitmq_queue_messages` | > 1000 → WARNING |
| **Outbox** | `outbox_pending_count` | > 100 > 5 min → CRITICAL |
| **SMS** | `sms_last_received_timestamp` | > 1h → WARNING |

### 10.3 SLO / SLI

| SLO | Cel | Error Budget (30d) |
|---|---|---|
| Dostarczalność alarmów < 2s | 99.9% | Max 43 miss |
| Dostępność API (non-5xx) | 99.95% | ~21 min downtime |
| Zero utraconych CRITICAL | 100% | 0 tolerancji |

### 10.4 Synthetic Probes

Co 5 min: test event → RabbitMQ → PostgreSQL → WebSocket → metryka latency. Brak dostarczenia > 10 min → CRITICAL.

### 10.5 Dashboardy Grafana

1. **Operations** — aktywne alarmy, czas reakcji, top 5 obiektów
2. **Infrastructure** — Satel connections, queue depth, Redis, HTTP latency
3. **Satel Worker** — TCP per panel, poll duration, CRC errors, reconnecty

---

## 11. Testowanie

### 11.1 Piramida testów

| Warstwa | Target coverage | Narzędzia |
|---|---|---|
| Unit | 80% backend, 70% frontend | Python: pytest, Flutter: widget tests |
| Integration | Krytyczne ścieżki | FastAPI test client, Docker `test` profile |
| E2E | 5 smoke scenarios | Flutter driver, mock services |
| Performance | Benchmarks per endpoint | locust / k6 |
| Security | OWASP Top 10 checklist | bandit, safety, manual |

### 11.2 E2E Smoke Tests (run na każdym deploy)

| # | Scenariusz |
|---|---|
| E2E-001 | Login → Dashboard |
| E2E-002 | Full Alarm Flow (event → close) |
| E2E-003 | Temp Alarm + Note |
| E2E-004 | Object CRUD |
| E2E-005 | Secret Reveal + AUDIT_LOG |

### 11.3 Chaos Engineering (STAGE only, 8 scenariuszy)

| # | Scenariusz | Metoda |
|---|---|---|
| CH-01 | Redis down | `docker compose stop redis` |
| CH-02 | RabbitMQ down | `docker compose stop rabbitmq` |
| CH-03 | PostgreSQL failover | `docker compose stop pg-primary` |
| CH-04 | Backend crash | `docker compose kill backend-1` |
| CH-05 | Satel Worker crash | `docker compose restart satel-worker` |
| CH-06 | Network partition | `iptables DROP` on satel_vlan |
| CH-07 | Disk full (RabbitMQ) | Fill to 95% |
| CH-08 | Burst 500 alarms | Simulator: 500 violations in 10s |

### 11.4 DR Drills

| Typ | Częstotliwość |
|---|---|
| Tabletop exercise | Miesięcznie |
| Failover drill (Patroni) | Kwartalnie |
| Full DR restore | Co pół roku |
| Chaos day (losowe scenariusze) | Co pół roku |

---

## 12. Procedury Operacyjne

### 12.1 Scenariusze awaryjne (Quick Reference)

| # | Scenariusz | Alert | Krok 1 | Eskalacja |
|---|---|---|---|---|
| RUN-001 | Satel Worker disconnect | SatelWorkerDown | Sprawdź kontener + logi | > 30 min → MASTER |
| RUN-002 | Redis down | RedisDown | `docker restart redis` | System działa w degraded mode |
| RUN-003 | RabbitMQ overflow | QueueBacklog | Sprawdź consumers, restart backend | Events bufferowane, brak utraty |
| RUN-004 | SMS modem silent | SMSModemSilent | Logi + fizyczna weryfikacja | > 2h → zapasowy modem |
| RUN-005 | Database full | DiskSpaceLow | `df -h` + archiwizacja starych eventów | — |
| RUN-014 | Outbox stuck | OutboxStuck | Sprawdź obie instancje relay | Alarmy w DB, nie docierają do operatora |
| RUN-015 | WS buffer overflow | WSBufferFull | Check `ws_replay_overflow_count` | Klienci przeładują z REST |
| RUN-016 | Auto-Arm failure | auto_arm_failure | `SELECT satel_commands WHERE source='AUTO_ARM'` | Ręczne uzbrojenie |
| RUN-017 | RabbitMQ disk alarm | RabbitMQ alarm | Management UI :15672 | Purge DLQ, resize volume |

### 12.2 Procedury administracyjne

| Procedura | Runbook | Kto |
|---|---|---|
| Onboarding obiektu | RUN-010 | ADMIN |
| Backup restore | RUN-011 | DevOps |
| Deploy nowej wersji | RUN-012 / RUN-018 | DevOps |
| Dodanie użytkownika | RUN-013 | ADMIN |

### 12.3 Eskalacja

| Poziom | Czas | Kto | Kanał |
|---|---|---|---|
| L1 | 0–10 min | On-call primary | SMS/Telefon (auto) |
| L2 | 10–30 min | On-call backup | SMS/Telefon |
| L3 | > 30 min | MASTER | Telefon bezpośredni |

### 12.4 Post-Incident

Po każdym CRITICAL: Blameless Postmortem → template w `18_RUNBOOK.md` (RUN-05).

---

## 13. Decyzje Architektoniczne

### 13.1 ADR — Podjęte decyzje

| # | Decyzja | Status | Uzasadnienie |
|---|---|---|---|
| ADR-001 | Flutter (Desktop + Mobile) | ✅ Accepted | Jeden codebase, szybki dev, native performance |
| ADR-002 | PostgreSQL jako core DB | ✅ Accepted | ACID, partycjonowanie, replikacja |
| ADR-003 | Redis jako live state cache | ✅ Accepted | Niski latency, TTL, pub/sub |
| ADR-004 | TCP/IP direct (nie WebSocket/MQTT) z ETHM-1 | ✅ Accepted | Dokumentowany protokół Satel, brak alternatyw |
| ADR-005 | Docker Secrets | ✅ Accepted | Native, bezpieczne, bez external vault |
| ADR-006 | Optimistic Locking (version) | ✅ Accepted | Eliminuje race conditions przy claim/ack |
| ADR-007 | Outbox Pattern | ✅ Accepted | Atomowość DB↔MQ, brak dual-write |
| ADR-008 | Tiered WebSocket Catch-Up | ✅ Accepted | Redis → PostgreSQL → REST, resilient |
| ADR-009 | Patroni (PostgreSQL HA) | ✅ Accepted | Automated failover < 30s |
| ADR-010 | RS256 JWT | ✅ Accepted | Asymetryczny, key rotation, `kid` |

### 13.2 Pytania otwarte (Remaining)

| # | Pytanie | Status |
|---|---|---|
| OQ-1 | Offline map strategy (Flutter) | OPEN — cache tiles vs dedicated SDK |
| OQ-2 | Migracja danych z STAM | OPEN — format eksportu do ustalenia |
| OQ-3 | DB HA tooling | ✅ RESOLVED — Patroni |

---

## Indeks Plików Źródłowych

| Plik | Sekcje w Master Guide |
|---|---|
| 01_PROJECT_OVERVIEW | §1 |
| 02_ARCHITECTURE | §2, §4, §8 |
| 03_FUNCTIONAL_MODULES | §5, §6 |
| 04_DATA_MODEL_ERD | §3 |
| 05_ALARM_LIFECYCLE | §4.4 |
| 06_INTEGRATIONS | §6.4, §6.5 |
| 07_TECH_STACK | §2.3 |
| 08_SECURITY_AND_ROLES | §7 |
| 09_HA_RTO_RPO | §8 |
| 10_API_HIGH_LEVEL | §4, §7.2 |
| 11_DEPLOYMENT | §9 |
| 12_DECISIONS_AND_OPEN | §13 |
| 13_EVENT_SCHEMAS | §4.1, §4.2, §4.3 |
| 14_SATEL_COMMANDS | §6.4 |
| 15_USER_STORIES_MVP | §5 (Faza 1) |
| 16_TESTING_STRATEGY | §11 |
| 17_MONITORING | §10 |
| 18_RUNBOOK | §12 |

---

> **Dokument wygenerowany:** 2026-02-20 | **Źródła:** 18 plików architektury STAM REPLACER v8.0 | **Status:** READY FOR IMPLEMENTATION 🚀
