# 02_ARCHITECTURE.md

## 1. Overview

System jest event-driven, wysokodostępny (HA), oparty o lokalną infrastrukturę firmy z możliwością integracji z chmurą w przyszłości.

Architektura została zaprojektowana pod:
- dużą liczbę alarmów w krótkim czasie,
- niskie opóźnienia (reakcja < 1s dzięki TCP/IP),
- odporność na awarie,
- możliwość pracy wielu operatorów równolegle,
- łatwą rozbudowę w przyszłości.

System NIE obsługuje live CCTV ani streamów wideo.
W bazie są wyłącznie dane o rejestratorach (loginy, IP, hasła, lokalizacja).

---

## 2. Warstwy systemu

### 2.1 Client Layer (Frontend)

Technologia:
- Flutter Desktop (Windows)
- Flutter Mobile (Android, opcjonalnie iOS)

Odpowiedzialności:
- UI operatora i admina
- Obsługa alarmów w czasie rzeczywistym
- Wyświetlanie map i danych obiektów
- Cache offline (SQLite)
- PIN + biometria
- Selektywne pobieranie plików (zdjęcia, mapy, plany)

Komunikacja:
- WebSocket / SSE (push alarmów i zmian stanu)
- REST API (dane, konfiguracja, raporty)

---

### 2.2 API Layer (Backend Core)

Technologia:
- Python + FastAPI

Odpowiedzialności:
- Autoryzacja i role
- Logika biznesowa alarmów
- Obsługa statusów alarmów
- Bundling / deduplikacja
- Synchronizacja danych
- API dla klientów
- Audit log

---

### 2.3 Satel Worker Layer (Hardware Driver)

Technologia:
- Python (Asyncio)
- Dedykowany mikroserwis (jeden proces lub pool procesów)

Odpowiedzialności:
- **Stałe połączenie TCP/IP** z modułami ETHM-1 (port 10004).
- Polling stanu wejść/wyjść/awarii (co 1-2s).
- Odbiór zdarzeń w czasie rzeczywistym.
- Wysyłanie komend (Arm/Disarm) z kolejki RabbitMQ.
- Reconnect logic (Exponential Backoff).
- Zrzut stanu centrali do Redis (Live Cache).

---

### 2.4 Event & Messaging Layer

Technologia:
- RabbitMQ

Odpowiedzialności:
- Kolejkowanie alarmów (Events Queue).
- Kolejkowanie komend sterujących (Command Queue).
- **Kolejkowanie powiadomień push (Notification Queue).**
- Gwarancja dostarczenia zdarzeń.
- Odporność na burst.

Zasada:
- Worker wrzuca zdarzenie → RabbitMQ → Backend → WebSocket → Frontend.
- Backend wrzuca `notification.push.*` → RabbitMQ → **Notification Worker** → FCM API → Mobile.

---

### 2.5 Integration Layer (Inne)

#### SMS (temperatura)
- GSM modem jako główne źródło SMS
- Parser SMS (Efento, Bluelog)
- Fallback: SMS API (future)

#### Push Notifications (FCM) — Faza 2
- **Architektura:** Backend core **NIE wysyła** HTTP do Firebase synchronicznie. Zamiast tego:
  1. Backend zapisuje zdarzenie `notification.push.alarm` do tabeli `OUTBOX` (Outbox Pattern).
  2. Outbox Relay publikuje do RabbitMQ (exchange: `system.notifications`, routing key: `notification.push.alarm`).
  3. **Notification Worker** (lekki mikroserwis lub background task w backendzie) konsumuje z kolejki `notifications.push` i wysyła do Firebase Cloud Messaging API.
- **Dlaczego async:** FCM API ma zmienne czasy odpowiedzi (50–500ms). Synchroniczny strzał blokowałby alarm pipeline. Async zapewnia, że alarm jest obsłużony natychmiast, a push dochodzi asynchronicznie.
- **Filtr:** Push TYLKO dla alarmów CRITICAL. Push NIE jest wysyłany dla obiektów w trybie serwisowym (`service_mode_active = true`).
- **Token registry:** Tabela `USER_DEVICES` (**04_DATA_MODEL_ERD.md**).
- **Feature flag:** `FEATURE_PUSH_ENABLED` (**19_DEV_TOOLING.md, sekcja 3**).

---

### 2.5b Feature Flags (Przełączanie Faz)

Mechanizm kontroli dostępności funkcji między fazami. Feature Flags pozwalają bezpiecznie deployować kod Fazy N+1 bez aktywacji nowych funkcji.

