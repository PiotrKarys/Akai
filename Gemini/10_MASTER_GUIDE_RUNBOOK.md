# MASTER GUIDE — STAM REPLACER

> **Wersja:** 1.0 | **Data:** 2026-02-20 | **Źródła:** Pliki 01–18 + MY_HONEST_OPINION.md
> Kompleksowy przewodnik implementacyjno-operacyjny systemu monitoringu bezpieczeństwa.

---

## SPIS TREŚCI

1. [Wizja i Zakres Systemu](#1-wizja-i-zakres-systemu)
2. [Architektura Systemu](#2-architektura-systemu)
3. [Model Danych](#3-model-danych)
4. [Przepływ Danych (Data Flow Masterplan)](#4-przepływ-danych)
5. [Harmonogram Implementacji (Fazy 1–6)](#5-harmonogram-implementacji)
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

> [!IMPORTANT]
> System wdrażany w **6 fazach**. Każda faza jest samodzielnie deployowalna. Feature Flags (`19_DEV_TOOLING.md`) kontrolują dostępność funkcji między fazami.

### Faza 1: Fundament (CRUD + Auth + Desktop + Mobile read-only)

> Cel: Backend z CRUD obiektów/paneli, Auth/RBAC, Flutter Desktop + Mobile (read-only, SQLite sync). **Zero alarmów, zero RabbitMQ, zero Redis.**

| # | Moduły | Opis |
|---|---|---|
| 1 | **Backend Core** | FastAPI, CRUD obiektów/paneli/stref, Auth JWT RS256, RBAC |
| 2 | **Flutter Desktop** | Lista obiektów, szczegóły, logowanie, Secret Reveal |
| 3 | **Flutter Mobile** | Read-only, SQLite sync, podgląd obiektów |
| 4 | **Auth & RBAC** | JWT RS256, 6 ról, permission matrix |
| 5 | **Baza danych** | PostgreSQL + schema + seed data (`seed_dev.py`) |
| 6 | **Deployment** | Docker Compose (DEV), Nginx, CI/CD basic |
| 7 | **Konfiguracja per obiekt** | `OBJECT_CONTACTS`, `OBJECT_PROCEDURES`, `OBJECT_ALARM_CONFIG` |
| 8 | **Tryb Serwisowy (config)** | Kolumny `service_mode_*` w OBJECTS, endpointy aktywacji/dezaktywacji |
| 9 | **Import Excel** | `POST /api/admin/import/objects` — import danych z .xlsx |
| 10 | **Structured Logging** | `structlog` + `request_id` od Fazy 1 |

**User Stories:** US-001 → US-016 + US-017 (Tryb Serwisowy config) + US-018 (Kontakty) + US-020 (Import Excel)

### Faza 2: Alarm Pipeline (Satel Worker + RabbitMQ + Redis + WebSocket)

> Cel: Działający pipeline alarmowy — od centrali ETHM-1 przez RabbitMQ do operatora w real-time.

| # | Funkcjonalność | Źródło |
|---|---|---|
| 1 | **Satel Worker** | TCP/IP ↔ ETHM-1, polling, events do RabbitMQ | 06_INTEGRATIONS |
| 2 | **RabbitMQ** | Message broker, topic/direct exchanges | 13_EVENT_SCHEMAS |
| 3 | **Redis** | Live state cache paneli + token blacklist | 07_TECH_STACK |
| 4 | **Outbox Relay** | Atomowe dostarczanie eventów DB→RabbitMQ | 02_ARCHITECTURE |
| 5 | **WebSocket** | Real-time push alarmów do operatorów + Tiered Catch-Up | 10_API_HIGH_LEVEL |
| 6 | **Alarm Lifecycle** | Bundling, dedup, statusy NEW→CLOSED | 05_ALARM_LIFECYCLE |
| 7 | **Tryb Serwisowy (efekt)** | Tłumienie priorytetu, brak dźwięku, szary styl + 🔧 | 05_ALARM_LIFECYCLE |
| 8 | **Push Notifications** | FCM Android — CRITICAL only, gdy app w tle | 06_INTEGRATIONS |
| 9 | **Dziennik Dyspozytora** | DISPATCH_LOG, wpisy z poziomu alarmu/obiektu | 03_FUNCTIONAL_MODULES |
| 10 | **Historia Stanu Centrali** | PANEL_STATE_HISTORY, timeline stanu partycji | 03_FUNCTIONAL_MODULES |

**User Stories:** US-019 (Dziennik Dyspozytora)

### Faza 3: Multi-Object + Monitoring

> Cel: Obsługa wielu obiektów i central jednocześnie. Pełny stack monitoringu.

| # | Funkcjonalność | Źródło |
|---|---|---|
| 1 | **Multi-object/panel** | Obsługa 50+ obiektów, wielu central | 03_FUNCTIONAL_MODULES |
| 2 | **Prometheus + Grafana** | Metryki biznesowe i infrastrukturalne | 17_MONITORING |
| 3 | **Alertmanager** | Reguły alertowania (CRITICAL/WARNING/INFO) | 17_MONITORING |
| 4 | **Synthetic Probes** | End-to-end health check co 5 min | 17_MONITORING |
| 5 | **Stress testy** | Burst 200+ events, 10 concurrent operators | 16_TESTING_STRATEGY |

### Faza 4: SMS Agent (Temperatury)

> Cel: Integracja SMS z czujnikami Efento i Bluelog.

| # | Funkcjonalność | Źródło |
|---|---|---|
| 1 | **SMS Agent** | Modem GSM, parser Efento/Bluelog, events do RabbitMQ | 06_INTEGRATIONS |
| 2 | **Alarmy temperaturowe** | Bundle z `requires_note = true`, TEMP_ALARM/TEMP_NORMAL | 05_ALARM_LIFECYCLE |
| 3 | **PII Isolation** | SHA-256 hash SMS → `SMS_RAW_ARCHIVE` | 06_INTEGRATIONS |

### Faza 5: Arm/Disarm z UI

> Cel: Zdalne uzbrajanie/rozbrajanie stref z Flutter.

| # | Funkcjonalność | Źródło |
|---|---|---|
| 1 | **Komendy sterujące** | ARM/DISARM/OUTPUT z UI → Satel Worker → ETHM-1 | 14_SATEL_COMMANDS |
| 2 | **Priority Queues** | `cmd.satel.high` / `cmd.satel.low` w RabbitMQ | 13_EVENT_SCHEMAS |
| 3 | **Stany przejściowe** | Redis: ARMING_STAY → ACK/NACK/TIMEOUT | 13_EVENT_SCHEMAS |

### Faza 6: HA + Raporty + Rozszerzenia

> Cel: Wysoka dostępność, raporty, Auto-Arm, Service Tickets, dokumentacja obiektów.

| # | Funkcjonalność | Źródło |
|---|---|---|
| 1 | **PostgreSQL HA** | Patroni + etcd + PgBouncer | 09_HA_RTO_RPO |
| 2 | **Multi-instance Backend** | 2+ instancje za Nginx | 11_DEPLOYMENT |
| 3 | **Redis Sentinel** | HA dla Redis (3 instancje) | 09_HA_RTO_RPO |
| 4 | **RabbitMQ Quorum Queues** | Cluster 3 nody | 09_HA_RTO_RPO |
| 5 | **Raporty** | Historia alarmów, eksport CSV/PDF, Read Replica | 03_FUNCTIONAL_MODULES |
| 6 | **Auto-Arm** | Harmonogramy uzbrajania z conflict resolution | 03_FUNCTIONAL_MODULES |
| 7 | **Service Tickets** | Moduł zgłoszeń serwisowych | 03_FUNCTIONAL_MODULES |
| 8 | **Object Documentation** | Pliki, schematy, zdjęcia per obiekt | 03_FUNCTIONAL_MODULES |
| 9 | **Offline Mode (Flutter)** | SQLite cache + Intent Queue | 03_FUNCTIONAL_MODULES |
| 10 | **Chaos Engineering** | Miesięczne testy awarii | 16_TESTING_STRATEGY |

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


---
---


# 18_RUNBOOK

## Cel
Procedury operacyjne dla typowych scenariuszy awaryjnych i administracyjnych. Przewodnik "co robić o 3 w nocy".

---

## 1. Scenariusze Awaryjne

---

### RUN-001: Satel Worker — utrata połączenia z centralą

**Alert:** `SatelWorkerDown` (CRITICAL)
**Symptom:** Dashboard: panel X → "DISCONNECTED" od > 5 min

**Procedura:**

| Krok | Akcja | Oczekiwany wynik |
|---|---|---|
| 1 | Sprawdź status kontenera `satel-worker` | Czy działa? Restartował się? |
| 2 | Sprawdź logi: `docker logs satel-worker --tail 100` | Szukaj: "Connection refused", "Timeout", "CRC error" |
| 3 | Sprawdź łączność sieciową z ETHM-1: `telnet {ethm_ip} 10004` | Czy port odpowiada? |
| 4 | Sprawdź czy ktoś inny nie jest podłączony do ETHM-1 (limit: 1 połączenie) | Odłącz inne narzędzia (DLOAD, STAM) |
| 5 | Restart kontenera: `docker restart satel-worker` | Worker powinien się połączyć w ciągu 30s |
| 6 | Jeśli nadal nie działa → sprawdź fizycznie ETHM-1 (zasilanie, kabel sieciowy) | Dioda ETHM-1 miga? |
| 7 | Jeśli ETHM-1 wymaga resetu → wyłącz/włącz zasilanie modułu | Worker połączy się automatycznie |

**Eskalacja:** Jeśli > 30 min bez połączenia → powiadom MASTER, ponieważ brak monitoringu obiektu.

---

### RUN-002: Redis Down

**Alert:** `RedisDown` (CRITICAL)
**Symptom:** Backend loguje "Redis connection refused", dashboard stanu central pusty

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź kontener: `docker ps \| grep redis` |
| 2 | Restart: `docker restart redis` |
| 3 | Weryfikacja: `docker exec redis redis-cli ping` → "PONG" |
| 4 | Po restarcie: Worker automatycznie odbuduje cache (fresh poll central) |
| 5 | Backend: fallback na direct DB queries (wolniejsze, ale działa bez Redis) |

**Wpływ:** System działa BEZ Redis (degraded mode). Stan live central niedostępny do momentu odbudowy cache. Alarmy nadal przechodzą (przez RabbitMQ → DB).

---

### RUN-003: RabbitMQ Queue Overflow

**Alert:** `QueueBacklog` (WARNING → CRITICAL)
**Symptom:** `rabbitmq_queue_messages > 1000` rośnie zamiast spadać

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź czy Backend konsumuje: `rabbitmq_consumers > 0`? |
| 2 | Jeśli consumers = 0 → kontener Backend padł. Restart: `docker restart backend` |
| 3 | Jeśli consumers > 0 ale queue rośnie → Backend nie nadąża. Sprawdź logi backendu pod kątem błędów DB |
| 4 | Sprawdź PostgreSQL: `docker exec db psql -U user -c "SELECT count(*) FROM pg_stat_activity"` |
| 5 | Jeśli DB zablokowana → restart DB + Backend |
| 6 | Po rozwiązaniu: kolejka przetworzy zaległe eventy automatycznie |

**Wpływ:** Eventy są buforowane. Żaden event nie jest tracony (RabbitMQ durable). Po rozwiązaniu — wszystkie zaległe alarmy pojawią się u operatora.

---

### RUN-004: Modem SMS nie odbiera

**Alert:** `SMSModemSilent` (WARNING)
**Symptom:** Brak SMSów > 1h (normalnie powinny przychodzić co ~15-30 min jeśli temperatura w normie)

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź kontener: `docker logs sms-agent --tail 50` |
| 2 | Sprawdź fizycznie: czy modem świeci? Czy ma sygnał? |
| 3 | Sprawdź kartę SIM: czy ma środki / nie jest zablokowana? |
| 4 | Test manualny: wyślij SMS na numer modemu z innego telefonu |
| 5 | Restart kontenera: `docker restart sms-agent` |
| 6 | Jeśli nie pomaga: odłącz/podłącz modem USB fizycznie |
| 7 | Sprawdź w dmesg czy system rozpoznaje urządzenie: `dmesg \| tail` |

**Eskalacja:** Jeśli modem martwy > 2h → użyj zapasowego modemu (na półce).

---

### RUN-005: Database Full

**Alert:** `DiskSpaceLow` (WARNING)
**Symptom:** PostgreSQL loguje "disk full" / "no space left"

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź disk: `df -h` |
| 2 | Wyczyść stare logi Dockera: `docker system prune --volumes` (UWAGA: nie usuwaj wolumenów bazy!) |
| 3 | Sprawdź rozmiar bazy: `docker exec db psql -U user -c "SELECT pg_size_pretty(pg_database_size('stam'))"` |
| 4 | Jeśli tabela EVENTS za duża → rozważ archiwizację starych eventów (> 90 dni) |
| 5 | Jeśli logi Dockera za duże → skonfiguruj log rotation w `daemon.json` |

---

## 2. Procedury Administracyjne

---

### RUN-010: Onboarding nowego obiektu

| Krok | Kto | Akcja |
|---|---|---|
| 1 | ADMIN | Dodaj obiekt w aplikacji (nazwa, adres, kontakty) |
| 2 | ADMIN | Dodaj centralę do obiektu (panel_id, model, IP ETHM-1) |
| 3 | ADMIN | Skonfiguruj ETHM-1 — włącz integrację, ustaw port 10004 |
| 4 | ADMIN | Dodaj konfigurację centrali do Satel Worker (config file / DB) |
| 5 | ADMIN | Restart Satel Worker → Worker połączy się z nową centralą |
| 6 | ADMIN | Sprawdź na dashboardzie: status "CONNECTED", stan stref widoczny |
| 7 | ADMIN | Test: uzbroj/rozbrój strefę na obiekcie → sprawdź czy event pojawił się w systemie |

---

### RUN-011: Backup Restore

| Krok | Akcja |
|---|---|
| 1 | Zatrzymaj kontenery: `docker compose down` |
| 2 | Zlokalizuj backup: `ls -la /backups/pg_dumps/` |
| 3 | Restore: `docker exec -i db psql -U user -d stam < backup_YYYYMMDD.sql` |
| 4 | Start usług: `docker compose up -d` |
| 5 | Weryfikacja: zaloguj się, sprawdź czy dane są aktualne |
| 6 | Satel Worker: automatycznie reconnect do central |
| 7 | Redis: automatycznie odbuduje cache |

---

### RUN-012: Deploy nowej wersji

| Krok | Akcja |
|---|---|
| 1 | Sprawdź czy CI/CD green na main |
| 2 | SSH do serwera PROD |
| 3 | Pull najnowsze obrazy: `docker compose pull` |
| 4 | Restart z nową wersją: `docker compose up -d --force-recreate` |
| 5 | Sprawdź logi: `docker compose logs -f --tail 50` |
| 6 | Smoke test: zaloguj się, sprawdź listę alarmów, stan central |
| 7 | Jeśli problemy → rollback: `docker compose up -d --force-recreate` z image digest z poprzedniej wersji |

---

### RUN-013: Dodanie użytkownika

| Krok | Kto | Akcja |
|---|---|---|
| 1 | ADMIN | Panel Admin → Użytkownicy → Dodaj nowego |
| 2 | ADMIN | Wypełnij: email, hasło tymczasowe, rola (OPERATOR/TECHNICIAN/etc.) |
| 3 | ADMIN | Jeśli FIELD_WORKER/TECHNICIAN → przypisz obiekty (sekcja „Przypisania obiektów") |
| 4 | ADMIN | Wymuś zmianę hasła przy pierwszym logowaniu |
| 5 | ADMIN | Weryfikacja: nowy użytkownik loguje się i widzi swój zakres danych |

---

### RUN-014: Outbox Relay zablokowany (alarmy nie docierają)

**Symptom:** Alert `OutboxStuck` — `outbox_pending_count > 100` przez > 5 min. Alarmy widoczne w PostgreSQL, ale operatorzy ich nie widzą.

> [!NOTE]
> Od v8.0 ruch Outbox Relay jest rozłożony na **2 instancje** (`outbox-relay-1`, `outbox-relay-2`). Obie instancje muszą być sprawdzone.

```
Diagnoza:
├── Sprawdź obie instancje relay:
│   ├── docker compose ps outbox-relay-1 outbox-relay-2
│   ├── Obie żyją? → Sprawdź logi obu instancji
│   └── Jedna padła? → Restart: docker compose up -d outbox-relay-{N}
│       (druga instancja kontynuuje przetwarzanie — brak utraty)
│   │         ├── "Connection refused" do RabbitMQ → idź do RUN-003
│   │         ├── "Timeout" publishing → RabbitMQ przeciążony? → RUN-017
│   │         └── Unhandled exception → restart: docker compose restart outbox-relay
│   └── NIE → docker compose up -d outbox-relay
│
├── Po restarcie: obserwuj outbox_pending_count (powinien spadać)
│   ├── Spada → OK, obserwuj 15 min
│   └── Nie spada → sprawdź RabbitMQ management: http://host:15672
│       └── Jeśli dead letter queue niepusta → ręcznie republish lub eskaluj
│
└── Szacowany czas: 5-15 min
```

---

### RUN-015: WebSocket buffer overflow (operatorzy tracą eventy)

**Symptom:** Alert `WSBufferFull` lub `ws_replay_overflow_count > 0`. Operatorzy zgłaszają "brak nowych alarmów".

```
Diagnoza:
├── Skala problemu: ile sesji WS dotknęło overflow?
│   └── Sprawdź metric: ws_replay_overflow_count (per instance)
│
├── Przyczyna:
│   ├── Operator offline > 2h → Tier 1 (Redis, 5000 eventów) i Tier 2 (PostgreSQL, 2h) wyczerpane
│   │   └── Normalne zachowanie. Klient po reconnect dostanie replay_overflow
│   │       z retry_after_ms (thundering herd protection).
│   │       Klient czeka retry_after_ms i przeładuje dane z REST API.
│   ├── Redis restart → Tier 2 (PostgreSQL fallback) przejmuje → brak overflow
│   │   └── Sprawdź: czy outbox.sequence_id rośnie normalnie
│   ├── Burst alarmów (np. 500 obiektów naraz) → chwilowy spike
│   │   └── Sprawdź: czy events.processing queue rośnie? → QueueBacklog
│   └── Backend memory leak → WS sesje nie zamykane
│       └── docker compose restart backend-1 (potem backend-2)
│
└── Szacowany czas: 5-10 min
```

---

### RUN-016: Auto-Arm failure (centrala nie uzbroiła się wg harmonogramu)

**Symptom:** Alert `satel_auto_arm_failure` lub raport poranny: obiekt niezabezpieczony w nocy.

```
Diagnoza:
├── Czy komenda została wysłana?
│   └── SELECT * FROM satel_commands 
│       WHERE source='AUTO_ARM' AND panel_id='...' ORDER BY created_at DESC LIMIT 5
│       ├── status=ACK → centrala potwierdziła, ale strefa otwarta?
│       │   └── Sprawdź: czy strefy zamknięte? (violation na wejściu)
│       │       └── Jeśli tak → zadzwoń do obiektu (drzwi otwarte?)
│       ├── status=NACK → centrala odrzuciła (np. strefa w alarmie)
│       │   └── Sprawdź event log centrali
│       ├── status=TIMEOUT → ETHM-1 nie odpowiedziało
│       │   └── Sprawdź Worker logs + satel_connection_status
│       └── Brak komendy → Auto-Arm scheduler nie odpalił
│           └── Sprawdź: czy scheduling cron działa? docker compose logs backend | grep auto_arm
│
├── Interwencja:
│   └── Ręczne uzbrojenie: POST /api/satel/command {command_type: "ARM_STAY", panel_id: "..."}
│
└── Szacowany czas: 5-20 min
```

---

### RUN-017: RabbitMQ disk alarm (kolejka zablokowana)

**Symptom:** Alert `RabbitMQ disk alarm` lub `rabbitmq_queue_messages` stale rośnie ale `consumers > 0`.

```
Diagnoza:
├── RabbitMQ Management UI: http://host:15672
│   ├── Zakładka "Overview" → czy jest "Disk space alarm"?
│   │   ├── TAK → dysk pełny
│   │   │   ├── Sprawdź: df -h /var/lib/rabbitmq
│   │   │   ├── Wyczyść stare logi: find /var/log/rabbitmq -mtime +7 -delete
│   │   │   ├── Jeśli dead letter queue duża → purge: rabbitmqadmin purge queue name=events.dead
│   │   │   └── Rozważ zwiększenie dysku (docker volume resize)
│   │   └── NIE → sprawdź consumers
│   │       ├── Consumers = 0 → Backend nie konsumuje → restart backend
│   │       └── Consumers > 0 ale messages rośnie → Backend za wolny
│   │           └── Sprawdź: pg_connections_active, response times
│   │               → Prawdopodobnie PostgreSQL bottleneck
│
└── Szacowany czas: 10-30 min
```

---

### RUN-018: Deploy z Rollback (szczegółowa procedura)

**Wymaganie:** Każdy deploy musi mieć zapisany image digest umożliwiający natychmiastowy rollback.

| Krok | Czas | Akcja |
|---|---|---|
| 1 | 0 min | Zapisz aktualny stan: `docker compose images > /var/log/deploy/pre_deploy_$(date +%Y%m%d_%H%M).txt` |
| 2 | 1 min | Pull nowe obrazy: `docker compose pull` |
| 3 | 2 min | Rolling restart backend-1: `docker compose up -d --no-deps backend-1` |
| 4 | 3 min | Poczekaj na `/readyz` → 200 OK (max 30s) |
| 5 | 4 min | Rolling restart backend-2: `docker compose up -d --no-deps backend-2` |
| 6 | 5 min | Restart Worker: `docker compose up -d --no-deps satel-worker` |
| 7 | 6 min | Smoke test: login, lista alarmów, stan central, WebSocket push |
| 8 | 7 min | ✅ Jeśli OK → gotowe. Zapisz: `docker compose images > /var/log/deploy/post_deploy_$(date +%Y%m%d_%H%M).txt` |

**Rollback (jeśli smoke test failed):**

| Krok | Akcja |
|---|---|
| 1 | Odczytaj image digest z `pre_deploy_*.txt` |
| 2 | `docker compose up -d --force-recreate` z pinowanymi digestami |
| 3 | Smoke test rollbacku |
| 4 | Loguj incydent: kto, co, dlaczego rollback |

---

## 3. Blameless Postmortem (RUN-05 Template)

> Po każdym incydencie CRITICAL stosuj ten szablon. **„Blameless"** = szukamy przyczyn systemowych, nie winnych ludzi.

```markdown
# Postmortem: [Tytuł incydentu]

## Podsumowanie
- **Data/czas:** YYYY-MM-DD HH:MM UTC
- **Czas trwania:** X min
- **Impact:** Co było niedostępne? Ilu operatorów dotkniętych?
- **Severity:** CRITICAL / WARNING

## Timeline
| Czas (UTC) | Zdarzenie |
|---|---|
| HH:MM | Pierwszy alert: [nazwa alertu] |
| HH:MM | On-call acknowledged |
| HH:MM | Diagnoza: [co ustalono] |
| HH:MM | Mitygacja: [co zrobiono] |
| HH:MM | Potwierdzenie rozwiązania |

## Root Cause
[Opis głównej przyczyny — technicznej, nie ludzkiej]

## Co zadziałało dobrze?
- [np. alert wykrył problem w < 5 min]

## Co nie zadziałało?
- [np. runbook nie obejmował tego scenariusza]

## Action Items
| # | Akcja | Priorytet | Odpowiedzialny | Termin |
|---|---|---|---|---|
| 1 | ... | CRITICAL | ... | ... |
| 2 | ... | HIGH | ... | ... |
```

---

## 4. Kontakty Eskalacyjne

> **Uwaga:** Dla pełnej konfiguracji rotacji dyżurów — patrz `17_MONITORING.md`, sekcja 10.

| Priorytet | Czas reakcji | Do kogo | Kanał |
|---|---|---|---|
| CRITICAL (system down) | < 10 min | On-call primary (PagerDuty) | SMS + Telefon (auto) |
| CRITICAL (eskalacja L2) | < 30 min | On-call backup | SMS + Telefon |
| CRITICAL (eskalacja L3) | > 30 min | MASTER (właściciel) | Telefon bezpośredni |
| WARNING (degradacja) | < 2h | ADMIN | Email + Push |
| INFO (obserwacja) | Następny dzień roboczy | ADMIN | Dashboard |

---

## 5. Nowe Metryki i Alerty (v8.0 Remediation)

| Metryka / Alert | Typ | Opis | Severity |
|---|---|---|---|
| `claiming_timeout_reverts_total` | Counter | Alarmy zrevertowane z CLAIMING do NEW (CLAIMING Reaper Job) | INFO (>10/h → WARNING) |
| `outbox_relay_active_instances` | Gauge | Liczba aktywnych instancji relay | CRITICAL jeśli <1 przez >30s |
| `ws_replay_tier2_queries_total` | Counter | Czerpanie z PostgreSQL zamiast Redis (Tier 2 Catch-Up) | INFO (wzrost → Redis problem) |
| `ws_replay_overflow_count` | Counter | Klienci otrzymujący replay_overflow (Tier 3) | WARNING jeśli >5/min |
| `stale_alarm_report_bundles` | Gauge | Liczba CRITICAL Bundle otwartych >24h | CRITICAL jeśli >0 |
| `intent_sync_rejected_total` | Counter | Odrzucone intencje offline (INTENT_REJECTED) | INFO |
| `sms_raw_archive_access_total` | Counter | Dostępy do surowych treści SMS | WARNING (każdy dostęp logowany) |


---
---


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