**Plik konfiguracyjny:** `backend/config/feature_flags.yaml`

| Flaga | Domyślnie | Włączona od | Co kontroluje |
|---|---|---|---|
| `FEATURE_REALTIME_ENABLED` | `false` | **Faza 2** | RabbitMQ, Redis, WebSocket, Outbox Relay |
| `FEATURE_PUSH_ENABLED` | `false` | **Faza 2** | FCM Push Notifications |
| `FEATURE_SMS_ENABLED` | `false` | **Faza 4** | SMS Agent, parser Efento/Bluelog |
| `FEATURE_ARM_DISARM_ENABLED` | `false` | **Faza 5** | Komendy ARM/DISARM z UI |
| `FEATURE_HA_ENABLED` | `false` | **Faza 6** | Patroni, Redis Sentinel, multi-instance |

> [!IMPORTANT]
> Zmiana flagi wymaga restartu serwisu. Endpoint `/readyz` sprawdza tylko włączone zależności. Szczegóły: **19_DEV_TOOLING.md, sekcja 3**.

---

### 2.6 Data Layer

Główna baza:
- **PostgreSQL** (core system of record: historia, konfiguracja, audit).

Live State Cache:
- **Redis** (aktualny stan wejść/wyjść, sesje TCP).
- Cel: Błyskawiczny dostęp do "Co jest uzbrojone?" bez katowania bazy SQL.
- **WebSocket Event Buffer:** Sorted Set `ws:events:buffer` (max 5000 wpisów) — catch-up Tier 1 po reconnect klienta. Przy >5000 utraconych eventów lub po restarcie Redisa, backend automatycznie odpytuje PostgreSQL (Tier 2, tabela `outbox`). Szczegóły: **10_API_HIGH_LEVEL.md, sekcja 8.5**.

Cache lokalny:
- SQLite (frontend offline).

Pliki:
- Storage lokalny na serwerze backendu.

### 2.6.1 Separacja Ruchu Transakcyjnego i Analitycznego

Dostępność: Faza 6

W celu uniknięcia przeciążenia głównej bazy danych przez ciężkie zapytania raportowe, system rozdziela ruch na dwa strumienie:

**Primary (read-write):**
- Zapisy: alarmy, komendy, audit log, CRUD obiektów.
- Odczyty operacyjne: lista alarmów dla operatora, szczegóły obiektu, WebSocket feed.
- Redis Cache jest nadal pierwszym źródłem dla stanów live (uzbrojenie, wejścia, awarie).

**Read-Only Replica (analityczna):**
- Wszystkie zapytania raportowe (`GET /api/reports/*`).
- Eksporty danych (CSV, PDF).
- Operacje administracyjne typu SELECT o dużym obciążeniu (np. statystyki, audyt).
- Timeout: 30s per zapytanie (patrz **10_API_HIGH_LEVEL.md, sekcja 10**).

**Mechanizm:**
- PostgreSQL Streaming Replication (asynchroniczna, lag < 1s w normalnych warunkach).
- Backend używa różnych connection stringów w zależności od typu zapytania:
  - `DATABASE_URL` — primary (read-write)
  - `DATABASE_REPLICA_URL` — replica (read-only)
- Konfiguracja replikacji: **11_DEPLOYMENT.md, sekcja 7**.

```mermaid
flowchart LR
    subgraph Backend
        API["API Layer"]
    end
    subgraph Data
        PG_PRIMARY[("PostgreSQL\nPrimary")]
        PG_REPLICA[("PostgreSQL\nReplica")]
        REDIS[("Redis\nLive Cache")]
    end
    API -- "Zapisy + Odczyty operacyjne" --> PG_PRIMARY
    API -- "Raporty + Eksporty" --> PG_REPLICA
    API -- "Stan live" --> REDIS
    PG_PRIMARY -- "Streaming Replication" --> PG_REPLICA
```

---

## 3. High Availability (HA)

Cel:
- 99.9% dostępności (SLO: `17_MONITORING.md`, sekcja 6)
- Automatyczny failover bez interwencji człowieka

Architektura HA (szczegóły: `09_HA_RTO_RPO.md`):

| Komponent | Strategia HA | RTO |
|---|---|---|
| PostgreSQL | Patroni (auto-failover) + etcd + PgBouncer | < 30s |
| Backend API | 2 instancje za Nginx (health-check routing) | < 15s |
| Satel Worker | Redis leader election per panel | < 60s |
| RabbitMQ | Durable queues + Quorum Queues (v2.0) | < 5s (restart) |
| Redis | AOF persistence + Sentinel (v2.0) | < 30s |

---

## 4. Failover & Recovery

### Automatyczny Failover
- **PostgreSQL:** Patroni promuje replikę → PgBouncer przekierowuje ruch → brak zmiany connection stringów.
- **Backend crash:** Nginx usuwa instancję z upstream (health-check `/readyz`), ruch kierowany na pozostałą.
- **Satel Worker:** Redis lock TTL wygaśnie → drugi Worker przejmuje panel.
- **Outbox Relay:** Docker restart policy `always` → automatyczny restart po crash.

### Graceful Shutdown
Wszystkie serwisy obsługują `SIGTERM`: dokończenie aktywnych requestów, zamknięcie połączeń WS, flush Redis pipeline. Szczegóły: `09_HA_RTO_RPO.md`, HA-04.

### Offline (Frontend)
- Frontend przechodzi w tryb **read-only** z danymi z SQLite cache.
- **Przyciski mutacji (Claim, Ack, Resolve, Close) są WYŁĄCZONE** w trybie offline.
- Brak sterowania centralą.
- Baner: "⚠️ Tryb offline — dane mogą być nieaktualne".
- **Intent Queue:** Operator może zapisywać lekkie intencje (np. "zamierzam obsłużyć alarm X"). Po reconnect, backend arbitruje intencje z bieżącym stanem. Szczegóły: **03_FUNCTIONAL_MODULES.md, sekcja 19.4**.

---

## 5. Tryby pracy systemu

### Normal Mode
- Pełna obsługa alarmów
- Push w czasie rzeczywistym
- Sterowanie centralą (Arm/Disarm)

### Test Mode
- Alarmy testowe
- Ukryte z głównego feedu

### Service / Maintenance Mode (per obiekt)
- Alarmy oznaczone jako serwisowe
- Ograniczone powiadomienia

### Degraded Mode
- Brak połączenia z backendem.
- Tylko dane lokalne (SQLite).
- Baner: "⚠️ Brak połączenia z serwerem".
- Akcje offline kolejkowane — sync po przywróceniu połączenia.
- Szczegóły: sekcja 4 (Failover & Recovery).

---

## 6. Push & Realtime

Mechanizm:
- WebSocket lub SSE

Zasady:
- Alarmy pushowane do wszystkich zalogowanych operatorów
- Zmiany stanu (Uzbrojono/Rozbrojono) pushowane natychmiast
- Statusy alarmów synchronizowane w czasie rzeczywistym

---

## 7. Scalability Assumptions

Zakładana skala:
- MVP: 5 obiektów (ETHM-1).
- Docelowo: 300–500 obiektów.
- Burst: 200+ alarmów w 15 minut.

Architektura pozwala na:
- Skalowanie Workerów (szarding po ID centrali).
- Redis radzi sobie z milionami operacji/s.

---

## 8. Security Architecture

- VPN dla dostępu do backendu (Nginx wystawia port 443)
- **JWT RS256** z rotacją kluczy (`kid` header) — szczegóły: `08_SECURITY_AND_ROLES.md`
- **Docker Secrets** dla wszystkich wrażliwych danych
- RBAC z granularnymi sub-rolami SYSTEM
- Scope enforcement dla FIELD_WORKER/TECHNICIAN
- Audit log (każda akcja logowana z `service_instance_id`)
- Timed Secret Reveal z MultiFernet szyfrowania
- PIN + biometria na urządzeniach mobilnych
- TLS wszędzie: API, replikacja PostgreSQL, RabbitMQ
- **VLAN izolacja** sieci ETHM-1 (`satel_vlan`)
- **WebSocket:** Ticket-based auth (nie `?token=`)

### 8.1 Resilience Patterns

| Pattern | Komponent | Opis |
|---|---|---|
| **Circuit Breaker** | Backend → Redis | Jeśli Redis nie odpowiada 3x pod rząd → otwórz circuit (fallback na direct DB). Auto-close po 30s. |
| **Circuit Breaker** | Backend → RabbitMQ | Jeśli publish failed 3x → Outbox pattern (Event ląduje w DB, Relay dostarczy później). |
| **Bulkhead** | API Layer | Osobne Thread Pool dla /api/reports/* (nie blokuje alarm pipeline). |
| **Retry + Backoff** | Worker → ETHM-1 | Exponential backoff: 1s → 2s → 4s → 8s → 16s → max 30s. |
| **Timeout** | Wszystkie HTTP calls | 5s default, 30s dla raportów. |

**Biblioteka:** `tenacity` (retry/backoff) + custom `CircuitBreaker` class. Szczegóły retry per komponent: `10_API_HIGH_LEVEL.md`, sekcja 7.3.

---

## 9. Non-Goals (wykluczenia)

System NIE:
- streamuje CCTV
- nagrywa wideo
- analizuje obraz

---

## 10. Architecture Principles

- Event-driven
- Single source of truth = PostgreSQL (historia) + Redis (stan live).
- RabbitMQ jako bufor i izolacja.
- Push zamiast polling (dla UI).
- Read-only offline.

---

## 11. Data Flow — End-to-End Alarm (Diagram)

```mermaid
sequenceDiagram
    participant C as Centrala SATEL
    participant W as Satel Worker
    participant R as Redis
    participant PG as PostgreSQL
    participant OB as Outbox Relay
    participant MQ as RabbitMQ
    participant B as Backend API
    participant WS as WebSocket
    participant F as Flutter (Operator)

    C->>W: Event binarna (TCP/IP, port 10004, VLAN)
    W->>W: Parse ramki, CRC check
    W->>R: UPDATE panel:{id}:zones
    W->>MQ: PUBLISH satel.alarm.{panel_id}
    MQ->>B: CONSUME events.processing
    B->>B: Dedup check (dedup_key)
    B->>PG: INSERT EVENTS + BUNDLE_ALARMS + OUTBOX (1 transakcja)
    OB->>PG: SELECT pending FROM outbox
    OB->>MQ: PUBLISH event notification
    B->>WS: PUSH alarm.new (via Redis pub/sub cross-instance)
    WS->>F: Alarm na ekranie operatora (<1s)
    F->>F: Dźwięk CRITICAL + migająca ikona
```

---

## 12. Performance Requirements (NFR)

| Metryka | Target MVP (5 obiektów) | Target v1.0 (100 obiektów) |
|---|---|---|
| GET /api/alarms (lista) | < 200ms (p95) | < 500ms (p95) |
| POST /api/alarms/{id}/claim | < 100ms (p95) | < 200ms (p95) |
| Event → ekran operatora (end-to-end) | < 1s | < 2s |
| Burst: 200 events w 1 min | Bez utraty | Bez utraty |
| Concurrent operators | 5 | 20 |
| WebSocket delivery | < 500ms | < 1s |
| Redis read (live state) | < 1ms | < 5ms |
| Daily event volume | ~500 events | ~10,000 events |



---
---


# 11_DEPLOYMENT

## Cel
Opis architektury wdrożeniowej i środowisk.

---

## 1. Środowiska

### DEV (Lokalnie u developera)
- Docker Compose.
- Hot-reload (FastAPI, Flutter).
- Baza danych w kontenerze (wolumen lokalny).
- Mock Satel Worker (symulator centrali).

### STAGE (Serwer testowy)
- Kopia środowiska PROD.
- Dane zanonimizowane lub kopia snapshotu PROD sprzed tygodnia.
- Testy integracyjne.

### PROD (Serwer docelowy)
- Bare Metal / VM w firmie.
- Docker Compose (tryb produkcyjny).
- Monitoring (Prometheus/Grafana).
- VPN Access Only.

---

## 2. Architektura Kontenerów (Docker Compose)

System składa się z **13 kontenerów** (PROD):

| Serwis | Obraz | Rola | Zależności | Dostępne od Fazy |
|---|---|---|---|---|
| **nginx** | nginx:alpine | Reverse proxy, TLS, health-check routing | backend-1, backend-2 | **Faza 1** |
| **backend-1** | stam-replacer-backend | API FastAPI, logika biznesowa, WebSocket (inst. 1) | pgbouncer, rabbitmq, redis | **Faza 1** |
| **backend-2** | stam-replacer-backend | API FastAPI, logika biznesowa, WebSocket (inst. 2) | pgbouncer, rabbitmq, redis | **Faza 6** (HA) |
| **outbox-relay-1** | stam-replacer-backend | Relay Outbox Pattern → RabbitMQ (instancja 1) | pgbouncer, rabbitmq | **Faza 2** |
| **outbox-relay-2** | stam-replacer-backend | Relay Outbox Pattern → RabbitMQ (instancja 2) | pgbouncer, rabbitmq | **Faza 6** (HA) |

> [!NOTE]
> **Multi-instance Outbox Relay:** Dwie instancje relay eliminują SPOF. Bezpieczne dzięki `FOR UPDATE SKIP LOCKED` — obie instancje konkurują o wiersze bez ryzyka podwójnego przetworzenia. Metryka: `outbox_relay_active_instances` (alert jeśli <1 instancja aktywna >30s).
| **satel-worker** | stam-satel-worker | Połączenie TCP/IP z centralami ETHM-1 | pgbouncer, rabbitmq, redis | **Faza 2** |
| **pg-primary** | postgres:15 (Patroni) | Primary PostgreSQL (read-write) | etcd | **Faza 1** |
| **pg-replica** | postgres:15 (Patroni) | Standby PostgreSQL (read-only) | etcd, pg-primary | **Faza 6** (HA) |
| **etcd** | bitnami/etcd | Consensus store dla Patroni failover | — | **Faza 6** (HA) |
| **pgbouncer** | pgbouncer | Connection pooler z auto-wykrywaniem primary | pg-primary, pg-replica | **Faza 1** |
| **redis** | redis:7-alpine | Live State Cache (stan central, sesje) | — | **Faza 2** |
| **rabbitmq** | rabbitmq:3-management | Kolejka zdarzeń i komend | — | **Faza 2** |
| **sms-agent** | stam-sms-agent | Odbiór SMS z modemu GSM | rabbitmq | **Faza 4** |

> **Uwaga DEV:** W środowisku DEV można używać uproszczonej konfiguracji: 1 backend, 1 PostgreSQL (bez Patroni), bez nginx.

### Nginx Reverse Proxy

```nginx
upstream backend {
    ip_hash;  # sticky sessions dla WebSocket
    server backend-1:8000;
    server backend-2:8000;
}

server {
    listen 443 ssl;
    # TLS config: certyfikaty w ./certs/
    
    location / {
        proxy_pass http://backend;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;
    }
    
    location /ws {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Health-check: usuń backend z puli jeśli /readyz zwraca != 200
}
```

### Satel Worker — szczegóły deploymentu
- **Restart policy:** always (automatyczny restart po awarii)
- **Zależności:** musi wystartować po Redis, RabbitMQ i PostgreSQL
- **Sieć:** `internal` + `satel_vlan` (szczegóły: sekcja Sieć)
- **Zasoby:** 1 CPU, 512MB RAM (dev), 2 CPU, 1GB RAM (prod)
- **Healthcheck:** sprawdzenie statusu połączeń TCP/IP z centralami
- **Leader election:** Redis lock per panel (patrz `09_HA_RTO_RPO.md`, HA-05)

### Redis — szczegóły deploymentu
- **Restart policy:** always
- **Porty:** 6379 (wewnętrzny, nie wystawiony na zewnątrz)
- **Persistence:** AOF + RDB snapshots (opcjonalnie, dane odtwarzalne z DB)
- **Pamięć:** 512MB (dev), 2GB (prod), eviction policy: allkeys-lru
- **Healthcheck:** ping co 10s
- **Recovery:** po restarcie cache odbudowywany automatycznie z PostgreSQL + fresh poll central

### SMS Agent — szczegóły
- **Devices:** wymaga dostępu do urządzenia modemu USB (np. /dev/ttyUSB0)
- **Restart policy:** always
- **Monitoring:** alert Prometheus jeśli brak SMSów > 1h

---

## 3. Aktualizacje (CI/CD)
1. Push do `main` -> GitHub Actions.
2. Build Docker images.
3. Testy automatyczne (Unit/Integration).
4. Jeśli OK -> Push to Registry.
5. Watchtower lub skrypt na serwerze wykonuje pull i restart kontenerów.

---

## 4. Backupy
- **Baza danych:** pg_dump co 4h → lokalne terytorium + kopia na NAS.
- **Pliki:** Rsync całego katalogu media/ na NAS co 24h.
- **Redis:** Nie wymaga backupu (dane tymczasowe, odtwarzalne).

---

## 5. Sieć

### Sieci Docker Compose

| Sieć | Driver | Cel |
|---|---|---|
| `internal` | bridge | Główna sieć wewnętrzna (backend, db, redis, rabbitmq) |
| `satel_vlan` | macvlan (eth1.100) | Izolowany VLAN dla ruchu Worker ↔ ETHM-1 (patrz `09_HA_RTO_RPO.md`, HA-09) |

### Reguły dostępu
- Nginx wystawia porty 443 (HTTPS) — jedyny punkt dostępu z VPN.
- Backend, Redis, RabbitMQ management **NIE** są wystawione publicznie.
- Satel Worker łączy się z centralami wyłącznie przez `satel_vlan`.
- Dostęp z zewnątrz **tylko przez VPN**.

### Docker Secrets

> Wszystkie sekrety przechowywane jako Docker Secrets (nie w `.env`). Szczegóły: `08_SECURITY_AND_ROLES.md`, sekcja 6.

```yaml
secrets:
  jwt_private_key:
    file: ./secrets/jwt_private.pem
  jwt_public_key:
    file: ./secrets/jwt_public.pem
  encryption_key:
    file: ./secrets/encryption.key
  postgres_password:
    file: ./secrets/pg_password.txt
  redis_password:
    file: ./secrets/redis_password.txt
  rabbitmq_password:
    file: ./secrets/rabbitmq_password.txt
```

---

## 6. Kubernetes (Opcjonalnie, v2.0+)
MVP i v1.0 działają na Docker Compose. Kubernetes jest opcjonalnym celem migracji dla wersji Enterprise (v2.0+), gdy skala przekroczy możliwości pojedynczego serwera.

---

## 7. Read-Only Replica PostgreSQL (Patroni)

Priorytet: v1.0

Cel: Rozdział ruchu transakcyjnego od analitycznego + automatyczny failover.

> **Patroni zarządza replikacją.** Nie konfiguruj ręcznie `pg_basebackup` ani `standby.signal` — Patroni robi to automatycznie. Szczegóły failover: `09_HA_RTO_RPO.md`, sekcja HA-01.

### 7.1 Konfiguracja replikacji

- **Typ:** PostgreSQL Streaming Replication (asynchroniczna), zarządzana przez Patroni.
- **Primary:** Kontener `pg-primary` — pełny read-write.
- **Replica:** Kontener `pg-replica` — read-only (hot standby).
- **Failover:** Automatyczny — Patroni promuje replikę jeśli primary padnie (< 30s).
- **TLS replikacji:** `sslmode=verify-full` między primary a repliką.

### 7.2 Kontener pg-replica

| Parametr | Wartość |
|---|---|
| Obraz | postgres:15 (zarządzany przez Patroni) |
| Rola | Hot Standby (read-only) |
| Restart policy | always |
| Zależności | pg-primary, etcd |
| Zasoby (prod) | 2 CPU, 2GB RAM |
| Port | 5433 (wewnętrzny, dostępny przez PgBouncer) |
| Healthcheck | `pg_isready` co 10s |

### 7.3 Konfiguracja PostgreSQL

> **Uwaga:** Patroni zarządza konfiguracją replikacji automatycznie (WAL, `pg_hba.conf`, `standby.signal`). Poniższe parametry są ustawiane przez Patroni — nie edytuj ręcznie.

**Kluczowe parametry (ustawione przez Patroni):**
```
wal_level = replica
max_wal_senders = 3
wal_keep_size = 256MB
ssl = on
ssl_cert_file = '/run/secrets/pg_server.crt'
ssl_key_file = '/run/secrets/pg_server.key'
```

### 7.4 Connection Strings w Backendzie

| Zmienna środowiskowa | Cel | Przykład |
|---|---|---|
| `DATABASE_URL` | Primary (read-write) — **przez PgBouncer** | `postgresql://app:***@pgbouncer:6432/stam` |
| `DATABASE_REPLICA_URL` | Replica (read-only) — raporty, eksporty | `postgresql://app:***@pg-replica:5433/stam` |

> **PgBouncer:** Backend łączy się z PgBouncer (nie bezpośrednio z pg-primary). PgBouncer automatycznie wykrywa aktualny primary przez etcd — po failover nie trzeba zmieniać connection stringów.

Backend automatycznie kieruje zapytania:
- Endpoint `/api/reports/*` → `DATABASE_REPLICA_URL`
- Pozostałe endpointy → `DATABASE_URL` (PgBouncer → aktualny primary)
- Jeśli replika niedostępna → fallback na primary (z logiem WARNING).

### 7.5 Monitoring replikacji

- Metryka Prometheus: `pg_replication_lag_seconds`
- Alert: jeśli lag > 10s → WARNING
- Alert: jeśli lag > 60s → CRITICAL
- Dashboard: patrz `17_MONITORING.md`, sekcja MON-06
